# Part 72: API Design Best Practices

## ทำไม API Design ถึงสำคัญ?

API ที่ออกแบบดีทำให้ developer อื่น (รวมถึงตัวเองในอนาคต) เข้าใจและใช้งานได้ง่าย ลดข้อผิดพลาด และทำให้ระบบ maintainable ยาวนาน

---

## 1. RESTful API Design Principles

### Resource Naming

```
# ดี - ใช้ noun (พหูพจน์)
GET    /api/users
GET    /api/users/123
POST   /api/users
PUT    /api/users/123
PATCH  /api/users/123
DELETE /api/users/123

# ดี - nested resources
GET    /api/users/123/posts
POST   /api/users/123/posts
GET    /api/users/123/posts/456

# ไม่ดี - ใช้ verb
GET    /api/getUsers
POST   /api/createUser
DELETE /api/deleteUser/123
```

### HTTP Methods ที่ถูกต้อง

| Method | ใช้สำหรับ | Idempotent | Safe |
|--------|-----------|------------|------|
| GET | ดึงข้อมูล | ✓ | ✓ |
| POST | สร้างข้อมูล | ✗ | ✗ |
| PUT | แทนที่ทั้งหมด | ✓ | ✗ |
| PATCH | แก้ไขบางส่วน | ✗ | ✗ |
| DELETE | ลบข้อมูล | ✓ | ✗ |

### HTTP Status Codes

```typescript
// สถานะที่ใช้บ่อย
const HTTP_STATUS = {
  // 2xx Success
  OK: 200,                // GET สำเร็จ
  CREATED: 201,           // POST สร้างสำเร็จ
  ACCEPTED: 202,          // Request รับแล้ว แต่ยังประมวลผล
  NO_CONTENT: 204,        // DELETE สำเร็จ (ไม่มี body)

  // 3xx Redirect
  MOVED_PERMANENTLY: 301,
  NOT_MODIFIED: 304,

  // 4xx Client Error
  BAD_REQUEST: 400,       // Input ผิด format
  UNAUTHORIZED: 401,      // ยังไม่ login
  FORBIDDEN: 403,         // login แล้ว แต่ไม่มีสิทธิ์
  NOT_FOUND: 404,         // ไม่พบข้อมูล
  METHOD_NOT_ALLOWED: 405,
  CONFLICT: 409,          // ข้อมูลซ้ำ
  UNPROCESSABLE: 422,     // Validation error
  TOO_MANY_REQUESTS: 429, // Rate limit

  // 5xx Server Error
  INTERNAL_ERROR: 500,
  BAD_GATEWAY: 502,
  SERVICE_UNAVAILABLE: 503
} as const
```

---

## 2. API Versioning

### URL Versioning (แนะนำ)

```
/api/v1/users
/api/v2/users
```

### Header Versioning

```
GET /api/users
Accept: application/vnd.myapi.v2+json
```

### Nuxt API Versioning Implementation

```typescript
// server/api/v1/users/index.get.ts
export default defineEventHandler(async (event) => {
  return { version: 'v1', users: [] }
})

// server/api/v2/users/index.get.ts  
export default defineEventHandler(async (event) => {
  // v2 มี pagination และ filters
  const query = getQuery(event)
  const { page = 1, limit = 20, search } = query

  const users = await prisma.user.findMany({
    where: search ? { name: { contains: String(search) } } : undefined,
    skip: (Number(page) - 1) * Number(limit),
    take: Number(limit)
  })

  const total = await prisma.user.count({
    where: search ? { name: { contains: String(search) } } : undefined
  })

  return {
    data: users,
    meta: {
      total,
      page: Number(page),
      limit: Number(limit),
      totalPages: Math.ceil(total / Number(limit))
    }
  }
})
```

### Version Deprecation Strategy

```typescript
// server/middleware/api-version.ts
export default defineEventHandler(async (event) => {
  const url = getRequestURL(event)
  
  if (url.pathname.startsWith('/api/v1/')) {
    // เพิ่ม deprecation header
    setHeader(event, 'Deprecation', 'true')
    setHeader(event, 'Sunset', 'Sat, 31 Dec 2024 00:00:00 GMT')
    setHeader(event, 'Link', '</api/v2>; rel="successor-version"')
    
    console.warn(`Deprecated API v1 called: ${url.pathname}`)
  }
})
```

---

## 3. Request/Response Schemas

### Zod Schema Validation

```typescript
// server/schemas/user.schema.ts
import { z } from 'zod'

export const CreateUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  name: z.string().min(2, 'Name must be at least 2 characters').max(100),
  role: z.enum(['admin', 'editor', 'viewer']).default('viewer'),
  metadata: z.record(z.unknown()).optional()
})

export const UpdateUserSchema = CreateUserSchema.partial().omit({ email: true })

export const UserQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
  search: z.string().optional(),
  role: z.enum(['admin', 'editor', 'viewer']).optional(),
  sortBy: z.enum(['name', 'email', 'createdAt']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc')
})

export type CreateUserInput = z.infer<typeof CreateUserSchema>
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>
export type UserQuery = z.infer<typeof UserQuerySchema>
```

### Response Schema

```typescript
// server/schemas/response.schema.ts
import { z } from 'zod'

// Generic response wrapper
export function createSuccessResponse<T>(data: T, meta?: object) {
  return {
    success: true,
    data,
    meta,
    timestamp: new Date().toISOString()
  }
}

export function createErrorResponse(
  code: string,
  message: string,
  details?: unknown
) {
  return {
    success: false,
    error: {
      code,
      message,
      details
    },
    timestamp: new Date().toISOString()
  }
}

// TypeScript types
export type ApiResponse<T> = {
  success: true
  data: T
  meta?: {
    total?: number
    page?: number
    limit?: number
    totalPages?: number
  }
  timestamp: string
} | {
  success: false
  error: {
    code: string
    message: string
    details?: unknown
  }
  timestamp: string
}
```

### Validation Helper

```typescript
// server/utils/validate.ts
import type { ZodSchema } from 'zod'

export async function validateBody<T>(
  event: H3Event,
  schema: ZodSchema<T>
): Promise<T> {
  const body = await readBody(event)
  const result = schema.safeParse(body)
  
  if (!result.success) {
    throw createError({
      statusCode: 422,
      message: 'Validation failed',
      data: {
        code: 'VALIDATION_ERROR',
        errors: result.error.flatten().fieldErrors
      }
    })
  }
  
  return result.data
}

export function validateQuery<T>(
  event: H3Event,
  schema: ZodSchema<T>
): T {
  const query = getQuery(event)
  const result = schema.safeParse(query)
  
  if (!result.success) {
    throw createError({
      statusCode: 400,
      message: 'Invalid query parameters',
      data: {
        code: 'INVALID_QUERY',
        errors: result.error.flatten().fieldErrors
      }
    })
  }
  
  return result.data
}
```

---

## 4. Error Response Format

### Standardized Error Format

```typescript
// server/utils/errors.ts

export interface ApiError {
  code: string        // Machine-readable error code
  message: string     // Human-readable message
  details?: unknown   // Additional context
  path?: string       // API path that caused error
  requestId?: string  // For tracing
}

// Error codes ที่ใช้ในระบบ
export const ERROR_CODES = {
  // Authentication
  UNAUTHORIZED: 'AUTH_001',
  TOKEN_EXPIRED: 'AUTH_002',
  INVALID_TOKEN: 'AUTH_003',

  // Authorization
  FORBIDDEN: 'AUTHZ_001',
  INSUFFICIENT_PERMISSIONS: 'AUTHZ_002',

  // Validation
  VALIDATION_ERROR: 'VAL_001',
  INVALID_FORMAT: 'VAL_002',
  REQUIRED_FIELD: 'VAL_003',

  // Resource
  NOT_FOUND: 'RES_001',
  CONFLICT: 'RES_002',
  ALREADY_EXISTS: 'RES_003',

  // Rate Limiting
  RATE_LIMIT_EXCEEDED: 'RATE_001',
  QUOTA_EXCEEDED: 'RATE_002',

  // Server
  INTERNAL_ERROR: 'SRV_001',
  SERVICE_UNAVAILABLE: 'SRV_002'
} as const

// Global error handler
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('error', async (error, { event }) => {
    const statusCode = error.statusCode || 500
    const requestId = getHeader(event, 'x-request-id') || generateId()
    
    const response = {
      success: false,
      error: {
        code: error.data?.code || ERROR_CODES.INTERNAL_ERROR,
        message: statusCode >= 500 
          ? 'An internal server error occurred' 
          : error.message,
        details: statusCode < 500 ? error.data?.errors : undefined,
        requestId
      },
      timestamp: new Date().toISOString()
    }

    // Log errors
    if (statusCode >= 500) {
      console.error('[API Error]', {
        statusCode,
        url: getRequestURL(event).pathname,
        requestId,
        error: error.message,
        stack: error.stack
      })
    }

    event.node.res.statusCode = statusCode
    event.node.res.setHeader('Content-Type', 'application/json')
    event.node.res.setHeader('X-Request-ID', requestId)
    event.node.res.end(JSON.stringify(response))
  })
})
```

---

## 5. Pagination Patterns

### Offset Pagination

```typescript
// server/utils/pagination.ts

export interface PaginationOptions {
  page: number
  limit: number
}

export interface PaginatedResult<T> {
  data: T[]
  meta: {
    total: number
    page: number
    limit: number
    totalPages: number
    hasNextPage: boolean
    hasPrevPage: boolean
  }
}

export async function paginate<T>(
  model: any,
  options: PaginationOptions,
  where?: object,
  include?: object,
  orderBy?: object
): Promise<PaginatedResult<T>> {
  const { page, limit } = options
  const skip = (page - 1) * limit

  const [data, total] = await Promise.all([
    model.findMany({ where, include, orderBy, skip, take: limit }),
    model.count({ where })
  ])

  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNextPage: page * limit < total,
      hasPrevPage: page > 1
    }
  }
}
```

### Cursor Pagination (สำหรับ Real-time Data)

```typescript
// server/utils/cursor-pagination.ts

export interface CursorPaginationOptions {
  cursor?: string
  limit: number
  direction?: 'forward' | 'backward'
}

export async function cursorPaginate<T extends { id: string; createdAt: Date }>(
  model: any,
  options: CursorPaginationOptions,
  where?: object
): Promise<{
  data: T[]
  pageInfo: {
    hasNextPage: boolean
    hasPrevPage: boolean
    startCursor?: string
    endCursor?: string
  }
}> {
  const { cursor, limit, direction = 'forward' } = options

  const queryWhere = cursor
    ? direction === 'forward'
      ? { ...where, id: { gt: cursor } }
      : { ...where, id: { lt: cursor } }
    : where

  const data = await model.findMany({
    where: queryWhere,
    take: limit + 1, // ดึงมา 1 มากกว่าเพื่อตรวจสอบ hasNextPage
    orderBy: { createdAt: direction === 'forward' ? 'asc' : 'desc' }
  })

  const hasMore = data.length > limit
  if (hasMore) data.pop()

  return {
    data,
    pageInfo: {
      hasNextPage: direction === 'forward' ? hasMore : true,
      hasPrevPage: direction === 'forward' ? !!cursor : hasMore,
      startCursor: data[0]?.id,
      endCursor: data[data.length - 1]?.id
    }
  }
}
```

---

## 6. Rate Limiting

### Rate Limiter Middleware

```typescript
// server/middleware/rate-limit.ts
import { redis } from '~/server/lib/redis'

interface RateLimitConfig {
  windowMs: number  // Time window in milliseconds
  max: number       // Max requests per window
  keyGenerator?: (event: H3Event) => string
}

const RATE_LIMIT_CONFIGS: Record<string, RateLimitConfig> = {
  // Default: 100 req/min
  default: { windowMs: 60 * 1000, max: 100 },
  // Auth endpoints: 10 req/min (strict)
  auth: { windowMs: 60 * 1000, max: 10 },
  // Upload: 20 req/min
  upload: { windowMs: 60 * 1000, max: 20 },
  // Webhook: 1000 req/min
  webhook: { windowMs: 60 * 1000, max: 1000 }
}

export default defineEventHandler(async (event) => {
  const url = getRequestURL(event)
  const path = url.pathname

  // เลือก config ตาม path
  let config = RATE_LIMIT_CONFIGS.default
  if (path.includes('/auth/')) config = RATE_LIMIT_CONFIGS.auth
  if (path.includes('/upload/')) config = RATE_LIMIT_CONFIGS.upload
  if (path.includes('/webhook/')) config = RATE_LIMIT_CONFIGS.webhook

  // Key = IP + Path prefix
  const ip = getHeader(event, 'x-forwarded-for') || 
              getHeader(event, 'x-real-ip') || 
              event.node.req.socket.remoteAddress || 'unknown'
  
  const tenantId = event.context.tenantId
  const keyPrefix = tenantId ? `tenant:${tenantId}` : `ip:${ip}`
  const key = `rate_limit:${keyPrefix}:${path.split('/').slice(0, 4).join('/')}`

  const now = Date.now()
  const windowStart = now - config.windowMs

  // Sliding window algorithm ด้วย Redis
  await redis.zremrangebyscore(key, 0, windowStart)
  const requests = await redis.zcard(key)

  if (requests >= config.max) {
    const oldestRequest = await redis.zrange(key, 0, 0, 'WITHSCORES')
    const resetTime = oldestRequest[1] 
      ? parseInt(oldestRequest[1]) + config.windowMs 
      : now + config.windowMs

    setHeader(event, 'X-RateLimit-Limit', config.max)
    setHeader(event, 'X-RateLimit-Remaining', 0)
    setHeader(event, 'X-RateLimit-Reset', Math.ceil(resetTime / 1000))
    setHeader(event, 'Retry-After', Math.ceil((resetTime - now) / 1000))

    throw createError({
      statusCode: 429,
      message: 'Too Many Requests',
      data: {
        code: 'RATE_LIMIT_EXCEEDED',
        retryAfter: Math.ceil((resetTime - now) / 1000)
      }
    })
  }

  // บันทึก request
  await redis.zadd(key, now, `${now}-${Math.random()}`)
  await redis.expire(key, Math.ceil(config.windowMs / 1000))

  // เพิ่ม headers
  setHeader(event, 'X-RateLimit-Limit', config.max)
  setHeader(event, 'X-RateLimit-Remaining', config.max - requests - 1)
})
```

---

## 7. API Documentation กับ Swagger/OpenAPI

### OpenAPI Schema Definition

```typescript
// server/lib/openapi.ts
import { OpenAPIV3 } from 'openapi-types'

export const openApiSpec: OpenAPIV3.Document = {
  openapi: '3.0.0',
  info: {
    title: 'My SaaS API',
    version: '2.0.0',
    description: 'RESTful API for My SaaS Platform',
    contact: {
      name: 'API Support',
      email: 'api@myapp.com',
      url: 'https://docs.myapp.com'
    },
    license: {
      name: 'MIT',
      url: 'https://opensource.org/licenses/MIT'
    }
  },
  servers: [
    {
      url: 'https://api.myapp.com/v2',
      description: 'Production'
    },
    {
      url: 'http://localhost:3000/api/v2',
      description: 'Development'
    }
  ],
  components: {
    securitySchemes: {
      BearerAuth: {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT'
      },
      ApiKeyAuth: {
        type: 'apiKey',
        in: 'header',
        name: 'X-API-Key'
      }
    },
    schemas: {
      User: {
        type: 'object',
        required: ['id', 'email', 'name'],
        properties: {
          id: { type: 'string', format: 'uuid' },
          email: { type: 'string', format: 'email' },
          name: { type: 'string' },
          role: { 
            type: 'string', 
            enum: ['admin', 'editor', 'viewer'] 
          },
          createdAt: { type: 'string', format: 'date-time' }
        }
      },
      PaginatedUsers: {
        type: 'object',
        properties: {
          data: {
            type: 'array',
            items: { $ref: '#/components/schemas/User' }
          },
          meta: { $ref: '#/components/schemas/PaginationMeta' }
        }
      },
      PaginationMeta: {
        type: 'object',
        properties: {
          total: { type: 'integer' },
          page: { type: 'integer' },
          limit: { type: 'integer' },
          totalPages: { type: 'integer' },
          hasNextPage: { type: 'boolean' },
          hasPrevPage: { type: 'boolean' }
        }
      },
      Error: {
        type: 'object',
        required: ['success', 'error'],
        properties: {
          success: { type: 'boolean', example: false },
          error: {
            type: 'object',
            properties: {
              code: { type: 'string' },
              message: { type: 'string' },
              details: { type: 'object' }
            }
          }
        }
      }
    }
  },
  paths: {
    '/users': {
      get: {
        tags: ['Users'],
        summary: 'List all users',
        security: [{ BearerAuth: [] }],
        parameters: [
          {
            name: 'page',
            in: 'query',
            schema: { type: 'integer', default: 1 }
          },
          {
            name: 'limit',
            in: 'query',
            schema: { type: 'integer', default: 20, maximum: 100 }
          },
          {
            name: 'search',
            in: 'query',
            schema: { type: 'string' }
          }
        ],
        responses: {
          '200': {
            description: 'Successful response',
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/PaginatedUsers' }
              }
            }
          },
          '401': {
            description: 'Unauthorized',
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/Error' }
              }
            }
          }
        }
      },
      post: {
        tags: ['Users'],
        summary: 'Create a new user',
        security: [{ BearerAuth: [] }],
        requestBody: {
          required: true,
          content: {
            'application/json': {
              schema: {
                type: 'object',
                required: ['email', 'name'],
                properties: {
                  email: { type: 'string', format: 'email' },
                  name: { type: 'string', minLength: 2 },
                  role: { type: 'string', enum: ['admin', 'editor', 'viewer'] }
                }
              }
            }
          }
        },
        responses: {
          '201': {
            description: 'User created successfully',
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/User' }
              }
            }
          }
        }
      }
    }
  }
}
```

### Swagger UI Endpoint

```typescript
// server/api/docs/index.get.ts
export default defineEventHandler(async (event) => {
  setHeader(event, 'Content-Type', 'text/html')
  
  return `<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist/swagger-ui.css">
</head>
<body>
  <div id="swagger-ui"></div>
  <script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist/swagger-ui-bundle.js"></script>
  <script>
    SwaggerUIBundle({
      url: '/api/docs/openapi.json',
      dom_id: '#swagger-ui',
      presets: [SwaggerUIBundle.presets.apis],
      layout: 'BaseLayout'
    })
  </script>
</body>
</html>`
})

// server/api/docs/openapi.json.get.ts
import { openApiSpec } from '~/server/lib/openapi'

export default defineEventHandler(() => openApiSpec)
```

---

## 8. ตัวอย่าง: Well-designed Nuxt API Layer

### Users API - Complete Implementation

```typescript
// server/api/v2/users/index.get.ts
import { UserQuerySchema } from '~/server/schemas/user.schema'
import { validateQuery, paginate } from '~/server/utils'

export default defineEventHandler(async (event) => {
  await requireAuth(event)
  await requirePermission(event, 'users:read')

  const query = validateQuery(event, UserQuerySchema)

  const where = {
    tenantId: event.context.tenantId,
    ...(query.search && {
      OR: [
        { name: { contains: query.search, mode: 'insensitive' } },
        { email: { contains: query.search, mode: 'insensitive' } }
      ]
    }),
    ...(query.role && { role: query.role })
  }

  const result = await paginate(
    prisma.user,
    { page: query.page, limit: query.limit },
    where,
    undefined,
    { [query.sortBy]: query.sortOrder }
  )

  return createSuccessResponse(result.data, result.meta)
})

// server/api/v2/users/index.post.ts
import { CreateUserSchema } from '~/server/schemas/user.schema'

export default defineEventHandler(async (event) => {
  await requireAuth(event)
  await requirePermission(event, 'users:create')

  // ตรวจสอบ plan limit
  await usageService.checkLimit(event.context.tenantId, 'users')

  const body = await validateBody(event, CreateUserSchema)

  // ตรวจสอบว่า email ซ้ำหรือไม่
  const existing = await prisma.user.findUnique({
    where: {
      tenantId_email: {
        tenantId: event.context.tenantId,
        email: body.email
      }
    }
  })

  if (existing) {
    throw createError({
      statusCode: 409,
      message: 'User with this email already exists',
      data: { code: 'USER_EXISTS' }
    })
  }

  const user = await prisma.user.create({
    data: {
      ...body,
      tenantId: event.context.tenantId
    }
  })

  // นับ usage
  await usageService.increment(event.context.tenantId, 'users')

  // ส่ง welcome email
  await emailService.sendWelcome(user)

  setResponseStatus(event, 201)
  return createSuccessResponse(user)
})

// server/api/v2/users/[id].get.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)
  
  const userId = getRouterParam(event, 'id')!

  // ผู้ใช้ดูได้เฉพาะ profile ตัวเอง หรือ admin ดูได้ทุกคน
  const canViewAll = event.context.user?.role === 'admin'
  if (!canViewAll && userId !== event.context.userId) {
    throw createError({ statusCode: 403, message: 'Forbidden' })
  }

  const user = await prisma.user.findFirst({
    where: {
      id: userId,
      tenantId: event.context.tenantId // ป้องกัน cross-tenant access
    }
  })

  if (!user) {
    throw createError({
      statusCode: 404,
      message: 'User not found',
      data: { code: 'USER_NOT_FOUND' }
    })
  }

  return createSuccessResponse(user)
})
```

### API Client Composable

```typescript
// composables/useApi.ts

interface FetchOptions extends RequestInit {
  params?: Record<string, string | number | boolean | undefined>
}

export function useApi() {
  const { token } = useAuth()
  
  async function apiFetch<T>(
    path: string, 
    options: FetchOptions = {}
  ): Promise<T> {
    const { params, ...fetchOptions } = options

    // สร้าง URL กับ query params
    const url = new URL(`/api/v2${path}`, window.location.origin)
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined) {
          url.searchParams.set(key, String(value))
        }
      })
    }

    const response = await fetch(url, {
      ...fetchOptions,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token.value}`,
        ...fetchOptions.headers
      }
    })

    const data = await response.json()

    if (!data.success) {
      throw new ApiError(data.error.code, data.error.message, data.error.details)
    }

    return data.data
  }

  return {
    get: <T>(path: string, params?: Record<string, any>) => 
      apiFetch<T>(path, { method: 'GET', params }),
    post: <T>(path: string, body: unknown) => 
      apiFetch<T>(path, { method: 'POST', body: JSON.stringify(body) }),
    put: <T>(path: string, body: unknown) => 
      apiFetch<T>(path, { method: 'PUT', body: JSON.stringify(body) }),
    patch: <T>(path: string, body: unknown) => 
      apiFetch<T>(path, { method: 'PATCH', body: JSON.stringify(body) }),
    delete: <T>(path: string) => 
      apiFetch<T>(path, { method: 'DELETE' })
  }
}

class ApiError extends Error {
  constructor(
    public code: string,
    message: string,
    public details?: unknown
  ) {
    super(message)
    this.name = 'ApiError'
  }
}
```

---

## สรุป

API ที่ดีควรมี:
1. **Consistent naming** - ใช้ noun พหูพจน์
2. **Proper status codes** - ใช้ HTTP status codes ที่ถูกต้อง
3. **Versioning** - รองรับการ upgrade โดยไม่ break clients เก่า
4. **Validation** - ตรวจสอบ input ทุกครั้ง
5. **Error handling** - Error format ที่สม่ำเสมอ
6. **Rate limiting** - ป้องกัน abuse
7. **Documentation** - Swagger/OpenAPI ที่ update อยู่เสมอ
