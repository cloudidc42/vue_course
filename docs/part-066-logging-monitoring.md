# Part 66: Logging และ Monitoring ใน Nuxt.js

## Logging Best Practices

Logging ที่ดีช่วยให้ debug และ monitor application ใน production ได้ง่ายขึ้น

### หลักการ Logging
1. Log ข้อมูลที่มีประโยชน์ ไม่ใช่ทุกอย่าง
2. ใช้ structured logging (JSON)
3. กำหนด log levels ที่เหมาะสม
4. อย่า log sensitive data

```typescript
// server/utils/logger.ts
import pino from 'pino'

const isDev = process.env.NODE_ENV === 'development'

export const logger = pino({
  level: process.env.LOG_LEVEL || (isDev ? 'debug' : 'info'),
  
  // Pretty print in development
  transport: isDev ? {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname'
    }
  } : undefined,
  
  // Production: JSON output
  formatters: {
    level: (label) => ({ level: label }),
    log: (obj) => ({
      ...obj,
      service: 'nuxt-app',
      environment: process.env.NODE_ENV,
      version: process.env.APP_VERSION || '1.0.0'
    })
  },
  
  // Redact sensitive fields
  redact: {
    paths: ['password', 'token', 'authorization', 'cookie', 'credit_card'],
    censor: '[REDACTED]'
  },
  
  serializers: {
    err: pino.stdSerializers.err,
    req: (req) => ({
      method: req.method,
      url: req.url,
      userAgent: req.headers?.['user-agent']
    })
  }
})

// Create child loggers for different modules
export const authLogger = logger.child({ module: 'auth' })
export const dbLogger = logger.child({ module: 'database' })
export const apiLogger = logger.child({ module: 'api' })
export const paymentLogger = logger.child({ module: 'payment' })
```

```typescript
// server/middleware/request-logger.ts
import { logger } from '~/server/utils/logger'

export default defineEventHandler(async (event) => {
  const startTime = Date.now()
  const requestId = crypto.randomUUID()
  
  // Add request ID to event context
  event.context.requestId = requestId
  
  const method = getMethod(event)
  const url = getRequestURL(event)
  
  logger.info({
    requestId,
    method,
    url: url.pathname + url.search,
    ip: getRequestIP(event, { xForwardedFor: true })
  }, 'Incoming request')
  
  // Wait for response
  const handler = event.node.res
  handler.on('finish', () => {
    const duration = Date.now() - startTime
    const status = handler.statusCode
    
    const logFn = status >= 500 ? logger.error.bind(logger) :
                  status >= 400 ? logger.warn.bind(logger) :
                  logger.info.bind(logger)
    
    logFn({
      requestId,
      method,
      url: url.pathname,
      status,
      duration,
      contentLength: handler.getHeader('content-length')
    }, `${method} ${url.pathname} ${status} ${duration}ms`)
  })
})
```

## Sentry Error Tracking

```bash
npm install @sentry/nuxt
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@sentry/nuxt/module'],
  
  sentry: {
    sourceMapsUploadOptions: {
      org: process.env.SENTRY_ORG,
      project: process.env.SENTRY_PROJECT,
      authToken: process.env.SENTRY_AUTH_TOKEN
    }
  }
})
```

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nuxt'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  
  // Performance monitoring
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  
  // Session replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true
    })
  ],
  
  // Filter out known non-issues
  beforeSend(event, hint) {
    const error = hint.originalException
    
    // Filter out network errors
    if (error instanceof Error) {
      if (error.message.includes('Network Error')) return null
      if (error.message.includes('ChunkLoadError')) return null
    }
    
    return event
  },
  
  // Add user context
  beforeSendTransaction(transaction) {
    return transaction
  }
})
```

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nuxt'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  
  // Server-side tracing
  tracesSampleRate: 0.1,
  
  integrations: [
    Sentry.prismaIntegration()
  ]
})
```

```typescript
// composables/useSentry.ts
import * as Sentry from '@sentry/nuxt'

export function useSentry() {
  const captureError = (error: Error, context?: Record<string, any>) => {
    Sentry.withScope((scope) => {
      if (context) {
        scope.setExtras(context)
      }
      Sentry.captureException(error)
    })
  }
  
  const captureMessage = (message: string, level: Sentry.SeverityLevel = 'info') => {
    Sentry.captureMessage(message, level)
  }
  
  const setUser = (user: { id: string; email?: string; username?: string } | null) => {
    Sentry.setUser(user)
  }
  
  const startTransaction = (name: string, op: string) => {
    return Sentry.startInactiveSpan({ name, op })
  }
  
  const addBreadcrumb = (breadcrumb: Sentry.Breadcrumb) => {
    Sentry.addBreadcrumb(breadcrumb)
  }
  
  return {
    captureError,
    captureMessage,
    setUser,
    startTransaction,
    addBreadcrumb
  }
}
```

## Application Metrics

```typescript
// server/services/metrics.ts
// Simple metrics using Redis counters
import { getRedis } from '~/server/utils/redis'

export const metrics = {
  increment: async (metric: string, tags?: Record<string, string>) => {
    const key = buildMetricKey(metric, tags)
    await getRedis().incr(key)
    await getRedis().expire(key, 86400) // 1 day TTL
  },
  
  timing: async (metric: string, durationMs: number, tags?: Record<string, string>) => {
    const redis = getRedis()
    const key = buildMetricKey(`${metric}_ms`, tags)
    const pipeline = redis.pipeline()
    pipeline.rpush(key, durationMs)
    pipeline.ltrim(key, -1000, -1) // Keep last 1000 values
    pipeline.expire(key, 86400)
    await pipeline.exec()
  },
  
  gauge: async (metric: string, value: number, tags?: Record<string, string>) => {
    const key = buildMetricKey(metric, tags)
    await getRedis().set(key, value, 'EX', 300) // 5 min
  }
}

function buildMetricKey(metric: string, tags?: Record<string, string>): string {
  if (!tags) return `metric:${metric}`
  const tagStr = Object.entries(tags).map(([k, v]) => `${k}=${v}`).join(',')
  return `metric:${metric}:${tagStr}`
}
```

## Health Checks

```typescript
// server/api/health.get.ts
interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy'
  timestamp: string
  version: string
  checks: Record<string, {
    status: 'pass' | 'warn' | 'fail'
    duration: number
    message?: string
  }>
}

export default defineEventHandler(async (event): Promise<HealthStatus> => {
  const startTime = Date.now()
  const checks: HealthStatus['checks'] = {}
  
  // Database check
  try {
    const dbStart = Date.now()
    await prisma.$queryRaw`SELECT 1`
    checks.database = {
      status: 'pass',
      duration: Date.now() - dbStart
    }
  } catch (err: any) {
    checks.database = {
      status: 'fail',
      duration: 0,
      message: err.message
    }
  }
  
  // Redis check
  try {
    const redisStart = Date.now()
    await getRedis().ping()
    checks.redis = {
      status: 'pass',
      duration: Date.now() - redisStart
    }
  } catch (err: any) {
    checks.redis = {
      status: 'warn',
      duration: 0,
      message: 'Redis unavailable, using fallback'
    }
  }
  
  // Memory check
  const memUsage = process.memoryUsage()
  const heapUsedMB = memUsage.heapUsed / 1024 / 1024
  checks.memory = {
    status: heapUsedMB < 512 ? 'pass' : heapUsedMB < 768 ? 'warn' : 'fail',
    duration: 0,
    message: `Heap used: ${heapUsedMB.toFixed(1)}MB`
  }
  
  // Determine overall status
  const hasFail = Object.values(checks).some(c => c.status === 'fail')
  const hasWarn = Object.values(checks).some(c => c.status === 'warn')
  
  const status: HealthStatus['status'] = hasFail ? 'unhealthy' : hasWarn ? 'degraded' : 'healthy'
  
  setResponseStatus(event, status === 'unhealthy' ? 503 : 200)
  
  return {
    status,
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0',
    checks
  }
})
```

## Uptime Monitoring

```typescript
// server/api/health/ready.get.ts
// Readiness check - is app ready to serve traffic?
export default defineEventHandler(async (event) => {
  // Check critical dependencies
  try {
    await prisma.$queryRaw`SELECT 1`
    return { ready: true }
  } catch {
    setResponseStatus(event, 503)
    return { ready: false, reason: 'Database unavailable' }
  }
})
```

```typescript
// server/api/health/live.get.ts
// Liveness check - is app alive?
export default defineEventHandler(() => {
  return {
    alive: true,
    uptime: process.uptime(),
    timestamp: new Date().toISOString()
  }
})
```

## Log Aggregation (ELK Stack)

```typescript
// server/utils/elasticsearch-logger.ts
import { Client } from '@elastic/elasticsearch'

const esClient = new Client({
  node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
  auth: {
    username: process.env.ELASTICSEARCH_USER || 'elastic',
    password: process.env.ELASTICSEARCH_PASSWORD || ''
  }
})

interface LogEntry {
  timestamp: string
  level: string
  message: string
  service: string
  environment: string
  requestId?: string
  userId?: string
  [key: string]: any
}

export async function indexLog(entry: LogEntry) {
  try {
    const indexName = `logs-${entry.service}-${new Date().toISOString().slice(0, 10)}`
    
    await esClient.index({
      index: indexName,
      document: {
        ...entry,
        '@timestamp': entry.timestamp
      }
    })
  } catch (err) {
    // Fail silently for logging errors
    console.error('Elasticsearch logging error:', err)
  }
}
```

## ตัวอย่าง: Production Monitoring Setup

```typescript
// server/plugins/monitoring.ts
export default defineNitroPlugin((nitro) => {
  // Track response times
  nitro.hooks.hook('request', (event) => {
    event.context._startTime = Date.now()
  })
  
  nitro.hooks.hook('afterResponse', async (event, response) => {
    const duration = Date.now() - (event.context._startTime || Date.now())
    const status = response?.status || 200
    const path = getRequestURL(event).pathname
    
    // Log slow requests (> 1s)
    if (duration > 1000) {
      logger.warn({
        path,
        duration,
        status,
        requestId: event.context.requestId
      }, 'Slow request detected')
    }
    
    // Track metrics
    await metrics.increment('http_requests_total', {
      method: getMethod(event),
      status: String(status),
      path: normalizePath(path)
    })
    
    await metrics.timing('http_request_duration', duration, {
      method: getMethod(event),
      status: String(status)
    })
  })
  
  nitro.hooks.hook('error', (error, context) => {
    logger.error({
      error: error.message,
      stack: error.stack,
      requestId: context.event?.context.requestId
    }, 'Unhandled server error')
  })
})

function normalizePath(path: string): string {
  // Normalize dynamic paths for metrics
  return path
    .replace(/\/[0-9a-f-]{36}/g, '/:id')
    .replace(/\/\d+/g, '/:id')
}
```

## สรุป

Logging และ Monitoring ที่ดีต้องมี:
1. Structured logging ด้วย pino
2. Error tracking ด้วย Sentry
3. Health checks สำหรับ load balancers
4. Application metrics
5. Request logging ทุก API call
6. Log aggregation ใน production
7. Alerting เมื่อเกิดปัญหา
