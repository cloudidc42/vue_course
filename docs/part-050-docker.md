# Part 50: Docker Deployment สำหรับ Nuxt.js

## Docker คืออะไร?

Docker เป็นแพลตฟอร์ม containerization ที่ช่วยให้แอปพลิเคชันทำงานได้เหมือนกันในทุก environment โดยไม่ขึ้นกับ OS หรือ dependencies ที่ติดตั้งในเครื่อง

## 1. Docker พื้นฐาน

```bash
# คำสั่ง Docker ที่ใช้บ่อย
docker build -t my-app .              # Build image
docker run -p 3000:3000 my-app        # Run container
docker ps                              # ดู running containers
docker stop <container-id>            # หยุด container
docker logs <container-id>            # ดู logs
docker exec -it <container-id> sh     # เข้าไปใน container
docker image prune                     # ลบ unused images
docker system prune                    # ลบทุกอย่างที่ไม่ใช้
```

## 2. Dockerfile สำหรับ Nuxt App

```dockerfile
# Dockerfile
# ============ Stage 1: Install Dependencies ============
FROM node:20-alpine AS deps

# ติดตั้ง system dependencies
RUN apk add --no-cache \
    libc6-compat \
    openssl

WORKDIR /app

# ติดตั้ง pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

# Copy package files เท่านั้น (layer caching)
COPY package.json pnpm-lock.yaml ./
COPY prisma ./prisma/

# ติดตั้ง dependencies
RUN pnpm install --frozen-lockfile

# ============ Stage 2: Build ============
FROM node:20-alpine AS builder

RUN apk add --no-cache libc6-compat openssl
WORKDIR /app

RUN corepack enable && corepack prepare pnpm@latest --activate

# Copy node_modules จาก deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma Client
RUN pnpm prisma generate

# Build arguments สำหรับ environment variables ที่ต้องการตอน build
ARG NUXT_PUBLIC_API_BASE
ARG NUXT_PUBLIC_SITE_URL
ARG NODE_ENV=production

ENV NUXT_PUBLIC_API_BASE=$NUXT_PUBLIC_API_BASE
ENV NUXT_PUBLIC_SITE_URL=$NUXT_PUBLIC_SITE_URL
ENV NODE_ENV=$NODE_ENV

# Build Nuxt app
RUN pnpm build

# ============ Stage 3: Production Runner ============
FROM node:20-alpine AS runner

RUN apk add --no-cache \
    libc6-compat \
    openssl \
    dumb-init  # Signal forwarding

WORKDIR /app

ENV NODE_ENV=production
ENV NITRO_PRESET=node-server

# สร้าง non-root user เพื่อความปลอดภัย
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nuxtjs

# Copy built output
COPY --from=builder --chown=nuxtjs:nodejs /app/.output ./

# Copy Prisma runtime
COPY --from=builder --chown=nuxtjs:nodejs /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder --chown=nuxtjs:nodejs /app/node_modules/@prisma/client ./node_modules/@prisma/client
COPY --from=builder --chown=nuxtjs:nodejs /app/prisma ./prisma

USER nuxtjs

EXPOSE 3000

ENV PORT=3000
ENV HOST=0.0.0.0

# ใช้ dumb-init เพื่อ handle signals อย่างถูกต้อง
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server/index.mjs"]

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget -q -O /dev/null http://localhost:3000/api/health || exit 1
```

```yaml
# .dockerignore
# Node
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Build outputs
.nuxt/
.output/
dist/

# Environment
.env
.env.*
!.env.example

# Git
.git/
.gitignore

# Development
*.md
.vscode/
.idea/

# Test
coverage/
tests/

# Misc
*.log
.DS_Store
Thumbs.db
```

## 3. Docker Compose - App + Database + Redis

```yaml
# docker-compose.yml
version: '3.9'

services:
  # ============ App ============
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runner
      args:
        NODE_ENV: production
        NUXT_PUBLIC_SITE_URL: ${NUXT_PUBLIC_SITE_URL:-http://localhost:3000}
    
    container_name: nuxt-app
    restart: unless-stopped
    
    ports:
      - "3000:3000"
    
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379
      NUXT_SESSION_PASSWORD: ${NUXT_SESSION_PASSWORD}
      NUXT_PUBLIC_SITE_URL: ${NUXT_PUBLIC_SITE_URL}
    
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    networks:
      - app-network
    
    volumes:
      - uploads:/app/public/uploads
    
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  
  # ============ PostgreSQL ============
  postgres:
    image: postgres:16-alpine
    container_name: nuxt-postgres
    restart: unless-stopped
    
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-appuser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
      POSTGRES_DB: ${POSTGRES_DB:-appdb}
      PGDATA: /var/lib/postgresql/data/pgdata
    
    ports:
      - "5432:5432"
    
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    
    networks:
      - app-network
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-appuser} -d ${POSTGRES_DB:-appdb}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
  
  # ============ Redis ============
  redis:
    image: redis:7-alpine
    container_name: nuxt-redis
    restart: unless-stopped
    
    command: redis-server --requirepass ${REDIS_PASSWORD:-changeme} --save 60 1 --loglevel warning
    
    ports:
      - "6379:6379"
    
    volumes:
      - redis_data:/data
    
    networks:
      - app-network
    
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # ============ Nginx ============
  nginx:
    image: nginx:alpine
    container_name: nuxt-nginx
    restart: unless-stopped
    
    ports:
      - "80:80"
      - "443:443"
    
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - uploads:/var/www/uploads:ro
    
    depends_on:
      - app
    
    networks:
      - app-network

# ============ Networks ============
networks:
  app-network:
    driver: bridge

# ============ Volumes ============
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  uploads:
    driver: local
```

```yaml
# docker-compose.override.yml (สำหรับ development)
version: '3.9'

services:
  app:
    build:
      target: deps  # ใช้ deps stage สำหรับ dev
    command: pnpm dev
    volumes:
      - .:/app
      - /app/node_modules
      - /app/.nuxt
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"
      - "24678:24678"  # HMR WebSocket
  
  postgres:
    ports:
      - "5432:5432"
  
  redis:
    ports:
      - "6379:6379"
  
  # PgAdmin สำหรับ development
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - app-network
```

## 4. Multi-stage Build Optimization

```dockerfile
# Dockerfile.optimized - ตัวอย่าง optimize สำหรับ production
FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Rebuild source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN \
  if [ -f yarn.lock ]; then yarn run build; \
  elif [ -f package-lock.json ]; then npm run build; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm run build; \
  else npm run build; \
  fi

# Production image, copy all the files and run
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nuxtjs

COPY --from=builder --chown=nuxtjs:nodejs /app/.output ./

USER nuxtjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server/index.mjs"]
```

## 5. Nginx Reverse Proxy

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Security Headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Hide server version
    server_tokens off;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
    
    # Upstream (Nuxt app)
    upstream nuxt_app {
        server app:3000;
        keepalive 32;
    }
    
    # HTTP -> HTTPS Redirect
    server {
        listen 80;
        server_name myapp.com www.myapp.com;
        
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
        
        location / {
            return 301 https://$server_name$request_uri;
        }
    }
    
    # HTTPS Server
    server {
        listen 443 ssl http2;
        server_name myapp.com www.myapp.com;
        
        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;
        
        # HSTS
        add_header Strict-Transport-Security "max-age=63072000" always;
        
        # Proxy to Nuxt
        location / {
            proxy_pass http://nuxt_app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
        
        # API Rate Limiting
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://nuxt_app;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # Login Rate Limiting
        location /api/auth/login {
            limit_req zone=login_limit burst=3 nodelay;
            proxy_pass http://nuxt_app;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }
        
        # Static Files Cache
        location /_nuxt/ {
            proxy_pass http://nuxt_app;
            proxy_cache_valid 200 1y;
            add_header Cache-Control "public, max-age=31536000, immutable";
        }
        
        # Uploads
        location /uploads/ {
            alias /var/www/uploads/;
            expires 30d;
            add_header Cache-Control "public, max-age=2592000";
        }
    }
}
```

## 6. Docker Networking

```yaml
# docker-compose.network.yml - ตัวอย่าง network configuration
version: '3.9'

networks:
  # Frontend network (Nginx -> App)
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  
  # Backend network (App -> DB/Redis)
  backend:
    driver: bridge
    internal: true  # ไม่สามารถเข้าถึงจาก outside
    ipam:
      config:
        - subnet: 172.21.0.0/16

services:
  nginx:
    networks:
      - frontend
  
  app:
    networks:
      - frontend
      - backend
  
  postgres:
    networks:
      - backend
  
  redis:
    networks:
      - backend
```

## 7. Production Setup พร้อม Scripts

```bash
#!/bin/bash
# scripts/deploy.sh - Production Deployment Script

set -e  # หยุดทันทีถ้ามี error

echo "🚀 Starting deployment..."

# ตรวจสอบ environment variables
required_vars=("DATABASE_URL" "NUXT_SESSION_PASSWORD" "POSTGRES_PASSWORD")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "❌ Error: $var is not set"
        exit 1
    fi
done

# Pull latest images
echo "📦 Pulling latest images..."
docker-compose pull

# Build new image
echo "🔨 Building application..."
docker-compose build --no-cache app

# Run database migrations
echo "🗄️ Running database migrations..."
docker-compose run --rm app sh -c "node_modules/.bin/prisma migrate deploy"

# Stop and remove old containers
echo "⏹️ Stopping old containers..."
docker-compose down --remove-orphans

# Start new containers
echo "▶️ Starting new containers..."
docker-compose up -d

# Wait for health check
echo "⏳ Waiting for application to be healthy..."
timeout 60 bash -c 'until docker-compose ps app | grep -q "healthy"; do sleep 2; done'

echo "✅ Deployment completed successfully!"
echo "🌐 Application is running at http://localhost:3000"
```

```bash
# scripts/backup.sh - Database Backup
#!/bin/bash

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/postgres_$DATE.sql.gz"

mkdir -p $BACKUP_DIR

echo "📦 Creating database backup..."
docker-compose exec -T postgres pg_dumpall -U $POSTGRES_USER | gzip > $BACKUP_FILE

echo "✅ Backup created: $BACKUP_FILE"

# ลบ backup ที่เก่ากว่า 30 วัน
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
echo "🗑️ Old backups cleaned up"
```

```bash
# .env.example
# Database
POSTGRES_USER=appuser
POSTGRES_PASSWORD=changeme_secure_password
POSTGRES_DB=myapp

DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}

# Redis
REDIS_PASSWORD=changeme_redis_password
REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379

# App
NUXT_SESSION_PASSWORD=at_least_32_characters_long_secret_key
NUXT_PUBLIC_SITE_URL=https://myapp.com

# OAuth
NUXT_OAUTH_GOOGLE_CLIENT_ID=
NUXT_OAUTH_GOOGLE_CLIENT_SECRET=
NUXT_OAUTH_GITHUB_CLIENT_ID=
NUXT_OAUTH_GITHUB_CLIENT_SECRET=

# Cloudinary
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
```

## 9. Docker Health Monitoring

```bash
# ดู status ของ containers
docker compose ps

# ดู resource usage
docker stats

# ดู logs realtime
docker compose logs -f --tail=100

# เข้าไปใน container
docker compose exec app sh

# ดู health check status
docker inspect --format='{{json .State.Health}}' nuxt-app | jq
```

```yaml
# docker-compose.monitoring.yml - เพิ่ม monitoring tools
services:
  # Prometheus - metrics collection
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring

  # Grafana - metrics visualization
  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3001:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring

  # Loki - log aggregation  
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

## 10. Docker Compose Commands

```bash
# Start ทุก service
docker compose up -d

# Rebuild image และ restart
docker compose up -d --build

# Stop ทุก service (เก็บ volumes)
docker compose down

# Stop และลบ volumes ด้วย
docker compose down -v

# Scale specific service
docker compose up -d --scale app=3

# ดู config ที่ merge แล้ว
docker compose config

# Run database migration
docker compose exec app npx prisma migrate deploy

# Seed database
docker compose exec app npx prisma db seed
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Docker Basics** - คำสั่งพื้นฐานที่จำเป็น
2. **Dockerfile** - Multi-stage build สำหรับ Nuxt
3. **Docker Compose** - App + PostgreSQL + Redis + Nginx
4. **Multi-stage Builds** - optimize image size
5. **Environment Variables** - จัดการ config ใน Docker
6. **Nginx Reverse Proxy** - SSL, rate limiting, caching
7. **Docker Networking** - ความปลอดภัยของ network
8. **Production Setup** - Deploy scripts และ backup
9. **Health Monitoring** - Prometheus, Grafana, Loki
10. **Docker Commands** - คำสั่งสำคัญสำหรับ production
