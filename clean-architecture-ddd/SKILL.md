---
name: clean-architecture-ddd
description: Clean Architecture + Domain-Driven Design patterns in Rust. Use when structuring new projects, separating concerns, designing domain models, or when the user asks about layered architecture, dependency inversion, aggregates, value objects, or domain events.
---

# Clean Architecture + DDD — The Friendly Guide

You're about to write (or refactor) code that needs to stay testable, grow without pain, and actually make sense to the business. Here's the mindset.

## When to Reach for This Skill

- "Where should this code live?" → **Clean Architecture** answers that
- "What should this thing be called?" → **DDD** answers that
- "How do I test business rules without a database?" → **Both** working together
- "How do I swap infrastructure without touching business logic?" → **Clean Architecture**

## The 30-Second Mental Model

```
┌────────────────────────────────────────────────────────┐
│  Infrastructure       Frameworks, DB, HTTP             │ ← outer
├────────────────────────────────────────────────────────┤
│  Application          Use Cases, Port interfaces       │
├────────────────────────────────────────────────────────┤
│  Domain               Entities, Value Objects, Events  │ ← inner
└────────────────────────────────────────────────────────┘

  Dependencies ALWAYS point inward →
  Inner layers NEVER know about outer layers
```

## The Recipe

Every feature follows this pattern. Start from the **domain** (inner), work outward.

### Step 1: Domain — The Core

These live in `domain/` and import NOTHING from your application or infrastructure.

**Value Objects** — Small, immutable, self-validating:
```rust
// domain/value_objects.rs
pub struct Email(String);

impl Email {
    pub fn new(email: impl Into<String>) -> Result<Self, EmailError> {
        // Validation here — fail early
    }
}

pub struct Money {
    amount: Decimal,
    currency: Currency,
}

impl Money {
    // Operations that prevent cross-currency accidents
    pub fn add(&self, other: &Money) -> Result<Money, MoneyError> { .. }
}
```

**Entities + Aggregate Root** — Objects with identity, controlling state changes:
```rust
// domain/entities.rs — Order is the Aggregate Root
pub struct Order {
    id: Uuid,                    // identity
    status: OrderStatus,         // state machine
    items: Vec<OrderLineItem>,   // child entities
    pending_events: Vec<DomainEvent>,  // events to publish
}

impl Order {
    // Factory: creates draft
    pub fn create(customer_email: Email) -> Self { .. }

    // Commands: state transitions with INVARIANTS
    pub fn add_item(&mut self, item: OrderLineItem) -> Result<(), OrderError> {
        if self.status != OrderStatus::Draft {
            return Err(OrderError::InvalidStateTransition(..));
        }
        // mutate & record event
    }

    pub fn submit(&mut self) -> Result<(), OrderError> {
        if self.items.is_empty() {
            return Err(OrderError::EmptyOrder);  // INVARIANT
        }
        // transition & record OrderPlaced event
    }
}
```

**Rust note:** Use an `enum` for Domain Events, not dyn trait objects. It's more ergonomic:
```rust
// domain/events.rs
#[derive(Debug, Clone)]
pub enum DomainEvent {
    OrderPlaced(OrderPlaced),
    OrderCancelled(OrderCancelled),
    ItemAddedToOrder(ItemAddedToOrder),
}
```

### Step 2: Application — Use Cases

These live in `application/`. They **orchestrate** but don't implement business logic (that's the domain's job).

**Ports first** — Define what you need from infrastructure as **traits**:
```rust
// application/ports.rs
pub trait OrderRepository: Send + Sync {
    fn save(&self, order: &Order) -> Result<(), RepositoryError>;
    fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, RepositoryError>;
}

pub trait EventBus: Send + Sync {
    fn publish(&self, events: Vec<DomainEvent>) -> Result<(), EventBusError>;
}
```

**Use cases** — One per business action, thin:
```rust
// application/use_cases.rs
pub struct SubmitOrderUseCase<R: OrderRepository, E: EventBus> {
    repository: Arc<R>,
    event_bus: Arc<E>,
}

impl<R: OrderRepository, E: EventBus> SubmitOrderUseCase<R, E> {
    pub fn execute(&self, order_id: Uuid) -> Result<SubmitOrderResult, SubmitOrderError> {
        let mut order = self.repository.find_by_id(order_id)?..ok_or(NotFound)?;
        
        order.submit()?;                              // domain validates
        let events = order.take_events();             // collect events
        self.event_bus.publish(events.clone())?;       // side effects
        self.repository.save(&order)?;                // persist
        
        Ok(SubmitOrderResult { order, events })
    }
}
```

### Step 3: Infrastructure — Implementation

These live in `infrastructure/` and **implement** the port traits:

```rust
// infrastructure/persistence.rs
pub struct InMemoryOrderRepository { .. }

impl OrderRepository for InMemoryOrderRepository {
    fn save(&self, order: &Order) -> Result<(), RepositoryError> { .. }
    fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, RepositoryError> { .. }
}
```

### Step 4: Wiring — Composition Root

In `main.rs` or your DI setup:
```rust
let repo = Arc::new(InMemoryOrderRepository::new());
let event_bus = Arc::new(KafkaEventBus::new());
let use_case = SubmitOrderUseCase::new(Arc::clone(&repo), event_bus);
let controller = OrderController::new(use_case);
```

### Step 5: UI Layer — GUIs Don't Change Anything

Here's the key insight: **a GUI is just another infrastructure adapter.**
The same use case that serves an HTTP controller serves a button click.

```
HTTP Controller ──┐
CLI Command     ──┼──→ SubmitOrderUseCase ──→ Domain (unchanged!)
GUI Button      ──┤
REST API        ──┘
```

All four call the **same** `use_case.execute(order_id)`. The domain **never**
knows about windows, buttons, or event loops.

The UI adapter pattern — identical structure to the HTTP controller:

```rust
// infrastructure/ui.rs

// UI state is SEPARATE from domain state — your domain Order doesn't know
// about error banners or loading spinners.
#[derive(Debug, Default)]
pub struct OrderFormState {
    pub error_message: Option<String>,
    pub success_message: Option<String>,
    pub pending_orders: Vec<UiOrderSummary>,
    pub is_loading: bool,
}

pub struct OrderView {
    submit_order: Arc<SubmitOrderUseCase<..>>,
    cancel_order: Arc<CancelOrderUseCase<..>>,
}

impl OrderView {
    // Called when user clicks "Submit Order" — same use case as HTTP!
    pub fn on_submit_clicked(&self, state: &mut OrderFormState, order_id: Uuid) {
        state.is_loading = true;
        match self.submit_order.execute(order_id) {
            Ok(result) => {
                state.success_message = Some(format!("Order placed! Total: ${}",
                    result.order.total().amount()));
            }
            Err(e) => {
                state.error_message = Some(e.to_string());
            }
        }
        state.is_loading = false;
    }
}
```

**Important distinction** — UI has its own state (`OrderFormState`) that tracks
display concerns (loading spinners, error banners). This is **not** the domain
state — the domain `Order` has no concept of "loading" or "error message."

### Wiring with a UI

Same composition root, just inject into UI instead of HTTP:

```rust
let repo = Arc::new(InMemoryOrderRepository::new());
let submit_use_case = Arc::new(SubmitOrderUseCase::new(
    Arc::clone(&repo), event_bus, email_service,
));

// Same use case, different adapter:
let api_controller = OrderController::new(submit_use_case.clone());  // HTTP
let order_view = OrderView::new(submit_use_case);                    // GUI
```

## Testing — The Superpower

Domain tests are **pure** — no mocks, no infrastructure:
```rust
#[test]
fn test_cannot_submit_empty_order() {
    let mut order = Order::create(Email::new("a@b.com").unwrap());
    assert!(order.submit().is_err());  // EmptyOrder error
}
```

Application tests use **mocked** traits:
```rust
#[test]
fn test_submit_publishes_events() {
    let repo = Arc::new(MockOrderRepository::new());
    let event_bus = Arc::new(MockEventBus::new());
    let uc = SubmitOrderUseCase::new(repo, event_bus.clone(), email_svc);
    
    let result = uc.execute(order_id).unwrap();
    assert_eq!(event_bus.published_count(), 2);
}
```

## Quick Decision Table

| Problem | Solution |
|---------|----------|
| "Where to put validation?" | Domain Value Objects + Entity methods |
| "How to handle cross-cutting concerns?" | Use cases orchestrate, ports define contracts |
| "How to swap Postgres for SQLite?" | Implement `OrderRepository` trait twice |
| "How to enforce business rules?" | Entity methods with `Result<(), Error>` |
| "Should this be an Entity or Value Object?" | Has identity? → Entity. Compare by attributes? → Value Object |
| "Trait object (dyn) or enum for events?" | **Enum** in Rust — it's Send+Sync+Clone friendly |
| "How do I add a GUI?" | Add `ui.rs` adapter — domain + use cases unchanged |

## Files You'll Actually Create

```
src/
├── lib.rs                    # Re-exports all modules
├── main.rs                   # Composition root, runs demo
├── domain/
│   ├── mod.rs
│   ├── entities.rs           # Aggregate roots, state machines
│   ├── value_objects.rs      # Email, Money, Sku, etc.
│   └── events.rs             # DomainEvent enum
├── application/
│   ├── mod.rs
│   ├── ports.rs              # Trait interfaces (OrderRepository, EventBus)
│   └── use_cases.rs          # One struct per business action
└── infrastructure/
    ├── mod.rs
    ├── persistence.rs        # Repo implementations
    ├── api.rs                # HTTP controllers
    └── ui.rs                 # GUI adapter (same use cases, different caller)
tests/
├── domain_tests.rs           # Zero infrastructure needed
├── application_tests.rs      # Mocked traits
└── (ui tests live in ui.rs   # Testable without rendering a window)
```

## References

For the complete reference with all patterns, rationale, and detailed Rust examples, see [reference.md](references/reference.md).

The full runnable example is in the project at `src/` — run it with `cargo run`.
