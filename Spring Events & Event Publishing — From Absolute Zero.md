# 🎯 Spring Events & Event Publishing — From Absolute Zero

---

## 🧩 1. Requirement Understanding — Feel the Pain First

You just joined a team. Your ticket:

```
TICKET-204: After a customer places an order, we need to:
  1. Deduct inventory
  2. Send a confirmation email
  3. Log it for audit
  4. Notify the warehouse
```

You're a junior dev. You look at `OrderService` and do the natural thing:

```java
@Service
public class OrderService {

    // You need all of these to do your job
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;   // ← injected
    private final EmailService emailService;           // ← injected
    private final AuditService auditService;           // ← injected
    private final WarehouseService warehouseService;   // ← injected

    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);

        inventoryService.deduct(order);                // ← you call this
        emailService.sendConfirmation(order);          // ← you call this
        auditService.log(order);                       // ← you call this
        warehouseService.notify(order);                // ← you call this

        return saved;
    }
}
```

This works. Ship it. ✅

---

**Six weeks later.** New ticket:

```
TICKET-389: When an order is placed, also:
  - Notify the fraud detection system
  - Push to analytics pipeline
  - Update loyalty points

Also, PM wants email sent ONLY if payment actually succeeds.
Also, warehouse notification must be async — it's slow.
Also, audit logging broke prod last week — it must never block order saving.
```

You open `OrderService`. It now looks like:

```java
public Order placeOrder(Order order) {
    Order saved = orderRepository.save(order);

    inventoryService.deduct(order);
    emailService.sendConfirmation(order);       // ← but what if payment failed?
    auditService.log(order);                    // ← broke prod last week
    warehouseService.notify(order);             // ← blocks for 3 seconds
    fraudService.check(order);                  // ← new
    analyticsService.push(order);               // ← new
    loyaltyService.addPoints(order);            // ← new

    return saved;
}
```

**This is the pain.** `OrderService` has become a god class. It knows about 7 other services. To add one new "side effect" of placing an order, you have to:
- open `OrderService`
- import a new service
- inject it
- call it
- pray nothing breaks

`OrderService` is supposed to know one thing: **how to place an order.** It shouldn't know who cares about that fact.

---

## 🧠 2. Design Thinking — The Naive Approaches

### Attempt 1: Extract it into a helper class

```java
@Service
public class OrderSideEffectsService {
    // Move all the calls here
    public void runAfterOrder(Order order) {
        inventoryService.deduct(order);
        emailService.sendConfirmation(order);
        // ... etc
    }
}
```

❌ **Rejected.** You just moved the problem. `OrderSideEffectsService` is still the one that knows about every downstream consumer. Adding a new side effect still means editing this class. The coupling still exists — it just lives one level away now.

---

### Attempt 2: Callbacks / Listeners list

```java
@Service
public class OrderService {
    private List<Consumer<Order>> postOrderCallbacks = new ArrayList<>();

    public void registerCallback(Consumer<Order> callback) {
        postOrderCallbacks.add(callback);
    }

    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);
        postOrderCallbacks.forEach(cb -> cb.accept(saved));
        return saved;
    }
}
```

Better! But now you've re-invented something manually. Who manages the list? What about ordering? Async? Errors? Transaction awareness? You'd spend weeks building what Spring already built.

---

### Attempt 3: Direct interface with `@Autowired List<OrderObserver>`

```java
public interface OrderObserver {
    void onOrderPlaced(Order order);
}

@Service
public class OrderService {
    @Autowired
    private List<OrderObserver> observers;   // Spring injects all implementations

    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);
        observers.forEach(o -> o.onOrderPlaced(saved));
        return saved;
    }
}
```

This is actually quite close! But there's still a structural problem: `OrderService` still knows the concept of `OrderObserver`. It still imports and holds that type. And this doesn't handle transactions ("only notify warehouse AFTER the DB commit succeeds"), async, ordering, or event hierarchies.

**The right answer is:** let Spring manage an event bus. `OrderService` shouts a fact into the void. Whoever cares, cares. `OrderService` doesn't know — or care — who's listening.

---

## 🧠 3. Concept Introduction — How It Works (No Code Yet)

Think of a **radio broadcast tower**.

```
Old way (tight coupling):
    OrderService has a telephone list
    It calls EmailService directly
    It calls InventoryService directly
    It must know EVERYONE's phone number

New way (event system):
    OrderService is a radio tower
    It broadcasts: "ORDER PLACED — Order #123, customer Alice"
    
    EmailService     has a radio tuned to "order placed" → reacts
    InventoryService has a radio tuned to "order placed" → reacts
    AuditService     has a radio tuned to "order placed" → reacts
    
    OrderService doesn't know who owns a radio.
    New listener? Just buy a radio. Nobody touches the tower.
```

**Three pieces:**

```
┌─────────────┐    ┌──────────────────────────┐    ┌──────────────────┐
│  Publisher  │───▶│  Event Bus               │───▶│  Listener(s)     │
│             │    │  (ApplicationContext)     │    │                  │
│ OrderService│    │  Holds: who listens to   │    │ EmailHandler     │
│             │    │  what. Routes events to  │    │ InventoryHandler │
│ "I placed   │    │  the right listeners.    │    │ AuditHandler     │
│  an order"  │    │                          │    │                  │
└─────────────┘    └──────────────────────────┘    └──────────────────┘
       │                      │                            ▲
       │    publishEvent()    │     invokeListener()       │
       └──────────────────────┴────────────────────────────┘
```

**The key insight about timing (this trips everyone up):**

By default, when `OrderService` calls `publishEvent()`, Spring **immediately** runs every listener, one by one, on the SAME thread, before returning. It's like the radio tower calling people sequentially on a loudspeaker. Async is opt-in, not the default.

---

## 🏗 4. Task Breakdown

```
TODO 1: Create the event — a plain Java object carrying order data
TODO 2: Make OrderService publish it (instead of calling services directly)
TODO 3: Create a basic listener that reacts to it
TODO 4: Add a transactional listener (email only after DB commit succeeds)
TODO 5: Add async handling for the slow warehouse notification
TODO 6: Chain events (payment triggers shipment)
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: Create the Event

```java
// An "event" is just a plain Java object that carries data about
// something that happened. No magic required.

public record OrderPlacedEvent(
    Order order,
    String customerId,
    LocalDateTime placedAt
) {}
// That's it. No special superclass needed (Spring 4.2+).
// It's just a data bag. A fact. "This thing happened, here's the data."
```

**What's a `record`?** Just a compact Java class where all fields are final and Spring doesn't care — it can publish any object as an event.

**Common mistake — defining it like this:**

```java
// ❌ Silently fails to share useful data
public class OrderPlacedEvent extends ApplicationEvent {
    public OrderPlacedEvent(Object source) {
        super(source);   // ← source is just "who published it"
    }
    // Forgot to add the actual order data!
    // Listeners get an event with no useful payload
}
```

The listener gets called but has nothing to work with.

---

### TODO 2: Publisher — OrderService Publishes the Event

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    // ApplicationEventPublisher is a Spring interface with one job:
    // "publish this event to whoever is listening"
    // Spring automatically wires in the ApplicationContext here,
    // because ApplicationContext implements this interface.
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);

        // This is the entire outbound API of OrderService.
        // It doesn't call EmailService. It doesn't call InventoryService.
        // It just announces: "an order was placed, here's the data."
        eventPublisher.publishEvent(
            new OrderPlacedEvent(saved, saved.getCustomerId(), LocalDateTime.now())
        );

        return saved;
        // OrderService is done. It has no idea what happens next.
    }
}
```

**What Spring does internally when `publishEvent()` is called:**

```
publishEvent(OrderPlacedEvent)
    │
    ▼
ApplicationContext wraps it (if it's not already an ApplicationEvent)
    │
    ▼
SimpleApplicationEventMulticaster.multicastEvent()
    │
    ├─ Is a TaskExecutor configured?
    │       NO  → call each listener ON THIS THREAD, one by one (default)
    │       YES → submit each listener to the thread pool
    │
    ▼
For each matching listener:
    invokeListener(listener, event)
```

**Common mistake:**

```java
// ❌ Injecting ApplicationContext directly — works but is too broad
@Autowired
private ApplicationContext context;

context.publishEvent(event);  // works, but now you depend on the entire context
// Always prefer the narrow interface: ApplicationEventPublisher
```

---

### TODO 3: Basic Listener — Inventory Deduction

```java
// @EventListener is the annotation that tells Spring:
// "when an event of this type is published, call this method"
// Spring figures out the event type from the method parameter.

@Component                           // must be a Spring-managed bean
@RequiredArgsConstructor
public class InventoryEventHandler {

    private final InventoryService inventoryService;

    @EventListener                   // ← this is the magic word
    public void onOrderPlaced(OrderPlacedEvent event) {
        // "event" is the exact object that OrderService published
        inventoryService.deduct(event.order());
    }
}
```

**What Spring does internally to make `@EventListener` work:**

```
During application startup, after ALL beans are created:

EventListenerMethodProcessor (a special Spring processor) scans
every bean for methods annotated with @EventListener.

For each @EventListener method it finds:
    → Creates an ApplicationListenerMethodAdapter
    → Registers it with the SimpleApplicationEventMulticaster

The multicaster builds a map:
    OrderPlacedEvent.class → [InventoryEventHandler.onOrderPlaced]
    UserCreatedEvent.class → [EmailHandler.onUserCreated, ...]
    ...

When publishEvent(OrderPlacedEvent) is called:
    Multicaster looks up: who listens to OrderPlacedEvent?
    → Calls each registered listener
```

**Common silent failure — wrong parameter type:**

```java
// ❌ This method NEVER gets called — wrong event type in parameter
@EventListener
public void onOrderPlaced(String event) {   // ← listens for String events
    inventoryService.deduct(...);            // never runs for OrderPlacedEvent
}
// No error. No warning. Just silence.
```

---

### TODO 4: Transactional Listener — Email Only After DB Commit

This is the most important concept in this topic. Here's the problem it solves:

```java
// Without @TransactionalEventListener:

@Transactional
public Order placeOrder(Order order) {
    Order saved = orderRepository.save(order);      // ① DB write (not committed yet)
    eventPublisher.publishEvent(new OrderPlacedEvent(saved)); // ② publish
        // → EmailHandler.onOrderPlaced() runs HERE, RIGHT NOW
        //   Email is sent
        //   But then...
    // ③ something fails — exception thrown — transaction ROLLS BACK
    // DB has NO order record. But email was already sent. ❌
    throw new RuntimeException("payment failed");
}
```

The email went out for an order that never existed. This is the problem `@TransactionalEventListener` solves.

```java
@Component
@RequiredArgsConstructor
public class NotificationEventHandler {

    private final EmailService emailService;

    // @TransactionalEventListener is @EventListener with transaction awareness.
    // Instead of running immediately when the event is published,
    // it registers itself with the transaction and asks:
    // "call me AFTER the transaction commits — not before."
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // This code only runs if the DB transaction committed successfully.
        // If the transaction rolled back, this is never called.
        emailService.sendOrderConfirmation(event.order(), event.customerId());
    }
}
```

**What happens under the hood:**

```
Transaction STARTS
    │
    ├─ orderRepository.save(order)    ← DB write
    │
    ├─ publishEvent(OrderPlacedEvent)
    │       │
    │       ▼
    │   @TransactionalEventListener sees: "is there an active transaction?"
    │   YES → don't run the listener now.
    │         Instead, register it with the transaction system:
    │         "hey, when this transaction finishes, call me"
    │         → Stored in TransactionSynchronizationManager (thread-local)
    │
    ├─ ... rest of method ...
    │
Transaction COMMITS ✅
    │
    ▼
Spring runs the registered callbacks:
    AFTER_COMMIT → EmailHandler.onOrderPlaced() runs NOW ← email sent safely

--- OR ---

Transaction ROLLS BACK ❌
    │
    ▼
AFTER_COMMIT listener → NOT called (order never existed, no email)
AFTER_ROLLBACK listener → called if you registered one
```

**The four phases:**

```
┌─────────────────┬─────────────────────────────────────┬──────────────────────┐
│ Phase           │ When                                │ Transaction state    │
├─────────────────┼─────────────────────────────────────┼──────────────────────┤
│ BEFORE_COMMIT   │ Just before commit decision         │ INSIDE — can rollback│
│ AFTER_COMMIT    │ After successful commit             │ OUTSIDE — tx is done │
│ AFTER_ROLLBACK  │ After rollback                      │ OUTSIDE — tx is done │
│ AFTER_COMPLETION│ After either outcome                │ OUTSIDE — tx is done │
└─────────────────┴─────────────────────────────────────┴──────────────────────┘
```

**The silent failure nobody expects:**

```java
// ❌ This test will never send the email and you won't know why

// In your test class (no @Transactional annotation on test):
@Test
void testPlaceOrder() {
    orderService.placeOrder(order);
    // Email never sent!
    // Why? No active transaction in the test.
    // @TransactionalEventListener with fallbackExecution=false (default)
    // SILENTLY DOES NOTHING when there's no transaction.
    // No exception. No log. Just... nothing.
}

// Fix:
@TransactionalEventListener(fallbackExecution = true)  // run even without tx
public void onOrderPlaced(OrderPlacedEvent event) { ... }
// OR: wrap test in @Transactional
// OR: use @EventListener for things that don't need tx-awareness
```

---

### TODO 5: Async Listener — Slow Warehouse Notification

```java
// @EnableAsync must be on a @Configuration class to activate async support
@Configuration
@EnableAsync
public class AppConfig { }
```

```java
@Component
@RequiredArgsConstructor
public class WarehouseEventHandler {

    private final WarehouseService warehouseService;

    // @Async tells Spring: don't run this on the publisher's thread.
    // Hand it off to a thread pool and return immediately.
    @Async
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Runs in a DIFFERENT thread from OrderService
        // OrderService.placeOrder() has already returned to the caller
        warehouseService.notify(event.order());  // slow — 3 seconds — doesn't block anymore
    }
}
```

**What happens under the hood with `@Async`:**

```
OrderService thread:
    publishEvent(OrderPlacedEvent)
        │
        ├─ InventoryHandler.onOrderPlaced()  → runs synchronously (no @Async)
        ├─ @Async WarehouseHandler:
        │       Spring submits task to ThreadPoolTaskExecutor
        │       Returns immediately — OrderService thread continues
        └─ publishEvent() returns

Meanwhile, in thread pool:
    WarehouseHandler.onOrderPlaced() runs independently
```

**The silent transaction trap with `@Async`:**

```java
// ❌ This will NOT work the way you think

@Async
@EventListener
@Transactional   // ← this is on a DIFFERENT thread!
public void onOrderPlaced(OrderPlacedEvent event) {
    // The @Transactional creates a NEW transaction on this new thread.
    // It does NOT continue the publisher's transaction.
    // The original transaction's data may not even be committed yet
    // when this runs! You could read stale data.
}
```

Transaction context is **thread-local** — it cannot travel across threads. An async listener always starts fresh with no transaction.

---

### TODO 6: Event Chaining — Payment Triggers Shipment

```java
@Component
public class OrderFulfillmentChain {

    // An @EventListener can RETURN a value.
    // Spring automatically publishes that return value as the NEXT event.
    // This is how you chain: Order → Payment → Shipment
    // without any service knowing about the next step.

    @EventListener
    public PaymentRequestedEvent onOrderPlaced(OrderPlacedEvent event) {
        log.info("Order {} placed — requesting payment", event.order().getId());
        // The returned event is automatically published by Spring
        return new PaymentRequestedEvent(event.order(), event.order().getTotalAmount());
    }

    @EventListener
    public ShipmentRequestedEvent onPaymentRequested(PaymentRequestedEvent event) {
        PaymentResult result = paymentGateway.process(event.amount(), event.order());
        if (result.isSuccessful()) {
            return new ShipmentRequestedEvent(event.order());
        }
        return null;  // returning null → no chained event published
    }

    @EventListener
    public void onShipmentRequested(ShipmentRequestedEvent event) {
        shippingService.schedulePickup(event.order());
    }
}
```

**Flow visualized:**

```
publishEvent(OrderPlacedEvent)
    │
    ▼
onOrderPlaced() runs → returns PaymentRequestedEvent
    │                           │
    │                           ▼
    │               Spring auto-publishes PaymentRequestedEvent
    │                           │
    │                           ▼
    │               onPaymentRequested() runs → returns ShipmentRequestedEvent
    │                                                   │
    │                                                   ▼
    │                               Spring auto-publishes ShipmentRequestedEvent
    │                                                   │
    │                                                   ▼
    │                               onShipmentRequested() runs → void (chain ends)
    │
    ▼
publishEvent() finally returns to OrderService
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

### Decision 1: `@EventListener` vs `@TransactionalEventListener`

```
Use @EventListener when:
    ✓ The side effect can safely happen during the transaction
    ✓ If the listener fails, you WANT the whole transaction to roll back
    ✓ Example: deducting inventory — failure should cancel the order

Use @TransactionalEventListener(AFTER_COMMIT) when:
    ✓ The side effect involves external systems (email, SMS, webhooks)
    ✓ You only want it to happen if the DB actually saved the record
    ✓ Failure of the listener should NOT cancel the order
    ✓ Example: sending confirmation email, pushing to analytics
```

### Decision 2: Sync vs Async

```
Sync (default):
    ✓ Simple, predictable
    ✓ Exception from listener reaches publisher (can roll back)
    ✗ Slow listeners block the publishing thread
    Use when: listener is fast AND failure should affect publisher

Async (@Async):
    ✓ Publisher returns immediately — better response times
    ✓ Listener failure isolated from publisher
    ✗ No transaction propagation
    ✗ No ordering guarantee
    Use when: listener is slow (HTTP calls, email) AND failure is independent
```

### Decision 3: POJO event vs `ApplicationEvent` subclass

```java
// Modern approach — POJO record (preferred)
public record OrderPlacedEvent(Order order, String customerId) {}

// Old approach — extend ApplicationEvent
public class OrderPlacedEvent extends ApplicationEvent { ... }
```

Prefer POJO records. Less boilerplate, cleaner code. Spring handles the wrapping internally via `PayloadApplicationEvent`.

---

## 🧪 7. Edge Cases & Testing

### Silent Break #1: `@TransactionalEventListener` in tests

```java
// ❌ Email never sent — test passes but feature is broken in disguise
@Test
void orderPlaced_shouldSendEmail() {
    orderService.placeOrder(testOrder);
    verify(emailService, times(1)).sendConfirmation(any());
    // FAILS — emailService was never called
    // because no transaction was active in the test
}

// ✅ Fix: mark the service method's transaction as the test transaction
@Test
@Transactional  // creates a transaction wrapping the test
void orderPlaced_shouldSendEmail() {
    orderService.placeOrder(testOrder);
    // BUT: @Transactional on test never commits — AFTER_COMMIT still never fires!
    // The real fix: use fallbackExecution=true for tests, OR
    // use TestTransaction.flagForCommit() + TestTransaction.end()
}

// ✅ Best fix for testing transactional events:
@Test
void orderPlaced_shouldSendEmail() {
    // Use @TransactionalEventListener(fallbackExecution = true)
    // OR test the listener in isolation, not through the whole stack
    orderEventHandler.onOrderPlaced(new OrderPlacedEvent(testOrder, "cust-1", now()));
    verify(emailService, times(1)).sendConfirmation(any());
}
```

### Silent Break #2: Exception in one listener blocks all others

```java
// ❌ If InventoryHandler throws, EmailHandler never runs
@EventListener @Order(1)
public void onOrderPlaced(OrderPlacedEvent e) {
    throw new RuntimeException("inventory system down"); // ← blocks everything after
}

@EventListener @Order(2)
public void onOrderPlaced2(OrderPlacedEvent e) {
    emailService.send(...); // ← NEVER REACHED
}

// ✅ Fix: isolate listener failures in the multicaster
@Bean
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
    multicaster.setErrorHandler(throwable ->
        log.error("Listener failed — continuing: {}", throwable.getMessage())
    );
    return multicaster;
}
```

### Silent Break #3: `ContextRefreshedEvent` fires twice

```java
// ❌ Cache warmup runs twice in parent-child context (Spring MVC + DispatcherServlet)
@EventListener
public void onRefresh(ContextRefreshedEvent event) {
    cacheWarmup.run();  // runs once for root context, once propagated from child
}

// ✅ Fix: check which context fired the event
@EventListener
public void onRefresh(ContextRefreshedEvent event) {
    if (event.getApplicationContext().getParent() != null) {
        return;  // skip events propagated from child context
    }
    cacheWarmup.run();
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version:** listeners are spread across random files with no structure.

**Senior improvement:** group related handlers, add resilience, add observability.

```java
// Before (scattered):
// InventoryEventHandler.java  — in /inventory package
// EmailEventHandler.java      — in /email package
// AuditEventHandler.java      — in /audit package
// Each has its own @Component, hard to see the full picture

// After (structured):
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventHandlers {

    private final InventoryService inventoryService;
    private final EmailService emailService;
    private final AuditRepository auditRepository;
    private final MeterRegistry meterRegistry;  // metrics

    // Fast, transactional — runs in same transaction, failure rolls back order
    @EventListener
    @Order(1)
    public void deductInventory(OrderPlacedEvent event) {
        log.info("[order={}] Deducting inventory", event.order().getId());
        inventoryService.deduct(event.order());
        meterRegistry.counter("order.inventory.deducted").increment();
    }

    // Safe — only after commit, email won't go out for failed orders
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Order(2)
    public void sendConfirmationEmail(OrderPlacedEvent event) {
        log.info("[order={}] Sending confirmation email", event.order().getId());
        try {
            emailService.sendConfirmation(event.order(), event.customerId());
        } catch (Exception e) {
            // Email failure should NOT break the order — log and continue
            log.error("[order={}] Email failed: {}", event.order().getId(), e.getMessage());
            meterRegistry.counter("order.email.failed").increment();
        }
    }

    // Audit — needs its own transaction after the main one commits
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // ← own fresh transaction
    @Order(3)
    public void auditOrder(OrderPlacedEvent event) {
        auditRepository.save(AuditEntry.from(event.order()));
    }

    // Async — slow, non-critical, fire and forget
    @Async
    @EventListener
    public void notifyWarehouse(OrderPlacedEvent event) {
        log.info("[order={}] Notifying warehouse (async)", event.order().getId());
        warehouseService.notify(event.order());
    }
}
```

**What changed and why:**
- All order-related handlers in one file → easy to see the full picture at a glance
- Metrics added → you can monitor which steps fail in production
- Email failure is caught → one listener failing doesn't kill others
- Audit gets `REQUIRES_NEW` → it runs in its own transaction after commit, so audit failure is independent of order

---

## 🧠 9. Mental Model Summary

### The jargon, decoded

| Term | What it actually means |
|---|---|
| **Event** | A plain Java object. A data bag. A recorded fact: "this thing happened." |
| **Publisher** | The class that shouts a fact into Spring's event bus. Uses `eventPublisher.publishEvent(event)`. |
| **Listener** | A method annotated with `@EventListener`. Spring calls it when the matching event type is published. |
| **`ApplicationEventPublisher`** | Spring interface with one method: `publishEvent()`. Your class injects this to announce events. |
| **`SimpleApplicationEventMulticaster`** | Spring's internal router. Knows who listens to what. Calls them when an event arrives. |
| **`@TransactionalEventListener`** | Like `@EventListener` but it waits. It registers with the active DB transaction and only runs at the specified transaction lifecycle phase. |
| **`AFTER_COMMIT`** | "Only call me if the database transaction actually succeeded." |
| **`BEFORE_COMMIT`** | "Call me while the transaction is still open — I can still cause a rollback." |
| **`AFTER_ROLLBACK`** | "Call me if the transaction failed." |
| **`fallbackExecution`** | "Should I still run if there's no active transaction?" Default is `false` — silent skip. |
| **`@Async`** | "Run this listener on a different thread — don't make the publisher wait." |
| **Event chaining** | Returning a value from an `@EventListener` method — Spring auto-publishes that return value as the next event. |
| **`PayloadApplicationEvent`** | Spring's internal wrapper around your POJO events. You never see it — Spring handles it. |

---

### The decision flowchart

```
You need a side effect when something happens in your system.
    │
    ▼
Does the side effect NEED to be part of the DB transaction?
    │
    ├── YES → use @EventListener
    │           (same thread, same transaction, failure = rollback)
    │
    └── NO → Does it need to happen ONLY if DB commit succeeds?
                │
                ├── YES → use @TransactionalEventListener(AFTER_COMMIT)
                │           Is it slow or external (email, HTTP)?
                │               ├── YES → also add @Async
                │               └── NO  → leave it synchronous
                │
                └── ALWAYS run it (even on failure)?
                        └── use @TransactionalEventListener(AFTER_COMPLETION)
                            or just @EventListener
```

---

### The one diagram to remember

```
placeOrder() called
    │
    ▼
@Transactional proxy: BEGIN TRANSACTION
    │
    ├─ orderRepository.save()        ← DB write (not committed yet)
    │
    ├─ publishEvent(OrderPlacedEvent)
    │       │
    │       ├─ @EventListener:                runs NOW (sync, in tx)
    │       │       InventoryHandler → deduct inventory (in same tx)
    │       │
    │       └─ @TransactionalEventListener:   QUEUED (not run yet)
    │               EmailHandler registered with tx synchronization
    │
    ├─ ... method returns ...
    │
@Transactional proxy: COMMIT
    │
    ├─ DB commit succeeds ✅
    │       │
    │       └─ AFTER_COMMIT callbacks fire:
    │               EmailHandler.onOrderPlaced() runs NOW
    │               (outside original tx — email sent safely)
    │
    OR
    │
    ├─ DB commit fails / exception ❌
    │       │
    │       └─ ROLLBACK
    │               AFTER_COMMIT callbacks: NOT called
    │               AFTER_ROLLBACK callbacks: called if any
```

The Spring event system is just **disciplined decoupling**. Your core service does its job and announces it. Everything else is someone else's problem.
