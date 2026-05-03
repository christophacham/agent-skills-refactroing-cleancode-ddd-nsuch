# Clean Architecture + DDD — Deep Reference

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Dependency Rule](#the-dependency-rule)
3. [Domain Layer — DDD Patterns](#domain-layer--ddd-patterns)
   - [Value Objects](#value-objects)
   - [Entities](#entities)
   - [Aggregate Root](#aggregate-root)
   - [Domain Events](#domain-events)
   - [Business Invariants](#business-invariants)
4. [Application Layer](#application-layer)
   - [Port Interfaces](#port-interfaces)
   - [Use Cases](#use-cases)
   - [Error Mapping](#error-mapping)
5. [Infrastructure Layer](#infrastructure-layer)
   - [Repository Implementations](#repository-implementations)
   - [API Controllers](#api-controllers)
   - [Composition Root](#composition-root)
6. [Testing Strategy](#testing-strategy)
7. [Rust-Specific Considerations](#rust-specific-considerations)
8. [Comparison: DDD vs Clean Architecture](#comparison)
9. [Common Anti-Patterns](#common-anti-patterns)
10. [Complete File Map](#complete-file-map)

---

## Architecture Overview

Clean Architecture and Domain-Driven Design are complementary. Clean Architecture answers
**"where does code go?"** while DDD answers **"what do we call it and how do we model it?"**

The combination creates a system where:
- Business logic is **independent** of frameworks, databases, and UI
- The domain uses the **same language** as the business
- Changes to infrastructure **cannot** break business rules
- Business rules are **testable** without any infrastructure

### Layer Diagram

```
┌───────────────────────────────────────────────────────────┐
│  Infrastructure                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Interface Adapters (API Controllers)               │  │
│  │  ┌───────────────────────────────────────────────┐  │  │
│  │  │  Application (Use Cases)                      │  │  │
│  │  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  │  Domain                                 │  │  │
│  │  │  │  • Entities (Aggregate Roots)           │  │  │
│  │  │  │  • Value Objects                       │  │  │
│  │  │  │  • Domain Events                       │  │  │
│  │  │  └─────────────────────────────────────────┘  │  │
│  │  └───────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

### Dependency Flow

```
Infrastructure ——→ Application ——→ Domain
     ↓                 ↓               ↓
 implements         depends         depends on
 port traits     on port traits    NOTHING else
```

All arrows point **inward**. The domain layer imports nothing from application or infrastructure.

---

## The Dependency Rule

> *Source code dependencies can only point inwards. Nothing in an inner circle
> can know anything at all about something in an outer circle.*

This means:
- `domain/` has **zero** crate imports from `application/` or `infrastructure/`
- `application/` depends on `domain/` types but **never** imports from `infrastructure/`
- `application/` defines traits (`OrderRepository`) that `infrastructure/` implements
- `infrastructure/` depends on everything but nothing depends on it

This is the **Dependency Inversion Principle** in action: high-level policy does not depend
on low-level details. Both depend on abstractions (traits defined in the application layer).

---

## Domain Layer — DDD Patterns

### Value Objects

**Definition:** Immutable objects defined by their attributes, not identity. Two value objects
with the same attributes are equal. They have no lifecycle.

**When to use:** When you need to wrap a primitive type with validation, or represent a 
business concept that has no identity (Email, Money, Address, SKU).

**Example from the codebase:**

```rust
// src/domain/value_objects.rs

/// Email — validates at construction, normalizes to lowercase
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Email(String);

impl Email {
    pub fn new(email: impl Into<String>) -> Result<Self, EmailError> {
        let email = email.into();
        if email.is_empty() { return Err(EmailError::Empty); }
        if !email.contains('@') { return Err(EmailError::MissingAtSign); }
        let parts: Vec<&str> = email.split('@').collect();
        if parts.len() != 2 || parts[1].is_empty() {
            return Err(EmailError::MissingDomain);
        }
        Ok(Self(email.to_lowercase()))
    }
}

/// Money — prevents cross-currency operations
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Money {
    amount: Decimal,
    currency: Currency,
}

impl Money {
    pub fn add(&self, other: &Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch(self.currency, other.currency));
        }
        Ok(Money::new(self.amount + other.amount, self.currency))
    }
}

/// SKU — always uppercase
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Sku(String);

impl Sku {
    pub fn new(sku: impl Into<String>) -> Self {
        Self(sku.into().to_uppercase())
    }
}
```

**Key rules for Value Objects:**
1. Immutable — no setters, only constructors
2. Validate at construction — impossible to create invalid state
3. Operations return new instances — `Money.add()` returns `Result<Money, Error>`
4. Derive `PartialEq` — compare by attributes for free

### Entities

**Definition:** Objects defined by their **identity**. An entity's identity remains constant
even as its attributes change. Two entities with the same attributes are NOT necessarily equal.

**Example:**

```rust
// In src/domain/entities.rs

/// OrderLineItem is a child entity within the Order aggregate
#[derive(Debug, Clone)]
pub struct OrderLineItem {
    pub sku: Sku,                  // Value Object
    pub product_name: String,
    pub unit_price: Money,         // Value Object
    pub quantity: u32,
}

impl OrderLineItem {
    pub fn subtotal(&self) -> Money {
        self.unit_price.multiply(self.quantity)
    }
}
```

**Key rules for Entities:**
1. Always have a unique identifier (`id: Uuid`)
2. Identities can be compared, not just attributes
3. Mutable within the Aggregate Root's control
4. Entity equality = `id()` equality

### Aggregate Root

**Definition:** A cluster of domain objects treated as a single unit. External code
**only references the aggregate root**, never its children directly.

**The Order Aggregate:**

```rust
// src/domain/entities.rs

pub struct Order {
    id: Uuid,                              // Identity
    customer_email: Email,                  // Value Object (WHO)
    status: OrderStatus,                   // State machine (WHERE in lifecycle)
    items: Vec<OrderLineItem>,             // Child entities (WHAT)
    pending_events: Vec<DomainEvent>,       // Events to publish
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}
```

**State Machine — only valid transitions allowed:**

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderStatus {
    Draft,
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled,
}

impl OrderStatus {
    pub fn can_transition_to(&self, new_status: OrderStatus) -> bool {
        matches!(
            (self, new_status),
            (Draft, Pending) | (Draft, Cancelled) |
            (Pending, Confirmed) | (Pending, Cancelled) |
            (Confirmed, Shipped) |
            (Shipped, Delivered)
        )
    }
}
```

**All mutations go through aggregate methods that enforce invariants:**

```rust
impl Order {
    /// Add item — INVARIANT: only to Draft orders
    pub fn add_item(&mut self, item: OrderLineItem) -> Result<(), OrderError> {
        if self.status != OrderStatus::Draft {
            return Err(OrderError::InvalidStateTransition(..));
        }
        self.items.push(item);
        self.pending_events.push(DomainEvent::ItemAddedToOrder(ItemAddedToOrder { .. }));
        Ok(())
    }

    /// Submit — INVARIANTS: not empty, state is Draft
    pub fn submit(&mut self) -> Result<(), OrderError> {
        if self.items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }
        if !self.status.can_transition_to(OrderStatus::Pending) {
            return Err(OrderError::InvalidStateTransition(..));
        }
        self.status = OrderStatus::Pending;
        self.pending_events.push(DomainEvent::OrderPlaced(OrderPlaced { .. }));
        Ok(())
    }

    /// Cancel — INVARIANT: only Draft or Pending
    pub fn cancel(&mut self, reason: impl Into<String>) -> Result<(), OrderError> {
        if !matches!(self.status, Draft | Pending) {
            return Err(OrderError::InvalidStateTransition(..));
        }
        self.status = OrderStatus::Cancelled;
        self.pending_events.push(DomainEvent::OrderCancelled(OrderCancelled { .. }));
        Ok(())
    }
}
```

**Aggregate Root Rules:**
1. Only the root has global identity
2. External objects reference ONLY the root
3. The root enforces all invariants for the aggregate
4. Children are accessed through root methods
5. The root decides which events to publish

### Domain Events

**Definition:** Immutable records of something that **happened** in the domain.
They decouple side effects from business logic.

**Rust pattern — enum, not trait objects:**

```rust
// src/domain/events.rs

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DomainEvent {
    OrderPlaced(OrderPlaced),
    OrderCancelled(OrderCancelled),
    ItemAddedToOrder(ItemAddedToOrder),
}

impl DomainEvent {
    pub fn event_type(&self) -> &'static str {
        match self {
            DomainEvent::OrderPlaced(_) => "OrderPlaced",
            DomainEvent::OrderCancelled(_) => "OrderCancelled",
            DomainEvent::ItemAddedToOrder(_) => "ItemAddedToOrder",
        }
    }
}

#[derive(Debug, Clone)]
pub struct OrderPlaced {
    pub order_id: Uuid,
    pub customer_email: String,
    pub total_amount: Decimal,
    pub occurred_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct OrderCancelled {
    pub order_id: Uuid,
    pub reason: String,
    pub occurred_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct ItemAddedToOrder {
    pub order_id: Uuid,
    pub sku: String,
    pub quantity: u32,
    pub occurred_at: DateTime<Utc>,
}
```

**Why enum instead of `Box<dyn DomainEvent>`:**
1. **dyn safety** — Serialize makes traits not object-safe
2. **Send + Sync** — Enums satisfy these automatically; dyn traits may not
3. **Clone** — Enums derive Clone; dyn traits can't
4. **Exhaustiveness** — Match statements guarantee all cases covered
5. **No allocation** — No Box heap allocation per event

**Pattern for collecting events:**

```rust
impl Order {
    /// Collect events from pending_events, reset the buffer
    pub fn take_events(&mut self) -> Vec<DomainEvent> {
        std::mem::take(&mut self.pending_events)
    }
}
```

### Business Invariants

Invariants are business rules enforced **within the aggregate** that must always be true.
They live at the domain level, not in use cases.

**Our invariants:**
1. Cannot add items to non-Draft orders
2. Cannot submit empty orders
3. Cannot cancel confirmed/shipped orders
4. Order total always equals sum of line item subtotals
5. State transitions follow the defined state machine

---

## Application Layer

### Port Interfaces

Ports define the contracts between the application and external world. They are **traits**
defined in `application/`, implemented in `infrastructure/`.

**Output ports (driven adapters) — what the application needs:**

```rust
// src/application/ports.rs

pub trait OrderRepository: Send + Sync {
    fn save(&self, order: &Order) -> Result<(), RepositoryError>;
    fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, RepositoryError>;
    fn find_by_customer(&self, email: &Email) -> Result<Vec<Order>, RepositoryError>;
    fn find_by_status(&self, status: OrderStatus) -> Result<Vec<Order>, RepositoryError>;
}

pub trait EventBus: Send + Sync {
    fn publish(&self, events: Vec<DomainEvent>) -> Result<(), EventBusError>;
}

pub trait EmailService: Send + Sync {
    fn send_order_confirmation(&self, order: &Order) -> Result<(), EmailError>;
    fn send_order_cancellation(&self, order: &Order, reason: &str) -> Result<(), EmailError>;
}
```

**Input ports (driving adapters) — how the outside world calls us:**

```rust
pub trait SubmitOrder: Send + Sync {
    type Output;
    fn execute(&self, order_id: Uuid) -> Self::Output;
}
```

### Use Cases

Use cases are the **application business rules**. They orchestrate domain operations with
infrastructure calls, but don't contain business logic themselves.

**Pattern:** One struct per use case, generic over port traits.

```rust
// src/application/use_cases.rs

pub struct SubmitOrderUseCase<R: OrderRepository, E: EventBus, M: EmailService> {
    repository: Arc<R>,
    event_bus: Arc<E>,
    email_service: Arc<M>,
}

impl<R: OrderRepository, E: EventBus, M: EmailService> SubmitOrderUseCase<R, E, M> {
    pub fn execute(&self, order_id: Uuid) -> Result<SubmitOrderResult, SubmitOrderError> {
        // 1. Fetch
        let mut order = self.repository.find_by_id(order_id)?
            .ok_or(SubmitOrderError::OrderNotFound)?;
        
        // 2. Operate (domain enforces invariants)
        order.submit()
            .map_err(SubmitOrderError::Domain)?;
        
        // 3. Collect events
        let events = order.take_events();
        
        // 4. Side effects (after successful operation)
        self.event_bus.publish(events.clone())?;
        
        // 5. Optional: email
        let _ = self.email_service.send_order_confirmation(&order);
        
        // 6. Persist
        self.repository.save(&order)?;
        
        Ok(SubmitOrderResult { order, events })
    }
}
```

**Use case execution order matters:**
1. Fetch aggregate from repository
2. Call aggregate methods (domain validates)
3. Take domain events
4. Publish events (if operation succeeded)
5. Trigger notifications (emails, webhooks)
6. Persist final state

### Error Mapping

Each layer has its own error types. Use cases map domain errors to application errors:

```rust
#[derive(Debug, thiserror::Error)]
pub enum SubmitOrderError {
    #[error("Order not found")]
    OrderNotFound,
    
    #[error("Invalid domain operation: {0}")]
    Domain(#[from] DomainOrderError),  // ← maps from domain
    
    #[error("Failed to publish events: {0}")]
    EventBus(#[from] EventBusError),
    
    #[error("Repository error: {0}")]
    Repository(#[from] RepositoryError),
}
```

---

## Infrastructure Layer

### Repository Implementations

Implement the port traits defined in the application layer:

```rust
// src/infrastructure/persistence.rs

pub struct InMemoryOrderRepository {
    orders: RwLock<HashMap<Uuid, Order>>,
}

impl OrderRepository for InMemoryOrderRepository {
    fn save(&self, order: &Order) -> Result<(), RepositoryError> {
        let mut orders = self.orders.write()
            .map_err(|_| RepositoryError::DatabaseError("Lock poisoned".into()))?;
        orders.insert(order.id(), order.clone());
        Ok(())
    }
    
    fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, RepositoryError> {
        let orders = self.orders.read()
            .map_err(|_| RepositoryError::DatabaseError("Lock poisoned".into()))?;
        Ok(orders.get(&id).cloned())
    }
}
```

**For real infrastructure:** Implement the same trait with Postgres, SQLite, DynamoDB, etc.
The application layer never changes.

### API Controllers

Translate HTTP requests → use case inputs, use case outputs → HTTP responses:

```rust
// src/infrastructure/api.rs

pub struct OrderController {
    service: Arc<OrderService>,
}

impl OrderController {
    pub fn submit_order(&self, req: SubmitOrderRequest) -> ApiResponse<SubmitOrderResponse> {
        match self.service.submit_order.execute(req.order_id) {
            Ok(result) => ApiResponse::success(SubmitOrderResponse {
                order_id: result.order.id().to_string(),
                status: format!("{:?}", result.order.status()),
                total: format!("{}", result.order.total().amount()),
                events_published: result.events.len(),
            }),
            Err(e) => ApiResponse::error(e.to_string()),
        }
    }
}
```

### Composition Root

The single place where all dependencies are wired together:

```rust
// Typically in main.rs or a dedicated wiring module

let repository = Arc::new(InMemoryOrderRepository::new());
let event_bus = Arc::new(LoggingEventBus::new());
let email_service = Arc::new(LoggingEmailService::new());

let submit_order = SubmitOrderUseCase::new(
    Arc::clone(&repository),
    event_bus,
    email_service,
);

let controller = OrderController { /* inject use cases */ };
```

---

## Testing Strategy

### Domain Tests — Pure, No Infrastructure

Domain tests don't need mocks, databases, or any infrastructure:

```rust
// tests/domain_tests.rs

#[test]
fn test_cannot_submit_empty_order() {
    let email = Email::new("test@example.com").unwrap();
    let mut order = Order::create(email);
    assert!(order.submit().is_err());  // EmptyOrder
}

#[test]
fn test_cannot_add_item_to_submitted_order() {
    let mut order = Order::create(Email::new("t@t.com").unwrap());
    order.add_item(OrderLineItem::new(Sku::new("X"), "P".into(), Money::usd(Decimal::from(10)), 1)).unwrap();
    order.submit().unwrap();
    
    // Can't add items after submit
    assert!(order.add_item(OrderLineItem::new(Sku::new("Y"), "Q".into(), Money::usd(Decimal::from(10)), 1)).is_err());
}

#[test]
fn test_order_total() {
    let mut order = Order::create(Email::new("t@t.com").unwrap());
    order.add_item(OrderLineItem::new(Sku::new("A"), "A".into(), Money::usd(Decimal::from(10)), 2)).unwrap(); // $20
    order.add_item(OrderLineItem::new(Sku::new("B"), "B".into(), Money::usd(Decimal::from(5)), 3)).unwrap();  // $15
    assert_eq!(order.total().amount(), Decimal::from(35));  // $35
}
```

### Application Tests — Mocked Ports

Test use cases by creating mock implementations of the port traits:

```rust
// tests/application_tests.rs

struct MockOrderRepository {
    orders: Mutex<Vec<Order>>,
}

impl OrderRepository for MockOrderRepository {
    fn save(&self, order: &Order) -> Result<(), RepositoryError> {
        self.orders.lock().unwrap().push(order.clone());
        Ok(())
    }
    fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, RepositoryError> {
        Ok(self.orders.lock().unwrap().iter().find(|o| o.id() == id).cloned())
    }
    // ...
}

struct MockEventBus {
    published: Mutex<Vec<DomainEvent>>,
}

impl EventBus for MockEventBus {
    fn publish(&self, events: Vec<DomainEvent>) -> Result<(), EventBusError> {
        self.published.lock().unwrap().extend(events);
        Ok(())
    }
}

#[test]
fn test_submit_order_publishes_events() {
    let repo = Arc::new(MockOrderRepository::new());
    let event_bus = Arc::new(MockEventBus::new());
    let email_svc = Arc::new(MockEmailService::new());
    
    // Setup: create order with item
    let mut order = Order::create(Email::new("t@t.com").unwrap());
    order.add_item(OrderLineItem::new(..)).unwrap();
    let order_id = order.id();
    repo.save(&order).unwrap();
    
    // Execute
    let uc = SubmitOrderUseCase::new(repo, event_bus.clone(), email_svc);
    let result = uc.execute(order_id).unwrap();
    
    // Assert
    assert!(result.events.iter().any(|e| e.event_type() == "OrderPlaced"));
    assert_eq!(event_bus.published_count(), 2);  // ItemAdded + OrderPlaced
}
```

### Test Result

```
running 21 tests
test test_cannot_submit_empty_order ... ok
test test_order_lifecycle ... ok
test test_submit_order_publishes_events ... ok
...all 21 passed...
```

---

## Rust-Specific Considerations

| Concern | Pattern | Rationale |
|---------|---------|-----------|
| **Trait objects vs enums** | Enum for domain events | dyn trait + Serialize = not object-safe; enums are Clone + Send + Sync |
| **Generics over port traits** | `struct UseCase<R: OrderRepository>` | Zero-cost abstraction; no dyn dispatch at call sites |
| **Thread safety** | `Arc<R>` for shared dependencies | Use cases are shared across requests/threads |
| **Error handling** | Layer-specific error types with `#[from]` | Domain errors propagate up via trait impls |
| **Serde** | `#[serde(skip)]` for event buffers | Events aren't persisted with the aggregate |
| **Interior mutability** | `RwLock<HashMap<..>>` in repos | Multi-reader, single-writer for in-mem stores |
| **Private fields** | `id: Uuid` (private), `pub fn id()` (getter) | Aggregate encapsulation — can't bypass method |

---

## Comparison: DDD vs Clean Architecture

| Aspect | Clean Architecture | DDD |
|--------|-------------------|-----|
| **Origin** | Robert C. Martin (2012) | Eric Evans (2003) |
| **Core concern** | Dependency direction | Domain language & boundaries |
| **Key concept** | Layers + Dependency Inversion | Ubiquitous Language + Bounded Contexts |
| **What it answers** | "Where does this code go?" | "What do we call this?" |
| **Primary artifact** | Use Cases, Ports/Adapters | Entities, Aggregates, Value Objects |
| **Structuring principle** | By dependency level | By bounded context |
| **Can be used alone?** | Yes (with anemic models) | Yes (without clean layering) |
| **Works best with** | Rich domain models | Explicit layering rules |

---

## Common Anti-Patterns

### 1. Anemic Domain Model
```rust
// ❌ WRONG: Entity is just a data bag, logic lives in "services"
pub struct Order {
    pub id: Uuid,
    pub status: OrderStatus,
    pub items: Vec<OrderLineItem>,
}
// "OrderService" elsewhere contains SubmitOrder logic

// ✅ RIGHT: Logic lives on the entity
impl Order {
    pub fn submit(&mut self) -> Result<(), OrderError> { .. }
}
```

### 2. Infrastructure Leaking into Domain
```rust
// ❌ WRONG: Domain depends on infrastructure
// src/domain/entities.rs
use sqlx::FromRow;  // NO! Database concern in domain!

// ✅ RIGHT: Domain is pure; infrastructure handles persistence
```

### 3. Use Case Doing Domain Logic
```rust
// ❌ WRONG: Use case checking business rules
impl SubmitOrderUseCase {
    fn execute(&self, order_id: Uuid) -> Result<..> {
        let order = self.repo.find_by_id(order_id)?;
        if order.items().is_empty() {  // This is domain logic!
            return Err(..);
        }
        order.status = OrderStatus::Pending;  // Direct mutation bypasses invariants!
    }
}

// ✅ RIGHT: Domain enforces invariants
impl Order {
    pub fn submit(&mut self) -> Result<(), OrderError> {
        if self.items.is_empty() { return Err(EmptyOrder); }
        // ... state machine check
    }
}
```

### 4. Bypassing Aggregates
```rust
// ❌ WRONG: Directly modifying child entities
repo.save_order_item(order_id, item);  // Bypasses Order aggregate!

// ✅ RIGHT: Go through aggregate root
order.add_item(item)?;
repo.save(&order)?;
```

### 5. Event Bus in Critical Path
```rust
// ❌ WRONG: Blocking on event publication
order.submit()?;
event_bus.publish(events)?;  // What if event bus is down?
repo.save(&order)?;          // Order is lost!

// ✅ RIGHT: Persist first, publish events after
order.submit()?;
let events = order.take_events();
repo.save(&order)?;          // Persist first
event_bus.publish(events)?;  // If this fails, event can be retried
```

---

## Complete File Map

```
src/
├── lib.rs                            # Module declarations
├── main.rs                           # Composition root + demo runner
│
├── domain/
│   ├── mod.rs                        # re-exports
│   ├── entities.rs                   # Order (aggregate root), OrderLineItem, OrderStatus
│   ├── value_objects.rs              # Email, Money, Currency, Sku
│   └── events.rs                     # DomainEvent enum + event structs
│
├── application/
│   ├── mod.rs                        # re-exports
│   ├── ports.rs                      # Traits: OrderRepository, EventBus, EmailService
│   └── use_cases.rs                  # CreateOrder, AddItem, SubmitOrder, CancelOrder use cases
│
└── infrastructure/
    ├── mod.rs                        # module declarations
    ├── persistence.rs                # InMemoryOrderRepository, LoggingEventBus, LoggingEmailService
    └── api.rs                        # OrderController, request/response types, OrderService composer

tests/
├── domain_tests.rs                   # 16 tests, zero infrastructure
└── application_tests.rs              # 5 tests with mocked ports
```

---

## Project Status

- ✅ 21 passing tests
- ✅ Running demo: `cargo run`
- ✅ Full Clean Architecture layering
- ✅ DDD patterns: Aggregate Root, Value Objects, Domain Events
- ✅ Dependency inversion: domain traits → infrastructure impls
- ✅ State machine with valid transitions only
- ✅ Business invariants enforced at domain level
