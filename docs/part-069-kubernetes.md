# Part 69: Kubernetes Deployment สำหรับ Nuxt.js

## Kubernetes พื้นฐาน

Kubernetes (K8s) คือระบบ container orchestration ที่ช่วยจัดการ containers ในระดับ production

### Kubernetes Objects หลัก
- **Pod** - หน่วยที่เล็กที่สุด ประกอบด้วย 1+ containers
- **Deployment** - จัดการ Pods ให้มีจำนวนตามที่กำหนด
- **Service** - expose Pods ให้เข้าถึงได้
- **Ingress** - routing traffic เข้า cluster
- **ConfigMap** - เก็บ configuration ที่ไม่ sensitive
- **Secret** - เก็บ sensitive data
- **HPA** - Horizontal Pod Autoscaler

## Deployment สำหรับ Nuxt

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
  namespace: production
  labels:
    app: nuxt-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nuxt-app
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # สร้าง Pod ใหม่ก่อน
      maxUnavailable: 0   # ไม่ให้ Pod ล้มเลย (zero-downtime)
  
  template:
    metadata:
      labels:
        app: nuxt-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/api/metrics"
    
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Image pull secrets
      imagePullSecrets:
        - name: registry-credentials
      
      containers:
        - name: nuxt-app
          image: registry.example.com/nuxt-app:1.0.0
          imagePullPolicy: Always
          
          ports:
            - containerPort: 3000
              protocol: TCP
          
          # Environment from ConfigMap and Secrets
          envFrom:
            - configMapRef:
                name: nuxt-app-config
            - secretRef:
                name: nuxt-app-secrets
          
          # Additional env vars
          env:
            - name: NODE_ENV
              value: "production"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          
          # Resource limits
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /api/health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          readinessProbe:
            httpGet:
              path: /api/health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3
          
          # Startup probe (for slow starting apps)
          startupProbe:
            httpGet:
              path: /api/health/live
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
          
          # Security
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          
          # Volumes
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: nuxt-cache
              mountPath: /app/.nuxt/cache
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: nuxt-cache
          emptyDir: {}
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
      
      # Topology spread (เพื่อกระจาย Pods ต่างๆ node)
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: nuxt-app
```

## Service และ Ingress

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nuxt-app-service
  namespace: production
  labels:
    app: nuxt-app
spec:
  type: ClusterIP  # Internal only - expose via Ingress
  selector:
    app: nuxt-app
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
      name: http
  
  # Session affinity (optional)
  sessionAffinity: None
```

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nuxt-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Compression
    nginx.ingress.kubernetes.io/enable-brotli: "true"
    
    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;

spec:
  tls:
    - hosts:
        - myapp.com
        - www.myapp.com
      secretName: myapp-tls
  
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nuxt-app-service
                port:
                  number: 80
    
    - host: www.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nuxt-app-service
                port:
                  number: 80
```

## ConfigMap และ Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nuxt-app-config
  namespace: production
data:
  APP_URL: "https://myapp.com"
  APP_NAME: "My Nuxt App"
  APP_VERSION: "1.0.0"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"
  SMTP_HOST: "smtp.sendgrid.net"
  SMTP_PORT: "587"
```

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nuxt-app-secrets
  namespace: production
type: Opaque
# Values must be base64 encoded
# echo -n 'value' | base64
stringData:
  DATABASE_URL: "postgresql://user:pass@postgres-service:5432/mydb"
  JWT_SECRET: "your-super-secret-jwt-key-min-64-chars-long"
  STRIPE_SECRET_KEY: "sk_live_xxx"
  STRIPE_WEBHOOK_SECRET: "whsec_xxx"
  RESEND_API_KEY: "re_xxx"
  SENTRY_DSN: "https://xxx@sentry.io/yyy"
```

```yaml
# External secrets (ใช้ External Secrets Operator)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: nuxt-app-external-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: nuxt-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: myapp/production
        property: DATABASE_URL
    - secretKey: JWT_SECRET
      remoteRef:
        key: myapp/production
        property: JWT_SECRET
```

## Horizontal Pod Autoscaling

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nuxt-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nuxt-app
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
    # Scale based on CPU usage
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Scale based on Memory usage
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Custom metric: requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

## Rolling Updates

```bash
# Update deployment image
kubectl set image deployment/nuxt-app \
  nuxt-app=registry.example.com/nuxt-app:2.0.0 \
  --namespace=production

# Watch rollout status
kubectl rollout status deployment/nuxt-app \
  --namespace=production \
  --timeout=300s

# Rollback if needed
kubectl rollout undo deployment/nuxt-app \
  --namespace=production

# Rollback to specific revision
kubectl rollout undo deployment/nuxt-app \
  --to-revision=3 \
  --namespace=production

# Check rollout history
kubectl rollout history deployment/nuxt-app \
  --namespace=production
```

## ตัวอย่าง: K8s Manifests สำหรับ Nuxt App

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
    environment: production
```

```yaml
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          volumeMounts:
            - name: redis-data
              mountPath: /data
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: production
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
```

```yaml
# k8s/kustomization.yaml
# Using Kustomize for environment-specific configs
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - namespace.yaml
  - configmap.yaml
  - secret.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - hpa.yaml
  - redis.yaml

images:
  - name: registry.example.com/nuxt-app
    newTag: "1.0.0"

commonLabels:
  app.kubernetes.io/managed-by: kustomize
  app.kubernetes.io/part-of: nuxt-app
```

## Network Policy

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nuxt-app-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: nuxt-app
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow traffic from Ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
  
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
    
    # Allow traffic to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    
    # Allow traffic to Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    
    # Allow external HTTPS (for Stripe, Sentry, etc.)
    - ports:
        - protocol: TCP
          port: 443
```

## Pod Disruption Budget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nuxt-app-pdb
  namespace: production
spec:
  minAvailable: 2  # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: nuxt-app
```

## Monitoring with Prometheus

```yaml
# k8s/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nuxt-app-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: nuxt-app
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: http
      path: /api/metrics
      interval: 30s
      scrapeTimeout: 10s
```

## ตัวอย่าง: kubectl Commands ที่ใช้บ่อย

```bash
# ดู pods ทั้งหมดใน namespace
kubectl get pods -n production

# ดู logs ของ pod
kubectl logs -f deployment/nuxt-app -n production

# Exec เข้า pod
kubectl exec -it pod/nuxt-app-xxx -n production -- /bin/sh

# Scale deployment
kubectl scale deployment nuxt-app --replicas=5 -n production

# ดู resource usage
kubectl top pods -n production

# ดู events
kubectl get events -n production --sort-by='.lastTimestamp'

# Apply configuration
kubectl apply -k k8s/

# Delete deployment (careful!)
kubectl delete deployment nuxt-app -n production

# Port forward สำหรับ debug
kubectl port-forward deployment/nuxt-app 3000:3000 -n production

# ดู horizontal pod autoscaler
kubectl get hpa -n production

# Describe pod สำหรับ troubleshoot
kubectl describe pod nuxt-app-xxx -n production
```

## สรุป

Kubernetes Deployment ที่ดีต้องมี:
1. Resource limits เสมอ เพื่อป้องกัน resource starvation
2. Health checks ทั้ง liveness, readiness, startup
3. Rolling update strategy เพื่อ zero-downtime
4. HPA สำหรับ auto-scaling ตาม load
5. Secrets management ที่ปลอดภัย ด้วย External Secrets
6. Topology spread constraints เพื่อ high availability
7. Graceful shutdown ก่อน pod termination
8. Network Policy เพื่อความปลอดภัย
9. Pod Disruption Budget สำหรับ maintenance
10. Monitoring ด้วย Prometheus/Grafana
