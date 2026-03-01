# TOPIC 3.1 — Types of Dependency Injection

---

## ① CONCEPTUAL EXPLANATION

### What is Dependency Injection?

Dependency Injection is the mechanism through which the Spring IoC container fulfills the dependencies declared by a bean. Instead of a class creating or locating its own dependencies, the container *pushes* them in. The class is passive — it declares what it needs, and the container satisfies those needs.

Spring supports five distinct injection mechanisms:

```
1. Constructor Injection
2. Setter Injection
3. Field Injection
4. Lookup Method Injection
5. Method Replacement (Arbitrary Method Injection)
```

Each has distinct internal mechanics, use cases, trade-offs, and pitfalls. Understanding all five at an architectural level is required for mastery.

---

### 3.1.1 — Constructor Injection

#### Mechanics

Constructor injection satisfies dependencies through the class constructor. The container calls `Constructor.newInstance(resolvedArgs)` via reflection, passing fully resolved dependencies as arguments.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final int timeout;

    // Single constructor — @Autowired optional in Spring 4.3+
    public OrderService(PaymentService paymentService,
                        InventoryService inventoryService,
                        @Value("${order.timeout}") int timeout) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
        this.timeout = timeout;
    }
}
```

#### Internal Processing — `ConstructorResolver`

When Spring creates a bean via constructor injection:

```
AbstractAutowireCapableBeanFactory.createBeanInstance()
        │
        ▼
ConstructorResolver.autowireConstructor()
        │
        ├── Step 1: Determine candidate constructors
        │       ├── If single constructor → use it
        │       ├── If multiple constructors:
        │       │       ├── Any @Autowired constructor? → use it
        │       │       ├── Single non-default constructor? → use it
        │       │       └── Multiple? → use default (no-arg) constructor
        │       └── SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()
        │           called to discover candidate constructors
        │
        ├── Step 2: For each candidate constructor:
        │       └── Score by number of matching arguments
        │           (more args matched → higher score → preferred)
        │
        ├── Step 3: Resolve each constructor parameter:
        │       ├── Check ConstructorArgumentValues (explicit XML/programmatic args)
        │       ├── Resolve autowired params via resolveDependency()
        │       │       └── Type → @Qualifier → name fallback (same algorithm as @Autowired)
        │       └── Apply TypeConverter for type conversion
        │
        └── Step 4: Invoke constructor via reflection
                Constructor.newInstance(resolvedArgs)
```

#### Constructor Selection Rules (Critical)

```java
public class MyService {
    // Rule 1: Single constructor → always used (no @Autowired needed from Spring 4.3+)
    public MyService(Dep dep) { }
}

public class MyService {
    // Rule 2: Multiple constructors, one @Autowired → use @Autowired one
    @Autowired
    public MyService(DepA a, DepB b) { }
    public MyService(DepA a) { }
}

public class MyService {
    // Rule 3: Multiple constructors, @Autowired(required=false) on multiple
    // Spring will try each in order of parameter count (more params = preferred)
    @Autowired(required = false)
    public MyService(DepA a, DepB b) { }  // preferred if both DepA and DepB available
    @Autowired(required = false)
    public MyService(DepA a) { }          // fallback if only DepA available
    public MyService() { }               // last resort
}

public class MyService {
    // Rule 4: Multiple constructors, NO @Autowired → uses no-arg constructor
    public MyService(DepA a) { }
    public MyService() { }  // this one is used
}

public class MyService {
    // Rule 5: Multiple constructors, NO @Autowired, NO no-arg → BeanCreationException
    public MyService(DepA a) { }
    public MyService(DepB b) { }
    // Spring cannot determine which to use → throws
}
```

#### Advantages of Constructor Injection

**1. Immutability:** Dependencies can be `final`. This is the single most important advantage for thread safety in singleton beans.

**2. Mandatory dependencies declared:** If a constructor dependency is not available, the bean cannot be created — the failure is explicit and immediate.

**3. Testability:** You can create instances with `new MyService(mockDep)` without a Spring container. No reflection needed for testing.

**4. Circular dependency detection:** Constructor injection circular dependencies FAIL FAST with `BeanCurrentlyInCreationException`. There is no way to create a partially-initialized object for the cycle to complete — the circular dependency is impossible to satisfy. This is actually a feature — it forces redesign.

**5. Forces single responsibility:** If a constructor grows beyond 3-4 parameters, it's a code smell signaling too many dependencies. Constructor injection makes this obvious in a way field injection hides.

---

### 3.1.2 — Setter Injection

#### Mechanics

Setter injection calls setter methods (or any method annotated with `@Autowired`) on an already-instantiated bean object. The bean is first created with its constructor (no-arg or otherwise), then Spring calls the setter methods via reflection: `Method.invoke(beanInstance, resolvedValue)`.

```java
@Service
public class OrderService {

    private PaymentService paymentService;
    private int timeout;

    // XML-driven setter injection — method must be public
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }
}
```

```java
@Service
public class OrderService {

    private PaymentService paymentService;

    // Annotation-driven setter injection
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    // @Autowired on ANY method — not just setters
    @Autowired
    public void configure(DataSource ds, CacheManager cm) {
        // Spring injects both params from container
    }
}
```

#### Internal Processing

```
AbstractAutowireCapableBeanFactory.populateBean()
        │
        ├── Call InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
        │       → can suppress property injection entirely if returns false
        │
        ├── Apply XML-defined PropertyValues:
        │       → MutablePropertyValues from BeanDefinition
        │       → BeanWrapper.setPropertyValues()
        │       → PropertyEditor / ConversionService for type conversion
        │
        └── AutowiredAnnotationBeanPostProcessor.postProcessProperties()
                → scans for @Autowired on methods and fields
                → resolves each dependency
                → injects via reflection
```

#### Three-Level Cache and Setter Injection

Setter injection enables **circular dependency resolution** through the three-level cache:

```
Bean A needs Bean B (setter)
Bean B needs Bean A (setter)

1. Start creating A:
   - Instantiate A (empty object via constructor)
   - Register A's ObjectFactory in singletonFactories (Level 3)

2. A needs B → start creating B:
   - Instantiate B (empty object)
   - Register B's ObjectFactory in singletonFactories (Level 3)

3. B needs A → check caches:
   - Level 1 (singletonObjects): not there (A not fully initialized)
   - Level 2 (earlySingletonObjects): not there yet
   - Level 3 (singletonFactories): A's ObjectFactory IS there
   → call A's ObjectFactory.getObject() → get early reference to A
   → move A to Level 2 (earlySingletonObjects)
   → inject early A reference into B's setter

4. B initialization completes → B moves to Level 1

5. Continue A's initialization:
   - B is now in Level 1 → inject B into A's setter
   - A initialization completes → A moves to Level 1
```

This is why setter injection CAN resolve circular dependencies while constructor injection CANNOT — at the time B needs A, A exists as a partially-initialized object (only constructor ran, setters not yet called). B gets the early reference, completes, and then A completes.

---

### 3.1.3 — Field Injection

#### Mechanics

Field injection directly sets a private (or any access level) field value via reflection, bypassing any setter method:

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;  // private field — Spring uses setAccessible(true)

    @Autowired
    private DataSource dataSource;

    @Value("${order.timeout:5000}")
    private int timeout;
}
```

Spring calls `Field.setAccessible(true)` and then `Field.set(beanInstance, resolvedValue)`. This bypasses Java's access control — deliberately.

#### Why Field Injection is Discouraged (Architectural Reasons)

**1. No immutability:** Cannot make fields `final` — `final` fields cannot be set by reflection after construction (without unsafe tricks).

**2. Hidden dependencies:** Looking at the class, you can't tell its dependencies without inspecting field annotations. The API (constructor/methods) doesn't reveal what the class needs.

**3. Untestable without container:** You cannot write `new OrderService()` and inject mocks — there's no setter or constructor to call. You must use Spring's test context or reflection-based test utilities (`ReflectionTestUtils.setField()`).

**4. Violates encapsulation principle:** The class appears to have no dependencies from its public API.

**5. Easy to abuse:** Field injection makes it trivially easy to add more dependencies (just add a field) — which hides class complexity and violates Single Responsibility Principle.

**6. Module system issues:** JDK 9+ module system can block `setAccessible(true)` on fields across module boundaries, requiring `--add-opens` JVM arguments.

Spring's official documentation explicitly recommends constructor injection over field injection for mandatory dependencies.

---

### 3.1.4 — Lookup Method Injection

This is an advanced and often misunderstood DI mechanism. It solves the **prototype-in-singleton problem**:

**The Problem:**
```java
@Component
@Scope("prototype")
public class ReportGenerator { }  // new instance needed each time

@Service
public class ReportService {
    @Autowired
    private ReportGenerator generator;
    // PROBLEM: generator is injected ONCE at startup
    // Every call to generateReport() uses the SAME ReportGenerator instance
    // The prototype is effectively a singleton from ReportService's perspective
}
```

**Lookup Method Injection — The Solution:**

Spring uses CGLIB to override the abstract or concrete lookup method. Every call to the method generates a new bean from the container:

```java
@Service
public abstract class ReportService {

    // Spring overrides this method via CGLIB
    // Returns a new ReportGenerator instance from the container each time
    @Lookup
    public abstract ReportGenerator createReportGenerator();

    public void generateReport(String type) {
        ReportGenerator gen = createReportGenerator(); // new instance every call
        gen.generate(type);
    }
}
```

#### Internal Mechanics of Lookup Method

1. Spring detects `@Lookup` via `AutowiredAnnotationBeanPostProcessor`
2. CGLIB subclasses the bean class
3. The CGLIB subclass overrides the `@Lookup` method
4. The generated override calls `beanFactory.getBean(ReportGenerator.class)` internally
5. For prototype-scoped target, this returns a new instance each time

In XML:
```xml
<bean id="reportService" class="com.example.ReportService">
    <lookup-method name="createReportGenerator" bean="reportGenerator"/>
</bean>

<bean id="reportGenerator" class="com.example.ReportGenerator"
      scope="prototype"/>
```

**Requirements for `@Lookup`:**
- The class containing `@Lookup` CANNOT be `final` (CGLIB subclassing required)
- The lookup method CANNOT be `final` or `private`
- The method must return the target bean type
- The method can be abstract or concrete (if concrete, the return is replaced)
- The class must be a Spring-managed bean

---

### 3.1.5 — Method Replacement (Arbitrary Method Injection)

This is the most advanced and rarely used DI mechanism. It replaces an existing method's implementation entirely with a different implementation at runtime, using CGLIB:

```java
public class OriginalService {
    public String computeValue(String input) {
        return "original: " + input;
    }
}
```

```java
public class ComputeValueReplacer implements MethodReplacer {
    @Override
    public Object reimplement(Object obj, Method method, Object[] args) {
        String input = (String) args[0];
        return "replaced: " + input.toUpperCase();
    }
}
```

```xml
<bean id="originalService" class="com.example.OriginalService">
    <replaced-method name="computeValue" replacer="computeValueReplacer">
        <arg-type>String</arg-type>
    </replaced-method>
</bean>

<bean id="computeValueReplacer" class="com.example.ComputeValueReplacer"/>
```

```java
OriginalService svc = ctx.getBean("originalService", OriginalService.class);
System.out.println(svc.computeValue("hello")); // "replaced: HELLO"
```

Method replacement has no annotation equivalent and is XML-only. It is almost never used in modern applications. However, it demonstrates Spring's capability to manipulate bean behavior at a bytecode level via CGLIB — the same mechanism underlies AOP.

---

### Choosing Between Injection Types — Architectural Decision

| | Constructor | Setter | Field |
|---|---|---|---|
| Immutability (`final`) | ✅ | ❌ | ❌ |
| Mandatory dependencies | ✅ clear | ❌ hidden | ❌ hidden |
| Optional dependencies | ❌ awkward | ✅ natural | ✅ easy |
| Circular dependency | ❌ fails fast | ✅ resolved | ✅ resolved |
| Testability | ✅ excellent | ✅ good | ❌ poor |
| Encapsulation | ✅ | ✅ | ❌ |
| Spring recommendation | ✅ preferred | For optional | ❌ avoid |
| Framework/legacy use | Common | Common | Common |

**Spring's official recommendation (current documentation):**
1. **Constructor injection** — for mandatory dependencies
2. **Setter injection** — for optional/reconfigurable dependencies
3. **Field injection** — discouraged, use sparingly if at all

---

### JVM and Reflection Deep Dive

**Constructor injection:**
```java
Constructor<?> ctor = MyService.class.getDeclaredConstructor(Dep.class);
ctor.setAccessible(true); // needed if non-public
Object instance = ctor.newInstance(resolvedDep);
```

**Setter injection:**
```java
Method setter = MyService.class.getDeclaredMethod("setDep", Dep.class);
setter.setAccessible(true);
setter.invoke(beanInstance, resolvedDep);
```

**Field injection:**
```java
Field field = MyService.class.getDeclaredField("dep");
field.setAccessible(true);
field.set(beanInstance, resolvedDep);
```

**Performance implications:**
- Reflection calls are ~10-50x slower than direct method calls
- Spring **caches** reflection metadata in `InjectionMetadata` objects per class
- `setAccessible(true)` results are cached (one-time cost per field/method)
- At steady state, the performance impact is negligible — reflection is only used during bean creation, not on every method call

---

### Common Misconceptions

**Misconception 1:** "`@Autowired` is required on constructors."
From Spring 4.3 onwards, if a class has a SINGLE constructor, `@Autowired` is not needed — Spring infers it automatically.

**Misconception 2:** "Field injection is faster than constructor injection."
Performance difference is negligible and only occurs at startup during bean creation. Runtime performance is identical — after injection, the field is set and accessed directly.

**Misconception 3:** "Setter injection is always optional."
Setter injection CAN be mandatory — `@Autowired` on a setter is required by default. Setter injection is optional only when `@Autowired(required=false)` is used.

**Misconception 4:** "Circular dependencies with `@Autowired` fields are always resolved."
Field injection CAN resolve circular dependencies (like setter injection) because the object is already instantiated before field injection. However, if BOTH circular beans use constructor injection, it always fails.

**Misconception 5:** "`@Lookup` requires the method to be abstract."
The method can be abstract OR concrete. If concrete, Spring's CGLIB override replaces it. If abstract, CGLIB provides the implementation.

**Misconception 6:** "You can make field-injected fields `final`."
Wrong. `final` fields must be set in the constructor. Reflection-based `field.set()` on a `final` field throws `IllegalAccessException` (or requires JVM unsafe tricks). Spring does not do this.

---

## ② CODE EXAMPLES

### Example 1 — Constructor Injection — All Variants

```java
// Variant 1: Single constructor, no @Autowired needed (Spring 4.3+)
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

```java
// Variant 2: Multiple constructors with @Autowired
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final AuditService auditService;

    @Autowired  // required — tells Spring which constructor to use
    public OrderService(PaymentService ps, AuditService as) {
        this.paymentService = ps;
        this.auditService = as;
    }

    public OrderService(PaymentService ps) {
        this(ps, null);
    }
}
```

```java
// Variant 3: Optional constructors with @Autowired(required=false)
@Service
public class OrderService {
    private PaymentService paymentService;
    private AuditService auditService;

    @Autowired(required = false)  // used if both available
    public OrderService(PaymentService ps, AuditService as) {
        this.paymentService = ps;
        this.auditService = as;
    }

    @Autowired(required = false)  // fallback if AuditService not available
    public OrderService(PaymentService ps) {
        this.paymentService = ps;
    }
}
```

```xml
<!-- XML constructor injection by index -->
<bean id="orderService" class="com.example.OrderService">
    <constructor-arg index="0" ref="paymentService"/>
    <constructor-arg index="1" value="5000"/>
</bean>

<!-- XML constructor injection by name -->
<bean id="orderService" class="com.example.OrderService">
    <constructor-arg name="paymentService" ref="paymentService"/>
    <constructor-arg name="timeout" value="5000"/>
</bean>
```

---

### Example 2 — Setter Injection — All Variants

```java
// Annotation-driven setter injection
@Service
public class NotificationService {

    private EmailService emailService;
    private SmsService smsService;   // optional

    @Autowired  // required by default
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }

    @Autowired(required = false)  // optional
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }

    // @Autowired on non-setter method — Spring injects all params
    @Autowired
    public void configure(DataSource ds,
                          @Qualifier("audit") AuditLogger logger) {
        // Both ds and logger are injected from container
    }
}
```

```xml
<!-- XML setter injection -->
<bean id="notificationService" class="com.example.NotificationService">
    <property name="emailService" ref="emailService"/>
    <!-- smsService omitted → field remains null -->
</bean>
```

---

### Example 3 — Field Injection (and why to avoid it)

```java
// Field injection — easy but problematic
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // cannot be final!

    @Autowired
    private DataSource dataSource;          // hidden dependency

    @Value("${timeout:5000}")
    private int timeout;
}

// Testing this WITHOUT Spring requires reflection:
@Test
public void test() throws Exception {
    OrderService service = new OrderService();  // can construct — no deps needed
    // But all fields are null! Must inject manually:
    ReflectionTestUtils.setField(service, "paymentService", mockPaymentService);
    ReflectionTestUtils.setField(service, "dataSource", mockDataSource);
    // Tedious and brittle
}

// Constructor injection testing — clean:
@Test
public void test() {
    OrderService service = new OrderService(mockPaymentService, mockDataSource, 5000);
    // Clean, no reflection needed
}
```

---

### Example 4 — Lookup Method Injection — Full Implementation

```java
// Prototype bean — new instance needed per use
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    private String sessionId;

    public void addItem(Item item) { items.add(item); }
    public List<Item> getItems() { return Collections.unmodifiableList(items); }
    public void setSessionId(String id) { this.sessionId = id; }
}
```

```java
// Singleton service that needs a new ShoppingCart per request
@Service
public abstract class CartService {

    // Spring CGLIB overrides this to call beanFactory.getBean(ShoppingCart.class)
    @Lookup
    public abstract ShoppingCart getCart();

    // Typed lookup — Spring infers bean type from return type
    @Lookup("shoppingCart")  // explicit bean name (optional if type is unambiguous)
    public abstract ShoppingCart getNamedCart();

    public void processSession(String sessionId) {
        ShoppingCart cart = getCart();  // NEW instance each call
        cart.setSessionId(sessionId);
        // ... process
    }

    public void demonstrateNewInstances() {
        ShoppingCart cart1 = getCart();
        ShoppingCart cart2 = getCart();
        System.out.println(cart1 == cart2);  // false — different instances
        System.out.println(cart1.getClass() == cart2.getClass()); // true — same type
    }
}
```

```xml
<!-- XML equivalent -->
<bean id="cartService" class="com.example.CartService">
    <lookup-method name="getCart" bean="shoppingCart"/>
</bean>

<bean id="shoppingCart" class="com.example.ShoppingCart" scope="prototype"/>
```

---

### Example 5 — Alternative Solutions to Prototype-in-Singleton Problem

```java
// Alternative 1: Inject ObjectFactory<T>
@Service
public class CartService {
    @Autowired
    private ObjectFactory<ShoppingCart> cartFactory;

    public void processSession(String sessionId) {
        ShoppingCart cart = cartFactory.getObject(); // new instance each call
        cart.setSessionId(sessionId);
    }
}
```

```java
// Alternative 2: Inject ObjectProvider<T> (Spring 4.3+ — preferred)
@Service
public class CartService {
    @Autowired
    private ObjectProvider<ShoppingCart> cartProvider;

    public void processSession(String sessionId) {
        ShoppingCart cart = cartProvider.getObject(); // new instance each call
        // Also supports: cartProvider.getIfAvailable(), cartProvider.stream()
    }
}
```

```java
// Alternative 3: Inject ApplicationContext (least elegant, most explicit)
@Service
public class CartService implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void processSession(String sessionId) {
        ShoppingCart cart = ctx.getBean(ShoppingCart.class); // new prototype instance
        cart.setSessionId(sessionId);
    }
}
```

```java
// Alternative 4: JSR-330 Provider<T>
@Service
public class CartService {
    @Autowired
    private Provider<ShoppingCart> cartProvider; // javax.inject.Provider

    public void processSession(String sessionId) {
        ShoppingCart cart = cartProvider.get(); // new instance each call
    }
}
```

**Comparison of Alternatives:**

| Approach | Spring-specific | Type-safe | Clean |
|---|---|---|---|
| `@Lookup` | Yes | ✅ | ✅ |
| `ObjectFactory<T>` | Yes | ✅ | ✅ |
| `ObjectProvider<T>` | Yes | ✅ | ✅✅ |
| `ApplicationContext.getBean()` | Yes | Partial | ❌ |
| `Provider<T>` (JSR-330) | No | ✅ | ✅ |

---

### Example 6 — Constructor vs Setter — Circular Dependency Contrast

```java
// CONSTRUCTOR circular dependency — FAILS
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { }  // needs B to be created
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { }  // needs A to be created
    // Deadlock — neither can be created first
    // Throws: BeanCurrentlyInCreationException
}
```

```java
// SETTER circular dependency — WORKS via three-level cache
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // injected after A is instantiated
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // gets early reference to partially-initialized A
}
// Both created successfully — but A is partially initialized when B gets it
// This is technically unsafe if B calls methods on A during its own initialization
```

---

### Example 7 — `@Autowired` on Multi-param Method

```java
@Service
public class DataProcessor {

    private DataSource dataSource;
    private TransactionManager txManager;
    private CacheManager cacheManager;

    // Spring injects ALL parameters from the container
    @Autowired
    public void initialize(DataSource ds,
                           TransactionManager txManager,
                           @Qualifier("localCache") CacheManager cacheManager) {
        this.dataSource = ds;
        this.txManager = txManager;
        this.cacheManager = cacheManager;
    }
}
```

This is valid — `@Autowired` on a method (not just setters). Spring treats each parameter as an injectable dependency. The method name does NOT need to start with "set".

---

### Example 8 — Edge Case: `@Autowired` with `Optional<T>` and `@Nullable`

```java
@Service
public class OrderService {

    // Option 1: Optional<T> — clean null handling
    @Autowired
    private Optional<AuditService> auditService;

    // Option 2: @Nullable — explicit null allowed
    @Autowired
    @Nullable
    private MetricsService metricsService;

    // Option 3: required=false — field remains null if bean not found
    @Autowired(required = false)
    private NotificationService notificationService;

    // Option 4: ObjectProvider — lazy + safe + stream support
    @Autowired
    private ObjectProvider<AnalyticsService> analyticsProvider;

    public void processOrder(Order order) {
        // Option 1 usage:
        auditService.ifPresent(audit -> audit.log(order));

        // Option 2 usage:
        if (metricsService != null) metricsService.record(order);

        // Option 3 usage:
        if (notificationService != null) notificationService.notify(order);

        // Option 4 usage:
        analyticsProvider.ifAvailable(analytics -> analytics.track(order));
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which injection type allows declaring dependencies as `final` fields?

A) Field injection
B) Setter injection
C) Constructor injection
D) Lookup method injection

**Answer: C**
Only constructor injection allows `final` fields because `final` must be set in the constructor. Reflection-based field/setter injection cannot set `final` fields.

---

**Q2 — MCQ:**
A class has two constructors: one with `PaymentService` parameter and one no-arg constructor. No `@Autowired` is present. Which constructor does Spring use?

A) The `PaymentService` constructor — more specific
B) The no-arg constructor — default
C) Throws `BeanCreationException` — ambiguous
D) Spring uses both — tries parameterized first, falls back to no-arg

**Answer: B**
When multiple constructors exist and NONE has `@Autowired`, Spring uses the **no-arg (default) constructor**. This is a critical rule. If no no-arg constructor exists in this scenario, Spring throws.

---

**Q3 — Select all that apply:**
Which of the following correctly describe constructor injection? (Select ALL that apply)

A) Dependencies can be declared `final`
B) Circular dependencies between two constructor-injected beans are resolved via three-level cache
C) From Spring 4.3+, `@Autowired` is optional when there is exactly one constructor
D) Constructor injection makes dependencies visible in the class API
E) Constructor-injected beans always have their dependencies fully initialized before use

**Answer: A, C, D, E**
B is wrong — constructor injection circular dependencies CANNOT be resolved. The three-level cache only applies to setter/field injection. Constructor circular dependency always throws `BeanCurrentlyInCreationException`.

---

**Q4 — Code Output Prediction:**

```java
@Service
public class ServiceA {
    public ServiceA(ServiceB b) {
        System.out.println("ServiceA created with: " + b);
    }
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) {
        System.out.println("ServiceB created with: " + a);
    }
}

// Both are @Service, detected via @ComponentScan
ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
```

What happens?

A) Both beans created successfully, output shows creation messages
B) `BeanCurrentlyInCreationException` thrown during `refresh()`
C) Beans created with null dependencies — circular reference detected and nulled out
D) Spring resolves it via three-level cache — both created successfully

**Answer: B**
Constructor injection circular dependencies ALWAYS fail. Spring cannot create A without B, and cannot create B without A. There's no way to provide a partially-initialized instance because the constructor needs to complete to produce an instance.

---

**Q5 — Trick Question:**

```java
@Component
@Scope("prototype")
public class Task { }

@Service
public class TaskManager {
    @Autowired
    private Task task;  // field injection

    public Task getTask() { return task; }
}
```

How many `Task` instances are created?

A) One per `TaskManager` request — prototype creates new instance each time
B) Exactly one — field injection happens once at container startup
C) Zero — `@Autowired` on prototype beans is not supported
D) One per `getTask()` call

**Answer: B**
The `task` field is set ONCE during `TaskManager` bean initialization (which happens once for a singleton). Despite `Task` being prototype-scoped, the field injection only happens at initialization time. Every call to `getTask()` returns the SAME `Task` instance. This is the **prototype-in-singleton problem** — the root cause for why `@Lookup` and `ObjectProvider` exist.

---

**Q6 — Select all that apply:**
Which are valid solutions to the prototype-in-singleton problem? (Select ALL that apply)

A) `@Lookup` method injection
B) Injecting `ObjectFactory<T>`
C) Injecting `ObjectProvider<T>`
D) Injecting `Provider<T>` (JSR-330)
E) Injecting `ApplicationContext` and calling `getBean()`
F) Changing the singleton to prototype scope

**Answer: A, B, C, D, E**
F technically "solves" the problem by eliminating the singleton, but it changes the design semantics and is not a solution to prototype-in-singleton — it's an elimination of the scenario.

---

**Q7 — Code Output Prediction:**

```java
@Configuration
public class AppConfig {

    @Bean
    @Scope("prototype")
    public ReportGenerator reportGenerator() {
        return new ReportGenerator();
    }

    @Bean
    public abstract ReportService reportService();
}

@Service
public abstract class ReportService {
    @Lookup
    public abstract ReportGenerator reportGenerator();

    public void runReport() {
        ReportGenerator g1 = reportGenerator();
        ReportGenerator g2 = reportGenerator();
        System.out.println(g1 == g2);
    }
}

ctx.getBean(ReportService.class).runReport();
```

What is printed?

A) `true`
B) `false`
C) `NullPointerException`
D) `BeanCreationException`

**Answer: B**
`@Lookup` causes Spring to generate a CGLIB override of `reportGenerator()` that calls `beanFactory.getBean(ReportGenerator.class)` each time. Since `ReportGenerator` is prototype-scoped, each call returns a new instance. `g1 != g2` → `false`.

---

**Q8 — Scenario: Multiple `@Autowired(required=false)` Constructors**

```java
@Service
public class NotificationService {

    private EmailSender emailSender;
    private SmsSender smsSender;

    @Autowired(required = false)
    public NotificationService(EmailSender e, SmsSender s) {
        this.emailSender = e;
        this.smsSender = s;
    }

    @Autowired(required = false)
    public NotificationService(EmailSender e) {
        this.emailSender = e;
    }
}
```

The container has `EmailSender` but NOT `SmsSender`. Which constructor is used?

A) The two-arg constructor — preferred as it has more parameters
B) The one-arg constructor — only one where all params can be satisfied
C) `NoSuchBeanDefinitionException` — SmsSender not found
D) No-arg constructor is used — Spring falls back

**Answer: B**
With `required=false` on multiple constructors, Spring tries to find the constructor that can be FULLY satisfied. The two-arg constructor requires `SmsSender` which isn't available. The one-arg constructor can be fully satisfied with `EmailSender`. Spring uses the maximally satisfiable constructor.

---

**Q9 — Compilation/Runtime Error:**

```java
@Service
public class OrderService {
    @Autowired
    private final PaymentService paymentService;  // final field, field injection
}
```

What happens?

A) Compiles and works — Spring handles `final` fields via reflection
B) Compile error — `final` fields must be initialized in the declaration or constructor
C) Compile error — `@Autowired` cannot be used on `final` fields
D) Runtime exception — Spring cannot inject into `final` fields

**Answer: B**
`final` fields must be initialized in their declaration or in the constructor. A `final` field without an initializer at the declaration site, and without being set in a constructor, is a **compile error** in Java. This has nothing to do with Spring — it's a Java language rule.

---

**Q10 — Drag-and-Drop: Order the steps when Spring creates a bean with setter injection:**

- BeanPostProcessor.postProcessAfterInitialization()
- Constructor called — bean instance created
- @PostConstruct / init-method called
- Dependencies injected via setter methods
- BeanPostProcessor.postProcessBeforeInitialization()
- Aware callbacks (BeanNameAware, ApplicationContextAware, etc.)
- Bean stored in singleton cache

**Answer:**
1. Constructor called — bean instance created
2. Dependencies injected via setter methods
3. Aware callbacks (BeanNameAware, ApplicationContextAware, etc.)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct / init-method called
6. BeanPostProcessor.postProcessAfterInitialization()
7. Bean stored in singleton cache

---

## ④ TRICK ANALYSIS

**Trap 1: Prototype-in-singleton with field injection**
The most common trap in DI topics. A prototype-scoped bean injected via `@Autowired` field into a singleton behaves as a singleton — injected once at startup, same instance forever. The solution is `@Lookup`, `ObjectProvider`, `ObjectFactory`, or `Provider`. Any question showing `@Autowired` field injection of a prototype into a singleton and asking "how many instances are created?" → ONE.

**Trap 2: Constructor selection with no `@Autowired`**
Multiple constructors + no `@Autowired` → Spring uses **no-arg constructor**. If no no-arg constructor → `BeanCreationException`. This surprises people who expect Spring to pick the most specific constructor. It only does that with `@Autowired`.

**Trap 3: Constructor circular dependency**
Constructor injection circular dependencies ALWAYS throw `BeanCurrentlyInCreationException`. The three-level cache ONLY helps with setter/field injection. Any question showing two beans with constructor injection of each other → exception.

**Trap 4: `final` field + `@Autowired`**
Cannot have `@Autowired` on a `final` field because `final` fields without initialization in declaration or constructor cause a compile error. This is a Java language constraint, not a Spring constraint. The question is answered at the compilation step.

**Trap 5: `@Lookup` method can be concrete**
Many assume `@Lookup` methods must be abstract. They can be concrete — Spring's CGLIB override replaces the method body regardless. The return value of a concrete `@Lookup` method is ignored at runtime.

**Trap 6: Setter injection is NOT always optional**
`@Autowired` on a setter is REQUIRED by default. "Setter injection = optional" is a common misconception. It is only optional when `@Autowired(required=false)` is explicitly set.

**Trap 7: `@Autowired` on multi-param method**
`@Autowired` is not restricted to single-param setter methods. It works on ANY method with ANY number of parameters. Each parameter is resolved from the container independently. This is a legitimate (if unusual) injection pattern.

**Trap 8: Field injection in JDK 9+ modules**
`Field.setAccessible(true)` across module boundaries requires `--add-opens` JVM flags. Field injection in modular JARs can fail at runtime without this flag. Constructor and setter injection don't have this problem because they use public APIs.

**Trap 9: Early reference in circular setter injection**
When Bean B receives an early reference to Bean A (via the three-level cache), Bean A's setters have NOT yet been called — only its constructor has run. If B's constructor or init method calls a method on A that depends on A's setter-injected fields, those fields will be null. This is a subtle runtime bug that can survive testing if the problematic code path isn't exercised during initialization.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Constructor injection allows `final` | ✅ Only injection type supporting `final` fields |
| Single constructor, no `@Autowired` | Spring 4.3+ — inferred automatically |
| Multiple constructors, no `@Autowired` | No-arg constructor used |
| Multiple constructors, no `@Autowired`, no no-arg | `BeanCreationException` |
| Constructor circular dependency | Always `BeanCurrentlyInCreationException` — no resolution |
| Setter/field circular dependency | Resolved via three-level cache |
| Prototype in singleton via `@Autowired` | PROBLEM — same instance always returned |
| Solutions to prototype-in-singleton | `@Lookup`, `ObjectFactory`, `ObjectProvider`, `Provider`, `ctx.getBean()` |
| `@Autowired` on setter | Required by default (`required=true`) |
| `@Lookup` method | Can be abstract OR concrete; class/method cannot be `final` |
| Field injection `final` | Compile error — `final` without initializer is Java compile error |

---

### Injection Type Comparison

```
Constructor Injection:
  ✅ Immutable (final)          ✅ Mandatory declared      ✅ Testable
  ✅ Fail-fast                  ❌ Circular dependency      ✅ Spring recommended

Setter Injection:
  ❌ No final                   ✅ Optional possible       ✅ Testable
  ✅ Circular dependency OK     ⚠️  Required by default    ✅ For optional deps

Field Injection:
  ❌ No final                   ❌ Hidden deps             ❌ Poor testability
  ✅ Circular dependency OK     ❌ Not recommended          ❌ Avoid in new code

Lookup Method Injection:
  ✅ Solves prototype-in-singleton problem
  ⚠️  Requires CGLIB (class/method cannot be final)
  ✅ Clean API — abstract method

ObjectProvider<T>:
  ✅ Solves prototype-in-singleton
  ✅ Optional dependency support
  ✅ Lazy resolution
  ✅ Stream support
  ✅ Spring recommended modern approach
```

---

### Three-Level Cache (Setter/Field Circular Dependency Resolution)

```
Level 1: singletonObjects        → fully initialized beans
Level 2: earlySingletonObjects   → early references (post-ObjectFactory, pre-init)
Level 3: singletonFactories      → ObjectFactory lambdas for early exposure

Flow for A→B→A circular (setter injection):
1. Create A: instantiate → register in Level 3
2. A needs B: create B → instantiate → register in Level 3
3. B needs A: check Level 1 (no) → Level 2 (no) → Level 3 (yes!)
4. Call A's ObjectFactory → get early A ref → move to Level 2
5. Inject early A into B → B completes → move to Level 1
6. Inject B into A → A completes → move to Level 1
```

---

### Prototype-in-Singleton Solutions

```
Problem: Singleton S has @Autowired Task task (prototype)
         → task injected ONCE → always same instance

Solutions (ordered by elegance):
1. ObjectProvider<Task>      @Autowired ObjectProvider<Task> p; p.getObject()
2. @Lookup                   @Lookup public abstract Task createTask()
3. ObjectFactory<Task>       @Autowired ObjectFactory<Task> f; f.getObject()
4. Provider<Task> (JSR-330)  @Inject Provider<Task> p; p.get()
5. ApplicationContext        ctx.getBean(Task.class)  ← least elegant
```

---

### Interview One-Liners

- "Constructor injection is Spring's recommended approach because it enables immutability, makes dependencies explicit, and provides fail-fast behavior for missing dependencies."
- "Circular dependencies with constructor injection are impossible — Spring throws `BeanCurrentlyInCreationException`. Use setter injection or redesign."
- "The prototype-in-singleton problem occurs when a prototype bean is `@Autowired` into a singleton — the prototype is injected once and becomes effectively a singleton. Solve with `@Lookup` or `ObjectProvider`."
- "Field injection uses `setAccessible(true)` — it bypasses Java's access control, hides dependencies, prevents `final` use, and makes unit testing without Spring painful."
- "From Spring 4.3, `@Autowired` is implicit for classes with a single constructor."
- "Multiple constructors without `@Autowired` → Spring uses the no-arg constructor."
- "The three-level singleton cache allows setter/field injection to resolve circular dependencies by providing early object references before initialization completes."
- "`@Lookup` uses CGLIB to override the annotated method, making it call `beanFactory.getBean()` internally — the class and method cannot be `final`."

---
