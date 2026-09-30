# Part 80: Enterprise Architecture

## Enterprise Architecture คืออะไร?

Enterprise applications ต้องการ architecture ที่ scalable, maintainable และรองรับ complex business requirements การออกแบบที่ดีต้องคำนึงถึง separation of concerns, testability และความสามารถในการ extend ระบบ

---

## 1. Enterprise Vue.js Architecture

### Layered Architecture

```
┌────────────────────────────────────────┐
│          Presentation Layer            │
│  (Vue Components, Pages, Layouts)      │
├────────────────────────────────────────┤
│          Application Layer             │
│  (Stores, Use Cases, Orchestration)    │
├────────────────────────────────────────┤
│           Domain Layer                 │
│  (Business Logic, Domain Models)       │
├────────────────────────────────────────┤
│        Infrastructure Layer            │
│  (API, Database, External Services)    │
└────────────────────────────────────────┘
```

### Directory Structure

```
enterprise-app/
├── src/
│   ├── domain/              # Domain models & business logic
│   │   ├── user/
│   │   │   ├── User.ts      # Domain model
│   │   │   ├── UserService.ts  # Domain service
│   │   │   └── UserRepository.ts  # Interface
│   │   └── order/
│   │       ├── Order.ts
│   │       ├── OrderService.ts
│   │       └── OrderRepository.ts
│   │
│   ├── application/         # Use cases / Application services
│   │   ├── user/
│   │   │   ├── CreateUserUseCase.ts
│   │   │   └── UpdateUserUseCase.ts
│   │   └── order/
│   │       ├── PlaceOrderUseCase.ts
│   │       └── CancelOrderUseCase.ts
│   │
│   ├── infrastructure/      # External implementations
│   │   ├── api/
│   │   │   ├── UserApiRepository.ts
│   │   │   └── OrderApiRepository.ts
│   │   └── cache/
│   │       └── RedisCacheService.ts
│   │
│   └── presentation/        # UI layer
│       ├── components/
│       ├── pages/
│       └── stores/
```

---

## 2. Domain-Driven Design กับ Nuxt

### Domain Model

```typescript
// domain/order/Order.ts

export class OrderItem {
  constructor(
    public readonly productId: string,
    public readonly productName: string,
    public readonly quantity: number,
    public readonly unitPrice: number
  ) {
    if (quantity <= 0) throw new Error('Quantity must be positive')
    if (unitPrice < 0) throw new Error('Unit price cannot be negative')
  }

  get totalPrice(): number {
    return this.quantity * this.unitPrice
  }
}

export type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled'

export class Order {
  private _items: OrderItem[] = []
  private _status: OrderStatus = 'pending'
  private _events: DomainEvent[] = []

  constructor(
    public readonly id: string,
    public readonly customerId: string,
    public readonly tenantId: string,
    private _createdAt: Date = new Date()
  ) {}

  addItem(item: OrderItem): void {
    if (this._status !== 'pending') {
      throw new DomainError('Cannot add items to non-pending order')
    }

    const existing = this._items.find(i => i.productId === item.productId)
    if (existing) {
      this._items = this._items.map(i =>
        i.productId === item.productId
          ? new OrderItem(i.productId, i.productName, i.quantity + item.quantity, i.unitPrice)
          : i
      )
    } else {
      this._items = [...this._items, item]
    }
  }

  confirm(): void {
    if (this._status !== 'pending') {
      throw new DomainError(`Cannot confirm order with status: ${this._status}`)
    }
    if (this._items.length === 0) {
      throw new DomainError('Cannot confirm empty order')
    }

    this._status = 'confirmed'
    this._events.push(new OrderConfirmedEvent(this.id, this.customerId))
  }

  cancel(reason: string): void {
    if (['shipped', 'delivered'].includes(this._status)) {
      throw new DomainError(`Cannot cancel order with status: ${this._status}`)
    }

    this._status = 'cancelled'
    this._events.push(new OrderCancelledEvent(this.id, reason))
  }

  get items(): readonly OrderItem[] { return this._items }
  get status(): OrderStatus { return this._status }
  get total(): number { return this._items.reduce((sum, item) => sum + item.totalPrice, 0) }
  get domainEvents(): readonly DomainEvent[] { return this._events }
  get createdAt(): Date { return this._createdAt }

  clearEvents(): void { this._events = [] }
}

// Domain errors
export class DomainError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'DomainError'
  }
}
```

### Repository Interface

```typescript
// domain/order/OrderRepository.ts
export interface OrderRepository {
  findById(id: string, tenantId: string): Promise<Order | null>
  findByCustomerId(customerId: string, tenantId: string): Promise<Order[]>
  save(order: Order): Promise<void>
  delete(id: string, tenantId: string): Promise<void>
}
```

---

## 3. CQRS Pattern

### Command/Query Separation

```typescript
// application/commands/PlaceOrderCommand.ts
export interface PlaceOrderCommand {
  customerId: string
  tenantId: string
  items: Array<{
    productId: string
    quantity: number
  }>
}

// application/queries/GetOrderQuery.ts
export interface GetOrderQuery {
  orderId: string
  tenantId: string
}

// application/handlers/PlaceOrderHandler.ts
import { inject, injectable } from 'tsyringe'

@injectable()
export class PlaceOrderHandler {
  constructor(
    @inject('OrderRepository') private orderRepo: OrderRepository,
    @inject('ProductRepository') private productRepo: ProductRepository,
    @inject('EventBus') private eventBus: EventBus
  ) {}

  async execute(command: PlaceOrderCommand): Promise<string> {
    // ดึงข้อมูล products
    const productIds = command.items.map(i => i.productId)
    const products = await this.productRepo.findByIds(productIds, command.tenantId)

    if (products.length !== productIds.length) {
      throw new DomainError('Some products were not found')
    }

    // สร้าง Order domain object
    const order = new Order(
      generateId(),
      command.customerId,
      command.tenantId
    )

    for (const item of command.items) {
      const product = products.find(p => p.id === item.productId)!
      order.addItem(new OrderItem(
        product.id,
        product.name,
        item.quantity,
        product.price
      ))
    }

    order.confirm()

    // Persist
    await this.orderRepo.save(order)

    // Publish domain events
    for (const event of order.domainEvents) {
      await this.eventBus.publish(event)
    }

    order.clearEvents()

    return order.id
  }
}

// Query Handler (read model)
@injectable()
export class GetOrderQueryHandler {
  constructor(
    @inject('OrderReadModel') private orderReadModel: OrderReadModel
  ) {}

  async execute(query: GetOrderQuery) {
    // Read model ถูก optimize สำหรับ reading
    return this.orderReadModel.findById(query.orderId, query.tenantId)
  }
}
```

---

## 4. Event Sourcing

### Event Store

```typescript
// infrastructure/EventStore.ts
interface StoredEvent {
  id: string
  aggregateId: string
  aggregateType: string
  eventType: string
  payload: unknown
  metadata: { tenantId: string; userId?: string; timestamp: Date }
  version: number
}

export class EventStore {
  async append(
    aggregateId: string,
    events: DomainEvent[],
    expectedVersion: number
  ): Promise<void> {
    // Optimistic concurrency check
    const currentVersion = await this.getVersion(aggregateId)
    if (currentVersion !== expectedVersion) {
      throw new ConcurrencyError(
        `Expected version ${expectedVersion}, got ${currentVersion}`
      )
    }

    const storedEvents = events.map((event, i) => ({
      id: generateId(),
      aggregateId,
      aggregateType: event.aggregateType,
      eventType: event.type,
      payload: event.payload,
      metadata: event.metadata,
      version: expectedVersion + i + 1
    }))

    await prisma.event.createMany({ data: storedEvents })
  }

  async getEvents(
    aggregateId: string,
    fromVersion = 0
  ): Promise<StoredEvent[]> {
    return prisma.event.findMany({
      where: { aggregateId, version: { gt: fromVersion } },
      orderBy: { version: 'asc' }
    })
  }

  private async getVersion(aggregateId: string): Promise<number> {
    const latest = await prisma.event.findFirst({
      where: { aggregateId },
      orderBy: { version: 'desc' },
      select: { version: true }
    })
    return latest?.version || 0
  }
}
```

---

## 5. Microservices Integration

### API Gateway Pattern

```typescript
// server/middleware/api-gateway.ts
const SERVICE_MAP: Record<string, string> = {
  'users': process.env.USER_SERVICE_URL!,
  'orders': process.env.ORDER_SERVICE_URL!,
  'payments': process.env.PAYMENT_SERVICE_URL!,
  'notifications': process.env.NOTIFICATION_SERVICE_URL!
}

export default defineEventHandler(async (event) => {
  const url = getRequestURL(event)
  const segments = url.pathname.split('/').filter(Boolean)

  if (segments[0] !== 'gateway') return

  const service = segments[1]
  const serviceUrl = SERVICE_MAP[service]

  if (!serviceUrl) {
    throw createError({ statusCode: 404, message: `Service '${service}' not found` })
  }

  // Forward request
  const targetPath = '/' + segments.slice(2).join('/')
  const response = await fetch(`${serviceUrl}${targetPath}${url.search}`, {
    method: event.method,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': getHeader(event, 'authorization') || '',
      'X-Tenant-ID': event.context.tenantId || '',
      'X-Request-ID': getHeader(event, 'x-request-id') || generateId()
    },
    body: ['GET', 'HEAD'].includes(event.method) 
      ? undefined 
      : await readRawBody(event)
  })

  const data = await response.json()

  setResponseStatus(event, response.status)

  return data
})
```

---

## 6. ตัวอย่าง: Enterprise App Architecture

### Dependency Injection Setup

```typescript
// infrastructure/container.ts
import { container } from 'tsyringe'

// Register implementations
container.register('OrderRepository', {
  useClass: PrismaOrderRepository
})

container.register('EventBus', {
  useClass: RedisEventBus
})

container.register('CacheService', {
  useClass: RedisCacheService
})

// Register use case handlers
container.register('PlaceOrderHandler', PlaceOrderHandler)
container.register('GetOrderHandler', GetOrderQueryHandler)
```

### Nuxt API Integration

```typescript
// server/api/orders/index.post.ts
import { container } from '~/infrastructure/container'

export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const body = await validateBody(event, PlaceOrderSchema)
  const handler = container.resolve(PlaceOrderHandler)

  const orderId = await handler.execute({
    customerId: event.context.userId,
    tenantId: event.context.tenantId,
    items: body.items
  })

  setResponseStatus(event, 201)
  return createSuccessResponse({ orderId })
})
```

### Event-Driven Integration

```typescript
// infrastructure/RedisEventBus.ts
import { createClient } from 'redis'

export class RedisEventBus implements EventBus {
  private publisher = createClient({ url: process.env.REDIS_URL })
  private subscriber = createClient({ url: process.env.REDIS_URL })
  private handlers: Map<string, EventHandler[]> = new Map()

  async publish(event: DomainEvent): Promise<void> {
    await this.publisher.publish(
      `domain:${event.type}`,
      JSON.stringify(event)
    )
  }

  async subscribe(eventType: string, handler: EventHandler): Promise<void> {
    const existing = this.handlers.get(eventType) || []
    this.handlers.set(eventType, [...existing, handler])

    await this.subscriber.subscribe(
      `domain:${eventType}`,
      async (message) => {
        const event = JSON.parse(message)
        await handler(event)
      }
    )
  }
}
```

---

## สรุป

Enterprise Architecture ต้องการ:
1. **Clear separation of concerns** - Domain, Application, Infrastructure layers
2. **DDD** - ภาษาเดียวกันระหว่าง business และ tech
3. **CQRS** - แยก read/write สำหรับ performance
4. **Event Sourcing** - Audit trail ที่สมบูรณ์
5. **Dependency Injection** - Testable และ loosely coupled
6. **Event-driven** - Decoupled services สื่อสารผ่าน events
