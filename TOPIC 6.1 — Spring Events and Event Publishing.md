# TOPIC 6.1 — Spring Events and Event Publishing

---

## ① CONCEPTUAL EXPLANATION

### What is the Spring Event System?

Spring's event system implements the **Observer pattern** (also called Publish-Subscribe) within the application context. It provides a mechanism for loosely coupled communication between components — publishers emit events without knowing who will handle them, and listeners react without knowing who published.

```
Without Events — tight coupling:
    OrderService → directly calls EmailService.sendConfirmation()
    OrderService → directly calls InventoryService.deduct()
    OrderService → directly calls AuditService.log()
    // OrderService knows about ALL downstream services

With Events — loose coupling:
    OrderService → publishes OrderPlacedEvent
    EmailService ← listens for OrderPlacedEvent → sends confirmation
    InventoryService ← listens for OrderPlacedEvent → deducts stock
    AuditService ← listens for OrderPlacedEvent → logs audit trail
    // OrderService knows about NONE of the downstream services
```

**Spring Event System Components:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Event Architecture                     │
├──────────────────────┬──────────────────────┬───────────────────┤
│  Event               │  Publisher           │  Listener         │
│                      │                      │                   │
│  ApplicationEvent    │  ApplicationContext  │  @EventListener   │
│  (or any Object      │  (extends            │  ApplicationListen│
│   since Spring 4.2)  │  ApplicationEventPub │  erAdapter        │
│                      │  lisher)             │                   │
└──────────────────────┴──────────────────────┴───────────────────┘
```

---

### 6.1.1 — Core Interfaces and Classes

#### ApplicationEvent

The base class for all Spring application events:

```java
public abstract class ApplicationEvent extends EventObject {

    private final long timestamp;

    public ApplicationEvent(Object source) {
        super(source);  // EventObject sets the 'source' field
        this.timestamp = System.currentTimeMillis();
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

**Custom event creation:**

```java
// Traditional: extend ApplicationEvent
public class OrderPlacedEvent extends ApplicationEvent {

    private final Order order;
    private final String customerId;

    public OrderPlacedEvent(Object source, Order order, String customerId) {
        super(source);  // source = the publisher (OrderService typically)
        this.order = order;
        this.customerId = customerId;
    }

    public Order getOrder() { return order; }
    public String getCustomerId() { return customerId; }
}

// Modern (Spring 4.2+): any POJO as event — no superclass needed
public record OrderPlacedEvent(Order order, String customerId) {}
// Plain Java class/record — no ApplicationEvent inheritance required
```

#### ApplicationEventPublisher

The interface for publishing events:

```java
public interface ApplicationEventPublisher {

    // Publish any ApplicationEvent
    void publishEvent(ApplicationEvent event);

    // Publish any Object (Spring 4.2+ — wraps in PayloadApplicationEvent)
    void publishEvent(Object event);
}
```

`ApplicationContext` extends `ApplicationEventPublisher` — the context itself IS the event publisher.

#### ApplicationListener

The interface for listening to events:

```java
public interface ApplicationListener<E extends ApplicationEvent>
        extends EventListener {

    void onApplicationEvent(E event);
}
```

#### ApplicationEventMulticaster

The infrastructure component that routes events to listeners:

```java
public interface ApplicationEventMulticaster {

    void addApplicationListener(ApplicationListener<?> listener);
    void addApplicationListenerBean(String listenerBeanName);
    void removeApplicationListener(ApplicationListener<?> listener);
    void removeAllListeners();

    void multicastEvent(ApplicationEvent event);
    void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);
}
```

The default implementation is `SimpleApplicationEventMulticaster`. It is registered in step 8 of `refresh()` (`initApplicationEventMulticaster()`).

---

### 6.1.2 — Event Publishing — Internal Mechanics

When `publishEvent()` is called:

```java
// AbstractApplicationContext.publishEvent():
@Override
public void publishEvent(Object event) {
    publishEvent(event, null);
}

protected void publishEvent(Object event, @Nullable ResolvableType typeHint) {
    // 1. Wrap in ApplicationEvent if needed
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent ae) {
        applicationEvent = ae;
    } else {
        applicationEvent = new PayloadApplicationEvent<>(this, event, typeHint);
        // PayloadApplicationEvent wraps POJO events
    }

    // 2. Multicast to all matching listeners
    getApplicationEventMulticaster()
        .multicastEvent(applicationEvent, eventType);

    // 3. If parent context exists — also publish to parent
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext aac) {
            aac.publishEvent(event, eventType);
        } else {
            this.parent.publishEvent(event);
        }
    }
}
```

**SimpleApplicationEventMulticaster.multicastEvent():**

```java
@Override
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : resolveDefaultEventType(event);

    // Get all listeners that match this event type
    Collection<ApplicationListener<?>> listeners =
        getApplicationListeners(event, type);

    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : listeners) {
        if (executor != null) {
            // ASYNC: submit to thread pool
            executor.execute(() -> invokeListener(listener, event));
        } else {
            // SYNC (default): invoke on current thread
            invokeListener(listener, event);
        }
    }
}
```

**Default event publishing is SYNCHRONOUS** — all listeners complete before `publishEvent()` returns to the caller.

---

### 6.1.3 — Listener Registration — Three Mechanisms

#### Mechanism 1 — @EventListener (Modern, Recommended)

```java
@Component
public class OrderEventHandler {

    // Listens for OrderPlacedEvent
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Order placed: " + event.getOrder().getId());
    }

    // Listens for multiple event types
    @EventListener({OrderPlacedEvent.class, OrderUpdatedEvent.class})
    public void onOrderChanged(ApplicationEvent event) {
        System.out.println("Order changed: " + event);
    }

    // With condition (SpEL)
    @EventListener(condition = "#event.order.amount > 1000")
    public void onLargeOrder(OrderPlacedEvent event) {
        System.out.println("Large order: " + event.getOrder().getAmount());
    }
}
```

**How `@EventListener` works internally:**

`@EventListener` methods are discovered by `EventListenerMethodProcessor` — a `BeanFactoryPostProcessor` that scans all beans for `@EventListener` methods and registers `ApplicationListenerMethodAdapter` beans for each:

```java
// EventListenerMethodProcessor (simplified):
public class EventListenerMethodProcessor
        implements SmartInitializingSingleton, ApplicationContextAware, BeanFactoryPostProcessor {

    @Override
    public void afterSingletonsInstantiated() {
        // After all singletons created — scan for @EventListener methods
        for (String beanName : beanNames) {
            Class<?> targetType = beanFactory.getType(beanName);
            processBean(beanName, targetType);
        }
    }

    private void processBean(String beanName, Class<?> targetType) {
        // Find all @EventListener methods
        Map<Method, EventListener> annotatedMethods =
            MethodIntrospector.selectMethods(targetType,
                (MethodIntrospector.MetadataLookup<EventListener>) method ->
                    AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));

        for (Map.Entry<Method, EventListener> entry : annotatedMethods.entrySet()) {
            // Create ApplicationListenerMethodAdapter for each method
            ApplicationListenerMethodAdapter listener =
                new ApplicationListenerMethodAdapter(beanName, targetType, entry.getKey());
            // Register with event multicaster
            applicationContext.addApplicationListener(listener);
        }
    }
}
```

#### Mechanism 2 — ApplicationListener Interface

```java
@Component
public class InventoryListener implements ApplicationListener<OrderPlacedEvent> {

    @Override
    public void onApplicationEvent(OrderPlacedEvent event) {
        inventoryService.deduct(event.getOrder());
    }
}

// Generic listener — receives ALL ApplicationEvents
@Component
public class AllEventsListener implements ApplicationListener<ApplicationEvent> {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("Event: " + event.getClass().getSimpleName());
    }
}
```

#### Mechanism 3 — Programmatic Registration

```java
// Programmatic registration before or after context refresh
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();

ctx.addApplicationListener(event -> {
    if (event instanceof OrderPlacedEvent ope) {
        System.out.println("Programmatic listener: " + ope.getOrder().getId());
    }
});

ctx.register(AppConfig.class);
ctx.refresh();
```

---

### 6.1.4 — @EventListener — Deep Dive

#### Return Value — Event Chaining

An `@EventListener` method can return a value — which is automatically published as a NEW event:

```java
@Component
public class OrderEventChain {

    @EventListener
    public ShipmentRequestedEvent onOrderPlaced(OrderPlacedEvent event) {
        // Process order placement...
        // Return value is automatically published as a new event
        return new ShipmentRequestedEvent(event.getOrder());
    }

    @EventListener
    public void onShipmentRequested(ShipmentRequestedEvent event) {
        // Handles the chained event
        shippingService.createShipment(event.getOrder());
    }
}
```

**Return types:**

```java
// Single event — published as-is
@EventListener
public OrderConfirmedEvent handleOrder(OrderPlacedEvent e) { ... }

// Collection — each element published as separate event
@EventListener
public List<ApplicationEvent> handleOrder(OrderPlacedEvent e) {
    return List.of(
        new InventoryDeductedEvent(e.getOrder()),
        new EmailScheduledEvent(e.getOrder())
    );
}

// Void — no chained event
@EventListener
public void handleOrder(OrderPlacedEvent e) { ... }
```

#### Conditional Listeners with SpEL

```java
@EventListener(condition = "#event.order.priority == 'HIGH'")
public void handleHighPriority(OrderPlacedEvent event) {
    // Only called for high priority orders
}

@EventListener(condition = "#event.amount > 500.0 && #event.currency == 'USD'")
public void handleLargeUsdOrder(PaymentEvent event) { }

// Access authentication:
@EventListener(condition = "authentication?.principal?.username == 'admin'")
public void handleAdminAction(AdminActionEvent event) { }

// Root variables in condition:
// #event       → the event argument
// #root.event  → same as #event
// #args[0]     → first argument (same as #event)
```

#### @EventListener Ordering

Multiple listeners for the same event can be ordered:

```java
@Component
public class ValidationListener {

    @EventListener
    @Order(1)  // runs first
    public void validate(OrderPlacedEvent event) {
        validateOrder(event.getOrder());
    }
}

@Component
public class InventoryListener {

    @EventListener
    @Order(2)  // runs second
    public void deductInventory(OrderPlacedEvent event) {
        inventoryService.deduct(event.getOrder());
    }
}

@Component
public class NotificationListener {

    @EventListener
    @Order(3)  // runs last
    public void notify(OrderPlacedEvent event) {
        notificationService.send(event);
    }
}
```

#### @TransactionalEventListener

A specialization of `@EventListener` that binds listener execution to transaction lifecycle:

```java
@Component
public class TransactionalOrderHandler {

    // AFTER_COMMIT (default): listener runs AFTER the current transaction commits
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Only runs if the publishing transaction COMMITTED successfully
        // Sending email AFTER we know the order was actually saved
        emailService.sendConfirmation(event.getOrder());
    }

    // AFTER_ROLLBACK: runs if transaction was rolled back
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderFailed(OrderPlacedEvent event) {
        alertService.notifyFailure(event.getOrder());
    }

    // AFTER_COMPLETION: runs after transaction completes (regardless of outcome)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void cleanup(OrderPlacedEvent event) {
        cacheService.invalidate(event.getOrder().getId());
    }

    // BEFORE_COMMIT: runs just before the current transaction commits
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommit(OrderPlacedEvent event) {
        auditService.log(event.getOrder()); // in same transaction!
    }
}
```

**Critical: `@TransactionalEventListener` requires an active transaction:**

```java
// If no transaction is active when event is published:
// → fallbackExecution = false (default): listener NOT called
// → fallbackExecution = true: listener called even without transaction

@TransactionalEventListener(fallbackExecution = true) // always execute
public void onEvent(MyEvent event) { }
```

---

### 6.1.5 — Async Event Handling

By default, events are handled synchronously. For async handling:

```java
// Method 1: @Async on @EventListener
@Configuration
@EnableAsync
public class AsyncConfig { }

@Component
public class AsyncEventHandler {

    @Async  // different thread
    @EventListener
    public void handleAsync(OrderPlacedEvent event) {
        // Runs in separate thread from ThreadPoolTaskExecutor
        emailService.sendConfirmation(event.getOrder()); // slow operation
    }
}
```

```java
// Method 2: Configure SimpleApplicationEventMulticaster with TaskExecutor
@Configuration
public class EventConfig {

    @Bean(name = "applicationEventMulticaster")
    public ApplicationEventMulticaster eventMulticaster() {
        SimpleApplicationEventMulticaster multicaster =
            new SimpleApplicationEventMulticaster();

        // Set executor — all events handled async in thread pool
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.initialize();

        multicaster.setTaskExecutor(executor);

        // Handle exceptions in async listeners
        multicaster.setErrorHandler(t ->
            log.error("Error in event listener: {}", t.getMessage(), t));

        return multicaster;
    }
}
```

**Async vs Sync trade-offs:**

```
SYNCHRONOUS (default):
    ✓ Transaction context propagates (same thread)
    ✓ Exception in listener propagates to publisher
    ✓ Publisher waits for all listeners
    ✗ Slow listeners block the publishing thread
    ✗ Tight temporal coupling

ASYNCHRONOUS:
    ✓ Publisher returns immediately
    ✓ Slow listeners don't block
    ✗ Transaction context does NOT propagate (different thread)
    ✗ Exceptions don't propagate to publisher
    ✗ Event ordering not guaranteed
```

---

### 6.1.6 — Built-in Spring Application Events

Spring itself publishes events during the `ApplicationContext` lifecycle:

```
Context startup sequence — events published during refresh():

1. ContextRefreshedEvent
   → Published at end of refresh()
   → Context fully initialized and ready
   → All beans created, all BPPs run

2. ContextStartedEvent
   → Published when ctx.start() is called
   → Lifecycle beans receive Lifecycle.start()
   → Separate from refresh — requires explicit start()

3. ContextStoppedEvent
   → Published when ctx.stop() is called
   → Lifecycle beans receive Lifecycle.stop()

4. ContextClosedEvent
   → Published when ctx.close() is called
   → Context being shut down
   → Destroy callbacks will run

Spring Boot adds more events:
   ApplicationStartingEvent      → very early — before logging, beans
   ApplicationEnvironmentPreparedEvent → Environment ready, context not yet
   ApplicationContextInitializedEvent  → context created, not refreshed
   ApplicationPreparedEvent      → context loaded, not refreshed
   ApplicationStartedEvent       → context refreshed, CommandLineRunners not run
   ApplicationReadyEvent         → application ready to service requests
   ApplicationFailedEvent        → startup failed
```

**Listening to built-in events:**

```java
@Component
public class ContextLifecycleListener {

    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        System.out.println("Context refreshed — application ready");
        ApplicationContext ctx = event.getApplicationContext();
        System.out.println("Total beans: " + ctx.getBeanDefinitionCount());
    }

    @EventListener
    public void onContextClosed(ContextClosedEvent event) {
        System.out.println("Context closing — cleanup time");
    }
}
```

**ContextRefreshedEvent published MULTIPLE TIMES in parent-child contexts:**

```
Root context refresh → ContextRefreshedEvent (from root context)
Child context refresh → ContextRefreshedEvent (from child context)
// Listeners in CHILD context see BOTH events (child's + parent's propagated up)
// Listeners in ROOT context see only ROOT context event
```

This is why `ContextRefreshedEvent` handlers should check `event.getApplicationContext()` to ensure they react to the correct context.

---

### 6.1.7 — Event Hierarchy and Listener Matching

Spring's event system supports type hierarchy — a listener for a parent event type receives child event types too:

```java
// Event hierarchy:
public class BaseOrderEvent extends ApplicationEvent { ... }
public class OrderPlacedEvent extends BaseOrderEvent { ... }
public class OrderCancelledEvent extends BaseOrderEvent { ... }

// Listener for base type receives ALL subtypes:
@EventListener
public void onAnyOrderEvent(BaseOrderEvent event) {
    // Called for OrderPlacedEvent AND OrderCancelledEvent
    System.out.println("Order event: " + event.getClass().getSimpleName());
}

// Listener for specific type:
@EventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    // Called ONLY for OrderPlacedEvent
}
```

**`ApplicationEvent` listener receives ALL events:**

```java
@EventListener
public void onAny(ApplicationEvent event) {
    // Receives EVERY event including built-in Spring events
    // ContextRefreshedEvent, ContextClosedEvent, etc.
}
```

**POJO event handling (Spring 4.2+):**

```java
// POJO event — not extending ApplicationEvent
public record UserRegisteredEvent(String email, LocalDateTime registeredAt) {}

@EventListener
public void onUserRegistered(UserRegisteredEvent event) {
    // Works! Spring wraps it in PayloadApplicationEvent
    welcomeEmailService.send(event.email());
}

// Publisher side:
applicationEventPublisher.publishEvent(
    new UserRegisteredEvent("alice@example.com", LocalDateTime.now()));
// Spring automatically wraps in PayloadApplicationEvent
```

---

### 6.1.8 — @ApplicationEventPublisher vs ApplicationContext

Both can publish events, but there are differences in how they are obtained:

```java
// Option 1: Inject ApplicationEventPublisher (narrower interface — preferred)
@Service
public class OrderService {

    @Autowired
    private ApplicationEventPublisher eventPublisher;
    // Spring injects the ApplicationContext which implements this interface
    // Better practice: depend on narrow interface, not wide ApplicationContext

    public void placeOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order, order.getCustomerId()));
    }
}

// Option 2: Inject ApplicationContext (wider interface)
@Service
public class OrderService implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void placeOrder(Order order) {
        orderRepository.save(order);
        ctx.publishEvent(new OrderPlacedEvent(this, order, order.getCustomerId()));
    }
}

// Option 3: ApplicationEventPublisherAware (Aware interface)
@Service
public class OrderService implements ApplicationEventPublisherAware {
    private ApplicationEventPublisher publisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

**Recommendation:** Use `@Autowired ApplicationEventPublisher` — follows the Interface Segregation Principle (depend on the narrowest needed interface).

---

### 6.1.9 — Event System and Transaction Integration

The interplay between events and transactions is one of the most complex parts of Spring's event system:

```
Scenario: OrderService.placeOrder() → @Transactional → publishes OrderPlacedEvent

Default (@EventListener):
    Transaction START
    → orderRepository.save(order)
    → publishEvent(OrderPlacedEvent)
        → EmailListener.onOrderPlaced() runs IN SAME TRANSACTION
        → emailRepository.save(emailLog)  ← part of SAME transaction!
        → if email fails → BOTH order AND email rolled back
    → Transaction COMMIT (or ROLLBACK if any listener threw)

Problem: Email listener is part of the transaction — failure affects order save

@TransactionalEventListener AFTER_COMMIT:
    Transaction START
    → orderRepository.save(order)
    → publishEvent(OrderPlacedEvent)
        → Event stored, NOT immediately dispatched
    → Transaction COMMIT
    → AFTER commit → EmailListener.onOrderPlaced() runs
        → email failure does NOT roll back the order
        → email runs OUTSIDE the original transaction context
        → needs its own @Transactional if it wants a transaction

@TransactionalEventListener BEFORE_COMMIT:
    Transaction START
    → orderRepository.save(order)
    → publishEvent(OrderPlacedEvent)
        → Event stored, NOT immediately dispatched
    → BEFORE commit → AuditListener.onOrderPlaced() runs IN SAME TRANSACTION
    → Transaction COMMIT
```

**`@TransactionalEventListener` — how it works internally:**

```java
// TransactionalEventListenerFactory creates TransactionalApplicationListenerMethodAdapter
// which registers a TransactionSynchronization:

public class TransactionalApplicationListenerMethodAdapter
        extends ApplicationListenerMethodAdapter
        implements TransactionalApplicationListener<ApplicationEvent> {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // If active transaction: register synchronization instead of running now
        if (TransactionSynchronizationManager.isSynchronizationActive() &&
            TransactionSynchronizationManager.isActualTransactionActive()) {

            TransactionSynchronization sync = createSynchronization(event);
            TransactionSynchronizationManager.registerSynchronization(sync);
            // Listener will be called when transaction reaches the specified phase
        } else {
            // No transaction active:
            if (this.annotation.fallbackExecution()) {
                super.onApplicationEvent(event); // run now
            }
            // else: skip (default — no transaction, no execution)
        }
    }
}
```

---

### 6.1.10 — Event System in Parent-Child Context Hierarchy

In a Spring MVC application with root + DispatcherServlet contexts:

```
Root ApplicationContext (parent):
    Beans: Services, Repositories, etc.
    Events: propagated UP to parent

Child ApplicationContext (DispatcherServlet):
    Beans: Controllers, ViewResolvers, etc.
    Events: published to child listeners + propagated to parent
```

**Event propagation rules:**

```
Event published in CHILD context:
    → Child listeners receive the event
    → Parent listeners receive the event (propagated up via parent.publishEvent())

Event published in PARENT context:
    → Parent listeners receive the event
    → Child listeners do NOT receive it (no downward propagation)
```

**Consequence for `ContextRefreshedEvent`:**

```
Root context refreshes → ContextRefreshedEvent from root
    → Root listeners receive it (once)

Child context refreshes → ContextRefreshedEvent from child
    → Child listeners receive it
    → Event propagated to parent → Root listeners receive it AGAIN

Result: Root context listeners may see ContextRefreshedEvent TWICE
        (once from root refresh, once from child refresh propagation)
```

Fix:

```java
@EventListener
public void onContextRefreshed(ContextRefreshedEvent event) {
    // Guard against double-execution in parent-child context:
    if (event.getApplicationContext() == this.applicationContext) {
        // Only react to our OWN context refresh, not propagated child refresh
        doInitialization();
    }
}
```

---

### Common Misconceptions

**Misconception 1:** "Events are always asynchronous in Spring."
Wrong. Events are SYNCHRONOUS by default. `publishEvent()` blocks until ALL listeners complete. To make them async, configure `SimpleApplicationEventMulticaster` with a `TaskExecutor` or use `@Async` on `@EventListener` methods.

**Misconception 2:** "`@TransactionalEventListener` always runs in the same transaction as the publisher."
Wrong. `@TransactionalEventListener` runs AFTER the publishing transaction completes (by default `AFTER_COMMIT`). It runs in a NEW transaction if it has its own `@Transactional`, or without a transaction at all. The original transaction is ALREADY committed/rolled back by the time the listener runs.

**Misconception 3:** "A listener for `ApplicationEvent` only receives custom application events."
Wrong. `ApplicationListener<ApplicationEvent>` (or `@EventListener` on `ApplicationEvent` parameter) receives ALL events — including Spring's built-in events like `ContextRefreshedEvent`, `ContextClosedEvent`, etc. This can cause unexpected behavior if not handled carefully.

**Misconception 4:** "POJO events (no `ApplicationEvent` superclass) are not supported."
Wrong. Since Spring 4.2, any POJO can be published as an event using `publishEvent(Object)`. Spring wraps it in `PayloadApplicationEvent<T>`. Listeners declare the POJO type as their parameter and Spring automatically unwraps it.

**Misconception 5:** "Child context listeners receive events published in the parent context."
Wrong. Event propagation is UPWARD only — child publishes to child listeners AND parent. Parent publishes to parent listeners ONLY. No downward propagation.

**Misconception 6:** "`@EventListener` methods must be in `@Component` classes."
Wrong. `@EventListener` works in any Spring-managed bean — `@Service`, `@Repository`, `@Controller`, `@Configuration`, etc. The bean just needs to be managed by the container.

**Misconception 7:** "`@TransactionalEventListener(fallbackExecution = false)` (default) throws an exception when no transaction is active."
Wrong. It SILENTLY does nothing — the listener is just not called. No exception is thrown. This silent behavior can lead to hard-to-debug issues where events appear to be ignored.

**Misconception 8:** "Events published in `@PostConstruct` are handled normally."
Nuanced. `@PostConstruct` runs during bean initialization — before `finishBeanFactoryInitialization()` completes and before `EventListenerMethodProcessor.afterSingletonsInstantiated()` registers all `@EventListener` methods. Events published in early initialization may miss listeners not yet registered. Events published in `ContextRefreshedEvent` listener or later are safe.

---

## ② CODE EXAMPLES

### Example 1 — Complete Event System Setup

```java
// 1. Event class (POJO — no superclass needed in Spring 4.2+)
public record OrderPlacedEvent(
    Order order,
    String customerId,
    LocalDateTime placedAt
) {}

// 2. Publisher
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);

        // Publish event — listeners run after this returns (sync default)
        eventPublisher.publishEvent(
            new OrderPlacedEvent(saved, saved.getCustomerId(), LocalDateTime.now()));

        return saved;
    }
}

// 3. Listener — plain @EventListener (synchronous, in same transaction)
@Component
@RequiredArgsConstructor
public class InventoryEventHandler {

    private final InventoryService inventoryService;

    @EventListener
    @Order(1) // runs first
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.deduct(event.order());
    }
}

// 4. Listener — after commit (no transaction context)
@Component
@RequiredArgsConstructor
public class NotificationEventHandler {

    private final EmailService emailService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Order(2)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Only runs if order was actually saved (transaction committed)
        emailService.sendOrderConfirmation(event.order(), event.customerId());
    }
}

// 5. Listener — async (non-blocking)
@Component
@RequiredArgsConstructor
public class AuditEventHandler {

    private final AuditRepository auditRepository;

    @Async
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Runs in separate thread — doesn't block order service
        auditRepository.save(AuditEntry.from(event));
    }
}
```

---

### Example 2 — Event Hierarchy and Listener Specificity

```java
// Event hierarchy
public abstract class AbstractOrderEvent {
    protected final Order order;
    public AbstractOrderEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

public class OrderPlacedEvent extends AbstractOrderEvent {
    private final String couponCode;
    public OrderPlacedEvent(Order order, String couponCode) {
        super(order);
        this.couponCode = couponCode;
    }
    public String getCouponCode() { return couponCode; }
}

public class OrderCancelledEvent extends AbstractOrderEvent {
    private final String reason;
    public OrderCancelledEvent(Order order, String reason) {
        super(order);
        this.reason = reason;
    }
    public String getReason() { return reason; }
}

// Listeners at different levels of hierarchy
@Component
public class OrderEventHandlers {

    // Handles ALL order events (both Placed and Cancelled)
    @EventListener
    public void onAnyOrderEvent(AbstractOrderEvent event) {
        log.info("Order event [{}]: orderId={}",
            event.getClass().getSimpleName(), event.getOrder().getId());
    }

    // Handles only OrderPlacedEvent
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        if (event.getCouponCode() != null) {
            couponService.markUsed(event.getCouponCode());
        }
    }

    // Handles only OrderCancelledEvent
    @EventListener
    public void onOrderCancelled(OrderCancelledEvent event) {
        refundService.processRefund(event.getOrder(), event.getReason());
    }

    // Multiple event types in one method
    @EventListener({OrderPlacedEvent.class, OrderCancelledEvent.class})
    public void onOrderStateChange(AbstractOrderEvent event) {
        analyticsService.trackOrderState(event.getOrder(), event.getClass());
    }
}
```

---

### Example 3 — Event Chaining

```java
@Component
public class OrderFulfillmentChain {

    // Step 1: Order placed → triggers payment
    @EventListener
    public PaymentRequestedEvent onOrderPlaced(OrderPlacedEvent event) {
        log.info("Order {} placed — requesting payment", event.order().getId());
        // Return value is auto-published as next event
        return new PaymentRequestedEvent(
            event.order(),
            event.order().getTotalAmount()
        );
    }

    // Step 2: Payment requested → process payment
    @EventListener
    public PaymentProcessedEvent onPaymentRequested(PaymentRequestedEvent event) {
        log.info("Processing payment for order {}", event.order().getId());
        PaymentResult result = paymentGateway.process(event.amount(), event.order());
        return new PaymentProcessedEvent(event.order(), result);
    }

    // Step 3: Payment processed → request shipment
    @EventListener
    public ShipmentRequestedEvent onPaymentProcessed(PaymentProcessedEvent event) {
        if (event.result().isSuccessful()) {
            log.info("Payment successful — requesting shipment");
            return new ShipmentRequestedEvent(event.order());
        }
        // Returning null → no chained event published
        // (or publish a PaymentFailedEvent instead)
        return null;
    }

    // Step 4: Shipment requested → final notification
    @EventListener
    public List<ApplicationEvent> onShipmentRequested(ShipmentRequestedEvent event) {
        // Can return multiple events — each published separately
        return List.of(
            new InventoryReservedEvent(event.order()),
            new CustomerNotificationEvent(event.order(), "Shipment arranged!")
        );
    }
}
```

---

### Example 4 — @TransactionalEventListener Patterns

```java
@Service
public class OrderService {

    @Autowired ApplicationEventPublisher publisher;
    @Autowired OrderRepository repository;

    @Transactional
    public Order placeOrder(Order order) {
        Order saved = repository.save(order);
        // Event published NOW — but @TransactionalEventListener handlers
        // will only fire AFTER transaction commits
        publisher.publishEvent(new OrderPlacedEvent(saved));
        return saved;
    }
}

@Component
public class TransactionalOrderHandlers {

    // AFTER_COMMIT: email sent only if order was actually saved
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendConfirmationEmail(OrderPlacedEvent event) {
        // Original transaction ALREADY committed
        // emailService runs WITHOUT a transaction by default
        emailService.send(event.order()); // no transaction
    }

    // AFTER_COMMIT + own @Transactional: runs in NEW transaction
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createAuditRecord(OrderPlacedEvent event) {
        // NEW transaction for the audit record
        auditRepository.save(AuditEntry.from(event.order()));
    }

    // BEFORE_COMMIT: runs in SAME transaction — can still roll back everything
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void validateBeforeCommit(OrderPlacedEvent event) {
        // Still inside original transaction!
        if (!fraudDetectionService.isLegitimate(event.order())) {
            throw new FraudDetectedException(event.order().getId());
            // This ROLLS BACK the entire transaction — order NOT saved!
        }
    }

    // AFTER_ROLLBACK: cleanup on failure
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderFailure(OrderPlacedEvent event) {
        alertingService.notifyOrderFailure(event.order().getId());
    }

    // fallbackExecution=true: runs even without transaction
    @TransactionalEventListener(fallbackExecution = true)
    public void alwaysRun(OrderPlacedEvent event) {
        metricsService.increment("order.placed.attempted");
    }
}
```

---

### Example 5 — Async Event Configuration

```java
@Configuration
@EnableAsync
public class AsyncEventConfig {

    // Register custom event multicaster with async executor
    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster() {
        SimpleApplicationEventMulticaster multicaster =
            new SimpleApplicationEventMulticaster();

        // Configure thread pool
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("async-event-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();

        multicaster.setTaskExecutor(executor);

        // Error handling for async listeners
        multicaster.setErrorHandler(throwable -> {
            log.error("Error in async event listener: {}", throwable.getMessage(), throwable);
            // Can notify alerting system here
        });

        return multicaster;
    }
}

// Individual async listener (requires @EnableAsync)
@Component
public class AsyncOrderHandlers {

    // All events to this service run async
    @Async("asyncEventExecutor")
    @EventListener
    public void processAnalytics(OrderPlacedEvent event) {
        // Runs in separate thread — doesn't block publisher
        analyticsService.track(event);
    }

    @Async("asyncEventExecutor")
    @EventListener
    public void syncToExternalSystem(OrderPlacedEvent event) {
        externalCrmService.createOrder(event.order()); // slow HTTP call
    }
}
```

---

### Example 6 — Built-in Context Event Handling

```java
@Component
public class ApplicationLifecycleHandler {

    // Cache warm-up after full context initialization
    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        // Guard for parent-child context duplication
        if (event.getApplicationContext().getParent() != null) {
            return; // skip child context events propagated to parent
        }
        log.info("Context refreshed — warming up caches");
        cacheWarmupService.warmAll();
    }

    // Graceful shutdown
    @EventListener
    public void onContextClosing(ContextClosedEvent event) {
        log.info("Context closing — draining in-flight requests");
        requestDrainer.drainAndWait(Duration.ofSeconds(30));
    }

    // React to Spring Boot startup complete
    @EventListener
    public void onApplicationReady(ApplicationReadyEvent event) {
        // Spring Boot specific — after everything is ready
        log.info("Application ready at {}",
            event.getTimestamp());
        healthCheckService.markHealthy();
    }
}
```

---

### Example 7 — Custom ApplicationEventMulticaster

```java
@Configuration
public class CustomMulticasterConfig {

    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster(
            BeanFactory beanFactory) {

        // Custom multicaster with error isolation
        return new SimpleApplicationEventMulticaster(beanFactory) {

            @Override
            protected void invokeListener(ApplicationListener<?> listener,
                                           ApplicationEvent event) {
                try {
                    super.invokeListener(listener, event);
                } catch (Exception e) {
                    // Isolate: one listener failure doesn't stop others
                    log.error("Listener {} failed for event {}: {}",
                        listener.getClass().getSimpleName(),
                        event.getClass().getSimpleName(),
                        e.getMessage(), e);
                    // Listener failure is logged but not propagated
                    // Other listeners continue receiving the event
                }
            }
        };
    }
}
```

---

### Example 8 — Generic Event with Type Safety

```java
// Generic event — carries any payload
public class DataChangedEvent<T> extends ApplicationEvent {

    private final T oldValue;
    private final T newValue;
    private final String entityType;

    public DataChangedEvent(Object source, T oldValue, T newValue, String entityType) {
        super(source);
        this.oldValue = oldValue;
        this.newValue = newValue;
        this.entityType = entityType;
    }

    public T getOldValue() { return oldValue; }
    public T getNewValue() { return newValue; }
    public String getEntityType() { return entityType; }
}

// Publishing typed events
@Service
public class ProductService {

    @Autowired ApplicationEventPublisher publisher;

    public void updatePrice(Product product, BigDecimal newPrice) {
        BigDecimal oldPrice = product.getPrice();
        product.setPrice(newPrice);
        productRepository.save(product);

        publisher.publishEvent(
            new DataChangedEvent<>(this, oldPrice, newPrice, "Product.price"));
    }
}

// Listening to generic events via ResolvableTypeProvider
@Component
public class PriceChangeHandler {

    @EventListener
    public void onPriceChanged(DataChangedEvent<BigDecimal> event) {
        // Type-safe handling of BigDecimal price changes
        BigDecimal change = event.getNewValue().subtract(event.getOldValue());
        log.info("Price changed by: {}", change);
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which of the following is TRUE about Spring's default event publishing behavior?

A) Events are published asynchronously — publisher returns immediately
B) Events are published synchronously — publisher blocks until all listeners complete
C) Events are published in a separate transaction
D) Events are queued and dispatched in bulk at end of request

**Answer: B**
Spring's `SimpleApplicationEventMulticaster` defaults to synchronous dispatch — all listeners are invoked on the SAME thread before `publishEvent()` returns. To make async, configure with a `TaskExecutor`.

---

**Q2 — Select all that apply:**
Which mechanisms register an `@EventListener` method as a listener? (Select ALL that apply)

A) `EventListenerMethodProcessor` scans beans and registers `ApplicationListenerMethodAdapter`
B) `@EventListener` directly registers the method with `SimpleApplicationEventMulticaster`
C) `SmartInitializingSingleton.afterSingletonsInstantiated()` is used for registration timing
D) The `@EventListener` annotation itself is processed by a `BeanPostProcessor`
E) `ApplicationListenerDetector` finds `ApplicationListener` interface implementations

**Answer: A, C, E**
`EventListenerMethodProcessor` (A) implements `SmartInitializingSingleton` (C) and scans for `@EventListener` after all singletons are instantiated. `ApplicationListenerDetector` (E) is a BPP that detects beans implementing `ApplicationListener` interface directly. B is wrong — `@EventListener` doesn't self-register. D is partially wrong — it's more about `afterSingletonsInstantiated()` timing.

---

**Q3 — Code Output Prediction:**

```java
@Component
public class EventPublisher {
    @Autowired ApplicationEventPublisher pub;

    @EventListener
    public String onMyEvent(MyEvent event) {
        System.out.println("Handling: " + event.getData());
        return "chained"; // what does this do?
    }
}

public record MyEvent(String data) {}
public record ChainedEvent(String value) {}
```

Wait — `onMyEvent` returns `String`, not a Spring event type. What happens?

A) The string "chained" is published as a new event (wrapped in `PayloadApplicationEvent`)
B) The string is ignored — return values must be `ApplicationEvent` subtypes
C) `ClassCastException` at runtime
D) `BeanInitializationException` at startup — invalid return type

**Answer: A**
Since Spring 4.2, any return value from `@EventListener` is published as a new event. A `String` is a valid return value and will be published as a `PayloadApplicationEvent<String>`. Listeners for `String` payloads would receive it.

---

**Q4 — Trick Question:**

```java
@Service
public class PaymentService {

    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    public void processPayment(Payment payment) {
        paymentRepository.save(payment);
        publisher.publishEvent(new PaymentCompletedEvent(payment));
        // Assume PaymentEmailHandler listens with @TransactionalEventListener
        // (phase = AFTER_COMMIT, fallbackExecution = false)
    }
}

@Component
public class PaymentEmailHandler {
    @TransactionalEventListener // AFTER_COMMIT, fallbackExecution=false
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        emailService.send("Payment confirmed: " + event.payment().getId());
    }
}
```

A test calls `paymentService.processPayment()` OUTSIDE of any transaction (no `@Transactional` on test, no transaction template). What happens?

A) Email is sent — `publishEvent` runs the listener synchronously
B) Email is NOT sent — `fallbackExecution=false` means listener skipped without active transaction
C) `IllegalTransactionStateException` thrown — `@Transactional` requires transaction
D) Email is sent after the test method completes

**Answer: B**
`@TransactionalEventListener` with `fallbackExecution=false` (default) SILENTLY skips execution when no active transaction is present. No exception thrown. `processPayment()` has `@Transactional` on the service, BUT the test is calling `paymentService.processPayment()` — if the test class has no transaction, and `@Transactional` is applied via proxy... actually the `@Transactional` on `processPayment()` WILL create a transaction. So the event IS fired AFTER_COMMIT. The correct answer depends on whether the test has transaction support. If `paymentService` is the proxy, `@Transactional` creates a transaction → event fires after commit → B is wrong. This is actually a trick — if the test MOCKS the repository and Spring transactions are active, AFTER_COMMIT fires. Answer: A (email IS sent if the transactional proxy creates/commits a transaction).

---

**Q5 — Select all that apply:**
Which `TransactionPhase` values run INSIDE the original transaction? (Select ALL that apply)

A) `BEFORE_COMMIT`
B) `AFTER_COMMIT`
C) `AFTER_ROLLBACK`
D) `AFTER_COMPLETION`

**Answer: A**
Only `BEFORE_COMMIT` runs while the original transaction is still active (before the commit decision). `AFTER_COMMIT`, `AFTER_ROLLBACK`, and `AFTER_COMPLETION` all run AFTER the transaction has already completed (committed or rolled back). Changes made in `BEFORE_COMMIT` are part of the original transaction.

---

**Q6 — MCQ:**
In a Spring MVC application with root context and child DispatcherServlet context, a `ContextRefreshedEvent` listener is registered in the root context. How many times does it fire during a normal startup?

A) Once — only when the root context is refreshed
B) Twice — once for root context, once propagated from child context
C) Three times — root context twice plus child once
D) Once — events don't propagate between contexts

**Answer: B**
Root context refresh → `ContextRefreshedEvent` → root listener fires (1st time). Child context refresh → `ContextRefreshedEvent` → propagated UP to parent → root listener fires (2nd time). The root listener receives the child's event because events propagate upward.

---

**Q7 — Select all that apply:**
Which statements about `@Async @EventListener` are TRUE? (Select ALL that apply)

A) Requires `@EnableAsync` configuration
B) The original transaction context is propagated to the async thread
C) Exceptions in the listener propagate to the publisher thread
D) The publisher thread returns before the listener finishes
E) A separate `TaskExecutor` thread handles the listener
F) `@Order` on async listeners guarantees execution order

**Answer: A, D, E**
B wrong — transaction context does NOT propagate across threads (new thread = no transaction). C wrong — async exceptions do NOT propagate to the publisher (different thread). F wrong — `@Order` affects which thread STARTS first but async execution order is non-deterministic.

---

**Q8 — Code Output Prediction:**

```java
@Component
public class OrderHandler {

    @EventListener
    @Order(1)
    public void first(OrderPlacedEvent e) {
        System.out.println("FIRST: " + e.getOrder().getId());
    }

    @EventListener
    @Order(3)
    public void third(OrderPlacedEvent e) {
        System.out.println("THIRD: " + e.getOrder().getId());
    }

    @EventListener
    @Order(2)
    public void second(OrderPlacedEvent e) {
        System.out.println("SECOND: " + e.getOrder().getId());
    }
}

// publisher.publishEvent(new OrderPlacedEvent(order));
```

What is the output order?

A) FIRST, SECOND, THIRD (ordered by @Order value)
B) FIRST, THIRD, SECOND (declaration order)
C) SECOND, FIRST, THIRD (undefined)
D) Undefined — @Order has no effect on @EventListener

**Answer: A**
`@Order` on `@EventListener` methods controls execution order — lower value runs first. `@Order(1)` → `@Order(2)` → `@Order(3)`. Output: `FIRST`, `SECOND`, `THIRD`.

---

**Q9 — MCQ:**
What happens when an `@EventListener` method throws a `RuntimeException`?

A) The exception is logged and other listeners continue
B) The exception propagates to the publisher — `publishEvent()` throws
C) The exception is swallowed — listeners always complete silently
D) The context is refreshed to recover from the error

**Answer: B**
In synchronous mode (default), exceptions from listeners propagate back to the publisher. `publishEvent()` throws the exception. Other listeners registered AFTER the failing one may NOT be called. To isolate listener failures, configure a custom error handler on `SimpleApplicationEventMulticaster`.

---

**Q10 — Drag-and-Drop: Match TransactionPhase to behavior:**

Phases: `BEFORE_COMMIT`, `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`

Behaviors:
- Runs in SAME transaction — can cause rollback by throwing
- Runs after successful commit — original transaction closed
- Runs after failed transaction — rollback has occurred
- Runs after any transaction outcome (commit or rollback)
- Best phase for sending external notifications (emails, webhooks)
- Best phase for audit logging that must be in same transaction
- Cannot be used to undo the completed transaction
- Can veto the commit by throwing exception

**Answer:**
- `BEFORE_COMMIT` → Runs in SAME transaction — can cause rollback by throwing
- `AFTER_COMMIT` → Runs after successful commit — original transaction closed
- `AFTER_ROLLBACK` → Runs after failed transaction — rollback has occurred
- `AFTER_COMPLETION` → Runs after any transaction outcome (commit or rollback)
- `AFTER_COMMIT` → Best phase for sending external notifications (emails, webhooks)
- `BEFORE_COMMIT` → Best phase for audit logging that must be in same transaction
- `AFTER_ROLLBACK` / `AFTER_COMMIT` → Cannot be used to undo the completed transaction
- `BEFORE_COMMIT` → Can veto the commit by throwing exception

---

## ④ TRICK ANALYSIS

**Trap 1: Events are async by default**
Wrong — the most common assumption. `SimpleApplicationEventMulticaster` with no executor configured dispatches events SYNCHRONOUSLY on the publisher's thread. The publisher blocks until ALL listeners complete. Async requires explicit configuration.

**Trap 2: @TransactionalEventListener always runs**
Wrong. `@TransactionalEventListener` with `fallbackExecution = false` (default) SILENTLY skips when no active transaction is present. No exception, no warning. This causes hard-to-debug scenarios in tests that don't have active transactions.

**Trap 3: @TransactionalEventListener AFTER_COMMIT runs in the original transaction**
Wrong. By the time `AFTER_COMMIT` runs, the original transaction is ALREADY committed and closed. The listener runs WITHOUT any transaction context (unless it has its own `@Transactional(propagation = REQUIRES_NEW)`).

**Trap 4: Child context events don't reach root context listeners**
Wrong. Event propagation goes UPWARD — child context publishes to its listeners AND propagates to parent. This causes the double-firing problem for `ContextRefreshedEvent` when root listeners receive both the root refresh event AND the child refresh event propagated up.

**Trap 5: @Async @EventListener propagates transaction context**
Wrong. `@Async` means a different thread. Transaction context is thread-bound. The async listener runs WITHOUT any transaction from the publisher.

**Trap 6: @EventListener return value must be ApplicationEvent**
Wrong. Since Spring 4.2, ANY return value is published as a new event — including POJOs, Strings, etc. They are wrapped in `PayloadApplicationEvent`. Only `null` and `void` suppress chaining.

**Trap 7: @Order has no effect on @EventListener**
Wrong. `@Order` on `@EventListener` methods (or on the class containing them) controls the execution order of listeners for the same event type. Lower order value = higher priority = runs first.

**Trap 8: publishEvent() is safe to call from @PostConstruct**
Partially wrong. Events published during early initialization may miss `@EventListener` methods because `EventListenerMethodProcessor.afterSingletonsInstantiated()` registers them AFTER all singletons are instantiated. Events published from `@PostConstruct` (during singleton creation) may miss some `@EventListener`-registered listeners.

**Trap 9: Exception in one listener prevents other listeners from running**
True by default — but only in sync mode. In synchronous dispatch, if listener 2 of 5 throws, listeners 3-5 do NOT run, and the exception propagates to the publisher. Configuring a custom `setErrorHandler()` on `SimpleApplicationEventMulticaster` can change this to log-and-continue.

**Trap 10: ApplicationListener<ApplicationEvent> receives only custom events**
Wrong. It receives ALL ApplicationEvents — custom AND Spring built-in (ContextRefreshedEvent, ContextClosedEvent, etc.). This can cause unexpected execution on context lifecycle events if not filtered.

---

## ⑤ SUMMARY SHEET

### Event System Architecture

```
Publisher                  EventMulticaster              Listeners
   │                           │                            │
publishEvent(event)  ──────→  multicastEvent()  ─────────→ @EventListener methods
   │                           │                            │ ApplicationListener impls
   │                    SimpleApplicationEvent              │
   │                    Multicaster:                        │
   │                    - If executor=null: SYNC            │
   │                    - If executor set: ASYNC            │
   │                           │                            │
   │◄──── blocks (sync) ───────│◄─── all listeners done ────┘
```

---

### @TransactionalEventListener Phase Reference

```
┌─────────────────────┬──────────────────────────┬───────────────────┐
│ Phase               │ When runs                 │ Transaction state │
├─────────────────────┼──────────────────────────┼───────────────────┤
│ BEFORE_COMMIT       │ Just before commit        │ INSIDE original tx│
│ AFTER_COMMIT        │ After successful commit   │ OUTSIDE (tx closed)│
│ AFTER_ROLLBACK      │ After rollback            │ OUTSIDE (tx closed)│
│ AFTER_COMPLETION    │ After any outcome         │ OUTSIDE (tx closed)│
└─────────────────────┴──────────────────────────┴───────────────────┘
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Default dispatch | SYNCHRONOUS — blocks publisher thread |
| POJO events | Supported since 4.2 — wrapped in `PayloadApplicationEvent` |
| `@TransactionalEventListener` default | `AFTER_COMMIT`, `fallbackExecution=false` |
| `fallbackExecution=false` with no tx | Listener SILENTLY skipped — no exception |
| `BEFORE_COMMIT` | Runs IN original transaction — can cause rollback |
| `AFTER_COMMIT` needs own tx | Needs `@Transactional(REQUIRES_NEW)` for its own transaction |
| Event propagation direction | Child→Parent ONLY — no downward propagation |
| `ContextRefreshedEvent` double-fire | Root listener fires twice in parent-child setup |
| `@EventListener` return value | Auto-published as new event (any type) |
| `@Order` on `@EventListener` | Controls ordering — lower = runs first |
| Exception in sync listener | Propagates to publisher — stops remaining listeners |
| `@Async @EventListener` tx context | NOT propagated — runs without transaction |
| `@EventListener` in `@PostConstruct` | May miss listeners not yet registered |

---

### Event Registration Mechanisms

```
@EventListener annotation:
    → Detected by EventListenerMethodProcessor
    → Registered in afterSingletonsInstantiated() (after all beans created)
    → Creates ApplicationListenerMethodAdapter per method

ApplicationListener interface:
    → Detected by ApplicationListenerDetector (BPP)
    → Registered as beans are created
    → Type parameter determines event type

Programmatic:
    → ctx.addApplicationListener(listener)
    → applicationEventMulticaster.addApplicationListener(listener)
```

---

### Interview One-Liners

- "Spring events are SYNCHRONOUS by default — `publishEvent()` blocks until ALL listeners complete. Async requires explicit configuration via `SimpleApplicationEventMulticaster.setTaskExecutor()` or `@Async`."
- "`@TransactionalEventListener` with default `fallbackExecution=false` SILENTLY skips execution when no active transaction is present — a common source of hard-to-debug test failures."
- "`BEFORE_COMMIT` runs inside the original transaction and CAN veto the commit by throwing. `AFTER_COMMIT` runs AFTER the transaction is committed and closed."
- "Event propagation goes UPWARD only — child context events reach parent listeners, but parent events do NOT reach child listeners."
- "Since Spring 4.2, any POJO can be an event — `publishEvent(anyObject)` wraps it in `PayloadApplicationEvent`. No `ApplicationEvent` superclass required."
- "An `@EventListener` method can return a value — Spring automatically publishes it as a new event, enabling event chaining without direct coupling."
- "Exceptions from synchronous listeners propagate to the publisher — stopping remaining listeners. Configure `errorHandler` on `SimpleApplicationEventMulticaster` to isolate listener failures."
- "`@EventListener` methods registered via `EventListenerMethodProcessor.afterSingletonsInstantiated()` — events published during `@PostConstruct` may miss some listeners not yet registered."

---
