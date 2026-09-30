# Part 68: Production Architecture สำหรับ Nuxt.js

## Production Checklist

### Performance
- [ ] Enable Nuxt compression
- [ ] Set up CDN สำหรับ static assets
- [ ] Optimize images ด้วย Nuxt Image
- [ ] Code splitting เปิดอยู่ (default)
- [ ] Tree shaking ทำงานถูกต้อง
- [ ] Route preloading ตั้งค่าแล้ว

### Security
- [ ] HTTPS บังคับใช้
- [ ] Security headers ตั้งค่าแล้ว
- [ ] CORS policy กำหนดแล้ว
- [ ] Rate limiting เปิดอยู่
- [ ] Input validation ทำงานทั้ง client/server

### Reliability
- [ ] Health checks endpoints พร้อม
- [ ] Error tracking (Sentry) ตั้งค่าแล้ว
- [ ] Logging ทำงานถูกต้อง
- [ ] Backup strategy กำหนดแล้ว

## Environment Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // Production optimizations
  nitro: {
    compressPublicAssets: {
      gzip: true,
      brotli: true
    },
    minify: true,
    timing: false // Disable timing headers in production
  },
  
  // Vite optimizations
  vite: {
    build: {
      target: 'es2020',
      rollupOptions: {
        output: {
          manualChunks: {
            'vendor': ['vue', 'vue-router', 'pinia'],
            'ui': ['@headlessui/vue']
          }
        }
      }
    }
  },
  
  // Route rules
  routeRules: {
    '/api/**': {
      cors: true,
      headers: {
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization'
      }
    }
  }
})
```

```typescript
// .env.production
NODE_ENV=production
APP_URL=https://myapp.com
APP_VERSION=1.0.0

# Database
DATABASE_URL=postgresql://user:pass@db-host:5432/mydb

# Redis
REDIS_URL=redis://redis-host:6379

# Authentication
JWT_SECRET=<strong-random-secret-min-64-chars>
JWT_EXPIRE=7d

# Email
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
EMAIL_FROM=noreply@myapp.com

# Monitoring
SENTRY_DSN=https://xxx@sentry.io/yyy
LOG_LEVEL=info
```

## Secret Management

```typescript
// server/utils/config.ts
function requireEnv(key: string): string {
  const value = process.env[key]
  if (!value) {
    throw new Error(`Missing required environment variable: ${key}`)
  }
  return value
}

export const serverConfig = {
  database: {
    url: requireEnv('DATABASE_URL')
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379'
  },
  jwt: {
    secret: requireEnv('JWT_SECRET'),
    expire: process.env.JWT_EXPIRE || '7d'
  },
  email: {
    from: requireEnv('EMAIL_FROM'),
    smtpHost: process.env.SMTP_HOST || '',
    resendKey: process.env.RESEND_API_KEY || ''
  },
  stripe: {
    secretKey: requireEnv('STRIPE_SECRET_KEY'),
    webhookSecret: requireEnv('STRIPE_WEBHOOK_SECRET')
  }
}
```

## Zero-downtime Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run type-check
      - run: npm run test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build \
            --build-arg APP_VERSION=${{ github.sha }} \
            -t myapp:${{ github.sha }} .
      
      - name: Push to registry
        run: |
          docker push registry.example.com/myapp:${{ github.sha }}
          docker push registry.example.com/myapp:latest
      
      - name: Deploy with zero-downtime
        run: |
          # Rolling update
          kubectl set image deployment/myapp \
            myapp=registry.example.com/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=300s
```

## Health Checks

```typescript
// server/api/health.get.ts
export default defineEventHandler(async (event) => {
  const checks = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || 'unknown',
    uptime: Math.floor(process.uptime()),
    memory: {
      used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
      total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024)
    }
  }
  
  return checks
})
```

## Graceful Shutdown

```typescript
// server/plugins/graceful-shutdown.ts
export default defineNitroPlugin((nitro) => {
  let isShuttingDown = false
  
  const shutdown = async (signal: string) => {
    if (isShuttingDown) return
    isShuttingDown = true
    
    console.log(`Received ${signal}. Starting graceful shutdown...`)
    
    // Stop accepting new requests
    // Complete existing requests
    
    try {
      // Close database connections
      await prisma.$disconnect()
      console.log('Database connections closed')
      
      // Close Redis
      await getRedis().quit()
      console.log('Redis connections closed')
      
    } catch (err) {
      console.error('Error during shutdown:', err)
    }
    
    console.log('Graceful shutdown completed')
    process.exit(0)
  }
  
  process.on('SIGTERM', () => shutdown('SIGTERM'))
  process.on('SIGINT', () => shutdown('SIGINT'))
  
  // Handle uncaught exceptions
  process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error)
    shutdown('uncaughtException')
  })
  
  process.on('unhandledRejection', (reason) => {
    console.error('Unhandled Rejection:', reason)
  })
})
```

## Production-ready Nuxt Configuration

```typescript
// nuxt.config.ts (production-ready)
export default defineNuxtConfig({
  devtools: { enabled: process.env.NODE_ENV === 'development' },
  
  typescript: {
    strict: true,
    typeCheck: true
  },
  
  // Security headers
  security: {
    headers: {
      contentSecurityPolicy: {
        'default-src': ["'self'"],
        'script-src': ["'self'", "'nonce-{{nonce}}'"],
        'style-src': ["'self'", "'unsafe-inline'"],
        'img-src': ["'self'", 'data:', 'https:'],
        'font-src': ["'self'", 'https://fonts.gstatic.com'],
        'connect-src': ["'self'", 'https://api.stripe.com'],
        'frame-src': ["'self'", 'https://js.stripe.com']
      },
      xFrameOptions: 'SAMEORIGIN',
      xContentTypeOptions: 'nosniff',
      referrerPolicy: 'strict-origin-when-cross-origin',
      permissionsPolicy: {
        camera: [],
        microphone: [],
        geolocation: []
      }
    },
    rateLimiter: {
      tokensPerInterval: 150,
      interval: 'hour',
      headers: true
    }
  },
  
  // Runtime config
  runtimeConfig: {
    // Private (server only)
    databaseUrl: process.env.DATABASE_URL,
    jwtSecret: process.env.JWT_SECRET,
    stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    
    // Public (client + server)
    public: {
      appUrl: process.env.APP_URL,
      stripePublicKey: process.env.STRIPE_PUBLIC_KEY,
      gaId: process.env.GA_MEASUREMENT_ID
    }
  },
  
  // Nitro configuration
  nitro: {
    preset: 'node-server', // or 'vercel', 'cloudflare', etc.
    
    compressPublicAssets: {
      gzip: true,
      brotli: true
    },
    
    storage: {
      redis: {
        driver: 'redis',
        url: process.env.REDIS_URL
      }
    },
    
    // Bundled dependencies
    externals: {
      inline: ['prisma', '@prisma/client']
    }
  },
  
  // Image optimization
  image: {
    provider: 'cloudinary',
    cloudinary: {
      baseURL: process.env.CLOUDINARY_BASE_URL
    }
  },
  
  // Modules
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    'nuxt-security',
    '@nuxt/image',
    '@nuxtjs/i18n'
  ]
})
```

## Docker Production Setup

```dockerfile
# Dockerfile
FROM node:20-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable

# Dependencies
FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile

# Build
FROM base AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG APP_VERSION
ENV APP_VERSION=$APP_VERSION
RUN pnpm build

# Production
FROM base AS production
WORKDIR /app
ENV NODE_ENV=production

# Copy only necessary files
COPY --from=build /app/.output ./

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/api/health || exit 1

USER node
CMD ["node", "server/index.mjs"]
```

## สรุป

Production Architecture ที่ดีต้องมี:
1. Environment configuration แยกชัดเจน
2. Secret management ที่ปลอดภัย
3. Zero-downtime deployment
4. Health checks
5. Graceful shutdown
6. Security headers
7. Performance optimizations
