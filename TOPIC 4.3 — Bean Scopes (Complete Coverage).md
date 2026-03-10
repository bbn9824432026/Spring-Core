# TOPIC 4.3 — Bean Scopes (Complete Coverage)

---

## ① CONCEPTUAL EXPLANATION

### What is a Bean Scope?

A bean scope defines the lifecycle and visibility of a bean instance within the Spring IoC container. Scope answers the question: **"How many instances of this bean exist, and for how long?"**

Spring provides six built-in scopes:

```
Core scopes (all ApplicationContext types):
    1. singleton   — one instance per ApplicationContext
    2. prototype   — new instance per getBean() call

Web-aware scopes (WebApplicationContext only):
    3. request     — one instance per HTTP request
    4. session     — one instance per HTTP session
    5. application — one instance per ServletContext
    6. websocket   — one instance per WebSocket session

Custom scopes:
    7+. Any scope you implement via Scope interface
```

---

### 4.3.1 — Singleton Scope

**Default scope** — if no scope is specified, Spring uses singleton.

```
One instance per ApplicationContext.
NOT one per JVM.
NOT one per ClassLoader.
NOT one per Spring application (if multiple contexts exist).
```

**Internal mechanism:**

```java
// In DefaultSingletonBeanRegistry:
private final Map<String, Object> singletonObjects =
    new ConcurrentHashMap<>(256);  // THE singleton cache

// On first getBean():
// 1. Check singletonObjects → not there
// 2. Create bean (full lifecycle)
// 3. Store in singletonObjects
// 4. Return from singletonObjects for all subsequent getBean() calls
```

**Thread safety:**
The singleton cache uses `ConcurrentHashMap` for thread-safe reads. The creation of a new singleton is synchronized to prevent duplicate creation:

```java
// In DefaultSingletonBeanRegistry.getSingleton():
synchronized (this.singletonObjects) {
    // double-check under lock
    singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
        // create and store
    }
}
```

**Singleton scope and thread safety of the bean itself:**
Spring guarantees ONE instance per context. It does NOT guarantee the bean itself is thread-safe. If multiple threads call methods on the singleton bean, the bean must manage its own thread safety.

```java
@Service  // singleton by default
public class OrderService {
    private int orderCount = 0; // DANGER — shared mutable state

    public void processOrder() {
        orderCount++; // NOT thread-safe — race condition
    }
}
```

**Singleton beans should be stateless** or use thread-safe state management (AtomicInteger, ThreadLocal, synchronized, etc.).

**Singleton lifecycle:**
```
Created: during preInstantiateSingletons() at context refresh (unless @Lazy)
Destroyed: when ApplicationContext.close() is called
```

---

### 4.3.2 — Prototype Scope

```
New instance per getBean() call.
Spring creates and returns a new instance every time.
Spring does NOT track prototype instances after creation.
```

**Internal mechanism:**

```java
// In AbstractBeanFactory.doGetBean():
if (mbd.isPrototype()) {
    Object prototypeInstance;
    try {
        beforePrototypeCreation(beanName);     // add to prototypesCurrentlyInCreation
        prototypeInstance = createBean(beanName, mbd, args);
    } finally {
        afterPrototypeCreation(beanName);      // remove from prototypesCurrentlyInCreation
    }
    return getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
    // No storage — instance returned directly
}
```

**Prototype lifecycle — critical differences from singleton:**

```
CREATION:  Identical to singleton (constructor → DI → Aware → BPP-before → @PostConstruct → BPP-after)
TRACKING:  NOT stored anywhere after creation
DESTROY:   @PreDestroy NEVER called
           DisposableBean.destroy() NEVER called
           Custom destroy-method NEVER called
```

**Memory management:**
Since Spring doesn't track prototypes, the developer is responsible for cleanup. If a prototype bean holds resources (connections, file handles, etc.), the developer must implement cleanup logic and call it manually.

**When to use prototype:**
- Stateful beans — each caller needs its own state
- Shopping carts, user sessions (when not using session scope)
- Command pattern — new command object per request
- Non-thread-safe beans used in multi-threaded contexts

---

### 4.3.3 — Web Scopes — Architecture

Web scopes are only available in a `WebApplicationContext`. They use a `Scope` implementation that ties bean lifecycle to HTTP constructs.

**How web scopes work internally:**

Spring uses a `Scope` interface:

```java
public interface Scope {
    Object get(String name, ObjectFactory<?> objectFactory);
    Object remove(String name);
    void registerDestructionCallback(String name, Runnable callback);
    Object resolveContextualObject(String key); // "request", "session", etc.
    String getConversationId();
}
```

For web scopes, the `Scope` implementation stores bean instances in the HTTP construct (HttpServletRequest attributes for request scope, HttpSession attributes for session scope, etc.).

**Registration of web scopes:**
Web scopes are registered in `postProcessBeanFactory()` — Step 4 of `refresh()`. For servlet-based web apps:

```java
// In AbstractRefreshableWebApplicationContext.postProcessBeanFactory():
ConfigurableListableBeanFactory beanFactory = getBeanFactory();
beanFactory.registerScope("request",  new RequestScope());
beanFactory.registerScope("session",  new SessionScope());
beanFactory.registerScope("application", new ServletContextScope(sc));
```

---

### 4.3.4 — Request Scope

```
One instance per HTTP request.
Bound to the lifecycle of a single HTTP request.
Destroyed when the request completes.
```

**Storage:**
Bean instances are stored as attributes of `HttpServletRequest`:
```java
// Key: "scopedTarget." + beanName
request.setAttribute("scopedTarget.myRequestBean", beanInstance);
```

**Internal mechanism — RequestScope:**

```java
public class RequestScope extends AbstractRequestAttributesScope {
    @Override
    protected int getScope() {
        return RequestAttributes.SCOPE_REQUEST;
    }
    // get() → RequestContextHolder.currentRequestAttributes().getAttribute(name, scope)
    // remove() → removes attribute from request
}
```

`RequestContextHolder` holds the current request in a `ThreadLocal<RequestAttributes>`. This is populated by `DispatcherServlet` (and `RequestContextListener`/`RequestContextFilter` for non-Spring-MVC contexts).

**Usage:**

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private final String requestId = UUID.randomUUID().toString();
    private Map<String, Object> attributes = new HashMap<>();
    // New instance per request — thread safe since one per request
}
```

**`proxyMode` — critical for injection into singletons:**

```java
// PROBLEM: Singleton cannot directly hold a request-scoped bean
@Service  // singleton
public class OrderService {
    @Autowired
    private RequestContext requestCtx; // request scope — injected ONCE at startup
    // requestCtx would always be the SAME instance — defeats the purpose!
}

// SOLUTION: ScopedProxy
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { ... }

@Service  // singleton
public class OrderService {
    @Autowired
    private RequestContext requestCtx;
    // requestCtx is now a CGLIB PROXY
    // Each method call on the proxy → looks up current request's RequestContext instance
    // Different request → different bean instance behind the proxy
}
```

---

### 4.3.5 — ScopedProxy — The Critical Mechanism

ScopedProxy is the mechanism that makes web scopes usable when injected into broader-scoped beans (singletons or session beans).

**The problem:**

```
Singleton (lives for application duration)
    ↓ @Autowired
Request bean (lives for one request)

At singleton creation time: which request bean? There's no request yet.
After singleton creation: singleton holds ONE request bean forever.
This defeats the purpose of request scope.
```

**The solution — ScopedProxy:**

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
// or: proxyMode = ScopedProxyMode.INTERFACES (if bean implements interface)
public class RequestUserContext {
    private User currentUser;
    // ...
}
```

Spring creates a CGLIB proxy (or JDK proxy) that implements the same type as the scoped bean. The singleton receives the PROXY. Every method call on the proxy:

1. Looks up the current scope context (current HTTP request)
2. Finds the actual bean instance for that request
3. If not found, creates a new instance (via ObjectFactory)
4. Delegates the method call to the actual instance

```
[OrderService singleton]
    → holds [ScopedProxy<RequestUserContext>]

Request 1: orderService.processOrder()
    → proxy.getCurrentUser()
    → looks in Request 1's attributes → finds RequestUserContext@1
    → delegates to RequestUserContext@1.getCurrentUser()

Request 2: orderService.processOrder()  (different thread)
    → proxy.getCurrentUser()
    → looks in Request 2's attributes → finds RequestUserContext@2
    → delegates to RequestUserContext@2.getCurrentUser()
```

**ScopedProxy modes:**

```java
// MODE 1: INTERFACES — JDK dynamic proxy (bean must implement interface)
@Scope(value = "request", proxyMode = ScopedProxyMode.INTERFACES)

// MODE 2: TARGET_CLASS — CGLIB proxy (works for classes, no interface required)
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)

// MODE 3: NO (default) — no proxy
// Use only when the bean is injected into same-scope or narrower-scope beans
@Scope(value = "request", proxyMode = ScopedProxyMode.NO)
```

**XML equivalent:**
```xml
<bean id="requestCtx" class="com.example.RequestContext"
      scope="request">
    <aop:scoped-proxy/>  <!-- creates TARGET_CLASS proxy by default -->
</bean>
```

---

### 4.3.6 — Session Scope

```
One instance per HTTP session.
Bound to HttpSession lifecycle.
Destroyed when session expires or is invalidated.
```

**Storage:**
```java
// Key in HttpSession: "scopedTarget." + beanName
session.setAttribute("scopedTarget.userPreferences", beanInstance);
```

**Usage:**

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserPreferences {
    private String theme = "light";
    private Locale locale = Locale.US;
    private List<String> recentSearches = new ArrayList<>();
    // One instance per user session
    // Thread safety: sessions are per-user so typically single-threaded
    // BUT: multiple tabs/requests from same user share same session bean
}
```

**Session scope and multi-tab access:**
A user with multiple browser tabs makes concurrent requests that share the SAME `HttpSession`. Therefore, session-scoped beans CAN be accessed concurrently and may require synchronization.

```java
@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    private final List<Item> items =
        Collections.synchronizedList(new ArrayList<>());
    // synchronized — multiple tabs may add items concurrently
}
```

---

### 4.3.7 — Application Scope

```
One instance per ServletContext.
Functionally similar to singleton for a single-application deployment.
Stored in ServletContext attributes.
```

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_APPLICATION,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class GlobalConfiguration {
    private Map<String, String> settings = new ConcurrentHashMap<>();
    // Shared across all requests and sessions
    // Distinct from singleton: multiple WebApplicationContexts (root + child)
    //   share the same ServletContext, so this is cross-context shared
}
```

**Application scope vs singleton:**
- Singleton: one per `ApplicationContext` (root context or servlet context — separate)
- Application: one per `ServletContext` — shared across ALL ApplicationContexts in the same web app

In a standard Spring MVC app with one root context and one DispatcherServlet context:
- Application-scoped beans are accessible from both contexts via `ServletContext`
- Singleton beans are separate in each context

---

### 4.3.8 — WebSocket Scope

```
One instance per WebSocket session.
Available with Spring WebSocket support.
```

```java
@Component
@Scope(value = "websocket",
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class WebSocketContext {
    private String sessionId;
    private User connectedUser;
    // New instance per WebSocket connection
    // Destroyed when WebSocket session ends
}
```

Requires `WebSocketScope` to be registered:
```java
beanFactory.registerScope("websocket", new WebSocketScope());
```

---

### 4.3.9 — Custom Scope Implementation

Spring's `Scope` interface allows defining custom scopes:

```java
public class ThreadScope implements Scope {

    private final ThreadLocal<Map<String, Object>> threadScope =
        new ThreadLocal<>() {
            @Override
            protected Map<String, Object> initialValue() {
                return new HashMap<>();
            }
        };

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = this.threadScope.get();
        Object object = scope.get(name);
        if (object == null) {
            object = objectFactory.getObject();  // create new instance
            scope.put(name, object);
        }
        return object;
    }

    @Override
    public Object remove(String name) {
        Map<String, Object> scope = this.threadScope.get();
        return scope.remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        // Could store callbacks and run them on thread end
        // Complex to implement correctly for thread pools
    }

    @Override
    public Object resolveContextualObject(String key) {
        return null; // for request/session, this would return the request/session
    }

    @Override
    public String getConversationId() {
        return Thread.currentThread().getName();
    }
}
```

**Registering a custom scope:**

```java
// Method 1: Via BeanFactoryPostProcessor
@Component
public class ThreadScopeRegistrar implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        factory.registerScope("thread", new ThreadScope());
    }
}

// Method 2: Via CustomScopeConfigurer (preferred)
@Bean
public static CustomScopeConfigurer customScopeConfigurer() {
    CustomScopeConfigurer configurer = new CustomScopeConfigurer();
    configurer.addScope("thread", new ThreadScope());
    return configurer; // MUST be static — CustomScopeConfigurer is a BFPP
}
```

**Using the custom scope:**
```java
@Component
@Scope("thread")
public class ThreadLocalUserContext {
    private User currentUser;
    // One instance per thread — useful for thread-pool-based processing
}
```

---

### 4.3.10 — Scope Injection Rules — The Narrowing Scope Problem

The fundamental rule:

```
SAFE: Inject wider-scope bean INTO narrower-scope bean
    singleton → injected into → prototype   ✓
    singleton → injected into → request     ✓
    session   → injected into → request     ✓

UNSAFE/PROBLEM: Inject narrower-scope bean INTO wider-scope bean
    prototype → injected into → singleton   ✗ (prototype-in-singleton problem)
    request   → injected into → singleton   ✗ (needs scoped proxy)
    session   → injected into → singleton   ✗ (needs scoped proxy)
    request   → injected into → session     ✗ (needs scoped proxy)
```

**Solutions for injecting narrower into wider:**

```
Solution 1: ScopedProxy (proxyMode = TARGET_CLASS or INTERFACES)
    → Bean declaration: @Scope(value="request", proxyMode=...)
    → Proxy looks up correct instance on each method call

Solution 2: @Lookup method injection
    → Singleton calls lookup method → returns current scoped instance

Solution 3: ObjectFactory<T> / ObjectProvider<T>
    → Singleton gets ObjectFactory → calls getObject() when needed

Solution 4: ApplicationContext.getBean()
    → Service locator pattern — least elegant
```

---

### 4.3.11 — Scoped Proxy Internal Architecture

When `proxyMode = TARGET_CLASS`, Spring:

1. Creates the proxy class at startup using CGLIB
2. Stores the proxy in the singleton cache (for injection into singletons)
3. Creates a `ScopedProxyFactoryBean` that manages the delegation

The proxy internally holds a reference to:
- The `BeanFactory` (to look up the scoped bean)
- The target bean name with `"scopedTarget."` prefix

At runtime, each method call on the proxy:
```java
// Inside generated CGLIB proxy:
@Override
public Object doSomething() {
    // Look up the real bean for the current scope
    Object target = this.targetSource.getTarget();
    // targetSource is an ScopedObjectTargetSource which calls:
    //    beanFactory.getBean("scopedTarget.myBean")
    // This triggers the Scope.get() mechanism for the current request/session
    try {
        return target.doSomething();
    } finally {
        this.targetSource.releaseTarget(target);
    }
}
```

**ScopedBeanTargetSource:**
```java
public class SimpleBeanTargetSource implements TargetSource {
    private final BeanFactory beanFactory;
    private final String targetBeanName;

    @Override
    public Object getTarget() throws Exception {
        return this.beanFactory.getBean(this.targetBeanName);
    }
}
```

Every method call on a scoped proxy involves a `beanFactory.getBean()` call — this is why scoped proxies have a small performance overhead compared to direct references.

---

### 4.3.12 — `@RequestScope`, `@SessionScope`, `@ApplicationScope`

Spring 4.3 introduced convenience annotations:

```java
// Old way:
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)

// New way (Spring 4.3+):
@RequestScope  // equivalent to above — proxyMode.TARGET_CLASS is default

@SessionScope  // one per HTTP session, TARGET_CLASS proxy

@ApplicationScope  // one per ServletContext, TARGET_CLASS proxy
```

These meta-annotations include `proxyMode = TARGET_CLASS` by default — a convenience that prevents the common mistake of forgetting `proxyMode` when injecting web-scoped beans into singletons.

---

### Scope and AOP Proxy Coexistence

When a scoped bean ALSO has AOP applied:

```java
@Service
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestAuditService {
    @Transactional // AOP proxy also needed
    public void audit(String action) { ... }
}
```

Two proxies are involved:
1. **Scoped proxy** — manages scope lookup (created by ScopedProxyFactoryBean)
2. **AOP proxy** — wraps the actual bean instance (created in postProcessAfterInitialization)

The typical arrangement:
```
[ScopedProxy] → [AOP Proxy] → [Real RequestAuditService]
```

The scoped proxy looks up the current scope context and finds the AOP proxy (which wraps the real bean).

---

### Common Misconceptions

**Misconception 1:** "Singleton scope means thread-safe."
Wrong. Singleton means one instance — it says nothing about thread safety. Thread safety of the bean's behavior is the developer's responsibility.

**Misconception 2:** "Prototype beans are destroyed by Spring."
Wrong. Spring NEVER calls destroy callbacks on prototype beans.

**Misconception 3:** "Web scopes work in any ApplicationContext."
Wrong. Web scopes (request, session, application) only work in a `WebApplicationContext`. Using `@RequestScope` in a non-web context throws `IllegalStateException` at runtime.

**Misconception 4:** "ScopedProxy creates a new instance of the proxy on every method call."
Wrong. The PROXY is a singleton — created once, stored in the singleton cache. The proxy LOOKS UP a new (or existing) scoped instance on each method call.

**Misconception 5:** "Application scope and singleton scope are equivalent."
Wrong. Singleton is per `ApplicationContext`. Application scope is per `ServletContext`. In a Spring MVC app with root context + DispatcherServlet context, singleton beans are separate in each context, but application-scoped beans are shared via `ServletContext`.

**Misconception 6:** "`@Scope("prototype")` with `proxyMode = TARGET_CLASS` makes each call get a new instance."
This is a nuanced one. If a prototype bean with scoped proxy is injected into a singleton, each METHOD CALL on the proxy calls `beanFactory.getBean()` for the prototype — which returns a NEW instance each time. But this is unusual behavior. Normally you would use `ObjectProvider` for this.

**Misconception 7:** "Request scope is always thread-safe."
Not always. A single request can have multiple threads (async processing, parallel streams). Request-scoped beans shared across such threads can have concurrent access issues.

---

## ② CODE EXAMPLES

### Example 1 — Scope Declaration — All Styles

```java
// Singleton (explicit)
@Component
@Scope("singleton") // same as default
public class SingletonBean { }

// Singleton (implicit default)
@Component
public class AlsoSingletonBean { } // singleton by default

// Prototype
@Component
@Scope("prototype")
public class PrototypeBean { }

// Prototype (Spring 4.3+ constant)
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PrototypeBean2 { }

// Request (old style)
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestBean { }

// Request (new style, Spring 4.3+)
@Component
@RequestScope
public class RequestBean2 { } // proxyMode = TARGET_CLASS included

// Session (new style)
@Component
@SessionScope
public class SessionBean { }

// Application (new style)
@Component
@ApplicationScope
public class AppBean { }

// Custom scope
@Component
@Scope("thread")
public class ThreadBean { }
```

---

### Example 2 — Prototype Lifecycle Proof

```java
@Component
@Scope("prototype")
public class StatefulTask {
    private static int instanceCount = 0;
    private final int id;

    public StatefulTask() {
        this.id = ++instanceCount;
        System.out.println("StatefulTask #" + id + " created");
    }

    @PostConstruct
    public void init() {
        System.out.println("StatefulTask #" + id + " @PostConstruct");
        // @PostConstruct IS called for prototypes
    }

    @PreDestroy
    public void destroy() {
        System.out.println("StatefulTask #" + id + " @PreDestroy");
        // @PreDestroy is NEVER called — Spring doesn't track prototypes
    }

    public int getId() { return id; }
}

@Service
public class TaskRunner {
    @Autowired
    private ApplicationContext ctx;

    public void run() {
        StatefulTask t1 = ctx.getBean(StatefulTask.class);
        StatefulTask t2 = ctx.getBean(StatefulTask.class);
        StatefulTask t3 = ctx.getBean(StatefulTask.class);

        System.out.println("Same instance? " + (t1 == t2)); // false
        System.out.println("IDs: " + t1.getId() + ", " + t2.getId() + ", " + t3.getId());
        // IDs: 1, 2, 3 — different instances
    }
}
```

Output:
```
StatefulTask #1 created
StatefulTask #1 @PostConstruct
StatefulTask #2 created
StatefulTask #2 @PostConstruct
StatefulTask #3 created
StatefulTask #3 @PostConstruct
Same instance? false
IDs: 1, 2, 3

[At context.close(): NO @PreDestroy output — never called for prototypes]
```

---

### Example 3 — ScopedProxy in Action

```java
// Request-scoped bean WITH proxy (safe to inject into singleton)
@Component
@RequestScope  // proxyMode = TARGET_CLASS included
public class RequestUserContext {
    private final String requestId = UUID.randomUUID().toString();
    private User currentUser;

    @PostConstruct
    public void init() {
        // Called once per request when this bean is first accessed
        System.out.println("RequestUserContext created for request: " + requestId);
    }

    public String getRequestId() { return requestId; }
    public User getCurrentUser() { return currentUser; }
    public void setCurrentUser(User user) { this.currentUser = user; }
}

// Singleton service holds a proxy — safe
@Service
public class AuditService {
    @Autowired
    private RequestUserContext userContext; // proxy injected at startup

    public void audit(String action) {
        // Each call to userContext.getRequestId() goes through proxy
        // → proxy looks up current request's RequestUserContext instance
        String requestId = userContext.getRequestId(); // different per request
        User user = userContext.getCurrentUser();
        System.out.println("Audit [" + requestId + "]: " + user + " → " + action);
    }

    @PostConstruct
    public void init() {
        // At startup: userContext is a proxy, NOT the real bean
        System.out.println("Is proxy: " +
            AopUtils.isAopProxy(userContext));        // true
        System.out.println("Proxy class: " +
            userContext.getClass().getSimpleName());  // RequestUserContext$$SpringCGLIB$$0

        // Calling getRequestId() here would fail — no HTTP request context!
        // Would throw: java.lang.IllegalStateException: No thread-bound request found
    }
}
```

---

### Example 4 — Session-Scoped Shopping Cart

```java
@Component
@SessionScope  // one cart per user session
public class ShoppingCart {

    private final List<CartItem> items = new ArrayList<>();
    private BigDecimal totalDiscount = BigDecimal.ZERO;

    public void addItem(Product product, int quantity) {
        items.add(new CartItem(product, quantity));
    }

    public List<CartItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    public BigDecimal getTotal() {
        return items.stream()
            .map(item -> item.getProduct().getPrice()
                .multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add)
            .subtract(totalDiscount);
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("ShoppingCart destroyed — session expired");
        // Called when HTTP session expires or invalidated
    }
}

// Singleton controller uses session-scoped cart via proxy
@RestController
public class CartController {

    @Autowired
    private ShoppingCart cart; // session-scoped proxy

    @PostMapping("/cart/add")
    public ResponseEntity<String> addToCart(@RequestBody AddItemRequest req) {
        // cart.addItem() → proxy → HttpSession → current user's ShoppingCart
        cart.addItem(req.getProduct(), req.getQuantity());
        return ResponseEntity.ok("Added to cart");
    }

    @GetMapping("/cart")
    public ResponseEntity<List<CartItem>> getCart() {
        return ResponseEntity.ok(cart.getItems());
    }
}
```

---

### Example 5 — Custom Thread Scope

```java
// Full custom thread scope implementation
public class ThreadScope implements Scope {

    public static final String THREAD_SCOPE = "thread";

    private final ThreadLocal<Map<String, Object>> threadLocal =
        ThreadLocal.withInitial(HashMap::new);

    private final ThreadLocal<Map<String, Runnable>> destructionCallbacks =
        ThreadLocal.withInitial(HashMap::new);

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = threadLocal.get();
        return scope.computeIfAbsent(name, k -> objectFactory.getObject());
    }

    @Override
    public Object remove(String name) {
        return threadLocal.get().remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        destructionCallbacks.get().put(name, callback);
    }

    @Override
    public Object resolveContextualObject(String key) {
        return switch (key) {
            case "thread" -> Thread.currentThread();
            default -> null;
        };
    }

    @Override
    public String getConversationId() {
        return Thread.currentThread().getName() + ":" + Thread.currentThread().getId();
    }

    // Call this when thread work is complete (e.g., end of task in thread pool)
    public void clear() {
        Map<String, Runnable> callbacks = destructionCallbacks.get();
        callbacks.values().forEach(Runnable::run);
        callbacks.clear();
        threadLocal.get().clear();
    }
}

// Register the scope
@Bean
public static CustomScopeConfigurer threadScopeConfigurer() {
    CustomScopeConfigurer configurer = new CustomScopeConfigurer();
    configurer.addScope(ThreadScope.THREAD_SCOPE, new ThreadScope());
    return configurer;
}

// Use the scope
@Component
@Scope(ThreadScope.THREAD_SCOPE)
public class ThreadUserContext {
    private String userId;
    private String tenantId;
    // One per thread — useful for batch processing with thread pools
}
```

---

### Example 6 — Scope Comparison Demo

```java
@Configuration
public class ScopeDemo {

    @Bean @Scope("singleton")
    public Counter singletonCounter() { return new Counter(); }

    @Bean @Scope("prototype")
    public Counter prototypeCounter() { return new Counter(); }
}

@Service
public class ScopeComparisonDemo {
    @Autowired ApplicationContext ctx;

    public void demonstrate() {
        // SINGLETON: same instance every time
        Counter s1 = ctx.getBean("singletonCounter", Counter.class);
        Counter s2 = ctx.getBean("singletonCounter", Counter.class);
        s1.increment();
        System.out.println("Singleton s1 count: " + s1.getCount()); // 1
        System.out.println("Singleton s2 count: " + s2.getCount()); // 1 (same instance!)
        System.out.println("s1 == s2: " + (s1 == s2));               // true

        // PROTOTYPE: new instance every time
        Counter p1 = ctx.getBean("prototypeCounter", Counter.class);
        Counter p2 = ctx.getBean("prototypeCounter", Counter.class);
        p1.increment();
        System.out.println("Prototype p1 count: " + p1.getCount()); // 1
        System.out.println("Prototype p2 count: " + p2.getCount()); // 0 (different instance!)
        System.out.println("p1 == p2: " + (p1 == p2));               // false
    }
}
```

---

### Example 7 — Scoped Proxy Failure (Without proxyMode)

```java
// WRONG: Request-scoped bean WITHOUT proxyMode
@Component
@Scope("request")  // no proxyMode!
public class RequestData {
    private String value = "data";
    public String getValue() { return value; }
}

// Injecting into singleton WITHOUT proxy
@Service
public class DataService {
    @Autowired
    private RequestData requestData; // PROBLEM!

    @PostConstruct
    public void init() {
        // BeanCreationException at startup:
        // Error creating bean with name 'dataService':
        // Unsatisfied dependency for requestData:
        // No thread-bound request found
        // (OR: scope 'request' is not active for current thread)
    }
}
```

```java
// CORRECT: With proxyMode
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestData {
    private String value = "data";
    public String getValue() { return value; }
}

@Service
public class DataService {
    @Autowired
    private RequestData requestData; // PROXY injected — works!
    // Each method call routes to correct request's RequestData
}
```

---

### Example 8 — Prototype Cleanup Pattern

```java
// When you need cleanup for prototype beans
@Component
@Scope("prototype")
public class ResourceHoldingTask implements AutoCloseable {

    private Connection connection;
    private FileOutputStream fileOut;

    @PostConstruct
    public void init() throws Exception {
        this.connection = dataSource.getConnection();
        this.fileOut = new FileOutputStream("/tmp/task-" + hashCode() + ".log");
        System.out.println("Resources acquired for task: " + hashCode());
    }

    public void execute() {
        // Use connection and fileOut
    }

    @Override
    public void close() {
        // Spring WON'T call this — developer must call manually
        // But by implementing AutoCloseable, can use try-with-resources
        try { if (connection != null) connection.close(); } catch (Exception e) { }
        try { if (fileOut != null) fileOut.close(); } catch (Exception e) { }
        System.out.println("Resources released for task: " + hashCode());
    }
}

// Usage pattern with try-with-resources
@Service
public class TaskService {
    @Autowired
    private ObjectProvider<ResourceHoldingTask> taskProvider;

    public void runTask() {
        try (ResourceHoldingTask task = taskProvider.getObject()) {
            // try-with-resources → task.close() called automatically
            task.execute();
        }
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What is the default scope of a Spring bean when no `@Scope` annotation is present?

A) prototype
B) request
C) singleton
D) application

**Answer: C**
Singleton is the default scope. Every `@Component`, `@Service`, `@Repository`, `@Controller`, and `@Bean` without explicit scope is singleton.

---

**Q2 — MCQ:**
A prototype-scoped bean implements `DisposableBean`. When does `destroy()` get called?

A) When `ApplicationContext.close()` is called
B) When the last reference to the prototype instance is garbage collected
C) When `ctx.getBean()` is called to create the next prototype instance
D) Never — Spring does not call destroy callbacks on prototype beans

**Answer: D**
Spring does NOT track prototype beans after creation. No destroy callbacks are ever invoked for prototype beans.

---

**Q3 — Select all that apply:**
Which scopes are ONLY available in a WebApplicationContext? (Select ALL that apply)

A) singleton
B) request
C) prototype
D) session
E) application
F) websocket

**Answer: B, D, E, F**
singleton and prototype are available in ALL ApplicationContext types. request, session, application, and websocket are web-specific scopes requiring `WebApplicationContext`.

---

**Q4 — Code Output Prediction:**

```java
@Component
@Scope("prototype")
public class MyBean {
    static int created = 0;
    static int destroyed = 0;

    @PostConstruct void init()    { created++; }
    @PreDestroy    void destroy() { destroyed++; }
}

@Test
public void test() {
    AnnotationConfigApplicationContext ctx =
        new AnnotationConfigApplicationContext(AppConfig.class);
    ctx.getBean(MyBean.class);
    ctx.getBean(MyBean.class);
    ctx.getBean(MyBean.class);
    ctx.close();
    System.out.println("Created: " + MyBean.created);
    System.out.println("Destroyed: " + MyBean.destroyed);
}
```

What is the output?

A) `Created: 3, Destroyed: 3`
B) `Created: 3, Destroyed: 0`
C) `Created: 1, Destroyed: 1`
D) `Created: 3, Destroyed: 1`

**Answer: B**
Each `getBean()` creates a new prototype instance (3 total), `@PostConstruct` called each time → `created = 3`. `@PreDestroy` NEVER called for prototypes → `destroyed = 0`.

---

**Q5 — Trick Question:**

```java
@Component
@RequestScope
public class RequestBean {
    public String getValue() { return "data"; }
}

@Service
public class MyService {
    @Autowired
    private RequestBean requestBean;

    public String getValue() { return requestBean.getValue(); }
}
```

The application is NOT a web application (plain `AnnotationConfigApplicationContext`). What happens?

A) Works fine — `RequestBean` is created as a regular singleton fallback
B) `BeanCreationException` at startup — request scope not registered
C) `IllegalStateException` only at runtime when `getValue()` is called
D) `NullPointerException` — `requestBean` is null

**Answer: B**
In a non-web `ApplicationContext`, the `request` scope is NOT registered. Attempting to instantiate `MyService` (which triggers the creation of a scoped proxy for `RequestBean`) fails with `BeanCreationException: No Scope registered for scope name 'request'`.

---

**Q6 — Select all that apply:**
Which are valid solutions when a session-scoped bean needs to be injected into a singleton bean? (Select ALL that apply)

A) `@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)`
B) `@Scope(value = "session", proxyMode = ScopedProxyMode.INTERFACES)` (if bean implements interface)
C) `@SessionScope` (which includes `proxyMode = TARGET_CLASS` by default)
D) Inject `ObjectProvider<SessionBean>` into singleton and call `getObject()` per use
E) `@Autowired SessionBean sessionBean` with no proxyMode (injected directly)
F) `@Scope("session")` with no `proxyMode` — Spring resolves it automatically

**Answer: A, B, C, D**
E is wrong — direct `@Autowired` injection of session-scoped bean into singleton without proxy causes `BeanCreationException` or uses wrong instance. F is wrong for the same reason.

---

**Q7 — Scenario-based:**

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String data = "initial";
    public String getData() { return data; }
    public void setData(String d) { this.data = d; }
}

@Service
public class ServiceA {
    @Autowired RequestContext requestContext;

    public void process() {
        requestContext.setData("from-A");
    }
}

@Service
public class ServiceB {
    @Autowired RequestContext requestContext;

    public String read() {
        return requestContext.getData();
    }
}
```

Within the SAME HTTP request, `serviceA.process()` is called, then `serviceB.read()` is called. What does `serviceB.read()` return?

A) `"initial"` — ServiceA and ServiceB have different RequestContext instances
B) `"from-A"` — same RequestContext instance is shared within the same request
C) `null` — data is not propagated through proxies
D) `BeanCreationException` — two beans cannot share the same scoped proxy

**Answer: B**
Within the same HTTP request, the request scope stores ONE `RequestContext` instance in `HttpServletRequest` attributes. Both `serviceA.requestContext` proxy and `serviceB.requestContext` proxy look up the SAME bean from the same request context. `setData("from-A")` mutates the shared instance → `read()` returns `"from-A"`.

---

**Q8 — MCQ:**
What is the difference between singleton scope and application scope in a Spring MVC application with a root `ApplicationContext` and a DispatcherServlet `ApplicationContext`?

A) No difference — both result in one instance per application
B) Singleton: one per each ApplicationContext (two separate instances total). Application: one per ServletContext (shared across both contexts)
C) Singleton: one per JVM. Application: one per ApplicationContext
D) Application scope beans are eagerly initialized; singletons may be lazy

**Answer: B**
In a standard Spring MVC setup, there are TWO ApplicationContexts (root + DispatcherServlet). Singleton beans in each context are SEPARATE — two instances total. Application-scoped beans are stored in `ServletContext` attributes and shared across both contexts — one instance shared.

---

**Q9 — Compilation/Runtime Error:**

```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public final class FinalPrototypeBean {
    public void doWork() { }
}
```

What happens?

A) Works fine — CGLIB can proxy final classes using different techniques
B) `BeanCreationException` at startup — CGLIB cannot subclass final classes
C) Compile error — `proxyMode` cannot be used with `prototype` scope
D) Runtime exception only when `doWork()` is called through the proxy

**Answer: B**
CGLIB creates proxies by subclassing. Final classes cannot be subclassed → `BeanCreationException` at startup when Spring tries to create the CGLIB proxy. Solution: remove `final`, use `ScopedProxyMode.INTERFACES` (if interface available), or use `ObjectProvider<FinalPrototypeBean>` instead.

---

**Q10 — Drag-and-Drop: Match scope to lifecycle boundary:**

Scopes: singleton, prototype, request, session, application

Lifecycle boundaries:
- HTTP session expiry or invalidation
- Each `getBean()` or injection point satisfied
- ApplicationContext `close()`
- First `getBean()` or injection to ApplicationContext `close()`
- `ServletContext` shutdown

**Answer:**
- singleton → `ApplicationContext close()` (created at startup, destroyed at context close)
- prototype → Each `getBean()` or injection (new instance per request; no tracking, no destroy)
- request → HTTP request completion (destroyed after response sent)
- session → HTTP session expiry or invalidation
- application → `ServletContext` shutdown

---

## ④ TRICK ANALYSIS

**Trap 1: Prototype @PreDestroy called at context close**
The most repeated prototype trap. `@PreDestroy`, `DisposableBean.destroy()`, and custom destroy methods are NEVER called for prototype beans. EVER. Spring does not track prototype beans after creation. Any answer suggesting destroy callbacks are called when the context closes is wrong for prototypes.

**Trap 2: Singleton is thread-safe**
Singleton means one instance. It says NOTHING about thread safety. A singleton bean with mutable state accessed from multiple threads is a classic race condition. Thread safety is the developer's responsibility.

**Trap 3: proxyMode default is TARGET_CLASS**
`@Scope("request")` WITHOUT explicit `proxyMode` has `proxyMode = ScopedProxyMode.NO` (no proxy). Direct injection of this unproxied bean into a singleton fails at startup (no request context) or injects the wrong instance. Only the convenience annotations `@RequestScope`, `@SessionScope`, `@ApplicationScope` include `proxyMode = TARGET_CLASS` by default.

**Trap 4: The proxy IS the singleton; the target is the scoped bean**
The scoped proxy itself is a SINGLETON stored in the singleton cache. The PROXY is injected into other singletons. The proxy delegates method calls to the correct SCOPED instance. Questions asking "how many proxy instances are created?" → ONE (singleton). "How many RequestContext instances are created?" → ONE PER REQUEST (scoped).

**Trap 5: @Scope("prototype") with proxyMode**
`@Scope("prototype", proxyMode = TARGET_CLASS)` creates a SINGLETON PROXY that calls `beanFactory.getBean()` (which returns a new prototype instance) on EVERY method call. This gives a different behavior than direct prototype injection — the proxy reference is shared (singleton), but each method call creates a new prototype. Unusual but valid. Exams may present this as a trick.

**Trap 6: Application scope vs singleton in single-context apps**
In a standalone application (non-web) or single-context Spring Boot app, there is effectively no difference between singleton and application scope — both result in one instance. The difference only matters in multi-context web apps (root + DispatcherServlet contexts).

**Trap 7: CustomScopeConfigurer must be static @Bean**
`CustomScopeConfigurer` implements `BeanFactoryPostProcessor`. Like all BFPPs, the `@Bean` method that produces it must be `static`. Non-static BFPP @Bean methods work unreliably. The exam may show a non-static `CustomScopeConfigurer` @Bean and ask what happens — it may not work correctly.

**Trap 8: Web scopes in non-web context**
`@RequestScope`, `@SessionScope`, and `@ApplicationScope` use scope names (`"request"`, `"session"`, `"application"`) that are only registered in `WebApplicationContext`. Using them in a plain `AnnotationConfigApplicationContext` throws `BeanCreationException: No Scope registered for scope name 'request'`. Exams love mixing web-scope annotations into non-web context questions.

**Trap 9: Session scope and multi-tab concurrent access**
Session-scoped beans are NOT inherently thread-safe. Multiple browser tabs from the same user share the same `HttpSession`, potentially causing concurrent access to session-scoped beans. Any exam answer suggesting session-scoped beans are always thread-safe is wrong.

**Trap 10: ScopedProxy type-checking**
`requestBean instanceof RequestBean` on a scoped proxy → may return `false` if using JDK proxy (which implements interfaces, not the class). With CGLIB proxy (`TARGET_CLASS`), `instanceof` returns `true`. `AopUtils.isAopProxy(requestBean)` → `true`. `AopUtils.getTargetClass(requestBean)` → `RequestBean.class`. This is similar to regular AOP proxies.

---

## ⑤ SUMMARY SHEET

### Scope Quick Reference

```
┌─────────────┬────────────────────────────┬────────────────┬─────────────────┐
│ Scope       │ Instance per               │ @PreDestroy    │ Web required?   │
├─────────────┼────────────────────────────┼────────────────┼─────────────────┤
│ singleton   │ ApplicationContext          │ YES (at close) │ No              │
│ prototype   │ getBean() call              │ NEVER          │ No              │
│ request     │ HTTP request               │ YES            │ YES             │
│ session     │ HTTP session               │ YES            │ YES             │
│ application │ ServletContext             │ YES            │ YES             │
│ websocket   │ WebSocket session          │ YES            │ YES             │
└─────────────┴────────────────────────────┴────────────────┴─────────────────┘
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Default scope | singleton |
| Prototype destroy callbacks | NEVER called |
| Web scopes in non-web context | `BeanCreationException` at startup |
| `@RequestScope` proxyMode | `TARGET_CLASS` included by default |
| `@Scope("request")` proxyMode | `NO` by default — must specify explicitly |
| Singleton vs application (multi-context) | Singleton = per context, Application = per ServletContext |
| ScopedProxy is | A SINGLETON proxy that looks up the scoped bean per call |
| Session scope thread safety | NOT guaranteed — multiple tabs share session |
| final class + proxyMode TARGET_CLASS | `BeanCreationException` — CGLIB cannot subclass final |
| CustomScopeConfigurer @Bean | MUST be static — it is a BFPP |

---

### Scope Injection Safety Matrix

```
                     INJECTED INTO:
                 singleton  prototype  request  session  application
BEAN SCOPE:
singleton    →       ✓         ✓         ✓        ✓         ✓
prototype    →       ⚠️*        ✓         ✓        ✓         ✓
request      →       ⚠️**       ⚠️**       ✓        ✓         ✓
session      →       ⚠️**       ⚠️**       ✗***     ✓         ✓
application  →       ✓         ✓         ✓        ✓         ✓

✓ = safe direct injection
⚠️* = prototype-in-singleton problem (use ObjectProvider/@Lookup)
⚠️** = needs scoped proxy (proxyMode = TARGET_CLASS/INTERFACES)
✗*** = session is wider than request — don't inject session into request
```

---

### ScopedProxy Architecture

```
@Service (singleton)
  └── @Autowired RequestContext requestContext
            │
            │  (is actually a singleton CGLIB proxy)
            ▼
       [ScopedProxy] ──────────────────────────────────┐
                                                        │ on each method call:
                                                        ▼
                                           beanFactory.getBean("scopedTarget.requestContext")
                                                        │
                                    Scope.get() → HttpServletRequest attributes
                                                        │
                                         ┌──────────────┴──────────────┐
                                         │                             │
                                  Request 1                      Request 2
                                  RequestContext@1               RequestContext@2
                                  (created on first             (created on first
                                   access in req 1)              access in req 2)
```

---

### Interview One-Liners

- "Singleton scope means one instance per ApplicationContext — NOT per JVM, NOT thread-safe, NOT stateless by definition."
- "Spring never calls destroy callbacks on prototype beans — it does not track prototype instances after creation."
- "Web scopes (request, session, application, websocket) are only available in a WebApplicationContext — using them in a plain ApplicationContext throws BeanCreationException."
- "`@RequestScope`, `@SessionScope`, and `@ApplicationScope` include `proxyMode = TARGET_CLASS` by default — unlike `@Scope('request')` which has no proxy mode."
- "A scoped proxy is a SINGLETON — it is the proxy itself that is stored in the singleton cache and injected into singletons. The proxy delegates each method call to the correct scoped instance."
- "Application scope and singleton scope differ in multi-context web apps: singleton is per ApplicationContext, application scope is per ServletContext (shared across all contexts in the web app)."
- "Session-scoped beans are not inherently thread-safe — multiple browser tabs from the same user can access the same session bean concurrently."
- "`CustomScopeConfigurer` must be a static `@Bean` — it is a `BeanFactoryPostProcessor` and must be instantiated before `@Configuration` CGLIB proxying."

---
