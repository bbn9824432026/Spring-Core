# TOPIC 3.4 — Dependency Resolution Order

---

## ① CONCEPTUAL EXPLANATION

### What is Dependency Resolution Order?

Dependency resolution order refers to the sequence in which Spring instantiates, wires, and initializes beans. In a production application with hundreds or thousands of beans, the order matters because:

1. A bean must be fully initialized before it can be injected into another bean
2. Some beans have side effects during initialization (database connections, cache warming, event registration)
3. Certain beans are infrastructure (BFPPs, BPPs) that MUST exist before ordinary beans
4. Application correctness can depend on initialization order in subtle ways

Spring determines instantiation order through a combination of:

```
Explicit ordering mechanisms:
    1. @DependsOn              → explicit dependency declaration
    2. @Order / Ordered        → for collections and BPP ordering
    3. @Priority               → JSR-250 variant of @Order
    4. PriorityOrdered         → highest priority ordering interface
    5. Lazy initialization     → deferred until first use

Implicit ordering:
    1. Dependency graph        → if A autowires B, B is created before A
    2. BeanDefinition registration order → baseline for ties
    3. Infrastructure-first    → BFPPs, BPPs always before ordinary beans
```

---

### How Spring Determines Basic Instantiation Order

During `finishBeanFactoryInitialization()`, Spring calls `preInstantiateSingletons()` which iterates `beanDefinitionNames` — an ordered list maintained in registration order. For each bean name:

```
for each beanName in beanDefinitionNames (registration order):
    if not abstract AND singleton AND not lazy:
        if FactoryBean:
            instantiate FactoryBean
            if SmartFactoryBean.isEagerInit(): call getObject()
        else:
            getBean(beanName)
                → dependency graph traversal
                → dependencies created first (recursive getBean)
```

The **dependency graph traversal** is what actually drives the order. When Spring creates bean A and A depends on B and C, Spring creates B and C first — regardless of where they appear in `beanDefinitionNames`. Registration order is just the baseline; actual order is determined by the dependency graph.

---

### 3.4.1 — Bean Instantiation Order — Complete Rules

#### Rule 1: Dependencies are always created before dependents

```java
@Service
public class OrderService {
    @Autowired PaymentService paymentService; // PaymentService created first
    @Autowired InventoryService inventoryService; // InventoryService created first
}
```

Spring cannot create `OrderService` until both `PaymentService` and `InventoryService` exist. The container recursively resolves the entire dependency tree before creating `OrderService`.

#### Rule 2: Registration order for independent beans

When two beans have NO dependency relationship between them, they are created in `beanDefinitionNames` registration order. For annotation-based config, this is roughly:
- Order of `@Bean` methods within a `@Configuration` class (top to bottom)
- Order of `@ComponentScan` discovery (alphabetical within package by default)
- Order of `@Import` processing (imported configs processed depth-first)

```java
@Configuration
public class AppConfig {
    @Bean public ServiceA serviceA() { return new ServiceA(); } // created first
    @Bean public ServiceB serviceB() { return new ServiceB(); } // created second
    // ServiceA and ServiceB have no dependency between them
}
```

#### Rule 3: Infrastructure beans always first

BFPPs are created in `invokeBeanFactoryPostProcessors()` (step 5 of `refresh()`), long before `finishBeanFactoryInitialization()` (step 11). BPPs are created in `registerBeanPostProcessors()` (step 6). Both happen before any ordinary singleton beans.

#### Rule 4: Singleton before prototype in initialization pass

The `preInstantiateSingletons()` pass only pre-instantiates SINGLETON beans. Prototype beans are not created in this pass — they are created lazily on demand.

#### Rule 5: `@Lazy` beans defer to first `getBean()` call

Beans annotated with `@Lazy` or `@Bean(lazyInit=true)` are skipped during `preInstantiateSingletons()` and created only when first requested.

---

### 3.4.2 — `@DependsOn` — Explicit Ordering

`@DependsOn` forces Spring to initialize specified beans BEFORE the annotated bean, even if there is no direct autowiring relationship between them.

```java
@Service
@DependsOn({"databaseInitializer", "cacheWarmupService"})
public class OrderService {
    // OrderService will be created only AFTER databaseInitializer
    // and cacheWarmupService are fully initialized
    // Even though OrderService does NOT autowire them
}
```

#### When to Use `@DependsOn`

`@DependsOn` is for **non-obvious side-effect dependencies** — situations where bean B must run some initialization code before bean A can function, but A doesn't directly autowire B:

```java
@Component
public class SchemaInitializer {
    @PostConstruct
    public void createSchema() {
        // Creates database tables
        jdbc.execute("CREATE TABLE IF NOT EXISTS orders (...)");
    }
}

@Service
@DependsOn("schemaInitializer") // Schema must exist before OrderRepository initializes
public class OrderRepository {
    @PostConstruct
    public void init() {
        // Assumes schema exists
        long count = jdbc.queryForObject("SELECT COUNT(*) FROM orders", Long.class);
    }
}
```

Without `@DependsOn`, if `OrderRepository` happened to be created before `SchemaInitializer`, the `@PostConstruct` would fail.

#### `@DependsOn` in XML

```xml
<bean id="orderService" class="com.example.OrderService"
      depends-on="databaseInitializer,cacheWarmupService"/>
```

#### `@DependsOn` on `@Bean` Methods

```java
@Configuration
public class AppConfig {

    @Bean
    public DatabaseInitializer databaseInitializer() {
        return new DatabaseInitializer();
    }

    @Bean
    @DependsOn("databaseInitializer")
    public OrderRepository orderRepository(DataSource ds) {
        return new OrderRepository(ds);
    }
}
```

#### `@DependsOn` Does NOT Resolve Circular Dependencies

```java
@Service
@DependsOn("serviceB") // Force B before A
public class ServiceA { }

@Service
@DependsOn("serviceA") // Force A before B — CIRCULAR!
public class ServiceB { }
```

This throws `BeanCreationException` — Spring detects the circular `@DependsOn` chain. `@DependsOn` is for ordering, not for dependency injection resolution.

#### `@DependsOn` and Destruction Order

`@DependsOn` also affects DESTRUCTION order. If A `@DependsOn` B, then:
- A is initialized AFTER B
- A is destroyed BEFORE B

This ensures that during shutdown, B is still alive while A is being destroyed (A might need B during cleanup).

---

### 3.4.3 — `@Lazy` — Deferred Initialization

`@Lazy` can be applied at multiple levels with different effects:

#### `@Lazy` on `@Component` / `@Service` / etc.

```java
@Service
@Lazy // Not created at startup — created on first getBean() or @Autowired injection
public class HeavyReportingService {
    @PostConstruct
    public void init() {
        // Expensive initialization — load ML models, warm connections, etc.
        System.out.println("HeavyReportingService initialized");
    }
}
```

Bean is created only when first needed — either:
- First explicit `ctx.getBean(HeavyReportingService.class)`
- First `@Autowired` injection point that resolves this bean is triggered

#### `@Lazy` on `@Bean` Method

```java
@Configuration
public class AppConfig {
    @Bean
    @Lazy
    public ExpensiveService expensiveService() {
        return new ExpensiveService(); // Only created when first requested
    }
}
```

#### `@Lazy` on Injection Point (Different Semantics!)

When `@Lazy` is at the injection point (not the bean definition), it creates a CGLIB proxy regardless of whether the target bean itself is `@Lazy`:

```java
@Service
public class OrderService {

    @Autowired @Lazy
    private ReportingService reportingService;
    // A CGLIB proxy for ReportingService is injected immediately
    // The REAL ReportingService is created on first method call on the proxy
    // ReportingService bean definition does NOT need to be @Lazy
}
```

This is useful for:
1. Breaking circular dependencies
2. Deferring initialization of expensive dependencies until actually needed
3. Preventing eager initialization during startup

#### `@Lazy` on `@Configuration` class

```java
@Configuration
@Lazy // ALL @Bean methods in this class become lazy by default
public class LazyConfig {

    @Bean // lazy by default (inherited from class-level @Lazy)
    public ServiceA serviceA() { return new ServiceA(); }

    @Bean @Lazy(false) // override — NOT lazy
    public ServiceB serviceB() { return new ServiceB(); }
}
```

#### Default Lazy Initialization in Spring Boot

```properties
# Make ALL beans lazy by default (Spring Boot 2.2+)
spring.main.lazy-initialization=true
```

This defers ALL bean creation to first use. Pros: faster startup, lower initial memory. Cons: first requests are slower, configuration errors surface at runtime (not startup).

---

### 3.4.4 — FactoryBean and Dependency Resolution

`FactoryBean` is a special interface that creates another object. It introduces a level of indirection in the dependency graph:

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;           // creates the product
    Class<?> getObjectType();                 // type of the product
    default boolean isSingleton() { return true; } // is product a singleton?
}
```

When Spring creates a bean defined by a `FactoryBean`:
1. The `FactoryBean` itself is instantiated as a regular bean
2. `getObject()` is called to produce the actual bean
3. The product is what gets registered under the original bean name
4. The `FactoryBean` itself is accessible via the `&` prefix

```java
@Component
public class ConnectionPoolFactoryBean implements FactoryBean<ConnectionPool> {

    @Value("${db.url}")
    private String url;

    @Override
    public ConnectionPool getObject() throws Exception {
        ConnectionPool pool = new ConnectionPool();
        pool.setUrl(url);
        pool.initialize(); // complex initialization
        return pool;
    }

    @Override
    public Class<?> getObjectType() {
        return ConnectionPool.class;
    }

    @Override
    public boolean isSingleton() { return true; }
}
```

```java
// getBean("connectionPoolFactoryBean") → returns ConnectionPool (the product)
// getBean("&connectionPoolFactoryBean") → returns ConnectionPoolFactoryBean
ConnectionPool pool = ctx.getBean("connectionPoolFactoryBean", ConnectionPool.class);
ConnectionPoolFactoryBean factory = ctx.getBean("&connectionPoolFactoryBean", ConnectionPoolFactoryBean.class);
```

#### FactoryBean Impact on Dependency Resolution

FactoryBean's `getObjectType()` is called during type-based dependency resolution (`getBeanNamesForType()`). If `getObjectType()` returns `null` (the type is unknown until `getObject()` is called), Spring may need to actually call `getObject()` to determine the type — which can cause the FactoryBean to be instantiated earlier than expected.

```java
// If getObjectType() returns null:
@Override
public Class<?> getObjectType() {
    return null; // BAD — forces early instantiation for type resolution
}

// Should always return a non-null type if possible:
@Override
public Class<?> getObjectType() {
    return ConnectionPool.class; // GOOD — type known without instantiation
}
```

#### SmartFactoryBean

`SmartFactoryBean` extends `FactoryBean` with:

```java
public interface SmartFactoryBean<T> extends FactoryBean<T> {
    default boolean isPrototype() { return false; }
    default boolean isEagerInit() { return false; } // force eager getObject() call
}
```

If `isEagerInit()` returns `true`, Spring calls `getObject()` during `preInstantiateSingletons()`, ensuring the product is created eagerly even though `FactoryBean` products are normally lazy (getObject() called on demand).

---

### Ordering Infrastructure Beans — `PriorityOrdered`, `Ordered`, `@Order`

When multiple BFPPs or BPPs exist, their execution order is determined by:

```
Priority 1: Implement PriorityOrdered → processed first (even before other Ordered)
Priority 2: Implement Ordered → processed in order value order
Priority 3: No ordering interface → processed in registration order
```

```java
// BeanFactoryPostProcessor ordering:
public class FirstBFPP implements BeanFactoryPostProcessor, PriorityOrdered {
    @Override public int getOrder() { return Ordered.HIGHEST_PRECEDENCE; } // runs first
    @Override public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) { }
}

public class SecondBFPP implements BeanFactoryPostProcessor, Ordered {
    @Override public int getOrder() { return 10; }
    @Override public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) { }
}

public class ThirdBFPP implements BeanFactoryPostProcessor {
    // No ordering — runs after all Ordered BFPPs
    @Override public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) { }
}
```

#### `@Order` for Regular Beans in Collections

`@Order` on regular beans (not BFPPs/BPPs) does NOT affect their instantiation order. It only affects:
- Ordering within `List<T>` and `T[]` injection
- Ordering of event listener execution

```java
@Component @Order(1) public class FirstHandler implements EventHandler { }
@Component @Order(2) public class SecondHandler implements EventHandler { }

@Service
public class EventProcessor {
    @Autowired
    List<EventHandler> handlers;
    // [FirstHandler, SecondHandler] — ordered by @Order value
    // But FirstHandler and SecondHandler may have been CREATED in any order
}
```

`@Order` does NOT guarantee creation order for regular singleton beans — only ordering within collections.

---

### Complete Initialization Order — The Full Picture

```
ApplicationContext.refresh()
│
├── Step 5: invokeBeanFactoryPostProcessors()
│       Order within BFPPs:
│       1. BeanDefinitionRegistryPostProcessors (implements PriorityOrdered)
│          → ConfigurationClassPostProcessor (highest priority)
│       2. BeanDefinitionRegistryPostProcessors (implements Ordered)
│       3. BeanDefinitionRegistryPostProcessors (no ordering)
│       4. BeanFactoryPostProcessors (implements PriorityOrdered)
│          → PropertySourcesPlaceholderConfigurer
│       5. BeanFactoryPostProcessors (implements Ordered)
│       6. BeanFactoryPostProcessors (no ordering)
│
├── Step 6: registerBeanPostProcessors()
│       Order within BPPs:
│       1. BPPs implementing PriorityOrdered
│          → AutowiredAnnotationBeanPostProcessor
│          → CommonAnnotationBeanPostProcessor
│       2. BPPs implementing Ordered
│       3. BPPs with no ordering
│       4. MergedBeanDefinitionPostProcessors (always last)
│
├── Step 11: finishBeanFactoryInitialization()
│       → preInstantiateSingletons()
│           For each beanName (registration order, depth-first dependency resolution):
│               Per bean:
│               a. createBeanInstance()        ← constructor
│               b. populateBean()              ← DI
│               c. Aware callbacks
│               d. BPP.postProcessBeforeInitialization()
│               e. @PostConstruct / afterPropertiesSet() / init-method
│               f. BPP.postProcessAfterInitialization()  ← AOP proxy may be created here
│           After all singletons:
│               g. SmartInitializingSingleton.afterSingletonsInstantiated()
│
└── Step 12: finishRefresh()
        → LifecycleProcessor.onRefresh()
            → SmartLifecycle beans with autoStartup=true, ordered by phase
```

---

### `SmartLifecycle` and Phase Ordering

`SmartLifecycle` extends `Lifecycle` with phase-based ordering:

```java
@Component
public class DataSourceLifecycle implements SmartLifecycle {

    private boolean running = false;

    @Override
    public int getPhase() {
        return Integer.MIN_VALUE; // start first (lowest phase = first to start)
    }

    @Override
    public boolean isAutoStartup() { return true; }

    @Override
    public void start() {
        // Start connection pool
        running = true;
    }

    @Override
    public void stop() {
        // Close connections
        running = false;
    }

    @Override
    public boolean isRunning() { return running; }
}
```

**Phase ordering rules:**
- Lower phase value → started FIRST, stopped LAST
- Higher phase value → started LAST, stopped FIRST
- Default phase = `Integer.MAX_VALUE / 2` (0 in some implementations)

```
Startup:  Phase MIN_VALUE → Phase -1 → Phase 0 → Phase MAX_VALUE
Shutdown: Phase MAX_VALUE → Phase 0 → Phase -1 → Phase MIN_VALUE
```

This is the mechanism used by Spring's embedded web server (Tomcat/Netty) in Spring Boot — it starts in a specific phase relative to other `SmartLifecycle` beans.

---

### `@PostConstruct` Order Across Beans

`@PostConstruct` runs PER BEAN during that bean's initialization. It does NOT have a cross-bean ordering guarantee (unless `@DependsOn` is used). If bean A `@DependsOn` bean B, then B's `@PostConstruct` runs before A's `@PostConstruct`.

```java
@Component
public class BeanB {
    @PostConstruct
    public void initB() { System.out.println("B initialized"); }
}

@Component
@DependsOn("beanB")
public class BeanA {
    @PostConstruct
    public void initA() { System.out.println("A initialized"); }
}
```

Output:
```
B initialized
A initialized
```

Without `@DependsOn`, the order of `@PostConstruct` calls across unrelated beans is determined by the dependency graph and registration order — not guaranteed to be alphabetical or definitionally ordered.

---

### Common Misconceptions

**Misconception 1:** "`@Order` controls bean creation order."
Wrong. `@Order` on regular `@Component` beans controls ordering WITHIN collections (`List<T>`, `T[]`). It does NOT control when the bean is instantiated.

**Misconception 2:** "`@DependsOn` resolves circular dependencies."
Wrong. `@DependsOn` specifies ordering, not dependency injection. Circular `@DependsOn` chains throw `BeanCreationException`. It cannot resolve injection-level circular dependencies.

**Misconception 3:** "Beans are always created in alphabetical order."
Wrong. The order is: dependency graph first, then registration order for independent beans.

**Misconception 4:** "FactoryBean products are always eagerly initialized."
Wrong. By default, FactoryBean products are lazily initialized — `getObject()` is called on first request. Only if `SmartFactoryBean.isEagerInit()` returns `true` are they eagerly initialized during `preInstantiateSingletons()`.

**Misconception 5:** "`@Lazy` on a bean definition means the bean is NEVER created at startup."
Partially wrong. `@Lazy` on a bean definition means it's skipped during `preInstantiateSingletons()`. BUT if another non-lazy bean has an `@Autowired` dependency on the lazy bean, the lazy bean IS created at startup (as a dependency of the eager bean). The lazy designation only prevents proactive creation — it doesn't prevent reactive creation as a dependency.

**Misconception 6:** "SmartLifecycle.start() is called during `refresh()`."
True — but only in `finishRefresh()` (the last step of `refresh()`). The beans themselves are fully initialized before `start()` is called.

---

## ② CODE EXAMPLES

### Example 1 — `@DependsOn` for Side-Effect Dependencies

```java
@Component
public class LiquibaseMigrationRunner {

    @Autowired DataSource dataSource;

    @PostConstruct
    public void runMigrations() {
        System.out.println("Running Liquibase migrations...");
        // Create/update schema
        Liquibase liquibase = new Liquibase("db/changelog.xml",
            new ClassLoaderResourceAccessor(), new JdbcConnection(dataSource.getConnection()));
        liquibase.update("");
        System.out.println("Migrations complete");
    }
}

@Repository
@DependsOn("liquibaseMigrationRunner")  // schema must exist first
public class UserRepository {

    @Autowired JdbcTemplate jdbc;

    @PostConstruct
    public void verifySchema() {
        Long count = jdbc.queryForObject(
            "SELECT COUNT(*) FROM users", Long.class);
        System.out.println("Users table has " + count + " rows");
        // Would fail with "table not found" if migrations hadn't run
    }
}
```

---

### Example 2 — `@DependsOn` Destruction Order

```java
@Component
public class MessageBrokerConnection {

    @PostConstruct
    public void connect() {
        System.out.println("Connecting to message broker");
    }

    @PreDestroy
    public void disconnect() {
        System.out.println("Disconnecting from message broker");
        // Can only disconnect after consumers have stopped
    }
}

@Component
@DependsOn("messageBrokerConnection") // init AFTER, destroy BEFORE
public class MessageConsumer {

    @PostConstruct
    public void startConsuming() {
        System.out.println("Starting to consume messages");
    }

    @PreDestroy
    public void stopConsuming() {
        System.out.println("Stopping message consumption");
    }
}
```

Output at startup:
```
Connecting to message broker
Starting to consume messages
```

Output at shutdown:
```
Stopping message consumption      ← MessageConsumer destroyed first (it depended on connection)
Disconnecting from message broker ← MessageBrokerConnection destroyed last
```

---

### Example 3 — `@Lazy` Behavior Demo

```java
@Service
@Lazy
public class ExpensiveService {
    public ExpensiveService() {
        System.out.println("ExpensiveService CONSTRUCTOR");
    }

    @PostConstruct
    public void init() {
        System.out.println("ExpensiveService @PostConstruct");
    }

    public String compute() { return "result"; }
}

@Service
public class AppService {

    @Autowired @Lazy  // proxy injected at AppService creation time
    private ExpensiveService expensiveService;

    public AppService() {
        System.out.println("AppService CONSTRUCTOR");
    }

    @PostConstruct
    public void init() {
        System.out.println("AppService @PostConstruct");
        // expensiveService is a CGLIB proxy here — NOT yet initialized
        System.out.println("Is proxy: " +
            expensiveService.getClass().getName().contains("CGLIB"));
    }

    public void doWork() {
        // First actual method call — triggers real ExpensiveService creation
        System.out.println("Calling expensiveService...");
        String result = expensiveService.compute(); // REAL bean created NOW
        System.out.println("Result: " + result);
    }
}
```

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
System.out.println("--- Context ready ---");
ctx.getBean(AppService.class).doWork();
```

Output:
```
AppService CONSTRUCTOR
AppService @PostConstruct
Is proxy: true
--- Context ready ---
Calling expensiveService...
ExpensiveService CONSTRUCTOR
ExpensiveService @PostConstruct
Result: result
```

---

### Example 4 — FactoryBean Dependency Resolution

```java
public class RedisConnectionFactoryBean implements FactoryBean<RedisConnection> {

    @Value("${redis.host}")
    private String host;

    @Value("${redis.port:6379}")
    private int port;

    private RedisConnection connection;

    @Override
    public RedisConnection getObject() throws Exception {
        System.out.println("Creating RedisConnection to " + host + ":" + port);
        if (connection == null) {
            connection = new RedisConnection(host, port);
        }
        return connection;
    }

    @Override
    public Class<?> getObjectType() {
        return RedisConnection.class; // Always return non-null type
    }

    @Override
    public boolean isSingleton() { return true; }
}
```

```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisConnectionFactoryBean redisConnection() {
        return new RedisConnectionFactoryBean();
    }
}
```

```java
// Access the product (RedisConnection):
RedisConnection conn = ctx.getBean("redisConnection", RedisConnection.class);
// → "Creating RedisConnection to localhost:6379"

// Access the factory itself:
RedisConnectionFactoryBean factory =
    ctx.getBean("&redisConnection", RedisConnectionFactoryBean.class);

// Type-based lookup works with the PRODUCT type:
RedisConnection conn2 = ctx.getBean(RedisConnection.class);
// → Returns same singleton (getObject() called once, isSingleton=true)
System.out.println(conn == conn2); // true
```

---

### Example 5 — `@Order` for Collections vs Creation Order

```java
@Component @Order(3) public class SlowFilter    implements RequestFilter { }
@Component @Order(1) public class SecurityFilter implements RequestFilter { }
@Component @Order(2) public class LoggingFilter  implements RequestFilter { }
```

```java
@Service
public class FilterChain {

    @Autowired
    private List<RequestFilter> filters;

    // filters order: [SecurityFilter(@Order=1), LoggingFilter(@Order=2), SlowFilter(@Order=3)]
    // Creation order: alphabetical by class name (component scan default)
    //     → LoggingFilter created, SecurityFilter created, SlowFilter created
    //     (DIFFERENT from list order!)

    public void process(Request req) {
        filters.forEach(f -> f.filter(req));
        // Processes in @Order order: Security → Logging → Slow
    }
}
```

**Key insight:** The CREATION order (alphabetical for component scanning) is different from the COLLECTION order (determined by `@Order`). Both are valid and correct — Spring creates beans in dependency/registration order, then sorts collections by `@Order` at injection time.

---

### Example 6 — SmartLifecycle Phase Ordering

```java
@Component
public class DatabaseLifecycle implements SmartLifecycle {
    private boolean running = false;

    @Override public int getPhase() { return 100; } // start early

    @Override public boolean isAutoStartup() { return true; }

    @Override
    public void start() {
        System.out.println("Phase 100: Starting database connections");
        running = true;
    }

    @Override
    public void stop() {
        System.out.println("Phase 100: Stopping database connections");
        running = false;
    }

    @Override public boolean isRunning() { return running; }
}

@Component
public class WebServerLifecycle implements SmartLifecycle {
    private boolean running = false;

    @Override public int getPhase() { return 1000; } // start after database

    @Override public boolean isAutoStartup() { return true; }

    @Override
    public void start() {
        System.out.println("Phase 1000: Starting web server");
        running = true;
    }

    @Override
    public void stop() {
        System.out.println("Phase 1000: Stopping web server");
        running = false;
    }

    @Override public boolean isRunning() { return running; }
}
```

Startup output:
```
Phase 100: Starting database connections     ← lower phase = first
Phase 1000: Starting web server              ← higher phase = second
```

Shutdown output:
```
Phase 1000: Stopping web server              ← higher phase = first to stop
Phase 100: Stopping database connections     ← lower phase = last to stop
```

---

### Example 7 — `@DependsOn` Circular Detection

```java
@Component
@DependsOn("serviceB")
public class ServiceA { }

@Component
@DependsOn("serviceA")
public class ServiceB { }
```

```
Result at startup:
BeanCreationException:
    Error creating bean with name 'serviceA':
    'serviceA' depends on missing bean 'serviceB';
    → actually: circular @DependsOn detected

Spring detects this during BeanDefinition processing and throws immediately.
```

---

### Example 8 — `@Lazy` Bean Still Created as Dependency

```java
@Service
@Lazy  // Defined as lazy
public class LazyService {
    public LazyService() {
        System.out.println("LazyService created");
    }
}

@Service
// NOT lazy — eager singleton
public class EagerService {
    @Autowired
    private LazyService lazyService; // non-lazy bean depends on lazy bean

    public EagerService() {
        System.out.println("EagerService created");
    }
}
```

Output during `preInstantiateSingletons()`:
```
LazyService created    ← Created eagerly as dependency of EagerService!
EagerService created
```

The `@Lazy` on `LazyService` is overridden by the eager autowiring from `EagerService`. `LazyService` is created at startup because `EagerService` needs it and `EagerService` is not lazy.

**The laziness only holds if NO eager bean directly autowires the lazy bean.**

---

### Example 9 — Full Initialization Order Proof

```java
@Component @DependsOn("c") public class BeanA {
    @PostConstruct void init() { System.out.println("A initialized"); }
}

@Component public class BeanB {
    @Autowired BeanC c; // B depends on C via autowiring
    @PostConstruct void init() { System.out.println("B initialized"); }
}

@Component public class BeanC {
    @PostConstruct void init() { System.out.println("C initialized"); }
}
```

Output (assuming A, B, C registered in alphabetical order):
```
C initialized    ← C is created first (dependency of B; DependsOn of A)
B initialized    ← B is created next (C is ready)
A initialized    ← A is created last (DependsOn C satisfied)
```

Why not A before B? `@DependsOn("c")` only guarantees C before A — it says nothing about B. Since B is independent of A and also has C as a dependency, B can be processed independently. The registration order (alphabetical: A, B, C) triggers A first, A needs C, C is created, then A. But B also needs C (already in L1), so B's order relative to A depends on which comes first in registration. In this case A comes before B alphabetically, so A is processed first — but to create A, C is needed. C is created. Then A. Then B (which finds C already in L1).

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What does `@Order(1)` on a `@Component` bean guarantee?

A) The bean is created first during `preInstantiateSingletons()`
B) The bean appears first in `List<T>` and `T[]` injection
C) The bean's `@PostConstruct` runs before beans with higher `@Order` values
D) The bean is given highest priority for `@Autowired` resolution

**Answer: B**
`@Order` on regular component beans ONLY affects ordering within collection injection (`List`, `Set`, array). It does NOT affect creation order or `@PostConstruct` timing.

---

**Q2 — MCQ:**
A bean is annotated with `@Lazy`. Another non-lazy singleton bean has `@Autowired` field of the lazy bean's type. What happens at startup?

A) The lazy bean is skipped at startup; the non-lazy bean receives null
B) The lazy bean is created at startup because it is required by the non-lazy bean
C) A CGLIB proxy is injected into the non-lazy bean; real lazy bean created on method call
D) `BeanCreationException` — lazy beans cannot be autowired into non-lazy beans

**Answer: B**
`@Lazy` on a bean definition only prevents proactive creation during `preInstantiateSingletons()`. When a non-lazy eager bean autowires the lazy bean, the lazy bean is created reactively as part of resolving the eager bean's dependencies. `@Lazy` at the INJECTION POINT (not bean definition) would cause proxy behavior (answer C).

---

**Q3 — Select all that apply:**
Which statements about `@DependsOn` are correct? (Select ALL that apply)

A) `@DependsOn("beanB")` ensures beanB's `@PostConstruct` completes before beanA is created
B) `@DependsOn` can resolve constructor injection circular dependencies
C) `@DependsOn` affects both initialization AND destruction order
D) Circular `@DependsOn` chains throw `BeanCreationException`
E) `@DependsOn` can be used on `@Bean` methods in `@Configuration` classes
F) `@DependsOn` creates an actual Spring dependency registration (dependentBeanMap)

**Answer: A, C, D, E, F**
B is wrong — `@DependsOn` controls ordering, not injection. It cannot resolve circular injection dependencies.

---

**Q4 — Code Output Prediction:**

```java
@Component
public class Alpha {
    public Alpha() { System.out.println("Alpha()"); }
    @PostConstruct void init() { System.out.println("Alpha @PostConstruct"); }
}

@Component
@DependsOn("alpha")
public class Beta {
    public Beta() { System.out.println("Beta()"); }
    @PostConstruct void init() { System.out.println("Beta @PostConstruct"); }
}

@Component
public class Gamma {
    @Autowired Beta beta;
    public Gamma() { System.out.println("Gamma()"); }
    @PostConstruct void init() { System.out.println("Gamma @PostConstruct"); }
}
```

What is the output order?

A)
```
Alpha()
Alpha @PostConstruct
Beta()
Beta @PostConstruct
Gamma()
Gamma @PostConstruct
```
B)
```
Gamma()
Alpha()
Alpha @PostConstruct
Beta()
Beta @PostConstruct
Gamma @PostConstruct
```
C)
```
Alpha()
Alpha @PostConstruct
Gamma()
Beta()
Beta @PostConstruct
Gamma @PostConstruct
```
D) Depends on JVM — unspecified

**Answer: A**
Registration order (alphabetical): Alpha, Beta, Gamma.
- Alpha processed: no deps → Alpha() → Alpha @PostConstruct
- Beta processed: @DependsOn("alpha") → alpha already done → Beta() → Beta @PostConstruct
- Gamma processed: needs Beta → beta already done → Gamma() → Gamma @PostConstruct

---

**Q5 — Trick Question:**

```java
@Component @Order(1) public class ServiceA implements MyService { }
@Component @Order(2) public class ServiceB implements MyService { }
@Component @Order(3) public class ServiceC implements MyService { }

@Service
public class Orchestrator {
    @Autowired List<MyService> services;
    @Autowired MyService myService; // single injection
}
```

For `myService` (single injection), which bean is injected?

A) `ServiceA` — `@Order(1)` makes it the primary candidate
B) `ServiceB` — `@Order(2)` is the median value
C) `NoUniqueBeanDefinitionException` — three beans of type `MyService`
D) `ServiceC` — last registered

**Answer: C**
`@Order` does NOT affect single-bean `@Autowired` resolution. For `List<MyService>`, it controls ordering. For `@Autowired MyService` (single), Spring finds three candidates, none is `@Primary`, name fallback fails → `NoUniqueBeanDefinitionException`.

---

**Q6 — Select all that apply:**
Which of the following correctly describe `FactoryBean` behavior? (Select ALL that apply)

A) `getBean("myFactory")` returns the FactoryBean's product, not the FactoryBean itself
B) `getBean("&myFactory")` returns the FactoryBean instance itself
C) `FactoryBean.getObjectType()` returning `null` forces Spring to call `getObject()` for type resolution
D) `SmartFactoryBean.isEagerInit()` returning `true` causes the product to be created during `preInstantiateSingletons()`
E) FactoryBean products are always eagerly initialized with singletons
F) A `FactoryBean` with `isSingleton()=true` returns the same product instance on every `getObject()` call

**Answer: A, B, C, D, F**
E is wrong — FactoryBean products are lazily initialized by default (on first `getBean()` call), unless `SmartFactoryBean.isEagerInit()` is `true`.

---

**Q7 — Scenario-based:**
The following beans are defined. In what order are they initialized?

```java
@Component public class X { @Autowired Z z; }
@Component public class Y { }
@Component @DependsOn("y") public class Z { }
```

A) X → Y → Z
B) Y → Z → X
C) Z → Y → X
D) Y → X → Z

**Answer: B**
Registration order (alphabetical): X, Y, Z.
- Processing X: needs Z → Z not ready → create Z
- Creating Z: @DependsOn("y") → create Y first
- Creating Y: no deps → Y initialized ✓
- Z initialized ✓ (Y is done)
- X initialized ✓ (Z is done)
Order: Y → Z → X

---

**Q8 — Compilation/Runtime Error:**

```java
@Configuration
public class AppConfig {

    @Bean
    @DependsOn("serviceB")
    public ServiceA serviceA() { return new ServiceA(); }

    @Bean
    @DependsOn("serviceA")
    public ServiceB serviceB() { return new ServiceB(); }
}
```

What happens?

A) `serviceA` initialized first (alphabetical order, DependsOn ignored for same-file beans)
B) `serviceB` initialized first (higher in the method order)
C) `BeanCreationException` — circular @DependsOn detected
D) Both initialized simultaneously — Spring detects independent resolution paths

**Answer: C**
Circular `@DependsOn` is detected by Spring during BeanDefinition processing. Spring builds a dependency graph from `dependsOn` attributes and detects the cycle. `BeanCreationException` with message about circular dependency in `@DependsOn`.

---

**Q9 — Drag-and-Drop: Order these events during ApplicationContext refresh:**

- `SmartInitializingSingleton.afterSingletonsInstantiated()`
- `BeanFactoryPostProcessor.postProcessBeanFactory()`
- `@PostConstruct` on first singleton bean
- `SmartLifecycle.start()` for autoStartup beans
- `BeanPostProcessor` beans instantiated and registered
- `ContextRefreshedEvent` published

**Answer:**
1. `BeanFactoryPostProcessor.postProcessBeanFactory()`
2. `BeanPostProcessor` beans instantiated and registered
3. `@PostConstruct` on first singleton bean
4. `SmartInitializingSingleton.afterSingletonsInstantiated()`
5. `SmartLifecycle.start()` for autoStartup beans
6. `ContextRefreshedEvent` published

---

**Q10 — MCQ:**
`SmartLifecycle` beans with a lower `getPhase()` value are:

A) Started last and stopped first
B) Started first and stopped last
C) Started first and stopped first
D) Stopped first and started last

**Answer: B**
Lower phase = started FIRST, stopped LAST. Higher phase = started LAST, stopped FIRST. This ensures that infrastructure (low phase) starts before application code (high phase), and application code stops before infrastructure during shutdown.

---

## ④ TRICK ANALYSIS

**Trap 1: `@Order` controls creation order**
The most common trap. `@Order` on `@Component` beans ONLY affects ordering in `List<T>`, `Set<T>`, and `T[]` injection. It does NOT affect when the bean is instantiated. Creation order is controlled by the dependency graph and registration order. Any question suggesting `@Order(1)` means "created first" is wrong.

**Trap 2: `@Order` as tiebreaker for `@Autowired` single bean**
`@Order` is NEVER used as a tiebreaker for single-bean `@Autowired` resolution. Only `@Primary` and `@Qualifier` affect single-bean resolution. Multiple beans of the same type with different `@Order` values → `NoUniqueBeanDefinitionException`, not "pick the lowest `@Order`."

**Trap 3: `@Lazy` bean definition vs `@Lazy` injection point**
Two completely different behaviors:
- `@Lazy` on bean definition: bean skipped in `preInstantiateSingletons()` — but still created if another non-lazy bean needs it
- `@Lazy` on injection point: CGLIB proxy injected immediately, real bean created on first method call

Exam questions showing `@Lazy` and asking about timing MUST identify which form is used.

**Trap 4: `@DependsOn` for circular dependency resolution**
`@DependsOn` cannot resolve circular injection dependencies. It only controls ordering. Circular `@DependsOn` is detected and throws. `@DependsOn` is for NON-INJECTION side-effect dependencies only.

**Trap 5: FactoryBean product laziness**
FactoryBean PRODUCTS are lazy by default — `getObject()` is called on first `getBean()` or `@Autowired` resolution. Questions suggesting FactoryBean products are eagerly created are wrong unless `SmartFactoryBean.isEagerInit()` is `true`.

**Trap 6: `SmartLifecycle` phase direction confusion**
Lower phase = started FIRST. This is counterintuitive — people assume "lower = later." Think of it as priority: phase 0 has higher priority than phase 100, so phase 0 starts first. Shutdown is always the reverse of startup — phase 100 stops first, phase 0 stops last.

**Trap 7: `@DependsOn` and destruction order guarantee**
Many candidates know `@DependsOn` controls initialization order but miss that it also controls DESTRUCTION order (opposite direction). `@DependsOn("b")` on bean A means: A initialized AFTER B, A destroyed BEFORE B. The exam may ask about destruction order of `@DependsOn` relationships.

**Trap 8: Registration order for component-scanned beans**
Component-scanned beans within the same package are discovered in a filesystem-dependent order (typically alphabetical on most systems, but not guaranteed by the spec). Questions assuming strict alphabetical registration order may be wrong on different OS or filesystems. The safe answer for ordering guarantees is `@DependsOn`.

**Trap 9: `PriorityOrdered` vs `Ordered` for BFPPs**
Both `PriorityOrdered` and `Ordered` provide ordering for BFPPs. BUT `PriorityOrdered` beans are processed in a COMPLETELY SEPARATE batch before ANY `Ordered` beans. It's not just "lower order value wins" — `PriorityOrdered` beans always run before `Ordered` beans, even if the `Ordered` bean has `getOrder()` returning `Integer.MIN_VALUE`.

**Trap 10: `afterSingletonsInstantiated` vs `@PostConstruct` timing**
`SmartInitializingSingleton.afterSingletonsInstantiated()` runs AFTER ALL singletons are fully initialized. `@PostConstruct` runs during each individual bean's initialization. If bean A's `@PostConstruct` tries to use bean B, bean B might or might not be initialized yet (depends on dependency order). `afterSingletonsInstantiated()` guarantees all beans are ready — it's the safe cross-bean post-initialization hook.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Instantiation order driver | Dependency graph (depth-first) > registration order |
| `@Order` on `@Component` | ONLY affects `List<T>`, `Set<T>`, `T[]` injection order — NOT creation order |
| `@Order` for `@Autowired` single bean | NEVER used — no effect on single-bean resolution |
| `@DependsOn` | Ordering only — cannot resolve circular injection |
| Circular `@DependsOn` | `BeanCreationException` — detected and thrown |
| `@DependsOn` destruction | Reverse of initialization — A depended on B → A destroyed before B |
| `@Lazy` on bean definition | Skips proactive creation; still created reactively as dependency |
| `@Lazy` on injection point | CGLIB proxy injected; real bean on first method call |
| FactoryBean product | Lazily created by default; eagerly if `SmartFactoryBean.isEagerInit()=true` |
| SmartLifecycle phase | Lower phase = started first = stopped last |
| `PriorityOrdered` vs `Ordered` | PriorityOrdered runs in completely separate (earlier) batch than Ordered |
| `afterSingletonsInstantiated()` | Runs after ALL singletons complete — safest cross-bean post-init hook |

---

### Complete Refresh Order (Condensed)

```
refresh()
│
├── [Step 5] BFPPs:
│   PriorityOrdered BDRPPs → Ordered BDRPPs → plain BDRPPs
│   PriorityOrdered BFPPs  → Ordered BFPPs  → plain BFPPs
│   (ConfigurationClassPostProcessor is PriorityOrdered BDRPP — runs first)
│
├── [Step 6] BPPs:
│   PriorityOrdered BPPs → Ordered BPPs → plain BPPs
│   (AABPP and CABPP are PriorityOrdered — registered first)
│
├── [Step 11] Singletons (preInstantiateSingletons):
│   For each bean in registration order:
│       → depth-first dependency resolution
│       → per-bean: construct → inject → aware → BPP-before → @PostConstruct → BPP-after
│   After all singletons:
│       → afterSingletonsInstantiated()
│
└── [Step 12] finishRefresh:
    → SmartLifecycle.start() (ordered by phase, lower first)
    → ContextRefreshedEvent published
```

---

### `@Order` Scope of Effect

```
@Order affects:
  ✅ List<T> injection — elements sorted by @Order
  ✅ T[] injection — elements sorted by @Order
  ✅ BPP execution order (within same priority tier)
  ✅ BFPP execution order (within same priority tier)
  ✅ EventListener execution order (@Order on @EventListener methods)

@Order does NOT affect:
  ❌ Bean instantiation/creation order
  ❌ @Autowired single-bean resolution
  ❌ @PostConstruct timing
  ❌ @PreDestroy timing
```

---

### FactoryBean Quick Reference

```
getBean("name")    → returns FactoryBean's PRODUCT (T)
getBean("&name")   → returns the FactoryBean itself
getObjectType()    → should NEVER return null if type is known at config time
isSingleton()      → true → getObject() result cached; false → new instance per call
isEagerInit()      → SmartFactoryBean only; true → product created in preInstantiateSingletons
```

---

### SmartLifecycle Phase Rules

```
Startup:  phase 0 starts before phase 100 before phase MAX_VALUE
                    ↓ lower = first to start
Shutdown: phase MAX_VALUE stops before phase 100 before phase 0
                    ↓ higher = first to stop (reverse of startup)

Default phase for SmartLifecycle: Integer.MAX_VALUE / 2 = 1073741823
Spring Boot embedded server: specific phases defined in WebServerGracefulShutdownLifecycle
```

---

### Interview One-Liners

- "Bean creation order is driven by the dependency graph — dependencies are always created before dependents, regardless of registration order."
- "`@Order` on `@Component` beans affects collection injection ordering, NOT instantiation order."
- "`@DependsOn` is for non-injection side-effect dependencies — it guarantees ordering but cannot resolve circular injection dependencies."
- "`@Lazy` on a bean definition only prevents proactive initialization — the bean is still created reactively when autowired by an eager bean."
- "`@Lazy` at the injection point injects a CGLIB proxy immediately — the real bean is created on first method invocation."
- "FactoryBean products are lazy by default — `getObject()` is called on first access, not at container startup."
- "SmartLifecycle phases: lower phase starts first and stops last — it's priority ordering, with infrastructure (low phase) outliving application code (high phase)."
- "`SmartInitializingSingleton.afterSingletonsInstantiated()` is the only callback guaranteed to run after ALL singleton beans are fully initialized — safer than `@PostConstruct` for cross-bean post-initialization logic."

---
