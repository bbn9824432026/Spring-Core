# 🎯 Spring Bean Lifecycle — From Absolute Zero

---

## 🧩 1. Requirement Understanding — Feel the Pain First

You join a new team. Your ticket:

```
TICKET-089: Our app crashes on startup with NullPointerException
            inside DatabaseConnectionPool.init().
            
            DatabaseConnectionPool tries to use its config values
            in the constructor — but they're always null.
```

You open the code:

```java
@Component
public class DatabaseConnectionPool {

    @Value("${db.url}")
    private String dbUrl;        // injected by Spring

    @Value("${db.maxConnections}")
    private int maxConnections;  // injected by Spring

    public DatabaseConnectionPool() {
        // You think: "I'll initialize right here in the constructor."
        connect(dbUrl, maxConnections);  // ← NullPointerException
        // dbUrl is null. maxConnections is 0.
        // Spring hasn't injected them yet.
        // The constructor runs BEFORE Spring fills in these fields.
    }

    private void connect(String url, int maxConn) {
        System.out.println("Connecting to: " + url);  // "Connecting to: null"
    }
}
```

The bug is silent. No compile error. Spring builds the object, then tries to inject values, but by then the constructor already ran and crashed (or silently used nulls).

---

**New ticket, same week:**

```
TICKET-091: In prod, we're getting "Cannot get a connection, pool error"
            even AFTER the app starts fine.
            
            Also, our cache warmup runs before the database is ready.
            And email clients are not being closed on shutdown —
            memory leak in production.
```

Three separate bugs. All of them are **lifecycle bugs** — code running at the wrong point in time. The developer didn't know:

- When can I safely use my injected fields?
- How do I run code AFTER everything is wired up?
- How do I guarantee my beans are cleaned up on shutdown?
- How do I control WHAT runs before WHAT?

Spring has a precise, ordered answer to all of these. That's the bean lifecycle.

---

## 🧠 2. Design Thinking — The Naive Approaches

### Attempt 1: Do everything in the constructor

```java
@Component
public class DatabasePool {
    @Value("${db.url}")
    private String url;

    public DatabasePool() {
        openConnection(url); // ← url is null here, always
    }
}
```

❌ **Rejected.** The constructor runs FIRST, before Spring injects `@Value` fields. The constructor is Step 1. Injection is Step 2. You're trying to use Step 2's result in Step 1. Impossible.

---

### Attempt 2: Use a static initializer block

```java
@Component
public class DatabasePool {
    @Value("${db.url}")
    private String url;

    static {
        // Can't even access url here — it's an instance field
        System.out.println("Static init: " + url); // compile error
    }
}
```

❌ **Rejected.** Static blocks run when the class is loaded by the JVM — before Spring even knows this class exists, let alone before it can inject instance fields.

---

### Attempt 3: Check values manually in every method

```java
@Service
public class DatabasePool {
    @Value("${db.url}")
    private String url;

    private boolean initialized = false;

    public Connection getConnection() {
        if (!initialized) {
            openConnection(url);
            initialized = true;
        }
        return connection;
    }
}
```

❌ **Rejected.** Every method now needs this guard. Thread-safety nightmare. What about shutdown? What about ordering between beans? You've re-invented a lifecycle framework — badly. Spring already built this.

**The right answer:** Spring has specific hooks that fire at exactly the right moment. You declare what you want to happen and WHEN. Spring orchestrates everything.

---

## 🧠 3. Concept Introduction — How It Works (No Code Yet)

Think of a bean's life like a **new employee joining a company**.

```
Day 1 - HR paperwork (BeanDefinition registered)
    HR reads your job description. Nothing real has happened yet.
    You don't exist as an employee yet — just a form.

Day 2 - You physically arrive (Constructor called)
    You exist as a person in the building.
    But your desk is empty. No computer. No access badge. Nothing.

Day 3 - Your desk is set up (Dependency Injection)
    IT gives you a laptop. Security gives you a badge.
    Now you have your tools. But you haven't started working yet.

Day 4 - Orientation / Training (@PostConstruct)
    You use your new laptop to set up your development environment.
    This is the FIRST moment you can safely use your tools.
    You run setup scripts, verify connections, warm caches.

Day 5+ - You work (Bean in use)
    Normal business operations. You're fully ready.

Resignation day - Handoff (@PreDestroy)
    Before leaving: close open tickets, hand off responsibilities.
    Clean exit. Nothing left hanging.
```

**The lifecycle in a picture:**

```
BeanDefinition        Constructor         Injection          @PostConstruct
(just metadata)  →   (you exist)  →  (tools arrive)  →  (you can work!)
     📄                  👤               🖥️ 🔑               ✅

                                                         Bean in use 🔄

                                                         @PreDestroy
                                                         (clean shutdown)
                                                              🧹
```

**The three init hooks — and when they're safe to use:**

```
Constructor:         ❌ Injected fields are NOT available yet
afterPropertiesSet:  ✅ All fields injected, fully safe to use
@PostConstruct:      ✅ All fields injected, fully safe to use (runs first)
```

**The AOP surprise (important!):**

```
The "you" that the outside world sees may not be the raw you.
It might be a version of you that intercepts every phone call
(a proxy). That proxy-you is created AFTER all your training is done.

During @PostConstruct: you are the raw you (no phone interception yet)
After the bean is ready: callers talk to proxy-you
```

---

## 🏗 4. Task Breakdown

```
TODO 1: @PostConstruct — run code AFTER all fields are injected
TODO 2: @PreDestroy — clean up before the bean dies
TODO 3: InitializingBean / init-method — the two alternatives to @PostConstruct
TODO 4: Aware interfaces — getting Spring's own objects into your bean
TODO 5: BeanPostProcessor — intercept EVERY bean's lifecycle
TODO 6: When the AOP proxy appears — and why it matters
TODO 7: Prototype beans — the lifecycle that ends at birth
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: `@PostConstruct` — The Safe Initialization Hook

```java
@Component
public class DatabaseConnectionPool {

    // Spring injects these BEFORE @PostConstruct runs
    @Value("${db.url}")
    private String dbUrl;

    @Value("${db.maxConnections:10}")
    private int maxConnections;

    private Connection connection;

    // Constructor: dbUrl is null here. maxConnections is 0.
    // DO NOT use injected fields here.
    public DatabaseConnectionPool() {
        System.out.println("Step 1 - Constructor: dbUrl=" + dbUrl); // null
    }

    // @PostConstruct marks a method that Spring calls AFTER:
    //   ① the constructor ran
    //   ② ALL @Value and @Autowired fields were filled in
    // This is the FIRST safe place to use injected values.
    @PostConstruct
    public void initialize() {
        System.out.println("Step 2 - @PostConstruct: dbUrl=" + dbUrl); // "jdbc:..."
        this.connection = openConnection(dbUrl, maxConnections);
        System.out.println("Connection pool ready with " + maxConnections + " connections");
    }

    public Connection getConnection() {
        return connection; // safely initialized by the time anyone calls this
    }
}
```

**What Spring does internally:**

```
Spring's internal sequence for DatabaseConnectionPool:

① new DatabaseConnectionPool()              ← constructor runs, fields all null/0
② field.set(bean, "${db.url}" value)        ← Spring fills @Value fields
③ field.set(bean, otherDependencies)        ← Spring fills @Autowired fields
④ bean.initialize()                         ← @PostConstruct called by Spring
                                               via CommonAnnotationBeanPostProcessor
⑤ Bean stored, available for use
```

**Common silent failure — using injected values in constructor:**

```java
// ❌ This compiles fine. Runs fine. But always uses null/defaults.
@Component
public class CacheService {

    @Value("${cache.size:1000}")
    private int cacheSize;

    public CacheService() {
        // cacheSize is 0 here — Java's default for int
        // @Value injection hasn't happened yet
        this.cache = new LinkedHashMap<>(cacheSize);
        // Creates a cache with size 0. Never actually limits entries.
        // App works but is silently wrong.
    }
}

// ✅ Fix:
@PostConstruct
public void init() {
    this.cache = new LinkedHashMap<>(cacheSize); // cacheSize is now correctly set
}
```

---

### TODO 2: `@PreDestroy` — Safe Shutdown Hook

```java
@Component
public class EmailClient {

    private SmtpConnection smtpConnection;

    @PostConstruct
    public void connect() {
        this.smtpConnection = SmtpConnection.open("smtp.company.com");
        System.out.println("SMTP connection opened");
    }

    public void sendEmail(String to, String body) {
        smtpConnection.send(to, body);
    }

    // @PreDestroy marks a method Spring calls when the container is shutting down.
    // This runs BEFORE the bean is destroyed.
    // Guaranteed to run before your dependencies are destroyed too.
    @PreDestroy
    public void disconnect() {
        if (smtpConnection != null) {
            smtpConnection.close();
            System.out.println("SMTP connection closed cleanly");
        }
    }
}
```

**What triggers `@PreDestroy`:**

```java
// Option 1: Explicit close
ConfigurableApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
ctx.close();  // ← triggers @PreDestroy on all singleton beans

// Option 2: JVM shutdown hook (Spring Boot registers this automatically)
// When JVM shuts down → Spring closes context → @PreDestroy runs
```

**The silent failure that caused the production memory leak:**

```java
// ❌ No cleanup — connection leaks on every restart
@Component
public class EmailClient {
    private SmtpConnection conn = SmtpConnection.open("smtp.company.com");
    // conn.close() is NEVER called
    // Every deployment leaks one connection
}

// ✅ Fix: @PreDestroy ensures close() is always called
```

**Destroy order is reverse of creation order:**

```
If DatabasePool is created BEFORE EmailClient:
    Startup:  DatabasePool init → EmailClient init
    Shutdown: EmailClient destroy → DatabasePool destroy

EmailClient is destroyed first (created last).
DatabasePool is still alive while EmailClient is cleaning up.
EmailClient could still use DatabasePool during its own @PreDestroy.
```

---

### TODO 3: The Three Init Mechanisms — and Their Order

Spring supports three ways to run code after injection. They always run in this exact order:

```
① @PostConstruct           — runs first (standard, recommended)
② afterPropertiesSet()     — runs second (Spring interface)
③ init-method              — runs third (XML or @Bean config)
```

```java
// Implementing all three at once (rare in practice — for learning purposes)
@Component
public class TripleInitDemo implements InitializingBean {

    // @PostConstruct: runs FIRST
    // Advantage: standard Java annotation (JSR-250), no Spring import needed
    // Use this in your own application code
    @PostConstruct
    public void initFirst() {
        System.out.println("① @PostConstruct — first always");
    }

    // InitializingBean is a Spring interface.
    // afterPropertiesSet() runs SECOND.
    // Disadvantage: your class now imports a Spring interface — coupled to Spring.
    // Use this when writing framework/library code that already depends on Spring.
    @Override
    public void afterPropertiesSet() {
        System.out.println("② afterPropertiesSet — second always");
    }

    // This method is declared as init-method in @Bean or XML config.
    // Runs THIRD.
    // Advantage: you don't touch the class at all — useful for third-party classes
    //            where you can't add annotations.
    public void initThird() {
        System.out.println("③ init-method — third always");
    }
}

@Configuration
public class Config {
    @Bean(initMethod = "initThird")  // ← tells Spring which method to call third
    public TripleInitDemo tripleInitDemo() {
        return new TripleInitDemo();
    }
}
```

**Output — always in this order, no exceptions:**

```
① @PostConstruct — first always
② afterPropertiesSet — second always
③ init-method — third always
```

**The same pattern for destroy:**

```
① @PreDestroy           — runs first
② DisposableBean.destroy() — runs second
③ destroyMethod          — runs third
```

**The `@Bean` `destroyMethod` silent failure — this surprises everyone:**

```java
// ❌ You think nothing happens at shutdown because you didn't set destroyMethod
@Bean
public HikariDataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
    return ds;
}
// But HikariDataSource has a close() method.
// Spring's @Bean default is destroyMethod="(inferred)".
// Spring automatically detects close() and CALLS IT at shutdown.
// This is actually correct behavior — but surprises people who didn't know.

// ✅ If you DON'T want close() called (externally managed connection pool):
@Bean(destroyMethod = "")  // empty string = no auto-detection
public DataSource sharedDataSource() {
    return externalPool; // managed by someone else
}
```

---

### TODO 4: Aware Interfaces — Getting Spring's Own Objects

Sometimes your bean needs to know things about Spring itself — its own name, the container it lives in, the environment it runs in.

```java
@Component
// Implementing these interfaces tells Spring:
// "When you build me, call these methods with Spring's internals"
public class SmartBean
        implements BeanNameAware,        // "tell me my own bean name"
                   BeanFactoryAware,     // "give me the factory that created me"
                   ApplicationContextAware {  // "give me the full application context"

    private String myBeanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;

    // BeanNameAware: Spring calls this right after construction,
    // before @PostConstruct.
    // Use case: logging, metrics, identifying yourself.
    @Override
    public void setBeanName(String name) {
        this.myBeanName = name;
        System.out.println("I am registered as: " + name);
    }

    // BeanFactoryAware: gives you the factory that manages beans.
    // You can call beanFactory.getBean() to fetch beans programmatically.
    @Override
    public void setBeanFactory(BeanFactory factory) {
        this.beanFactory = factory;
    }

    // ApplicationContextAware: gives you the full context —
    // includes event publishing, resource loading, everything.
    // This is called AFTER setBeanName and setBeanFactory.
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
    }

    @PostConstruct
    public void verify() {
        // By @PostConstruct time, ALL aware callbacks have already run
        System.out.println("My name: " + myBeanName);       // filled ✅
        System.out.println("Factory: " + (beanFactory != null)); // true ✅
        System.out.println("Context: " + (applicationContext != null)); // true ✅
    }
}
```

**Aware callback order — fixed, memorize this:**

```
Called by Spring DIRECTLY (before any BPP step):
    ① BeanNameAware.setBeanName()
    ② BeanClassLoaderAware.setBeanClassLoader()
    ③ BeanFactoryAware.setBeanFactory()

Called by ApplicationContextAwareProcessor (a BPP, runs just before @PostConstruct):
    ④ EnvironmentAware.setEnvironment()
    ⑤ ResourceLoaderAware.setResourceLoader()
    ⑥ ApplicationEventPublisherAware.setApplicationEventPublisher()
    ⑦ ApplicationContextAware.setApplicationContext()

Then:
    ⑧ @PostConstruct
```

**When to use Aware vs `@Autowired`:**

```java
// ❌ Unnecessary — just use @Autowired
@Component
public class MyService implements ApplicationContextAware {
    private ApplicationContext ctx;
    public void setApplicationContext(ApplicationContext ctx) { this.ctx = ctx; }
}

// ✅ Simpler:
@Component
public class MyService {
    @Autowired
    private ApplicationContext ctx;
}

// ✅ Use Aware when:
// 1. You're writing framework/library code (can't control injection order)
// 2. You need it BEFORE @PostConstruct (Aware fires slightly earlier)
// 3. You need BeanFactory specifically (not the full ApplicationContext)
```

---

### TODO 5: BeanPostProcessor — Intercepting Every Bean's Lifecycle

So far you've seen lifecycle hooks ON individual beans. A `BeanPostProcessor` (BPP) is different: it runs for **every single bean** in the container.

```
New employee joins the company (a bean is created)
    Before their first day:   HR reviews their paperwork (BPP before-init)
    Training completes:       IT assigns equipment (init callbacks)
    After their first day:    Security upgrades their badge if needed (BPP after-init)
    
This HR/IT/Security review happens for EVERY new employee.
```

```java
// A BPP intercepts every bean's lifecycle at two points:
// 1. Just before initialization callbacks run (before @PostConstruct)
// 2. Just after initialization callbacks run (after init-method)

@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {

    // postProcessBeforeInitialization: called for EVERY bean,
    // BEFORE @PostConstruct / afterPropertiesSet() / init-method.
    // The bean is fully injected but init hasn't happened yet.
    // Return value is what continues through the lifecycle.
    // Return the original bean (or a modified version).
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("Before init: " + beanName
            + " [" + bean.getClass().getSimpleName() + "]");
        return bean; // return the same bean — no change
    }

    // postProcessAfterInitialization: called for EVERY bean,
    // AFTER all init callbacks.
    // THIS is where Spring creates AOP proxies.
    // You can return a completely different object here — it REPLACES the bean.
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("After init: " + beanName
            + " [" + bean.getClass().getSimpleName() + "]");

        // Return a wrapper/proxy instead of the original:
        if (bean instanceof PaymentService) {
            return new LoggingPaymentServiceWrapper((PaymentService) bean);
            // Now ctx.getBean(PaymentService.class) returns the WRAPPER, not original
        }
        return bean;
    }
}
```

**What Spring does internally:**

```
For every singleton bean, in order:

postProcessBeforeInitialization(bean, name)  ← BPP step
    → @PostConstruct called HERE (by CommonAnnotationBeanPostProcessor)

@PostConstruct / afterPropertiesSet() / init-method  ← init callbacks

postProcessAfterInitialization(bean, name)   ← BPP step
    → AOP PROXY CREATED HERE (by AbstractAutoProxyCreator)
    → If proxy created, the proxy goes into singleton cache — not the raw bean
```

**Common silent failure — returning null from BPP:**

```java
// ❌ This breaks everything downstream
@Component
public class BrokenBPP implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (shouldProcess(beanName)) {
            doSomethingWithBean(bean);
            return null;  // ← SILENT BUG: null means "use the original"
                          // Actually Spring uses original, but you've lost any changes
                          // you wanted to make — confusing behavior
        }
        return bean;
    }
}

// ✅ Fix: always return something meaningful
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    doSomethingWithBean(bean);
    return bean;  // or return wrappedBean
}
```

---

### TODO 6: When the AOP Proxy Appears — And Why It Matters

This is the #1 lifecycle surprise for developers.

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;

    // @Transactional means Spring will wrap this in an AOP proxy
    @Transactional
    public void placeOrder(Order order) {
        paymentService.charge(order);
    }

    @PostConstruct
    public void verify() {
        // What is "this" here?
        System.out.println(this.getClass().getSimpleName());
        // prints: OrderService   ← the RAW bean, not the proxy
        // The proxy doesn't exist yet!
    }
}
```

**The timeline:**

```
Spring building OrderService:

    ① new OrderService()              ← raw bean, no proxy
    ② paymentService injected         ← raw bean has its dependency
    ③ @PostConstruct verify() called  ← "this" is still raw OrderService
                                         no AOP yet, no proxy yet
    ④ afterPropertiesSet() (if any)   ← still raw bean
    ⑤ init-method (if any)           ← still raw bean
    ⑥ AbstractAutoProxyCreator runs  ← PROXY CREATED HERE
       "Does any @Transactional advice apply to OrderService?"
       YES → wrap raw bean in CGLIB proxy
       Return proxy instead of raw bean.
    ⑦ OrderService$$SpringCGLIB$$0 stored in singleton cache

Now when you do:
    ctx.getBean(OrderService.class) → returns the PROXY
    @Autowired OrderService service  → injects the PROXY

Calling service.placeOrder(order):
    → PROXY intercepts the call
    → Starts transaction
    → Delegates to raw OrderService.placeOrder()
    → Commits transaction
```

**The self-invocation trap this causes:**

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order); // internal call — bypasses proxy!
    }

    @Transactional  // ← this annotation is IGNORED for internal calls
    public void validateOrder(Order order) {
        // This runs WITHOUT a transaction
        // because "this.validateOrder()" skips the proxy entirely
    }
}

// ✅ Fix: inject self-reference to go through proxy
@Service
public class OrderService {

    @Autowired
    private OrderService self;  // Spring injects the PROXY of myself

    @Transactional
    public void placeOrder(Order order) {
        self.validateOrder(order);  // goes through proxy → transaction works
    }

    @Transactional
    public void validateOrder(Order order) { ... }
}
```

---

### TODO 7: Prototype Beans — The Lifecycle That Ends at Birth

```java
@Component
@Scope("prototype")  // new instance on every getBean() call
public class ShoppingCart {

    @PostConstruct
    public void init() {
        System.out.println("ShoppingCart created: " + this.hashCode());
        // ✅ @PostConstruct DOES run for prototype beans — every time
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("ShoppingCart cleanup: " + this.hashCode());
        // ❌ @PreDestroy NEVER runs for prototype beans
        // Spring creates prototypes but doesn't track them afterward
        // You are responsible for cleanup
    }
}
```

```java
@Service
public class StoreService {
    @Autowired
    private ApplicationContext ctx;

    public void demonstratePrototype() {
        ShoppingCart c1 = ctx.getBean(ShoppingCart.class);
        // prints: ShoppingCart created: 12345

        ShoppingCart c2 = ctx.getBean(ShoppingCart.class);
        // prints: ShoppingCart created: 67890

        System.out.println(c1 == c2); // false — different instances ✅
    }
}

// When context.close() is called:
// NOTHING printed for ShoppingCart — @PreDestroy never fires
```

**The complete prototype lifecycle:**

```
Spring creates prototype:
    ① Constructor
    ② Dependency injection
    ③ Aware callbacks
    ④ @PostConstruct ← DOES run ✅
    ⑤ Bean returned to caller

Spring FORGETS about it. No reference kept.
Spring NEVER calls:
    ❌ @PreDestroy
    ❌ DisposableBean.destroy()
    ❌ Custom destroyMethod

You must clean up prototypes yourself.
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

### Decision 1: Which init mechanism to use?

```
@PostConstruct (recommended for application code):
    ✓ No Spring import — just javax.annotation
    ✓ Clearest signal of intent ("this runs after DI")
    ✓ Works in any Java EE / Jakarta EE context too
    Use: 99% of the time in your own beans

afterPropertiesSet() (for framework code):
    ✓ Programmatic guarantee — compiler enforces the method exists
    ✗ Imports Spring interface — your class is coupled to Spring
    Use: when you're writing code that already deeply depends on Spring internals

init-method (for third-party classes):
    ✓ Zero modification to the class
    ✓ Can apply to classes you don't own
    ✗ Name is just a string — typos fail at runtime, not compile time
    Use: when you can't annotate the class (legacy, third-party library)
```

### Decision 2: `@PreDestroy` vs `DisposableBean` vs destroyMethod

Same pattern as init. Always prefer `@PreDestroy`. Use `DisposableBean` in framework code. Use `destroyMethod` for third-party classes that have a `close()` method.

### Decision 3: Do you need Aware interfaces?

```
95% of the time: use @Autowired instead — simpler and testable.

Use Aware when:
    - You need the value BEFORE @PostConstruct (rare)
    - You're writing infrastructure that can't use @Autowired safely
    - You need BeanFactory (not ApplicationContext) specifically
```

---

## 🧪 7. Edge Cases & Testing

### Silent Break #1: @PostConstruct ordering between beans

```java
// ❌ Cache tries to warm up before DB is ready
@Component
public class CacheWarmer {
    @Autowired DatabasePool db;

    @PostConstruct
    public void warmCache() {
        db.query("SELECT * FROM products"); // might fail if DB not ready
        // Spring doesn't guarantee WHICH @PostConstruct runs first
        // across unrelated beans
    }
}

// ✅ Fix: use @DependsOn to explicitly order
@Component
@DependsOn("databasePool")  // run DatabasePool's @PostConstruct first
public class CacheWarmer {
    @Autowired DatabasePool db;

    @PostConstruct
    public void warmCache() {
        db.query("SELECT * FROM products"); // DB is guaranteed ready ✅
    }
}
```

### Silent Break #2: Prototype `@PreDestroy` never runs

```java
// ❌ Memory leak — FileHandle never closed
@Component @Scope("prototype")
public class ReportGenerator {
    private FileHandle fileHandle = FileHandle.open("temp.dat");

    @PreDestroy
    public void close() {
        fileHandle.close(); // NEVER CALLED for prototypes
    }
}

// ✅ Fix: implement AutoCloseable and use try-with-resources
@Component @Scope("prototype")
public class ReportGenerator implements AutoCloseable {
    private FileHandle fileHandle = FileHandle.open("temp.dat");

    @Override
    public void close() {
        fileHandle.close();
    }
}

// Caller is responsible:
try (ReportGenerator gen = ctx.getBean(ReportGenerator.class)) {
    gen.generate();
} // close() called here by try-with-resources
```

### Silent Break #3: AOP doesn't work on raw `this`

```java
// ❌ @Cacheable is ignored — direct internal call bypasses proxy
@Service
public class ProductService {
    public List<Product> getProductsForCategory(String cat) {
        return getAllProducts().stream()  // this.getAllProducts() — bypasses proxy!
            .filter(p -> p.category().equals(cat))
            .toList();
    }

    @Cacheable("products")  // ← never activated by internal call
    public List<Product> getAllProducts() {
        return db.findAll(); // expensive — called every time
    }
}

// ✅ Catch in test:
@Test
void getAllProducts_shouldUseCacheOnSecondCall() {
    service.getProductsForCategory("electronics");
    service.getProductsForCategory("electronics");
    verify(db, times(1)).findAll(); // FAILS — called twice, cache not working
    // Test reveals the bug before production does
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version:**

```java
// ❌ Junior version — scattered lifecycle hooks, no cleanup
@Service
public class DataPipeline {
    @Value("${pipeline.threads}")
    private int threads;

    @Autowired
    private DataSource dataSource;

    private ExecutorService executor;
    private Connection connection;

    public DataPipeline() {
        // trying to use threads here — it's 0
        executor = Executors.newFixedThreadPool(threads);  // creates 0-thread pool!
    }

    public void process(Data data) {
        // connection is null if constructor-initialization failed
        connection.execute(data); // NullPointerException
    }
}
```

**Senior refactored version:**

```java
@Service
@Slf4j
public class DataPipeline {

    // All config values — injected by Spring, available at @PostConstruct
    @Value("${pipeline.threads:4}")
    private int threads;

    @Value("${pipeline.timeout:30}")
    private int timeoutSeconds;

    // Mandatory dependency — constructor injected (final, immutable)
    private final DataSource dataSource;

    // Resources that require setup — initialized in @PostConstruct
    private ExecutorService executor;
    private Connection connection;

    // Constructor: only mandatory dependencies, nothing that uses @Value
    public DataPipeline(DataSource dataSource) {
        this.dataSource = dataSource;
        // DO NOT use threads here — it's 0 until Spring injects it
    }

    // @PostConstruct: safe to use ALL injected values
    @PostConstruct
    public void initialize() {
        log.info("Initializing pipeline with {} threads", threads);
        this.executor = Executors.newFixedThreadPool(threads);
        this.connection = dataSource.getConnection();
        log.info("Pipeline ready. Timeout: {}s", timeoutSeconds);
    }

    public void process(Data data) {
        // executor and connection are guaranteed non-null here
        executor.submit(() -> connection.execute(data));
    }

    // @PreDestroy: always clean up, even on unexpected shutdown
    @PreDestroy
    public void shutdown() {
        log.info("Pipeline shutting down...");
        if (executor != null) {
            executor.shutdown();
            try {
                executor.awaitTermination(timeoutSeconds, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        if (connection != null) {
            connection.close();
        }
        log.info("Pipeline shut down cleanly");
    }
}
```

**What changed and why:**
- Constructor only takes mandatory dependency (`dataSource`) — not `@Value` fields
- `@PostConstruct` initializes all resources — fields are guaranteed filled
- `@PreDestroy` cleans up with timeout and interrupt handling
- `executor.awaitTermination` — graceful shutdown, not abrupt kill
- Logging at each phase — visible in production when things go wrong

---

## 🧠 9. Mental Model Summary

### The jargon, decoded

| Term | What it actually means |
|---|---|
| **Bean Lifecycle** | The sequence of events from "Spring reads your class" to "Spring destroys your object" |
| **BeanDefinition** | Spring's metadata record about a class — read at startup, before any objects are made |
| **`@PostConstruct`** | "Call this method after you've finished injecting all my fields" |
| **`@PreDestroy`** | "Call this method before you destroy me so I can clean up" |
| **`InitializingBean`** | A Spring interface. Implementing it = adding an `afterPropertiesSet()` method that Spring calls during init |
| **`DisposableBean`** | A Spring interface. Implementing it = adding a `destroy()` method that Spring calls during shutdown |
| **`init-method`** | A method name you tell Spring to call at init time — useful for third-party classes |
| **`destroy-method`** | A method name you tell Spring to call at shutdown — defaults to `(inferred)` which auto-detects `close()` |
| **BeanPostProcessor (BPP)** | A special bean that intercepts EVERY other bean's lifecycle at two points — before and after init |
| **`BeanNameAware`** | Interface: Spring calls `setBeanName()` to tell your bean its registered name |
| **`ApplicationContextAware`** | Interface: Spring calls `setApplicationContext()` to give your bean access to the full context |
| **AOP Proxy** | A wrapper object Spring creates around your bean to intercept method calls (for `@Transactional`, `@Cacheable`, etc.) |
| **`postProcessAfterInitialization`** | The BPP method where AOP proxies are born — AFTER all init callbacks on the raw bean |
| **Prototype scope** | A bean that is created fresh on every request — Spring doesn't track it after creation |

---

### The lifecycle as a single diagram

```
Spring reads class → BeanDefinition
        │
        ▼
new MyBean()                          ← Constructor
        │ (all fields null/0)
        ▼
@Autowired / @Value filled            ← Dependency Injection
        │ (all fields ready)
        ▼
setBeanName() / setBeanFactory()      ← BeanNameAware, BeanFactoryAware (direct)
        │
        ▼
setApplicationContext()               ← ApplicationContextAware (via BPP)
        │
        ▼
@PostConstruct                        ← FIRST SAFE INIT POINT
        │
        ▼
afterPropertiesSet()                  ← Init (if implements InitializingBean)
        │
        ▼
init-method                           ← Init (if configured)
        │
        ▼
AOP Proxy created HERE ────────────── postProcessAfterInitialization
        │                             (if any advice applies)
        ▼
Bean (or Proxy) stored in cache       ← READY FOR USE
        │
        │ ... application runs ...
        │
        ▼ (on context.close())
@PreDestroy                           ← FIRST SAFE CLEANUP POINT
        │
        ▼
DisposableBean.destroy()
        │
        ▼
destroy-method
        │
        ▼
Bean gone 🪦
```

---

### The init/destroy order — the one thing to memorize

```
INIT ORDER:        @PostConstruct → afterPropertiesSet() → init-method
DESTROY ORDER:     @PreDestroy   → DisposableBean.destroy() → destroy-method

Easy memory trick:
    "Post before Properties, Pre before Disposable"
```

---

### Decision rule — which hook to use

```
Writing your own application bean?
    Use @PostConstruct / @PreDestroy. Always. Done.

Writing a framework/library that deeply integrates with Spring?
    Use InitializingBean / DisposableBean — programmatic guarantee.

Configuring a third-party class you can't annotate?
    Use @Bean(initMethod="start", destroyMethod="stop").

Need access to Spring internals (context, bean name)?
    Implement the relevant Aware interface.
    But first ask: can @Autowired solve this more simply?

Need to intercept every bean in the container?
    Implement BeanPostProcessor.
```
