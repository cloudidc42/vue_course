# Part 49: CI/CD Pipeline สำหรับ Nuxt.js

## CI/CD คืออะไร?

- **CI (Continuous Integration)** - การรวม code โดยอัตโนมัติ พร้อม run tests ทุกครั้งที่ push
- **CD (Continuous Delivery/Deployment)** - การ deploy อัตโนมัติเมื่อผ่าน tests

## 1. GitHub Actions Setup

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch: # สามารถ trigger manual ได้

env:
  NODE_VERSION: '20'
  PNPM_VERSION: '8'

jobs:
  # ============ Code Quality ============
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Run ESLint
        run: pnpm lint
      
      - name: Run TypeScript check
        run: pnpm typecheck
      
      - name: Run Prettier check
        run: pnpm format:check
  
  # ============ Tests ============
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: quality
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    env:
      DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
      NODE_ENV: test
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Run database migrations
        run: pnpm prisma migrate deploy
      
      - name: Run unit tests
        run: pnpm test:unit --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
  
  # ============ Build ============
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [quality, test]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Build application
        run: pnpm build
        env:
          NUXT_PUBLIC_API_BASE: ${{ vars.NUXT_PUBLIC_API_BASE }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: .output/
          retention-days: 7
  
  # ============ Deploy to Staging ============
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: .output/
      
      - name: Deploy to Vercel (Staging)
        run: |
          npx vercel --token ${{ secrets.VERCEL_TOKEN }} \
            --yes \
            --env NODE_ENV=staging
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
  
  # ============ Deploy to Production ============
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: .output/
      
      - name: Deploy to Vercel (Production)
        run: |
          npx vercel --token ${{ secrets.VERCEL_TOKEN }} \
            --prod \
            --yes
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
      
      - name: Notify on Slack
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "Deployment to production: ${{ job.status }}\nCommit: ${{ github.sha }}\nAuthor: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 2. Automated Testing

```typescript
// tests/unit/utils.test.ts
import { describe, it, expect } from 'vitest'
import { generateSlug, formatFileSize, validateEmail } from '~/utils'

describe('generateSlug', () => {
  it('แปลงข้อความเป็น slug', () => {
    expect(generateSlug('Hello World')).toBe('hello-world')
  })
  
  it('ลบอักขระพิเศษ', () => {
    expect(generateSlug('Hello, World!')).toBe('hello-world')
  })
  
  it('จัดการภาษาไทย', () => {
    expect(generateSlug('สวัสดี Vue')).toContain('vue')
  })
})

describe('formatFileSize', () => {
  it('แสดงขนาดใน Bytes', () => {
    expect(formatFileSize(500)).toBe('500 B')
  })
  
  it('แสดงขนาดใน KB', () => {
    expect(formatFileSize(1500)).toBe('1.5 KB')
  })
  
  it('แสดงขนาดใน MB', () => {
    expect(formatFileSize(1500000)).toBe('1.4 MB')
  })
})
```

```typescript
// tests/api/posts.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import { setup, $fetch } from '@nuxt/test-utils'

describe('Posts API', () => {
  let testPostId: string
  
  beforeEach(async () => {
    // สร้าง test data
    const post = await prisma.post.create({
      data: {
        title: 'Test Post',
        slug: 'test-post-' + Date.now(),
        content: 'Test content',
        published: true,
        publishedAt: new Date(),
        authorId: 'test-user-id'
      }
    })
    testPostId = post.id
  })
  
  afterEach(async () => {
    // ลบ test data
    await prisma.post.delete({ where: { id: testPostId } }).catch(() => {})
  })
  
  it('GET /api/posts - ดึงรายการโพสต์', async () => {
    const result = await $fetch('/api/posts')
    
    expect(result).toHaveProperty('posts')
    expect(result).toHaveProperty('pagination')
    expect(Array.isArray(result.posts)).toBe(true)
  })
  
  it('GET /api/posts - รองรับ pagination', async () => {
    const result = await $fetch('/api/posts?page=1&limit=5')
    
    expect(result.pagination.page).toBe(1)
    expect(result.pagination.limit).toBe(5)
    expect(result.posts.length).toBeLessThanOrEqual(5)
  })
  
  it('POST /api/posts - ต้อง authenticate', async () => {
    try {
      await $fetch('/api/posts', {
        method: 'POST',
        body: { title: 'Test', content: 'Content' }
      })
      expect(true).toBe(false) // Should not reach here
    } catch (error: any) {
      expect(error.response.status).toBe(401)
    }
  })
})
```

## 3. Docker Build

```dockerfile
# Dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# ติดตั้ง pnpm
RUN corepack enable && corepack prepare pnpm@8 --activate

# Copy package files
COPY package.json pnpm-lock.yaml ./
COPY prisma ./prisma/

# Install dependencies
RUN pnpm install --frozen-lockfile

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@8 --activate

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma Client
RUN pnpm prisma generate

# Build application
ENV NODE_ENV=production
RUN pnpm build

# Stage 3: Runner (Production)
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# สร้าง non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nuxtjs

# Copy built files
COPY --from=builder --chown=nuxtjs:nodejs /app/.output ./

# Copy Prisma files
COPY --from=builder --chown=nuxtjs:nodejs /app/prisma ./prisma
COPY --from=builder --chown=nuxtjs:nodejs /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder --chown=nuxtjs:nodejs /app/node_modules/@prisma ./node_modules/@prisma

USER nuxtjs

EXPOSE 3000

ENV PORT=3000
ENV HOST=0.0.0.0

CMD ["node", "server/index.mjs"]
```

```yaml
# .dockerignore
node_modules
.git
.nuxt
.output
*.log
.env
.env.*
README.md
```

## 4. Deploy to Vercel

```json
// vercel.json
{
  "version": 2,
  "framework": "nuxtjs",
  "buildCommand": "pnpm build",
  "outputDirectory": ".output/public",
  "installCommand": "pnpm install",
  "regions": ["sin1"],
  "env": {
    "NODE_ENV": "production"
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        }
      ]
    },
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "no-store, max-age=0"
        }
      ]
    }
  ],
  "rewrites": [
    {
      "source": "/((?!api/).*)",
      "destination": "/"
    }
  ]
}
```

## 5. Deploy to Netlify

```toml
# netlify.toml
[build]
  command = "pnpm build"
  publish = ".output/public"

[build.environment]
  NODE_VERSION = "20"
  NODE_ENV = "production"

[[plugins]]
  package = "@netlify/plugin-nuxt"

[functions]
  directory = ".output/server"
  node_bundler = "esbuild"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Cache-Control = "public, max-age=0, must-revalidate"

[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

## 6. Environment Variables ใน CI/CD

```yaml
# .github/workflows/deploy.yml
- name: Deploy
  run: pnpm deploy
  env:
    # จาก GitHub Secrets
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    NUXT_SESSION_PASSWORD: ${{ secrets.NUXT_SESSION_PASSWORD }}
    
    # จาก GitHub Variables (non-sensitive)
    NUXT_PUBLIC_API_BASE: ${{ vars.NUXT_PUBLIC_API_BASE }}
    NUXT_PUBLIC_SITE_URL: ${{ vars.NUXT_PUBLIC_SITE_URL }}
```

```bash
# ตั้งค่า Secrets ผ่าน GitHub CLI
gh secret set DATABASE_URL --body "postgresql://..."
gh secret set NUXT_SESSION_PASSWORD --body "$(openssl rand -base64 32)"

# ตั้งค่า Variables (non-sensitive)
gh variable set NUXT_PUBLIC_API_BASE --body "https://api.myapp.com"
```

## 7. Complete GitHub Actions Pipeline

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: ['**']
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      app: ${{ steps.filter.outputs.app }}
      docs: ${{ steps.filter.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            app:
              - '*.json'
              - 'src/**'
              - 'server/**'
              - 'pages/**'
              - 'components/**'
            docs:
              - 'docs/**'
              - '*.md'
  
  lint-and-test:
    needs: changes
    if: ${{ needs.changes.outputs.app == 'true' }}
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: 8
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      
      - run: pnpm install --frozen-lockfile
      
      - name: Lint
        run: pnpm lint
      
      - name: Type Check
        run: pnpm typecheck
      
      - name: Test
        run: pnpm test
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - name: Build
        run: pnpm build
        env:
          NUXT_PUBLIC_SITE_URL: https://myapp.com
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run security audit
        run: pnpm audit --audit-level high
      
      - name: SAST scan
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript
      
      - uses: github/codeql-action/analyze@v3
  
  deploy:
    needs: [lint-and-test, security-scan]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: 8
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      
      - run: pnpm install --frozen-lockfile
      
      - name: Run database migrations
        run: pnpm prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
      
      - name: Deploy to Vercel
        id: deploy
        run: |
          URL=$(npx vercel --token ${{ secrets.VERCEL_TOKEN }} --prod --yes 2>&1 | tail -1)
          echo "url=$URL" >> $GITHUB_OUTPUT
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          NUXT_SESSION_PASSWORD: ${{ secrets.NUXT_SESSION_PASSWORD }}
      
      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.run_number }}
          release_name: Release v${{ github.run_number }}
          draft: false
          prerelease: false
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GitHub Actions** - ตั้งค่า CI/CD pipeline
2. **Automated Testing** - Unit tests และ API tests
3. **Build Process** - Build และ cache artifacts
4. **Docker Build** - Multi-stage Dockerfile
5. **Deploy to Vercel/Netlify** - การ deploy อัตโนมัติ
6. **Environment Variables** - จัดการ secrets ใน CI/CD
7. **Complete Pipeline** - Pipeline ที่ครบถ้วน
