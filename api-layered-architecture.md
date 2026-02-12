# API Layered Architecture: Route → Service → Repository

## Overview

Proper separation of concerns in API design ensures maintainability, testability, and scalability. The three-layer pattern is a proven approach for organizing backend code.

```
┌──────────────────────────────────────────────┐
│ Route Layer (Thin)                           │
│ - Input validation (schema)                  │
│ - Call service                               │
│ - Return response                            │
└──────────────────┬───────────────────────────┘
                   ↓
┌──────────────────────────────────────────────┐
│ Service Layer (Business Logic)               │
│ - Validation (business rules)                │
│ - Orchestration                              │
│ - Call repositories                          │
└──────────────────┬───────────────────────────┘
                   ↓
┌──────────────────────────────────────────────┐
│ Repository Layer (Data Access)               │
│ - Raw queries                                │
│ - CRUD operations                            │
│ - No business logic                          │
└──────────────────────────────────────────────┘
```

---

## Layer Responsibilities

### Route Layer (Controller)

**Responsibility**: HTTP interface only

✅ **Should**:
- Define request/response schemas
- Parse request data
- Call service functions
- Return HTTP responses
- Handle HTTP-specific concerns (headers, status codes)

❌ **Should NOT**:
- Contain business logic
- Make database queries directly
- Validate business rules
- Orchestrate multiple operations

**Example**:
```typescript
// ✅ Good: Thin route handler
router.post('/orders', async (req, res) => {
  const { userId, items } = req.body;

  // Call service (business logic happens there)
  const order = await orderService.createOrder({
    db: req.db,
    userId,
    items,
  });

  return res.status(201).json(order);
});

// ❌ Bad: Business logic in route
router.post('/orders', async (req, res) => {
  const { userId, items } = req.body;

  // Validation logic (should be in service!)
  if (items.length === 0) {
    throw new Error('Order must have items');
  }

  // Calculate total (should be in service!)
  const total = items.reduce((sum, item) => sum + item.price, 0);

  // Direct DB access (should be in repository!)
  const order = await db.insert('orders').values({ userId, total });

  return res.json(order);
});
```

---

### Service Layer (Business Logic)

**Responsibility**: Application business rules and orchestration

✅ **Should**:
- Validate business rules
- Orchestrate multiple repository calls
- Handle complex business logic
- Manage transactions
- Throw domain-specific errors

❌ **Should NOT**:
- Write raw SQL/ORM queries
- Know about HTTP details (status codes, headers)
- Access external APIs directly (use adapters)

**Example**:
```typescript
// ✅ Good: Service with business logic
export async function createOrder(params: {
  db: Database;
  userId: string;
  items: OrderItem[];
}): Promise<Order> {
  const { db, userId, items } = params;

  // Business validation
  if (items.length === 0) {
    throw new ValidationError('Order must contain at least one item');
  }

  // Business logic: Calculate total
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

  // Business rule: Check minimum order amount
  if (total < 10) {
    throw new BusinessRuleError('Minimum order is $10');
  }

  // Orchestrate repository calls
  const order = await orderRepository.create(db, { userId, total });
  await orderItemRepository.createBatch(db, items.map(item => ({
    orderId: order.id,
    ...item,
  })));

  return order;
}

// ❌ Bad: Service with raw queries
export async function createOrder(db: Database, userId: string) {
  // Raw query in service (should be in repository!)
  const result = await db.query(
    'INSERT INTO orders (user_id) VALUES ($1) RETURNING *',
    [userId]
  );
  return result.rows[0];
}
```

---

### Repository Layer (Data Access)

**Responsibility**: Database operations only

✅ **Should**:
- Perform CRUD operations
- Write efficient queries
- Return raw data
- Be entity-centric (one repository per entity)

❌ **Should NOT**:
- Contain business logic
- Validate business rules
- Orchestrate multiple entities
- Format data for API responses

**Example**:
```typescript
// ✅ Good: Repository with pure data access
export async function create(
  db: Database,
  order: CreateOrderInput
): Promise<Order> {
  return await db
    .insertInto('orders')
    .values({
      userId: order.userId,
      total: order.total,
      status: 'pending',
      createdAt: new Date(),
    })
    .returningAll()
    .executeTakeFirstOrThrow();
}

export async function findById(
  db: Database,
  orderId: string
): Promise<Order | undefined> {
  return await db
    .selectFrom('orders')
    .selectAll()
    .where('id', '=', orderId)
    .executeTakeFirst();
}

// ❌ Bad: Repository with business logic
export async function create(db: Database, order: CreateOrderInput) {
  // Business validation (should be in service!)
  if (order.total < 10) {
    throw new Error('Minimum order is $10');
  }

  // Business logic (should be in service!)
  const discount = order.total > 100 ? 0.1 : 0;
  const finalTotal = order.total * (1 - discount);

  return await db.insert('orders').values({ ...order, total: finalTotal });
}
```

---

## Dependency Rules

### Import Direction

```
Route → Service → Repository
  ↓       ↓         ↓
  ✅      ✅        ✅

Route ← Service ← Repository
  ❌      ❌        ❌
```

**Rules**:
- Routes can import services (not repositories)
- Services can import repositories (not routes)
- Repositories import nothing (just DB client)

**Example**:
```typescript
// ✅ routes/orders.ts
import * as orderService from '../services/order-service.js';
// NOT: import * as orderRepository from '../repositories/order-repository.js';

// ✅ services/order-service.ts
import * as orderRepository from '../repositories/order-repository.js';
import * as userRepository from '../repositories/user-repository.js';
// NOT: import { router } from '../routes/orders.js';

// ✅ repositories/order-repository.ts
import { db } from '../db.js';
// NO imports of services or routes
```

---

## Where to Put Logic

### Validation Logic

**Format Validation** → Route Layer (Schema)
```typescript
// routes/orders.ts
const CreateOrderSchema = {
  body: {
    type: 'object',
    properties: {
      userId: { type: 'string', format: 'uuid' }, // ✅ Format
      items: { type: 'array', minItems: 1 },      // ✅ Structure
    },
  },
};
```

**Business Validation** → Service Layer
```typescript
// services/order-service.ts
export async function createOrder(params) {
  // ✅ Business rule validation
  if (params.total < MINIMUM_ORDER_AMOUNT) {
    throw new BusinessRuleError('Order below minimum');
  }

  // ✅ Cross-entity validation
  const user = await userRepository.findById(params.userId);
  if (!user.isActive) {
    throw new BusinessRuleError('User account is inactive');
  }
}
```

---

### Calculation Logic

**Always** → Service Layer
```typescript
// ✅ services/order-service.ts
function calculateTotal(items: OrderItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

function applyDiscount(total: number, userTier: string): number {
  const discount = DISCOUNT_RATES[userTier] || 0;
  return total * (1 - discount);
}
```

---

### Database Queries

**Always** → Repository Layer
```typescript
// ✅ repositories/order-repository.ts
export async function findByUserId(
  db: Database,
  userId: string
): Promise<Order[]> {
  return await db
    .selectFrom('orders')
    .selectAll()
    .where('userId', '=', userId)
    .orderBy('createdAt', 'desc')
    .execute();
}
```

---

## Testing Strategy

### Route Layer Tests
Focus on HTTP interface:
```typescript
describe('POST /orders', () => {
  it('returns 201 with order', async () => {
    const response = await request(app)
      .post('/orders')
      .send({ userId: 'user-1', items: [...] });

    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty('id');
  });

  it('returns 400 for invalid input', async () => {
    const response = await request(app)
      .post('/orders')
      .send({ invalid: 'data' });

    expect(response.status).toBe(400);
  });
});
```

### Service Layer Tests
Focus on business logic:
```typescript
describe('createOrder', () => {
  it('rejects orders below minimum', async () => {
    await expect(
      orderService.createOrder({
        db: mockDb,
        userId: 'user-1',
        items: [{ price: 5, quantity: 1 }], // Total: $5 < $10
      })
    ).rejects.toThrow('Minimum order is $10');
  });

  it('applies correct discount for premium users', async () => {
    const order = await orderService.createOrder({
      db: mockDb,
      userId: 'premium-user',
      items: [{ price: 100, quantity: 1 }],
    });

    expect(order.total).toBe(90); // 10% discount
  });
});
```

### Repository Layer Tests
Focus on data access:
```typescript
describe('orderRepository.create', () => {
  it('inserts order into database', async () => {
    const order = await orderRepository.create(db, {
      userId: 'user-1',
      total: 100,
    });

    expect(order.id).toBeDefined();
    expect(order.userId).toBe('user-1');

    // Verify in DB
    const saved = await db
      .selectFrom('orders')
      .where('id', '=', order.id)
      .executeTakeFirst();
    expect(saved).toBeDefined();
  });
});
```

---

## Common Mistakes

### ❌ Mistake 1: Fat Controllers
```typescript
// BAD: All logic in route
router.post('/orders', async (req, res) => {
  const { userId, items } = req.body;

  // Validation
  if (!userId) throw new Error('User ID required');
  if (items.length === 0) throw new Error('No items');

  // Calculation
  const total = items.reduce((sum, item) => sum + item.price, 0);

  // Discount logic
  const user = await db.selectFrom('users').where('id', '=', userId).executeTakeFirst();
  const discount = user.tier === 'premium' ? 0.1 : 0;
  const finalTotal = total * (1 - discount);

  // DB operations
  const order = await db.insertInto('orders').values({ userId, total: finalTotal }).returning('*').executeTakeFirst();

  // More DB operations
  await Promise.all(
    items.map(item =>
      db.insertInto('order_items').values({ orderId: order.id, ...item }).execute()
    )
  );

  res.json(order);
});
```

**Fix**: Extract to service and repository layers

---

### ❌ Mistake 2: Anemic Services
```typescript
// BAD: Service just passes through to repository
export async function createOrder(db, data) {
  return await orderRepository.create(db, data); // No business logic!
}
```

**Fix**: Add business logic to service

---

### ❌ Mistake 3: Smart Repositories
```typescript
// BAD: Repository with business logic
export async function createOrder(db, order) {
  // Business logic in repository!
  if (order.items.length > 10) {
    order.discount = 0.05;
  }

  return await db.insert('orders').values(order);
}
```

**Fix**: Move business logic to service

---

## Benefits of Layered Architecture

1. **Testability**: Each layer can be tested independently
2. **Maintainability**: Clear where to add/modify code
3. **Reusability**: Services can be called from multiple routes
4. **Separation of Concerns**: Each layer has single responsibility
5. **Team Scaling**: Different teams can work on different layers
6. **Easy to Refactor**: Changes localized to specific layers

---

## Quick Reference

| Concern | Layer | Example |
|---------|-------|---------|
| HTTP request parsing | Route | `req.body.userId` |
| Schema validation | Route | TypeBox/Zod schemas |
| Business rules | Service | "Order minimum is $10" |
| Calculations | Service | `calculateTotal(items)` |
| Orchestration | Service | Call multiple repositories |
| Database queries | Repository | `db.selectFrom('orders')` |
| CRUD operations | Repository | `create()`, `findById()` |

---

## Checklist

- [ ] Routes are thin (< 10 lines per handler)
- [ ] No business logic in routes
- [ ] No DB queries in routes
- [ ] Services contain all business logic
- [ ] Services orchestrate repository calls
- [ ] Repositories only do data access
- [ ] No business logic in repositories
- [ ] Clear dependency direction (Route → Service → Repository)
- [ ] Each layer has appropriate tests
- [ ] Code is easy to locate ("where does X logic go?")

---

## Related Patterns

- **Clean Architecture**: Similar layering principle
- **Hexagonal Architecture**: Ports and adapters approach
- **Domain-Driven Design**: Rich domain models with services
- **CQRS**: Separate read/write operations

---

## Further Reading

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Layered Architecture Pattern](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html)
- [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html)
