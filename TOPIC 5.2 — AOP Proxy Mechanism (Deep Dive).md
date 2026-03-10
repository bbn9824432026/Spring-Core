# TOPIC 5.2 — AOP Proxy Mechanism (Deep Dive)

---

## ① CONCEPTUAL EXPLANATION

### Why Proxies — The Fundamental Architecture Decision

Spring AOP implements cross-cutting concerns through **proxy-based interception** at runtime. Understanding the proxy mechanism deeply means understanding every implication — when proxies work, when they silently fail, and why Spring made these architectural choices.

```
The Core Contract:
    ctx.getBean(OrderService.class) → returns a PROXY, not OrderService
    The proxy is type-compatible with OrderService
    The proxy intercepts method calls and delegates to advice + target
    The target (real OrderService) is hidden inside the proxy
```

Spring AOP uses two proxy strategies:

```
Strategy 1: JDK Dynamic Proxy
    └── Uses java.lang.reflect.Proxy
    └── Creates proxy implementing target's INTERFACES
    └── Requires: target implements at least one interface

Strategy 2: CGLIB Proxy
    └── Uses Byte Buddy (Spring 4.0+) or CGLIB library
    └── Creates proxy by SUBCLASSING the target class
    └── Requires: target class is not final, methods not final
    └── Default in Spring Boot 2.0+
```

---

### 5.2.1 — JDK Dynamic Proxy — Complete Mechanics

JDK dynamic proxies are created by `java.lang.reflect.Proxy.newProxyInstance()`:

```java
// What Spring does internally to create a JDK proxy:
ClassLoader classLoader = target.getClass().getClassLoader();
Class<?>[] interfaces = getAllInterfaces(target);  // all interfaces implemented by target
InvocationHandler handler = new JdkDynamicAopProxy(advised); // Spring's invocation handler

Object proxy = Proxy.newProxyInstance(classLoader, interfaces, handler);
```

**How JDK Dynamic Proxy works:**

The JVM generates a new class at runtime (e.g., `$Proxy42`) that:
1. Implements ALL interfaces of the target
2. Delegates every method call to the `InvocationHandler`
3. The `InvocationHandler` runs advice + delegates to target

```java
// Spring's JdkDynamicAopProxy (simplified):
public class JdkDynamicAopProxy implements InvocationHandler, AopProxy {

    private final AdvisedSupport advised; // holds target + list of advisors

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // 1. Handle special methods (equals, hashCode, toString)
        if (AopUtils.isEqualsMethod(method)) {
            return equals(args[0]);
        }
        if (AopUtils.isHashCodeMethod(method)) {
            return hashCode();
        }

        // 2. Handle Advised methods (getTargetSource, getAdvisors, etc.)
        if (method.getDeclaringClass() == Advised.class) {
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        // 3. Get the list of applicable interceptors (advice chain)
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
            method, targetClass);

        MethodInvocation invocation;
        if (chain.isEmpty()) {
            // No advice — call target directly (fast path)
            invocation = new ReflectiveMethodInvocation(proxy, target, method, args,
                targetClass, chain);
            return invocation.proceed();
        } else {
            // Build interceptor chain — run advice + target
            invocation = new ReflectiveMethodInvocation(proxy, target, method, args,
                targetClass, chain);
            return invocation.proceed();
        }
    }
}
```

**The method invocation chain (`ReflectiveMethodInvocation`):**

```java
// ReflectiveMethodInvocation.proceed() — the core of Spring AOP interception:
public Object proceed() throws Throwable {
    // If all interceptors have been invoked — call the target method
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint(); // calls target.method(args) via reflection
    }

    // Get next interceptor in chain
    Object interceptorOrInterceptionAdvice =
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    if (interceptorOrInterceptionAdvice instanceof MethodInterceptor mi) {
        // It's a MethodInterceptor — invoke it
        // Each interceptor calls proceed() to continue the chain
        return mi.invoke(this);
    } else {
        // Dynamic interception advice — evaluate pointcut at runtime
        InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        } else {
            // Advice doesn't match at runtime — skip, continue chain
            return proceed();
        }
    }
}
```

**The chain is a recursive call stack:**

```
ReflectiveMethodInvocation.proceed()
    → Interceptor1.invoke(this)
        → this.proceed()  (call from Interceptor1)
            → Interceptor2.invoke(this)
                → this.proceed()  (call from Interceptor2)
                    → invokeJoinpoint()  (actual target method)
                → return result (Interceptor2 post-processing)
            → return result
        → return result (Interceptor1 post-processing)
    → return result
```

This recursive chain is how `@Around` advice wraps both before and after logic around `pjp.proceed()`.

---

### 5.2.2 — CGLIB Proxy — Complete Mechanics

CGLIB (Code Generation Library) — now powered by Byte Buddy in Spring 6 — generates bytecode at runtime to create a SUBCLASS of the target:

```java
// What Spring does to create a CGLIB proxy (simplified):
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(targetClass);          // subclass target
enhancer.setInterfaces(additionalInterfaces); // + Advised, SpringProxy
enhancer.setCallbacks(callbacks);             // array of interceptors
enhancer.setCallbackFilter(filter);           // which callback for which method

Object proxy = enhancer.create();             // instantiate the subclass
```

**CGLIB callback dispatch:**

CGLIB uses a callback filter to decide which `Callback` handles each method:

```java
// CglibAopProxy.ProxyCallbackFilter (simplified):
// Determines callback index for each method:
public int accept(Method method) {
    if (AopUtils.isEqualsMethod(method))     return INVOKE_EQUALS;     // special handling
    if (AopUtils.isHashCodeMethod(method))   return INVOKE_HASHCODE;
    if (isFinalize(method))                  return INVOKE_TARGET;      // pass-through
    if (Advised.class.isAssignableFrom(..))  return INVOKE_ADVISED;
    if (isNoOverrideMethod(method))          return INVOKE_TARGET;      // no interceptors
    if (hasInterceptors(method))             return AOP_PROXY;          // run advice chain
    else                                     return INVOKE_TARGET;      // direct target
}
```

**CGLIB proxy class structure:**

```
Generated class: OrderService$$SpringCGLIB$$0
    extends OrderService
    implements SpringProxy, Advised

    // CGLIB-generated override for each non-final method:
    @Override
    public void placeOrder(Order order) {
        // Get interceptor chain for this method
        MethodInterceptor[] interceptors = CGLIB$CALLBACK_0; // AopProxy callback
        if (interceptors == null || interceptors.length == 0) {
            super.placeOrder(order); // fast path — no advice
        } else {
            // Create CglibMethodInvocation (extends ReflectiveMethodInvocation)
            // Run the same interceptor chain as JDK proxy
            new CglibMethodInvocation(proxy, target, method, args,
                targetClass, interceptors, methodProxy).proceed();
        }
    }
```

**CGLIB uses `MethodProxy` for direct invocation:**

```java
// In CglibMethodInvocation:
@Override
protected Object invokeJoinpoint() throws Throwable {
    // Fast path: MethodProxy.invoke() is FASTER than reflection
    // Does NOT use Method.invoke() — uses bytecode-generated direct call
    return this.methodProxy.invoke(this.target, this.arguments);
}
```

This is why CGLIB proxies are generally faster than JDK proxies — direct method invocation instead of reflection.

---

### 5.2.3 — ProxyFactory — Programmatic Proxy Creation

`ProxyFactory` is Spring's API for creating AOP proxies programmatically:

```java
// Create a proxy programmatically
OrderService target = new OrderServiceImpl();

ProxyFactory factory = new ProxyFactory(target);

// Add advice
factory.addAdvice(new MethodBeforeAdvice() {
    @Override
    public void before(Method method, Object[] args, Object target) {
        System.out.println("Before: " + method.getName());
    }
});

// Add advisor (advice + pointcut)
factory.addAdvisor(new DefaultPointcutAdvisor(
    new AnnotationMatchingPointcut(Transactional.class), // pointcut
    new TransactionInterceptor()                          // advice
));

// Control proxy type
factory.setProxyTargetClass(true);  // force CGLIB
// factory.setInterfaces(OrderRepository.class); // add interface to proxy

// Create the proxy
OrderService proxy = (OrderService) factory.getProxy();
proxy.placeOrder(new Order()); // advice runs ✓
```

**ProxyFactory hierarchy:**

```
ProxyConfig (base config)
    └── AdvisedSupport (adds advisors list + target)
            └── ProxyCreatorSupport (adds AopProxyFactory)
                    ├── ProxyFactory (simplest — create one proxy)
                    ├── ProxyFactoryBean (FactoryBean for Spring container integration)
                    └── AspectJProxyFactory (creates proxy from @Aspect classes)
```

---

### 5.2.4 — Advisor, Advice, and Pointcut Internals

Spring's AOP infrastructure uses these core interfaces:

```java
// Advisor: combines a Pointcut with an Advice
public interface Advisor {
    Advice getAdvice();
}

// PointcutAdvisor: Advisor that has a Pointcut
public interface PointcutAdvisor extends Advisor {
    Pointcut getPointcut();
}

// Advice: the action to take at a JoinPoint
// Base marker interface — subtypes:
//   BeforeAdvice → MethodBeforeAdvice
//   AfterAdvice  → AfterReturningAdvice
//                  ThrowsAdvice
//   Interceptor  → MethodInterceptor (used by @Around)
public interface Advice {}

// Pointcut: defines which joinpoints the advice applies to
public interface Pointcut {
    ClassFilter getClassFilter();  // filters by class
    MethodMatcher getMethodMatcher(); // filters by method
}
```

**How `@Before`, `@After`, etc. are converted to `MethodInterceptor`:**

When Spring processes `@Aspect` classes, each `@Before`, `@AfterReturning`, etc. is wrapped in a `MethodInterceptor` adapter:

```
@Before advice    → AspectJMethodBeforeAdvice → wrapped in MethodBeforeAdviceInterceptor
@AfterReturning   → AspectJAfterReturningAdvice → AfterReturningAdviceInterceptor
@AfterThrowing    → AspectJAfterThrowingAdvice
@After            → AspectJAfterAdvice → implements MethodInterceptor directly
@Around           → AspectJAroundAdvice → implements MethodInterceptor directly
```

All advice types ultimately become `MethodInterceptor` instances that fit into `ReflectiveMethodInvocation`'s chain.

---

### 5.2.5 — The AdvisedSupport — The Brain of Every Proxy

Every proxy holds a reference to an `AdvisedSupport` (or `Advised`) object that maintains:

```java
public interface Advised extends TargetClassAware {

    // Was proxyTargetClass used?
    boolean isProxyTargetClass();

    // All interfaces the proxy implements
    Class<?>[] getProxiedInterfaces();

    // Add/remove/get advisors at RUNTIME (dynamic proxy modification!)
    void addAdvisor(Advisor advisor) throws AopConfigException;
    void removeAdvisor(int index) throws AopConfigException;
    Advisor[] getAdvisors();

    // Get the target source (wraps the actual target object)
    TargetSource getTargetSource();

    // Is the proxy frozen (no more advisor changes)?
    boolean isFrozen();

    // Does this proxy have this advisor?
    boolean adviceIncluded(Advice advice);
}
```

**Runtime advisor modification:**

```java
// You can add/remove advisors to a proxy AFTER creation!
OrderService proxy = ctx.getBean(OrderService.class);

if (proxy instanceof Advised advised) {
    // Add a new advisor at runtime
    advised.addAdvisor(new DefaultPointcutAdvisor(
        Pointcut.TRUE,
        new MethodBeforeAdvice() {
            public void before(Method m, Object[] args, Object t) {
                System.out.println("Dynamically added: " + m.getName());
            }
        }
    ));
    // All subsequent calls to proxy now include this advice
}
```

This is used by Spring Test's `@MockitoBean` / `@SpyBean` and dynamic proxy modification patterns.

---

### 5.2.6 — TargetSource — Advanced Target Management

By default, the proxy holds the target as a `SingletonTargetSource` — the same target object every time. But Spring's `TargetSource` abstraction allows the target itself to be dynamic:

```java
public interface TargetSource extends TargetClassAware {
    Class<?> getTargetClass();
    boolean isStatic();  // true = always same target, can be cached
    Object getTarget() throws Exception;   // called on each invocation
    void releaseTarget(Object target) throws Exception; // return target to pool
}
```

**Built-in TargetSource implementations:**

```java
// 1. SingletonTargetSource (default)
// Same target object every time
new SingletonTargetSource(myService);

// 2. PrototypeTargetSource
// New instance from context for EACH method call
PrototypeTargetSource pts = new PrototypeTargetSource();
pts.setBeanFactory(beanFactory);
pts.setTargetBeanName("myPrototypeBean");
// Every method call creates a new prototype instance

// 3. CommonsPool2TargetSource
// Pools target objects — gets one from pool, releases after call
CommonsPool2TargetSource pool = new CommonsPool2TargetSource();
pool.setMaxSize(10);
pool.setTargetBeanName("expensiveBean");

// 4. ThreadLocalTargetSource
// Different target per thread (thread-local storage)
ThreadLocalTargetSource tl = new ThreadLocalTargetSource();
tl.setTargetBeanName("threadSafeBean");

// 5. HotSwappableTargetSource
// Target can be swapped at runtime without stopping the application
HotSwappableTargetSource swappable =
    new HotSwappableTargetSource(initialTarget);
// ... later:
swappable.swap(newTarget); // all future calls go to newTarget
```

**ScopedProxy uses `TargetSource`:**

The scoped proxy mechanism (Topic 4.3) uses `SimpleBeanTargetSource` which calls `beanFactory.getBean(targetBeanName)` on every method invocation — looking up the request/session-scoped bean from the current context.

---

### 5.2.7 — AspectJ Integration — @EnableAspectJAutoProxy Internals

When `@EnableAspectJAutoProxy` is used, Spring registers `AnnotationAwareAspectJAutoProxyCreator` (AAAPC):

```
@EnableAspectJAutoProxy
    → @Import(AspectJAutoProxyRegistrar.class)
    → registers AnnotationAwareAspectJAutoProxyCreator BeanDefinition
```

**AAAPC's role:**

```java
// AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator
//     extends AbstractAdvisorAutoProxyCreator
//         extends AbstractAutoProxyCreator
//             extends SmartInstantiationAwareBeanPostProcessor

// Key method — postProcessAfterInitialization:
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

// wrapIfNecessary:
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // Skip infrastructure beans (BFPPs, BPPs, aspect beans themselves)
    if (isInfrastructureClass(bean.getClass())) return bean;

    // Find all advisors applicable to this bean
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // Create proxy wrapping this bean with these advisors
        Object proxy = createProxy(bean.getClass(), beanName,
            specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean; // no advice → return raw bean
}
```

**Advisor discovery:**

AAAPC discovers advisors from two sources:

```
1. @Aspect beans:
   → Scans all beans for @Aspect annotation
   → Processes each @Before, @After, etc. method
   → Creates AspectJPointcutAdvisor for each
   → Applies pointcut expression to determine applicability

2. Programmatic advisors:
   → Beans implementing Advisor interface directly
   → Beans implementing PointcutAdvisor
```

**Advisor applicability check:**

```java
// For each bean, check each advisor's pointcut:
// 1. ClassFilter: does this advisor apply to this class?
boolean classMatch = advisor.getPointcut().getClassFilter().matches(targetClass);

// 2. MethodMatcher: does this advisor apply to any method in this class?
MethodMatcher mm = advisor.getPointcut().getMethodMatcher();
for (Method method : allMethods(targetClass)) {
    if (mm.matches(method, targetClass)) {
        // At least one method matches → create proxy for this bean
    }
}
```

---

### 5.2.8 — Proxy Creation Flow — Step by Step

Complete proxy creation walkthrough for a `@Transactional` service:

```
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) { ... }
}
```

```
Step 1: OrderService BeanDefinition registered
         (by ConfigurationClassPostProcessor)

Step 2: AnnotationAwareAspectJAutoProxyCreator registered as BPP
         (by AspectJAutoProxyRegistrar during @EnableAspectJAutoProxy processing)

Step 3: OrderService instantiated (constructor)

Step 4: OrderService populated (DI)

Step 5: Aware callbacks, @PostConstruct, afterPropertiesSet

Step 6: postProcessAfterInitialization() called on AAAPC:
    a. wrapIfNecessary(orderService, "orderService", ...)
    b. Find applicable advisors:
       - BeanFactoryTransactionAttributeSourceAdvisor matches:
         → Pointcut: methods with @Transactional on OrderService → YES
       - Collect all matching advisors
    c. Sort advisors by order
    d. Create proxy:
       - OrderService has no interfaces (assume) → CGLIB
       - Enhancer.create() generates OrderService$$SpringCGLIB$$0
       - Proxy holds: target=orderService, advisors=[TxAdvisor, ...]
    e. Return CGLIB proxy

Step 7: CGLIB proxy stored in singleton cache
         (not the raw OrderService — the PROXY)

Step 8: ctx.getBean(OrderService.class) → returns CGLIB proxy
         @Autowired OrderService → injects CGLIB proxy
```

---

### 5.2.9 — Proxy Introspection — AopUtils and AopProxyUtils

Spring provides utilities for working with proxies:

```java
// Check if object is an AOP proxy
AopUtils.isAopProxy(obj)           // true if JDK or CGLIB proxy
AopUtils.isJdkDynamicProxy(obj)    // true if JDK dynamic proxy
AopUtils.isCglibProxy(obj)         // true if CGLIB proxy

// Get the ultimate target class (unwraps proxy chains)
AopUtils.getTargetClass(obj)        // returns OrderServiceImpl.class
// Note: NOT OrderService$$SpringCGLIB$$0.class

// Get target class (Advised interface)
((TargetClassAware) proxy).getTargetClass()

// Unwrap the actual target object
AopProxyUtils.getSingletonTarget(proxy)  // returns the raw target

// Get all interfaces (including ones added by proxy)
AopProxyUtils.completeProxiedInterfaces(advised, decorateProxy)

// Invoke method on target (bypass proxy)
AopUtils.invokeJoinpointUsingReflection(target, method, args)

// Cast to Advised to inspect proxy config
if (proxy instanceof Advised advised) {
    Advisor[] advisors = advised.getAdvisors();
    TargetSource ts = advised.getTargetSource();
    boolean isFrozen = advised.isFrozen();
}
```

---

### 5.2.10 — exposeProxy and AopContext

`AopContext.currentProxy()` enables a bean to access its own proxy:

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true) // REQUIRED
public class AopConfig { }

@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        // Must route internal calls through proxy for @Transactional to apply
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.validateOrder(order); // through proxy → @Transactional applies
        doPlace(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) {
        // This REQUIRES a new transaction — but only works if called through proxy
    }
}
```

**How `exposeProxy` works:**

When `exposeProxy = true`, the proxy stores itself in a `ThreadLocal` before delegating to the target:

```java
// Inside JdkDynamicAopProxy / CglibAopProxy:
if (this.advised.exposeProxy) {
    // Store proxy in ThreadLocal
    oldProxy = AopContext.setCurrentProxy(proxy);
    setProxyContext = true;
}
try {
    // Invoke target
    retVal = invocation.proceed();
} finally {
    if (setProxyContext) {
        AopContext.setCurrentProxy(oldProxy); // restore previous
    }
}

// AopContext:
public class AopContext {
    private static final ThreadLocal<Object> currentProxy = new ThreadLocal<>();

    public static Object currentProxy() throws IllegalStateException {
        Object proxy = currentProxy.get();
        if (proxy == null) {
            throw new IllegalStateException(
                "Cannot find current proxy: Set 'exposeProxy' property to 'true'");
        }
        return proxy;
    }
}
```

**Performance note:** `exposeProxy = true` adds ThreadLocal read/write overhead to EVERY proxied method call. Use only when necessary.

---

### 5.2.11 — Proxy and equals()/hashCode()/toString()

Special method handling in proxies:

```java
// JdkDynamicAopProxy:
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
        // Special: proxy's equals() compares the ADVISORS and TARGET, not object identity
        return equals(args[0]);
    }
    if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
        // Special: hashCode consistent with equals
        return hashCode();
    }
    if (method.getDeclaringClass() == Advised.class) {
        // Route to Advised implementation
        return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
    }

    // ... normal advice chain invocation
}
```

**Critical implications:**

```java
OrderService bean1 = ctx.getBean("orderService1", OrderService.class);
OrderService bean2 = ctx.getBean("orderService2", OrderService.class);

// Two different proxies wrapping the same target type:
bean1.equals(bean2); // false — different targets
bean1 == bean2;      // false — different proxy objects

// The proxy's hashCode is based on proxy identity, not target identity
// Storing a proxy in a HashSet and retrieving works correctly

// BUT: target.equals(proxy) — calls target.equals() which doesn't know about proxies
// Could return false even if same underlying object
// ALWAYS compare proxy to proxy (not proxy to target)
```

---

### 5.2.12 — Proxy Overhead and Performance Characteristics

```
JDK Dynamic Proxy overhead per call:
    1. Method.invoke() — reflection (expensive: security check + native call)
    2. InvocationHandler.invoke() dispatch
    3. Interceptor chain traversal
    Typical overhead: 2-5x slower than direct call for simple methods

CGLIB overhead per call:
    1. Callback filter lookup (O(1) — precomputed)
    2. MethodProxy.invoke() — faster than reflection (generated bytecode)
    3. Interceptor chain traversal (same as JDK)
    Typical overhead: 1.5-3x slower than direct call

After JIT compilation:
    Both proxy types become very fast — JIT inlines and optimizes the chains
    Measured overhead in production: often < 1 microsecond per call

Performance tips:
    → @Around advice is slightly faster than combining @Before + @After
      (one chain entry vs two)
    → isStatic() TargetSource avoids target lookup overhead per call
    → Frozen proxies (isFrozen=true) allow internal caching optimizations
    → exposeProxy=true adds ThreadLocal overhead to every call
```

---

### 5.2.13 — Proxy Limitations — Complete List

```
1. Self-invocation bypass:
   this.method() inside the bean → bypasses proxy → advice doesn't run

2. Final class/method bypass:
   CGLIB: cannot subclass final class
   CGLIB: cannot override final method → advice doesn't apply

3. Private method bypass:
   Cannot be overridden → not intercepted by CGLIB
   Not in interface → not intercepted by JDK proxy

4. Static method bypass:
   Static methods belong to class, not instance → proxy irrelevant

5. Constructor bypass:
   Spring AOP doesn't intercept constructors
   Use AspectJ CTW/LTW for constructor interception

6. Field access bypass:
   Direct field reads/writes bypass proxy
   Use accessor methods + pointcut expressions

7. Non-Spring-managed object bypass:
   new OrderService() → raw object, no proxy
   Only beans managed by Spring container get proxied

8. Serialization issues:
   JDK proxies are serializable if target and InvocationHandler are
   CGLIB proxies: serialization is complex, often avoided

9. Type casting issues:
   JDK proxy: cannot cast to concrete class (ClassCastException)
       OrderServiceImpl impl = (OrderServiceImpl) ctx.getBean(OrderService.class);
       // ClassCastException — proxy is $Proxy42, not OrderServiceImpl

   CGLIB proxy: CAN cast to concrete class (proxy IS a subclass)
       OrderServiceImpl impl = (OrderServiceImpl) ctx.getBean(OrderService.class);
       // Works with CGLIB — proxy extends OrderServiceImpl

10. Aspect beans themselves are not proxied:
    @Aspect beans are infrastructure — not subject to AOP proxy creation
```

---

### Common Misconceptions

**Misconception 1:** "CGLIB proxy IS the target — they are the same object."
Wrong. CGLIB proxy is a SUBCLASS instance. The target is a SEPARATE object (the `super` of the proxy). `pjp.getTarget()` ≠ `pjp.getThis()`. They occupy different memory addresses.

**Misconception 2:** "JDK proxy is always slower than CGLIB."
Nuanced. Modern JVMs with JIT compilation minimize the performance gap. For most production workloads with reasonable method complexity, the proxy type makes negligible difference. CGLIB was historically faster due to bypassing reflection, but this advantage is smaller in modern JVMs.

**Misconception 3:** "Spring always creates a proxy for beans with @Transactional."
Partially wrong. Spring creates a proxy ONLY if at least one pointcut actually matches the bean. For `@Transactional`, `BeanFactoryTransactionAttributeSourceAdvisor` checks if the class or any of its methods has `@Transactional`. If a class has `@Transactional` but the advisor is not registered (e.g., `@EnableTransactionManagement` not present), no proxy is created.

**Misconception 4:** "Proxy creation happens when `getBean()` is called."
Wrong for singletons. Singleton bean proxies are created during `finishBeanFactoryInitialization()` at `refresh()` time — which calls `postProcessAfterInitialization()` for AAAPC. The proxy is created ONCE at startup, not on every `getBean()`.

**Misconception 5:** "`Advised` interface is only for internal Spring use."
Wrong. Application code can cast proxies to `Advised` to inspect and modify advisors at runtime. This is a legitimate and documented Spring API.

**Misconception 6:** "exposeProxy works with any proxy configuration."
Wrong. `exposeProxy = true` must be explicitly configured in `@EnableAspectJAutoProxy`. Without it, `AopContext.currentProxy()` throws `IllegalStateException`. It also only exposes the proxy for the current thread — it is thread-bound via `ThreadLocal`.

---

## ② CODE EXAMPLES

### Example 1 — Programmatic Proxy Creation (ProxyFactory)

```java
public class ProxyFactoryDemo {

    public void createProxies() {
        OrderServiceImpl target = new OrderServiceImpl();

        // ===== JDK Dynamic Proxy =====
        ProxyFactory jdkFactory = new ProxyFactory(target);
        jdkFactory.setProxyTargetClass(false); // JDK proxy
        jdkFactory.addInterface(OrderService.class); // must specify interface

        jdkFactory.addAdvice(new MethodBeforeAdvice() {
            @Override
            public void before(Method method, Object[] args, Object t) {
                System.out.println("JDK Before: " + method.getName());
            }
        });

        OrderService jdkProxy = (OrderService) jdkFactory.getProxy();
        System.out.println("JDK proxy class: " + jdkProxy.getClass().getName());
        // prints: com.sun.proxy.$Proxy0

        // ===== CGLIB Proxy =====
        ProxyFactory cglibFactory = new ProxyFactory(target);
        cglibFactory.setProxyTargetClass(true); // CGLIB proxy

        cglibFactory.addAdvice(new MethodBeforeAdvice() {
            @Override
            public void before(Method method, Object[] args, Object t) {
                System.out.println("CGLIB Before: " + method.getName());
            }
        });

        OrderService cglibProxy = (OrderService) cglibFactory.getProxy();
        System.out.println("CGLIB proxy class: " + cglibProxy.getClass().getName());
        // prints: com.example.OrderServiceImpl$$SpringCGLIB$$0

        // ===== With Pointcut =====
        ProxyFactory pointcutFactory = new ProxyFactory(target);
        pointcutFactory.setProxyTargetClass(true);

        // Advisor: only intercept methods starting with "place"
        NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
        advisor.setMappedName("place*");
        advisor.setAdvice((MethodBeforeAdvice) (method, args, t) ->
            System.out.println("Intercepted: " + method.getName()));

        pointcutFactory.addAdvisor(advisor);
        OrderService pointcutProxy = (OrderService) pointcutFactory.getProxy();

        pointcutProxy.placeOrder(new Order());  // intercepted ✓
        pointcutProxy.cancelOrder(1L);          // NOT intercepted (doesn't match "place*")
    }
}
```

---

### Example 2 — Interceptor Chain Visualization

```java
// Custom MethodInterceptor to visualize the chain
public class ChainVisualizerInterceptor implements MethodInterceptor {

    private final String name;
    private final int position;

    public ChainVisualizerInterceptor(String name, int position) {
        this.name = name;
        this.position = position;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println(indent(position) + name + " BEFORE");
        try {
            Object result = invocation.proceed(); // continue chain
            System.out.println(indent(position) + name + " AFTER (result=" + result + ")");
            return result;
        } catch (Exception e) {
            System.out.println(indent(position) + name + " AFTER-THROW");
            throw e;
        }
    }

    private String indent(int n) {
        return "  ".repeat(n);
    }
}

// Setup:
ProxyFactory factory = new ProxyFactory(new SimpleService());
factory.setProxyTargetClass(true);
factory.addAdvice(new ChainVisualizerInterceptor("Security",   1));
factory.addAdvice(new ChainVisualizerInterceptor("Transaction", 2));
factory.addAdvice(new ChainVisualizerInterceptor("Logging",    3));

SimpleService proxy = (SimpleService) factory.getProxy();
proxy.doWork();

// Output:
// Security BEFORE
//   Transaction BEFORE
//     Logging BEFORE
//       [doWork() executes]
//     Logging AFTER (result=null)
//   Transaction AFTER (result=null)
// Security AFTER (result=null)
```

---

### Example 3 — HotSwappableTargetSource

```java
@Configuration
public class HotSwapConfig {

    @Bean
    public HotSwappableTargetSource targetSource() {
        return new HotSwappableTargetSource(initialPaymentGateway());
    }

    @Bean
    public PaymentGateway paymentGatewayProxy(HotSwappableTargetSource targetSource) {
        ProxyFactory factory = new ProxyFactory();
        factory.setTargetSource(targetSource);
        factory.addInterface(PaymentGateway.class);
        return (PaymentGateway) factory.getProxy();
    }

    @Bean
    public PaymentGateway initialPaymentGateway() {
        return new StripePaymentGateway();
    }
}

@RestController
public class AdminController {

    @Autowired
    private HotSwappableTargetSource targetSource;

    @PostMapping("/admin/switch-gateway/{name}")
    public ResponseEntity<String> switchGateway(@PathVariable String name) {
        PaymentGateway newGateway = switch (name) {
            case "stripe"  -> new StripePaymentGateway();
            case "paypal"  -> new PayPalPaymentGateway();
            case "braintree" -> new BraintreePaymentGateway();
            default -> throw new IllegalArgumentException("Unknown gateway: " + name);
        };

        // Swap target at runtime — zero downtime!
        Object oldGateway = targetSource.swap(newGateway);
        return ResponseEntity.ok("Switched from " +
            oldGateway.getClass().getSimpleName() + " to " +
            newGateway.getClass().getSimpleName());
    }
}
```

---

### Example 4 — Proxy Introspection

```java
@Service
public class ProxyInspectorService {

    @Autowired
    private OrderService orderService;

    @PostConstruct
    public void inspect() {
        // Basic proxy detection
        System.out.println("Is AOP proxy: " + AopUtils.isAopProxy(orderService));
        System.out.println("Is JDK proxy: " + AopUtils.isJdkDynamicProxy(orderService));
        System.out.println("Is CGLIB proxy: " + AopUtils.isCglibProxy(orderService));

        // Get real target class
        Class<?> targetClass = AopUtils.getTargetClass(orderService);
        System.out.println("Target class: " + targetClass.getName());

        // Get proxy class name
        System.out.println("Proxy class: " + orderService.getClass().getName());

        // Cast to Advised for deep inspection
        if (orderService instanceof Advised advised) {
            System.out.println("Proxy target class mode: " + advised.isProxyTargetClass());
            System.out.println("Expose proxy: " + advised.isExposeProxy());
            System.out.println("Frozen: " + advised.isFrozen());

            System.out.println("Advisors:");
            for (Advisor advisor : advised.getAdvisors()) {
                System.out.println("  " + advisor.getClass().getSimpleName());
                if (advisor instanceof PointcutAdvisor pa) {
                    System.out.println("    Pointcut: " + pa.getPointcut());
                }
            }

            // Get the target source
            TargetSource ts = advised.getTargetSource();
            System.out.println("TargetSource: " + ts.getClass().getSimpleName());
            System.out.println("TargetSource is static: " + ts.isStatic());
        }
    }
}
```

---

### Example 5 — Runtime Advisor Addition

```java
@Service
public class DynamicAdviceService {

    @Autowired
    private UserService userService;

    public void addSecurityAdvice() {
        if (!(userService instanceof Advised advised)) {
            throw new IllegalStateException("userService is not a Spring proxy");
        }

        // Check if already frozen (frozen proxies reject changes)
        if (advised.isFrozen()) {
            throw new AopConfigException("Proxy is frozen — cannot add advisors");
        }

        // Add a new advisor at runtime
        MethodBeforeAdvice securityAdvice = (method, args, target) -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.isAuthenticated()) {
                throw new SecurityException("Not authenticated: " + method.getName());
            }
        };

        advised.addAdvisor(0, // position 0 = first = outermost
            new DefaultPointcutAdvisor(Pointcut.TRUE, securityAdvice));

        System.out.println("Security advice added to userService proxy at runtime");
    }

    public void removeAllCustomAdvisors() {
        if (userService instanceof Advised advised) {
            Advisor[] advisors = advised.getAdvisors();
            for (Advisor advisor : advisors) {
                if (advisor instanceof DefaultPointcutAdvisor) {
                    try {
                        advised.removeAdvisor(advisor);
                    } catch (AopConfigException e) {
                        // Advisor might be required — skip
                    }
                }
            }
        }
    }
}
```

---

### Example 6 — Proxy With PrototypeTargetSource

```java
// Expensive stateful service — need new instance per call
@Component
@Scope("prototype")
public class ExpensiveStatefulService {
    private final String id = UUID.randomUUID().toString();

    public String processData(String input) {
        System.out.println("Processing in instance: " + id);
        return input.toUpperCase();
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Cleaning up instance: " + id);
    }
}

@Configuration
public class PrototypeProxyConfig {

    @Bean
    public PrototypeTargetSource expensiveServiceTargetSource(
            ConfigurableListableBeanFactory beanFactory) {
        PrototypeTargetSource pts = new PrototypeTargetSource();
        pts.setBeanFactory(beanFactory);
        pts.setTargetBeanName("expensiveStatefulService");
        return pts;
    }

    @Bean
    public ExpensiveStatefulService expensiveServiceProxy(
            PrototypeTargetSource targetSource) {
        ProxyFactory factory = new ProxyFactory();
        factory.setTargetSource(targetSource);
        factory.setProxyTargetClass(true);
        return (ExpensiveStatefulService) factory.getProxy();
    }
}

// Usage: every method call creates a new prototype instance
@Service
public class DataProcessor {
    @Autowired
    private ExpensiveStatefulService service; // singleton proxy!

    public void process(List<String> data) {
        for (String item : data) {
            service.processData(item); // new ExpensiveStatefulService per call!
        }
    }
}
```

---

### Example 7 — Self-Invocation via AopContext

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true) // REQUIRED
public class AppConfig { }

@Service
public class OrderProcessingService {

    @Transactional
    public void processOrders(List<Order> orders) {
        // Self-invocation through AopContext.currentProxy()
        OrderProcessingService proxy =
            (OrderProcessingService) AopContext.currentProxy();

        for (Order order : orders) {
            try {
                // Each order in its own transaction (REQUIRES_NEW)
                proxy.processSingleOrder(order); // through proxy → @Transactional works
            } catch (Exception e) {
                // Individual order failure doesn't fail the entire batch
                log.error("Failed to process order {}: {}", order.getId(), e.getMessage());
            }
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processSingleOrder(Order order) {
        // Independent transaction for each order
        orderRepository.save(order);
        inventoryService.deduct(order);
        notificationService.send(order);
    }
}
```

---

### Example 8 — JDK Proxy vs CGLIB Proxy Type Casting

```java
// Setup: both proxy types for same bean
public interface PaymentService {
    void pay(BigDecimal amount);
}

public class PaymentServiceImpl implements PaymentService {
    @Override
    public void pay(BigDecimal amount) { /* ... */ }

    public void refund(BigDecimal amount) { /* only on concrete class */ }
}

@Test
public void proxyTypeCastingTest() {
    PaymentServiceImpl target = new PaymentServiceImpl();

    // ===== JDK PROXY =====
    ProxyFactory jdkFactory = new ProxyFactory(target);
    jdkFactory.setProxyTargetClass(false);
    Object jdkProxy = jdkFactory.getProxy();

    System.out.println(jdkProxy instanceof PaymentService);     // true ✓
    System.out.println(jdkProxy instanceof PaymentServiceImpl); // FALSE ✗ (JDK proxy doesn't extend impl)

    try {
        ((PaymentServiceImpl) jdkProxy).refund(BigDecimal.TEN); // ClassCastException!
    } catch (ClassCastException e) {
        System.out.println("JDK: Cannot cast to concrete class — ClassCastException");
    }

    // ===== CGLIB PROXY =====
    ProxyFactory cglibFactory = new ProxyFactory(target);
    cglibFactory.setProxyTargetClass(true);
    Object cglibProxy = cglibFactory.getProxy();

    System.out.println(cglibProxy instanceof PaymentService);     // true ✓
    System.out.println(cglibProxy instanceof PaymentServiceImpl); // TRUE ✓ (CGLIB extends impl)

    // Can call non-interface method through concrete type cast
    ((PaymentServiceImpl) cglibProxy).refund(BigDecimal.TEN); // works with CGLIB ✓
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What type of object does `ctx.getBean(OrderService.class)` return when `@Transactional` is applied to `OrderService`?

A) An instance of `OrderService` — Spring always returns the real bean
B) A JDK dynamic proxy implementing all interfaces of `OrderService`
C) Either a JDK dynamic proxy or CGLIB subclass — depending on proxy configuration
D) A CGLIB proxy, always — Spring never uses JDK dynamic proxies

**Answer: C**
The proxy type depends on: (1) whether `proxyTargetClass=true` is set, (2) whether the bean implements an interface. Default pure Spring: JDK if interface available, CGLIB otherwise. Default Spring Boot 2+: always CGLIB.

---

**Q2 — Select all that apply:**
Which of the following are TRUE about JDK dynamic proxies? (Select ALL that apply)

A) They require the target to implement at least one interface
B) They can proxy final classes
C) They are created by `java.lang.reflect.Proxy.newProxyInstance()`
D) Method calls go through `InvocationHandler.invoke()`
E) They can intercept final methods
F) Casting to the concrete implementation class works

**Answer: A, C, D**
B wrong — JDK proxy creates a new class, irrelevant to whether target is final. E wrong — JDK proxy only intercepts interface methods; final or not is irrelevant since it only implements the interface. F wrong — JDK proxy class is `$ProxyN`, NOT the concrete class — cast to concrete class throws `ClassCastException`.

---

**Q3 — Code Output Prediction:**

```java
@Aspect
@Component
public class TrackingAspect {
    @Before("execution(* com.example.OrderService.*(..))")
    public void track(JoinPoint jp) {
        System.out.println("Tracking: " + jp.getSignature().getName());
        System.out.println("Target class: " + jp.getTarget().getClass().getSimpleName());
        System.out.println("This class: " + jp.getThis().getClass().getSimpleName());
    }
}

@Service
public class OrderService {
    public void placeOrder() { System.out.println("ORDER PLACED"); }
}
```

With `proxyTargetClass=true` (CGLIB), when `orderService.placeOrder()` is called:

A)
```
Tracking: placeOrder
Target class: OrderService$$SpringCGLIB$$0
This class: OrderService
ORDER PLACED
```
B)
```
Tracking: placeOrder
Target class: OrderService
This class: OrderService$$SpringCGLIB$$0
ORDER PLACED
```
C)
```
Tracking: placeOrder
Target class: OrderService
This class: OrderService
ORDER PLACED
```
D)
```
ORDER PLACED
Tracking: placeOrder
Target class: OrderService
This class: OrderService$$SpringCGLIB$$0
```

**Answer: B**
`jp.getTarget()` = the TARGET (raw bean) = `OrderService`. `jp.getThis()` = the PROXY = `OrderService$$SpringCGLIB$$0`. `@Before` runs before the method → `ORDER PLACED` is last.

---

**Q4 — Trick Question:**

```java
@Service
public class ItemService {
    public final void findItem(Long id) {  // FINAL method
        System.out.println("Finding: " + id);
    }
}

@Aspect
@Component
public class ItemAspect {
    @Before("execution(* com.example.ItemService.findItem(..))")
    public void before(JoinPoint jp) {
        System.out.println("BEFORE findItem");
    }
}
```

With CGLIB proxy (`proxyTargetClass=true`), what happens when `itemService.findItem(1L)` is called?

A) `BEFORE findItem` then `Finding: 1`
B) `Finding: 1` only — final method bypasses CGLIB proxy
C) `BeanCreationException` — cannot proxy class with final methods
D) `UnsupportedOperationException` at runtime

**Answer: B**
CGLIB creates a subclass and overrides methods to apply advice. `final` methods CANNOT be overridden → CGLIB cannot intercept them. The proxy is still created (Spring doesn't fail — it just cannot intercept final methods), but calls to `findItem()` bypass the proxy and go directly to the target. The `@Before` advice NEVER runs. No exception — just silent bypass.

---

**Q5 — MCQ:**
A `@Transactional` singleton bean is requested from the context. At what point is the CGLIB proxy created?

A) When `ctx.getBean()` is called — lazy proxy creation per request
B) During `postProcessAfterInitialization()` — at context startup for all singletons
C) When the first `@Transactional` method is invoked
D) During `postProcessBeforeInitialization()` — before init callbacks

**Answer: B**
`AnnotationAwareAspectJAutoProxyCreator.postProcessAfterInitialization()` creates the proxy during `finishBeanFactoryInitialization()` at context refresh. All singleton beans get their proxies created at startup — NOT lazily on first call.

---

**Q6 — Select all that apply:**
Which of the following scenarios result in the AOP advice NOT being applied? (Select ALL that apply)

A) Calling a `final` method on a CGLIB-proxied bean
B) Calling a method through `@Autowired` injection (the normal way)
C) Calling `this.method()` from within the same bean
D) Calling a method on an object created with `new OrderService()`
E) Calling a `private` method through the proxy
F) Calling a `static` method on the target class

**Answer: A, C, D, E, F**
B is the correct way — advice IS applied. A: final method bypasses CGLIB. C: self-invocation bypasses proxy. D: `new` creates raw object — not a Spring-managed proxy. E: private methods can't be overridden or placed in interfaces — never intercepted. F: static methods bypass proxy.

---

**Q7 — MCQ:**
What does `advised.isFrozen()` returning `true` mean?

A) The proxy's target has been garbage collected
B) No more advisors can be added to or removed from this proxy
C) The proxy cannot intercept any more method calls
D) The `TargetSource` has been frozen and cannot be changed

**Answer: B**
`isFrozen()` returns `true` when the proxy configuration is frozen — no more advisor additions or removals are allowed via the `Advised` interface. The proxy still intercepts method calls normally. It is an optimization hint that allows the proxy to cache the interceptor chain.

---

**Q8 — Scenario-based:**

```java
public interface Auditable {
    String getLastModifiedBy();
}

@Service
public class DocumentService { // Does NOT implement Auditable
    public void save(Document doc) { ... }
}

@Aspect
@Component
public class AuditAspect {
    @DeclareParents(
        value = "com.example.service.DocumentService",
        defaultImpl = DefaultAuditable.class
    )
    public static Auditable auditable;
}
```

After context startup, which of the following works?

A) `ctx.getBean(DocumentService.class).getLastModifiedBy()` — fails, DocumentService doesn't declare method
B) `((Auditable) ctx.getBean(DocumentService.class)).getLastModifiedBy()` — works via introduction
C) `ctx.getBean(Auditable.class)` — works, Auditable is now a registered bean type
D) Both B and C work

**Answer: B**
`@DeclareParents` adds the `Auditable` interface to the proxy of `DocumentService`. The proxy implements both `DocumentService` behavior AND `Auditable`. Casting the proxy to `Auditable` and calling `getLastModifiedBy()` works. However, `ctx.getBean(Auditable.class)` does NOT work — `Auditable` is introduced via the proxy, not registered as a distinct bean. C is wrong.

---

**Q9 — Select all that apply:**
Which statements about `AopContext.currentProxy()` are TRUE? (Select ALL that apply)

A) It requires `exposeProxy = true` in `@EnableAspectJAutoProxy`
B) It returns the proxy for the CURRENT THREAD
C) It works without any special configuration
D) It uses a `ThreadLocal` to store the proxy reference
E) It can be called from outside the bean's own methods
F) It throws `IllegalStateException` if `exposeProxy` is not enabled

**Answer: A, B, D, F**
C wrong — requires `exposeProxy = true`. E partially wrong — it returns whatever proxy is currently on the stack for the thread; if called from a nested AOP invocation it works, but it ALWAYS requires `exposeProxy = true`.

---

**Q10 — Drag-and-Drop: Match proxy mechanism to characteristic:**

Mechanisms: JDK Dynamic Proxy, CGLIB Proxy

Characteristics:
- Requires target to implement at least one interface
- Creates a subclass of the target class
- Uses `java.lang.reflect.Proxy`
- Uses Byte Buddy / CGLIB library
- Cannot proxy final classes
- Cannot intercept final methods (both cannot, but which?)
- Default in Spring Boot 2.0+
- Proxy can be cast to concrete target type
- Invokes target via `MethodProxy` (faster than reflection)
- InvocationHandler.invoke() is the dispatch mechanism

**Answer:**
- JDK: Requires target to implement at least one interface
- CGLIB: Creates a subclass of the target class
- JDK: Uses `java.lang.reflect.Proxy`
- CGLIB: Uses Byte Buddy / CGLIB library
- CGLIB: Cannot proxy final classes (JDK doesn't care about final classes)
- CGLIB: Cannot intercept final methods (JDK cannot intercept non-interface methods)
- CGLIB: Default in Spring Boot 2.0+
- CGLIB: Proxy can be cast to concrete target type
- CGLIB: Invokes target via `MethodProxy` (faster than reflection)
- JDK: InvocationHandler.invoke() is the dispatch mechanism

---

## ④ TRICK ANALYSIS

**Trap 1: CGLIB proxy IS the same object as the target**
Wrong. The CGLIB proxy is a DIFFERENT object — a generated subclass INSTANCE. The target is a SEPARATE object (the `super` state lives in the proxy's inherited fields, but the proxy object reference is distinct). `pjp.getTarget()` ≠ `pjp.getThis()`.

**Trap 2: JDK proxy can cast to concrete class**
Wrong. JDK proxy class is `$ProxyN` which only implements the specified interfaces. Casting to the concrete implementation class throws `ClassCastException`. CGLIB proxies CAN be cast to the concrete class because the proxy IS a subclass.

**Trap 3: Spring creates a proxy for ALL beans**
Wrong. Spring only creates a proxy when at least one advisor's pointcut matches the bean. Beans with no applicable advice get NO proxy — the raw bean object is stored and injected. This is an important performance characteristic.

**Trap 4: exposeProxy works automatically**
Wrong. `exposeProxy = true` MUST be explicitly set on `@EnableAspectJAutoProxy`. Without it, `AopContext.currentProxy()` throws `IllegalStateException` with message "Cannot find current proxy: Set 'exposeProxy' property to 'true'".

**Trap 5: final methods throw exception at proxy creation**
Wrong. CGLIB does NOT fail when it encounters final methods — it simply skips overriding them. The proxy is created successfully. Final methods just bypass the proxy silently. This silent bypass is the dangerous part — no error, no warning, advice just doesn't run.

**Trap 6: Proxy creation is lazy (happens on first call)**
Wrong for singletons. Singleton bean proxies are created eagerly during `finishBeanFactoryInitialization()` at context refresh time. `postProcessAfterInitialization()` runs for ALL singletons. Only `@Lazy` beans would have proxy creation deferred.

**Trap 7: AAAPC is a BFPP**
Wrong. `AnnotationAwareAspectJAutoProxyCreator` is a `SmartInstantiationAwareBeanPostProcessor` (BPP), NOT a BFPP. It registers as a BPP in Step 6 and creates proxies during Step 11 (in `postProcessAfterInitialization()`).

**Trap 8: Advised interface is internal only**
Wrong. Application code can legitimately cast Spring proxies to `Advised` to inspect advisors, check proxy configuration, and even add/remove advisors at runtime. This is a documented public API.

**Trap 9: Both proxy types have identical performance**
Nuanced. CGLIB historically was faster due to `MethodProxy` bypassing reflection. Modern JVMs with JIT compilation largely eliminate this gap. For high-frequency calls (thousands per second), CGLIB may still be marginally faster. For typical application code, the difference is negligible.

**Trap 10: The proxy REPLACES the target in the container**
Partially correct but nuanced. The proxy is what's stored in the singleton cache and returned by `getBean()`. The target still EXISTS as an object — it is held by the proxy's `TargetSource`. The target is NOT garbage collected. The proxy WRAPS, not REPLACES.

---

## ⑤ SUMMARY SHEET

### JDK vs CGLIB Comparison

```
┌──────────────────────────────┬─────────────────────┬─────────────────────┐
│ Characteristic               │ JDK Dynamic Proxy   │ CGLIB Proxy         │
├──────────────────────────────┼─────────────────────┼─────────────────────┤
│ Mechanism                    │ Implements interfaces│ Subclasses target   │
│ Requires interface           │ YES                  │ NO                  │
│ Final class support          │ N/A (not subclassing)│ NO — fails          │
│ Final method interception    │ NO (not in interface)│ NO — silent bypass  │
│ Private method interception  │ NO                   │ NO                  │
│ Static method interception   │ NO                   │ NO                  │
│ Cast to concrete class       │ ClassCastException   │ Works — it's subclass│
│ Invocation dispatch          │ InvocationHandler    │ Callback/MethodProxy│
│ Library                      │ JDK (java.lang.reflect)│ Byte Buddy/CGLIB  │
│ Spring Boot 2+ default       │ NO                   │ YES                 │
│ Performance                  │ Slightly slower      │ Slightly faster     │
│ Serialization                │ Easier               │ Complex             │
└──────────────────────────────┴─────────────────────┴─────────────────────┘
```

---

### Proxy Creation Flow

```
refresh() → finishBeanFactoryInitialization() → preInstantiateSingletons()
    → for each singleton:
        createBean()
            → constructor, DI, Aware, @PostConstruct, afterPropertiesSet
            → postProcessAfterInitialization() [AAAPC]:
                → wrapIfNecessary()
                    → findEligibleAdvisors() — check all advisors' pointcuts
                    → if advisors match → createProxy()
                        → ProxyFactory
                        → proxyTargetClass=true? → CGLIB
                        → has interface? → JDK
                        → Enhancer/Proxy.newInstance()
                    → return proxy (or raw bean if no advisors)
        → store in singletonObjects cache
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Proxy created when | `postProcessAfterInitialization()` — at startup for singletons |
| Proxy created by | `AnnotationAwareAspectJAutoProxyCreator` (BPP, not BFPP) |
| Proxy stored in | L1 singleton cache (in place of raw bean) |
| `pjp.getTarget()` | Raw target bean (NOT the proxy) |
| `pjp.getThis()` | The proxy itself |
| Final method + CGLIB | Silent bypass — no error, no advice |
| JDK proxy + concrete cast | `ClassCastException` |
| CGLIB proxy + concrete cast | Works — proxy IS a subclass |
| No applicable advisors | No proxy created — raw bean injected |
| `AopContext.currentProxy()` | Requires `exposeProxy=true` |
| `Advised` interface | Public API — cast proxy to inspect/modify advisors |
| `isFrozen()` | No more advisor changes allowed — chain cached |

---

### Interceptor Chain Flow

```
Client calls proxy.method()
    ↓
[Proxy] intercepts (InvocationHandler.invoke / CGLIB Callback)
    ↓
ReflectiveMethodInvocation.proceed()
    ↓
Interceptor 1 (Security advice):
    → before code
    → this.proceed()  ←─────────────────────────────────────┐
        ↓                                                   │
    Interceptor 2 (Transaction):                            │
        → begin transaction                                 │
        → this.proceed()  ←──────────────────────────────┐  │
            ↓                                            │  │
        Interceptor 3 (Logging):                         │  │
            → log entry                                  │  │
            → this.proceed()                             │  │
                ↓                                        │  │
            [TARGET METHOD EXECUTES]                     │  │
                ↓                                        │  │
            → log exit                                   │  │
            → return result ──────────────────────────────┘  │
        → commit/rollback transaction                         │
        → return result ──────────────────────────────────────┘
    → after code
    → return result to Client
```

---

### Interview One-Liners

- "Spring creates a proxy ONLY when at least one advisor's pointcut matches the bean — beans with no applicable advice receive NO proxy."
- "JDK dynamic proxy cannot be cast to the concrete implementation class (`ClassCastException`); CGLIB proxy CAN be cast because it IS a subclass."
- "CGLIB silently bypasses `final` methods — no error, no warning, no advice. The method executes directly on the target."
- "`pjp.getTarget()` returns the raw target bean; `pjp.getThis()` returns the proxy — they are different objects at different memory addresses."
- "`AopContext.currentProxy()` requires `exposeProxy = true` in `@EnableAspectJAutoProxy` — without it, the method throws `IllegalStateException`."
- "Spring proxies implement the `Advised` interface — application code can cast to `Advised` to inspect advisors, check configuration, and add/remove advisors at runtime."
- "The interceptor chain in `ReflectiveMethodInvocation` is a recursive structure — each interceptor calls `invocation.proceed()` to pass control to the next interceptor, eventually reaching the target method."
- "`AnnotationAwareAspectJAutoProxyCreator` is a `SmartInstantiationAwareBeanPostProcessor` (BPP) — it creates proxies in `postProcessAfterInitialization()` after all init callbacks run on the raw bean."

---
