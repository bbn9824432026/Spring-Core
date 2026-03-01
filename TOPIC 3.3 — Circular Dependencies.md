# TOPIC 3.3 — Circular Dependencies

---

## ① CONCEPTUAL EXPLANATION

### What is a Circular Dependency?

A circular dependency occurs when bean A depends on bean B, and bean B depends on bean A (directly or transitively). The dependency graph forms a cycle:

```
Direct circular dependency:
A → B → A

Transitive circular dependency:
A → B → C → A
```

Circular dependencies are generally a design smell — they often indicate that two classes are too tightly coupled and should be refactored. However, in large enterprise applications they do occur, and understanding how Spring handles them (and when it cannot) is critical.

---

### Why Circular Dependencies Are Fundamentally Problematic

Consider the lifecycle of object creation:

```
To create A: need B fully initialized
To create B: need A fully initialized
→ Deadlock — neither can be created first
```

This is not a Spring-specific problem. In plain Java:

```java
// Plain Java — impossible circular construction
public class A {
    private final B b;
    public A() { this.b = new B(this); } // pass "this" before A is fully constructed
}

public class B {
    private final A a;
    public B(A a) { this.a = a; } // receives partially constructed A
}
```

Even in plain Java, passing `this` from a constructor means the recipient gets a **partially constructed object** — all fields below the `new B(this)` call are uninitialized. This is dangerous and undefined behavior territory.

Spring faces the same challenge — but solves it (partially) through the **three-level singleton cache**.

---

### Three Categories of Circular Dependencies in Spring

```
Category 1: Constructor injection circular dependency
            → ALWAYS FAILS
            → BeanCurrentlyInCreationException
            → No workaround except redesign or @Lazy

Category 2: Setter/field injection circular dependency (singleton scope)
            → RESOLVED by three-level cache
            → Works but has subtle safety implications

Category 3: Prototype scope circular dependency
            → ALWAYS FAILS regardless of injection type
            → BeanCurrentlyInCreationException
```

---

### 3.3.1 — Constructor Injection Circular Dependency — Why It Always Fails

With constructor injection, Spring must have a **fully constructed** instance of the dependency before it can invoke the constructor. There is no way to provide a "partially ready" instance for constructor injection because:

1. The constructor itself IS the act of creating the instance
2. Before `new A(b)` completes, there is no `A` instance to provide to `B`
3. Spring has nothing to put in the singleton cache until the constructor returns

```
Attempt to create A (constructor needs B):
    → Check singleton cache for B → not there
    → Start creating B (constructor needs A):
            → Check singleton cache for A → not there
            → A is in "currently being created" set
            → Detect circular reference → throw BeanCurrentlyInCreationException
```

The `singletonsCurrentlyInCreation` set in `DefaultSingletonBeanRegistry` tracks beans whose creation is in progress. When Spring tries to create a bean that is already in this set, it immediately throws:

```java
// In DefaultSingletonBeanRegistry:
protected void beforeSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName)
            && !this.singletonsCurrentlyInCreation.add(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
}
```

The exception message is explicit:
```
Error creating bean with name 'a': Requested bean is currently in creation:
Is there an unresolvable circular reference?
```

---

### 3.3.2 — Setter/Field Injection Circular Dependency — Three-Level Cache Solution

Setter and field injection work differently from constructor injection because:

1. The bean is **instantiated first** (via no-arg or other non-circular constructor)
2. The instance exists (even if incomplete) before dependencies are injected
3. Spring can expose this early reference to break the cycle

This is the three-level singleton cache:

```java
// In DefaultSingletonBeanRegistry:

// Level 1: Fully initialized singletons
private final Map<String, Object> singletonObjects =
    new ConcurrentHashMap<>(256);

// Level 2: Early singleton references (post-ObjectFactory, pre-initialization)
private final Map<String, Object> earlySingletonObjects =
    new ConcurrentHashMap<>(16);

// Level 3: Singleton factories (ObjectFactory for early reference creation)
private final Map<String, ObjectFactory<?>> singletonFactories =
    new HashMap<>(16);
```

---

### 3.3.3 — Three-Level Cache Internals — Step-by-Step

Let's trace the complete flow for setter injection circular dependency between A and B:

```
Setup: Bean A has @Autowired field of type B
       Bean B has @Autowired field of type A
       Both are singleton scope
```

**Step 1: Container starts creating Bean A**
```
getSingleton("a")
    → check singletonObjects (L1): not there
    → check earlySingletonObjects (L2): not there
    → check singletonFactories (L3): not there
    → proceed to create A
    → add "a" to singletonsCurrentlyInCreation set
```

**Step 2: Bean A is instantiated (constructor runs)**
```
AbstractAutowireCapableBeanFactory.doCreateBean("a"):
    → createBeanInstance() → new A() → A instance created (fields not yet injected)
```

**Step 3: A's ObjectFactory is registered in Level 3 cache**
```java
// This happens BEFORE populateBean() (before field injection)
// Critical: early exposure happens here
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

The `ObjectFactory` lambda captures the early A reference. `getEarlyBeanReference()` calls `SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()` — this is where AOP proxies can wrap the early reference.

**Step 4: Bean A's dependencies are being injected (populateBean)**
```
AutowiredAnnotationBeanPostProcessor processes A:
    → field B needs injection → getSingleton("b")
    → B not in any cache → start creating B
    → add "b" to singletonsCurrentlyInCreation
```

**Step 5: Bean B is instantiated**
```
createBeanInstance() → new B() → B instance created
addSingletonFactory("b", () -> getEarlyBeanReference("b", mbd, b));
```

**Step 6: Bean B's dependencies are being injected**
```
AutowiredAnnotationBeanPostProcessor processes B:
    → field A needs injection → getSingleton("a")
```

**Step 7: A's early reference is retrieved from Level 3**
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // Level 1 check
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // Level 2 check
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                // Double-check under lock
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // Level 3: call ObjectFactory
                        ObjectFactory<?> singletonFactory =
                            this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject(); // calls lambda
                            // Promote to Level 2
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

→ A's `ObjectFactory.getObject()` is called → returns early A reference
→ A's early reference is promoted from L3 to L2 (earlySingletonObjects)
→ A's early reference is injected into B's field

**Step 8: B completes initialization**
```
B's @PostConstruct runs (if any)
B's BPP.postProcessAfterInitialization() runs
B is moved from "in creation" to Level 1 (singletonObjects)
```

**Step 9: Back to A — B is now fully ready**
```
B was just created → getSingleton("b") → found in L1
B injected into A's field
```

**Step 10: A completes initialization**
```
A's @PostConstruct runs
A's BPP.postProcessAfterInitialization() runs
A is moved from "in creation" to Level 1
A is removed from L2 (earlySingletonObjects) if it was promoted there
```

---

### Why Three Levels? Why Not Two?

**Two levels (L1 + L2) would suffice for simple cases.** The third level (singletonFactories) exists for **AOP proxy support**.

When Spring creates an AOP proxy for a bean (via `AnnotationAwareAspectJAutoProxyCreator`), the proxy must be the object that gets injected everywhere — not the raw target. But proxies are created in `postProcessAfterInitialization()` (after initialization), while early references are needed before initialization.

The `ObjectFactory` lambda in L3 calls `getEarlyBeanReference()` which asks AOP post-processors:

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}
```

`AbstractAutoProxyCreator.getEarlyBeanReference()` creates the proxy at this early stage if needed. This ensures that even when a circular dependency is resolved via early reference, the injected object is the AOP **proxy** (with all advice applied), not the raw target.

**Without L3 (if we stored raw references in L2 directly):**
- The raw object would be exposed early
- AOP proxies would be created later (in postProcessAfterInitialization)
- B would have a reference to the raw A (no AOP), while the container's L1 would have the proxy A
- Inconsistency — different parts of the application would see different objects for the same bean

**With L3 (ObjectFactory):**
- The ObjectFactory is a lazy computation that creates the early reference
- When first requested, it asks AOP processors to potentially wrap the reference
- The proxy (or raw object if no proxy) is stored in L2
- Consistency — everything gets the proxy

---

### Spring 6 Change — `allowCircularReferences = false` by Default in Spring Boot 3

Starting with Spring Boot 3 (Spring Framework 6), circular dependencies in singleton beans are **disallowed by default** in Spring Boot applications:

```
BeanCurrentlyInCreationException: Error creating bean with name 'a':
Requested bean is currently in creation: Is there an unresolvable circular reference?
```

Even setter/field injection circular dependencies that previously worked now fail at startup. This was a deliberate design decision to eliminate the subtle bugs that early references can cause.

**To re-enable (not recommended):**
```properties
# application.properties
spring.main.allow-circular-references=true
```

```java
// Programmatically:
SpringApplication app = new SpringApplication(MyApp.class);
app.setAllowCircularReferences(true);
app.run(args);
```

---

### 3.3.4 — `@Lazy` to Break Circular Dependencies

`@Lazy` is the cleanest Spring-native way to break a circular dependency without redesigning:

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;

    public ServiceA(@Lazy ServiceB serviceB) {  // @Lazy on constructor parameter
        this.serviceB = serviceB;
    }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;

    public ServiceB(ServiceA serviceA) {
        this.serviceA = serviceA;
    }
}
```

**How `@Lazy` breaks the cycle:**

Instead of injecting `ServiceB` directly, Spring injects a **CGLIB proxy** for `ServiceB`. The proxy is created immediately (no `ServiceB` instantiation yet). `ServiceA` receives the proxy and completes its construction.

Only when a method is first called on the proxy does the real `ServiceB` get instantiated and wired. At that point, `ServiceA` already exists in the singleton cache and can be injected into `ServiceB` normally.

```
Without @Lazy:
Create A → needs B → create B → needs A → CIRCULAR DEPENDENCY EXCEPTION

With @Lazy on B in A's constructor:
Create A → needs B → create PROXY for B (no real B yet) → A completes
Later, when A calls a method on the B proxy:
    → proxy creates real B → needs A → A already in L1 cache → inject A into B
    → B completes → proxy delegates to real B ✓
```

**`@Lazy` placement options:**

```java
// Option 1: On constructor parameter (breaks circular for constructor injection)
public ServiceA(@Lazy ServiceB serviceB) { ... }

// Option 2: On field (breaks circular for field injection)
@Autowired @Lazy
private ServiceB serviceB;

// Option 3: On setter
@Autowired @Lazy
public void setServiceB(ServiceB serviceB) { ... }

// Option 4: On the @Bean definition (affects all injections of this bean)
@Bean @Lazy
public ServiceB serviceB() { return new ServiceB(); }
```

**`@Lazy` internals:**
Spring's `AutowiredAnnotationBeanPostProcessor` detects `@Lazy` on an injection point and calls `buildLazyResolutionProxy()`. This creates a CGLIB proxy that implements the bean's type interface. The proxy's method calls are intercepted and delegate to `beanFactory.getBean(targetBeanName)` on first invocation.

---

### 3.3.5 — Prototype Scope Circular Dependency — Always Fails

For prototype beans, there is NO three-level cache. Each `getBean()` call creates a fresh instance. There's no early reference to expose.

```java
@Component @Scope("prototype")
public class ProtoA {
    @Autowired ProtoB protoB; // field injection
}

@Component @Scope("prototype")
public class ProtoB {
    @Autowired ProtoA protoA;
}

ctx.getBean(ProtoA.class); // → BeanCurrentlyInCreationException!
```

Even with setter/field injection, prototype circular dependencies fail because Spring detects the cycle differently:

```java
// In AbstractBeanFactory.doGetBean():
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```

Prototype beans are tracked in `prototypesCurrentlyInCreation` (a ThreadLocal `Set`). When creating `ProtoA` requires `ProtoB` which requires `ProtoA` again, the check fires.

---

### Detecting Circular Dependencies

Spring's detection mechanism:

**For singletons (during creation):**
```java
Set<String> singletonsCurrentlyInCreation  // in DefaultSingletonBeanRegistry
```

**For prototypes (during creation):**
```java
ThreadLocal<Object> prototypesCurrentlyInCreation  // in AbstractBeanFactory
// Object is either a String (single) or Set<String> (multiple)
```

When `getBean()` is called for a bean whose name is in these sets, `BeanCurrentlyInCreationException` is thrown.

---

### The Subtle Safety Problem with Early References

Even when Spring successfully resolves a circular dependency, there's a subtle danger:

```java
@Service
public class ServiceA {
    @Autowired private ServiceB serviceB;

    @PostConstruct
    public void init() {
        serviceB.doSomethingWithA(); // calls back to A
    }
}

@Service
public class ServiceB {
    @Autowired private ServiceA serviceA;

    public void doSomethingWithA() {
        serviceA.someMethod(); // accessing A
    }
}
```

During A's `@PostConstruct`, B is fully initialized. B has a reference to A. But A's `@PostConstruct` has NOT yet completed — it's currently executing. If `doSomethingWithA()` accesses state set up by A's `@PostConstruct` that runs after this line, that state is not yet set.

This is a subtle initialization-time bug that doesn't appear in testing if the problematic code path isn't exercised during the specific initialization sequence.

---

### Common Misconceptions

**Misconception 1:** "Circular dependencies always fail in Spring."
Wrong. Setter/field injection between singleton beans resolves successfully via the three-level cache (before Spring Boot 3 defaults).

**Misconception 2:** "The three-level cache resolves constructor injection circular dependencies."
Wrong. The three-level cache is used ONLY for setter/field injection. Constructor circular dependencies always fail regardless of the cache.

**Misconception 3:** "`@Lazy` prevents the bean from being created."
Wrong. `@Lazy` at an injection point creates a CGLIB proxy immediately. The actual bean is created on first method call. `@Lazy` on a `@Bean` definition means the bean is not created at startup — but the proxy mechanism is separate.

**Misconception 4:** "Circular dependency resolution works the same for all scopes."
Wrong. Only SINGLETON scope has the three-level cache. Prototype scope circular dependencies ALWAYS fail, even with setter/field injection.

**Misconception 5:** "Spring 6 removed circular dependency support entirely."
Wrong. The mechanism still exists. Spring Boot 3 (using Spring Framework 6) simply sets `allowCircularReferences=false` by default. The capability exists and can be re-enabled.

**Misconception 6:** "The three-level cache always injects the raw object."
Wrong. Level 3 uses an `ObjectFactory` lambda that calls `getEarlyBeanReference()`, which allows AOP post-processors to wrap the early reference in a proxy. Circular dependencies between AOP-proxied beans inject the proxy, not the raw object.

---

## ② CODE EXAMPLES

### Example 1 — Constructor Circular Dependency — The Failure

```java
@Service
public class BillingService {

    private final InvoiceService invoiceService;

    public BillingService(InvoiceService invoiceService) {
        this.invoiceService = invoiceService;
    }

    public void generateBill() {
        invoiceService.createInvoice();
    }
}

@Service
public class InvoiceService {

    private final BillingService billingService;

    public InvoiceService(BillingService billingService) {
        this.billingService = billingService;
    }

    public void createInvoice() {
        billingService.generateBill();
    }
}
```

```
Result at startup:
BeanCurrentlyInCreationException:
    Error creating bean with name 'billingService':
    Requested bean is currently in creation:
    Is there an unresolvable circular reference?
```

---

### Example 2 — Setter Injection Circular Dependency — Resolution

```java
@Service
public class BillingService {

    private InvoiceService invoiceService;

    @Autowired
    public void setInvoiceService(InvoiceService invoiceService) {
        this.invoiceService = invoiceService;
    }
}

@Service
public class InvoiceService {

    private BillingService billingService;

    @Autowired
    public void setBillingService(BillingService billingService) {
        this.billingService = billingService;
    }
}
```

```
Execution trace:
1. Start creating BillingService
2. Instantiate BillingService (no-arg constructor)
3. Register ObjectFactory for early BillingService in L3
4. populateBean(BillingService) → needs InvoiceService
5. Start creating InvoiceService
6. Instantiate InvoiceService (no-arg constructor)
7. Register ObjectFactory for early InvoiceService in L3
8. populateBean(InvoiceService) → needs BillingService
9. getSingleton("billingService") → L1: no → L2: no → L3: YES
10. ObjectFactory.getObject() → early BillingService reference
11. early BillingService → L2, remove from L3
12. Inject early BillingService into InvoiceService setter
13. InvoiceService init completes → L1
14. Back to BillingService → inject InvoiceService (now in L1)
15. BillingService init completes → L1

Result: Both beans created successfully.
BillingService.invoiceService = fully initialized InvoiceService ✓
InvoiceService.billingService = fully initialized BillingService ✓
```

---

### Example 3 — Field Injection Circular Dependency

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // field injection — same resolution as setter
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;
}
```

Same three-level cache resolution as setter injection. Both beans created successfully.

---

### Example 4 — Breaking Constructor Circular Dependency with `@Lazy`

```java
@Service
public class CheckoutService {

    private final PaymentService paymentService;

    // @Lazy on the parameter — CGLIB proxy injected instead of real PaymentService
    public CheckoutService(@Lazy PaymentService paymentService) {
        this.paymentService = paymentService;
        // paymentService is a CGLIB proxy here, NOT the real PaymentService
        System.out.println("CheckoutService created. PaymentService is proxy: " +
            paymentService.getClass().getName().contains("CGLIB"));
        // prints: true
    }

    public void checkout(Order order) {
        paymentService.processPayment(order); // proxy delegates to real PaymentService
        // Real PaymentService created on first method call
    }
}

@Service
public class PaymentService {

    private final CheckoutService checkoutService;

    public PaymentService(CheckoutService checkoutService) {
        this.checkoutService = checkoutService;
        // checkoutService is the real, fully-initialized CheckoutService
        // because CheckoutService was created first (using the proxy for PaymentService)
    }
}
```

---

### Example 5 — Transitive Circular Dependency

```java
@Service public class A {
    @Autowired B b;
}

@Service public class B {
    @Autowired C c;
}

@Service public class C {
    @Autowired A a; // closes the cycle: A → B → C → A
}
```

```
Resolution trace:
1. Create A → needs B
2. A instantiated → A registered in L3
3. Create B → needs C
4. B instantiated → B registered in L3
5. Create C → needs A
6. C instantiated → C registered in L3
7. getSingleton("a") → L3 hit → early A reference
8. early A → L2, inject into C
9. C completes → L1
10. Back to B → inject C (L1) into B
11. B completes → L1
12. Back to A → inject B (L1) into A
13. A completes → L1

All three beans resolved successfully.
```

---

### Example 6 — Prototype Circular Dependency — Always Fails

```java
@Component
@Scope("prototype")
public class ProtoAlpha {
    @Autowired
    private ProtoBeta protoBeta;
}

@Component
@Scope("prototype")
public class ProtoBeta {
    @Autowired
    private ProtoAlpha protoAlpha;
}

// Attempting to get ProtoAlpha:
ctx.getBean(ProtoAlpha.class);
```

```
Result:
BeanCurrentlyInCreationException:
    Error creating bean with name 'protoAlpha': Requested bean is currently in creation
    (Happens because prototypesCurrentlyInCreation ThreadLocal detects the cycle)
```

---

### Example 7 — Self-Referencing Bean (Special Case)

```java
@Service
public class TreeService {

    @Autowired
    private TreeService self; // inject itself

    @Transactional
    public void doTransactionalWork() { ... }

    public void doWorkWithTransaction() {
        self.doTransactionalWork(); // goes through AOP proxy → @Transactional works
    }
}
```

```
Resolution:
1. Create TreeService
2. TreeService instantiated → ObjectFactory registered in L3
3. populateBean → needs TreeService (self-reference)
4. getSingleton("treeService") → L3 hit → early reference (possibly AOP proxy)
5. Self-reference injected
6. TreeService completes → L1

self is the AOP proxy of TreeService (if @Transactional AOP is active)
→ Calling self.doTransactionalWork() properly triggers transaction management
```

---

### Example 8 — Verifying Three-Level Cache Behavior (Diagnostic Code)

```java
@Component
public class CircularDiagnostic implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    @PostConstruct
    public void diagnose() {
        ConfigurableListableBeanFactory factory =
            ((ConfigurableApplicationContext) ctx).getBeanFactory();

        // Access the internal DefaultSingletonBeanRegistry
        // (DefaultListableBeanFactory extends it)
        DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) factory;

        // Use reflection to inspect the caches (internal, not public API)
        try {
            Field soField = DefaultSingletonBeanRegistry.class
                .getDeclaredField("singletonObjects");
            soField.setAccessible(true);
            Map<?, ?> l1 = (Map<?, ?>) soField.get(dlbf);
            System.out.println("L1 cache size: " + l1.size());
            // At this point, all singleton beans are in L1
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

### Example 9 — Spring Boot 3 — Circular Reference Error and Fix

```java
// This will FAIL in Spring Boot 3 by default:
@Service
public class OrderService {
    @Autowired ShippingService shippingService;
}

@Service
public class ShippingService {
    @Autowired OrderService orderService;
}
```

```
Error starting ApplicationContext. To display the condition evaluation report,
re-run your application with 'debug' enabled.
BeanCurrentlyInCreationException: Error creating bean with name 'orderService':
Requested bean is currently in creation: Is there an unresolvable circular reference?
```

**Fix Option 1: Enable circular references (not recommended):**
```properties
spring.main.allow-circular-references=true
```

**Fix Option 2: Break the cycle with `@Lazy`:**
```java
@Service
public class OrderService {
    @Autowired @Lazy ShippingService shippingService; // proxy injected
}
```

**Fix Option 3: Redesign — extract shared logic:**
```java
// Extract shared behavior into a third service
@Service
public class OrderShippingCoordinator {
    // Contains logic that both OrderService and ShippingService need
}

@Service
public class OrderService {
    @Autowired OrderShippingCoordinator coordinator; // no cycle
}

@Service
public class ShippingService {
    @Autowired OrderShippingCoordinator coordinator; // no cycle
}
```

---

### Example 10 — AOP + Circular Dependency — Early Proxy Reference

```java
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.ServiceA.*(..))")
    public void logBefore(JoinPoint jp) {
        System.out.println("Before: " + jp.getSignature().getName());
    }
}

@Service
public class ServiceA {
    @Autowired private ServiceB serviceB;
}

@Service
public class ServiceB {
    @Autowired private ServiceA serviceA;
    // serviceA will be the AOP PROXY of ServiceA, not the raw object
    // This is because L3 ObjectFactory calls getEarlyBeanReference()
    // which asks AbstractAutoProxyCreator to wrap the early reference in a proxy
}
```

```java
// Verify: the injected serviceA in ServiceB IS the proxy
ServiceB b = ctx.getBean(ServiceB.class);
ServiceA aFromB = b.getServiceA();
ServiceA aFromCtx = ctx.getBean(ServiceA.class);

System.out.println(aFromB == aFromCtx);               // true — same proxy object
System.out.println(AopUtils.isAopProxy(aFromB));      // true — it's an AOP proxy
System.out.println(AopUtils.isAopProxy(aFromCtx));    // true
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which injection type can NEVER resolve circular dependencies, regardless of scope?

A) Setter injection
B) Field injection
C) Constructor injection
D) Lookup method injection

**Answer: C**
Constructor injection circular dependencies always fail. The constructor cannot receive a bean that hasn't been constructed yet, and there's no way to provide an early reference before the constructor completes.

---

**Q2 — MCQ:**
At which level of the three-level cache does Spring first expose an early bean reference to resolve circular dependencies?

A) Level 1 (singletonObjects) — fully initialized singletons
B) Level 2 (earlySingletonObjects) — early references
C) Level 3 (singletonFactories) — ObjectFactory lambdas
D) A separate circular reference cache not part of the three levels

**Answer: C**
The early reference is first registered as an `ObjectFactory` in Level 3 (singletonFactories). When another bean needs the early reference, the `ObjectFactory.getObject()` is called, and the result is promoted to Level 2 (earlySingletonObjects).

---

**Q3 — Select all that apply:**
Which of the following statements about the three-level singleton cache are correct? (Select ALL that apply)

A) Level 1 stores fully initialized singleton instances
B) Level 2 stores `ObjectFactory` lambdas for early reference creation
C) Level 3 stores partially initialized (early) singleton references
D) The Level 3 `ObjectFactory` allows AOP post-processors to wrap early references in proxies
E) Prototype beans use a separate three-level cache
F) Level 2 is populated by calling the Level 3 `ObjectFactory`

**Answer: A, D, F**
B is wrong — Level 2 stores early singleton references (not ObjectFactory). Level 3 stores ObjectFactory lambdas. C is wrong (that's Level 2). E is wrong — prototype beans have NO three-level cache.

---

**Q4 — Code Output Prediction:**

```java
@Service
public class Alpha {
    @Autowired Beta beta;

    @PostConstruct
    public void init() {
        System.out.println("Alpha init. Beta is null: " + (beta == null));
    }
}

@Service
public class Beta {
    @Autowired Alpha alpha;

    @PostConstruct
    public void init() {
        System.out.println("Beta init. Alpha is null: " + (alpha == null));
    }
}
```
(Assuming Spring Boot 2.x / Spring 5 where circular references are allowed by default)

What is the output?

A)
```
Alpha init. Beta is null: false
Beta init. Alpha is null: false
```
B)
```
Beta init. Alpha is null: true
Alpha init. Beta is null: false
```
C)
```
Alpha init. Beta is null: true
Beta init. Alpha is null: false
```
D) `BeanCurrentlyInCreationException`

**Answer: A**
Both field injections complete before `@PostConstruct` runs. When `Alpha.init()` runs, Beta is fully initialized and injected. When `Beta.init()` runs, the early reference to Alpha has been replaced by the fully initialized Alpha. Neither field is null.

---

**Q5 — Trick Question:**

```java
@Service
public class ServiceX {
    public ServiceX(@Lazy ServiceY serviceY) {
        System.out.println("ServiceX created. ServiceY class: " +
            serviceY.getClass().getSimpleName());
    }
}

@Service
public class ServiceY {
    public ServiceY(ServiceX serviceX) {
        System.out.println("ServiceY created");
    }
}
```

What is printed (assuming no Spring Boot 3 restrictions)?

A)
```
ServiceX created. ServiceY class: ServiceY
ServiceY created
```
B)
```
ServiceX created. ServiceY class: ServiceY$$SpringCGLIB$$0
ServiceY created
```
C)
```
ServiceY created
ServiceX created. ServiceY class: ServiceY$$SpringCGLIB$$0
```
D) `BeanCurrentlyInCreationException`

**Answer: B**
`@Lazy` causes Spring to inject a CGLIB proxy for `ServiceY` into `ServiceX`'s constructor. `ServiceX` is created first (receiving the proxy). Then `ServiceY` is created (receiving the fully-initialized `ServiceX`). The proxy class name contains `SpringCGLIB` or `EnhancerBySpring`.

---

**Q6 — Select all that apply:**
Which approaches correctly break a constructor injection circular dependency between ServiceA and ServiceB? (Select ALL that apply)

A) Add `@Lazy` to ServiceA's constructor parameter of type ServiceB
B) Change one of the constructor injections to setter injection
C) Change both to setter injection
D) Add `@Primary` to one of the services
E) Add `@DependsOn("serviceB")` to ServiceA
F) Extract shared logic into a third service that both depend on

**Answer: A, B, C, F**
D is wrong — `@Primary` is for disambiguation between multiple candidates, not circular dependencies. E is wrong — `@DependsOn` controls initialization ORDER but cannot break a circular dependency; if anything it makes it fail faster.

---

**Q7 — Scenario-based:**

```java
@Component @Scope("prototype")
public class TaskA {
    @Autowired TaskB taskB;
}

@Component @Scope("prototype")
public class TaskB {
    @Autowired TaskA taskA;
}

ctx.getBean(TaskA.class);
```

What happens?

A) Both prototypes created successfully via three-level cache
B) `BeanCurrentlyInCreationException` — prototype circular dependency always fails
C) Both prototypes created but with null cross-references
D) The first `getBean()` succeeds; subsequent ones fail

**Answer: B**
Prototype beans have NO three-level cache. Spring detects the cycle via `prototypesCurrentlyInCreation` ThreadLocal and throws immediately, even with field/setter injection.

---

**Q8 — MCQ:**
In Spring Boot 3, what happens when a setter injection circular dependency is detected?

A) Resolved via three-level cache as in previous versions
B) Warning logged but application starts successfully
C) `BeanCurrentlyInCreationException` thrown — circular references disallowed by default
D) `@Lazy` is automatically applied to break the cycle

**Answer: C**
Spring Boot 3 (Spring Framework 6) sets `allowCircularReferences=false` by default. Even setter/field injection circular dependencies now fail unless explicitly re-enabled.

---

**Q9 — Compilation/Runtime Error:**

```java
@Service
public class ReportService {

    private final AuditService auditService;
    private final ReportService self; // inject self via constructor

    public ReportService(AuditService auditService,
                         ReportService self) { // circular via constructor
        this.auditService = auditService;
        this.self = self;
    }
}
```

What happens?

A) Compiles and works — self-reference via constructor is special-cased
B) `BeanCurrentlyInCreationException` — self-referencing constructor is also circular
C) Compile error — constructor cannot reference its own class type
D) Works in Spring 5, fails in Spring 6

**Answer: B**
Self-referencing via constructor injection is treated the same as any other constructor circular dependency. Spring cannot provide `ReportService` as a constructor argument because `ReportService` hasn't been constructed yet. Use `@Autowired private ReportService self` (field injection) for self-reference.

---

**Q10 — Drag-and-Drop: Order the three-level cache resolution steps for setter injection circular dependency between A and B (A created first):**

Steps (unordered):
- B is moved to Level 1 (singletonObjects)
- A's ObjectFactory is registered in Level 3
- A injects B (now in Level 1) via setter
- A is instantiated (constructor called)
- B's ObjectFactory.getObject() is called for A — early A moved to Level 2
- B is instantiated, needs A
- A is moved to Level 1, removed from Level 2
- B completes its initialization

**Answer:**
1. A is instantiated (constructor called)
2. A's ObjectFactory is registered in Level 3
3. B is instantiated, needs A
4. B's ObjectFactory.getObject() is called for A — early A moved to Level 2
5. B completes its initialization
6. B is moved to Level 1 (singletonObjects)
7. A injects B (now in Level 1) via setter
8. A is moved to Level 1, removed from Level 2

---

## ④ TRICK ANALYSIS

**Trap 1: "Three-level cache resolves ALL circular dependencies"**
The three-level cache ONLY resolves singleton setter/field injection circular dependencies. Constructor injection circular dependencies always fail. Prototype circular dependencies always fail. The exam loves to present "all circular deps are resolved" as a true statement — it's false.

**Trap 2: Level 2 vs Level 3 role confusion**
Level 3 stores `ObjectFactory` (the lambda). Level 2 stores the actual early reference (the result of calling the lambda). Many candidates confuse which level stores what. Memory aid: L3 = factory, L2 = early product, L1 = finished product.

**Trap 3: Prototype circular dependency with setter injection**
Setter injection normally resolves singleton circular dependencies — this leads candidates to think it also resolves prototype circular dependencies. It does NOT. Prototype beans have no three-level cache. Any circular dependency between prototype beans always fails regardless of injection type.

**Trap 4: `@Lazy` and when the real bean is created**
`@Lazy` at an injection point creates a CGLIB proxy IMMEDIATELY (at injection time). The REAL bean is created on first METHOD CALL on the proxy. Exam questions may ask "when is the real ServiceY created?" — the answer is "on first method call," not "at context startup."

**Trap 5: `@DependsOn` as a circular dependency solution**
`@DependsOn("beanB")` on `beanA` forces `beanB` to be initialized before `beanA`. It does NOT break circular dependencies — it just specifies ordering. If A depends on B AND B depends on A, `@DependsOn` makes Spring try to create B first, fail to create B (because B needs A), and then throw. `@DependsOn` can actually make circular dependency errors appear faster/clearer, not resolve them.

**Trap 6: Spring Boot 3 default behavior change**
Before Spring Boot 3, setter/field singleton circular dependencies were silently resolved. Spring Boot 3 disallows them by default. Exam questions may present circular dependency scenarios and ask whether they work — the answer depends on the Spring version. Always note the version context.

**Trap 7: AOP proxy and early reference consistency**
When a circular dependency involves AOP-proxied beans, the early reference (from L3 ObjectFactory) should be the PROXY, not the raw object. If the proxy isn't created early enough, inconsistent references can occur — some parts of the application holding the proxy, others holding the raw object. The three-level cache's `getEarlyBeanReference()` mechanism ensures the proxy is used everywhere.

**Trap 8: Self-reference via constructor vs field**
Self-reference via CONSTRUCTOR → always fails (circular constructor injection).
Self-reference via FIELD (`@Autowired private MyService self`) → works via three-level cache.
The exam may present both and ask which works.

**Trap 9: Circular dependency error message**
The exception is `BeanCurrentlyInCreationException`, NOT `CircularDependencyException` (which doesn't exist). If an exam option includes a non-existent exception class, it's always wrong.

**Trap 10: `allowCircularReferences` location**
`spring.main.allow-circular-references=true` is a Spring Boot property, not a Spring Framework property. In plain Spring Framework (non-Boot), you configure this on `AbstractAutowireCapableBeanFactory.setAllowCircularReferences(boolean)`. Exam questions about how to configure this must specify the context.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Scenario | Result |
|---|---|
| Constructor + constructor circular dep | ALWAYS `BeanCurrentlyInCreationException` |
| Setter/field + setter/field (singleton, Spring 5) | RESOLVED via three-level cache |
| Setter/field + setter/field (singleton, Spring Boot 3) | FAILS by default — `allowCircularReferences=false` |
| Any injection type + prototype scope | ALWAYS `BeanCurrentlyInCreationException` |
| Constructor + `@Lazy` | RESOLVED — CGLIB proxy injected immediately |
| Self-reference via field | RESOLVED via three-level cache |
| Self-reference via constructor | ALWAYS fails |

---

### Three-Level Cache — What Lives Where

```
Level 1: singletonObjects      → fully initialized singleton instances
Level 2: earlySingletonObjects → early references (ObjectFactory result, possibly AOP proxy)
Level 3: singletonFactories    → ObjectFactory<T> lambdas (called when early ref first needed)

Flow:
Bean created → registered in L3
Another bean needs it → L3 ObjectFactory called → result in L2
Bean finishes → moved to L1 (removed from L2)
```

---

### Circular Dependency Decision Tree

```
Is it constructor injection?
    YES → ALWAYS FAILS (BeanCurrentlyInCreationException)
    NO (setter/field) →
        Is it prototype scope?
            YES → ALWAYS FAILS
            NO (singleton) →
                Spring Boot 3 with defaults?
                    YES → FAILS (allowCircularReferences=false)
                    NO → RESOLVED via three-level cache
```

---

### Breaking Circular Dependencies — Options

```
Option 1: @Lazy on injection point
          → Proxy injected immediately, real bean created on first call
          → Works for constructor AND setter/field injection
          → Best for breaking constructor circular deps

Option 2: Change constructor to setter/field
          → Enables three-level cache resolution
          → Only works in Spring 5 (not Spring Boot 3 defaults)

Option 3: Redesign — extract shared logic to third bean
          → Best long-term solution
          → Eliminates the circular dependency entirely

Option 4: Use @Lookup / ObjectProvider
          → For prototype-in-singleton (different problem, same symptom)
```

---

### Exception Reference

```
BeanCurrentlyInCreationException
    → Circular dependency detected
    → Parent: BeanCreationException → BeansException → RuntimeException
    → NOT: CircularDependencyException (doesn't exist)

UnsatisfiedDependencyException
    → Wrapper exception that wraps circular dependency / missing dependency errors
    → Contains root cause (BeanCurrentlyInCreationException)
```

---

### Interview One-Liners

- "Constructor injection circular dependencies always fail — Spring cannot provide an instance that hasn't been constructed yet."
- "The three-level singleton cache resolves setter/field injection circular dependencies by exposing early bean references before initialization completes."
- "Level 3 stores ObjectFactory lambdas — when called, they may return an AOP proxy to ensure consistency throughout the application."
- "Prototype scope circular dependencies always fail regardless of injection type — prototypes have no three-level cache."
- "`@Lazy` breaks constructor circular dependencies by injecting a CGLIB proxy instead of the real bean — the real bean is created on first method call."
- "Spring Boot 3 disallows circular references by default (`allowCircularReferences=false`) — even setter injection circular dependencies now fail."
- "Self-reference via field injection works via three-level cache and is a legitimate pattern for AOP self-invocation."
- "The three-level cache ensures that circular dependencies involving AOP-proxied beans inject the proxy, not the raw object, maintaining consistency."

---

