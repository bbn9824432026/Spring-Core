# TOPIC 5.1 — AOP Fundamentals and Core Concepts

---

## ① CONCEPTUAL EXPLANATION

### What is AOP?

Aspect-Oriented Programming (AOP) is a programming paradigm that addresses **cross-cutting concerns** — functionality that cuts across multiple modules and cannot be cleanly encapsulated in a single class using object-oriented design alone.

```
WITHOUT AOP — cross-cutting concern scattered everywhere:

OrderService.placeOrder() {
    log.info("Entering placeOrder");          ← logging (duplicated)
    checkSecurity();                           ← security (duplicated)
    long start = System.currentTimeMillis();   ← timing (duplicated)
    // actual business logic
    log.info("Exiting placeOrder");            ← logging (duplicated)
}

UserService.createUser() {
    log.info("Entering createUser");           ← logging (duplicated AGAIN)
    checkSecurity();                           ← security (duplicated AGAIN)
    long start = System.currentTimeMillis();   ← timing (duplicated AGAIN)
    // actual business logic
    log.info("Exiting createUser");            ← logging (duplicated AGAIN)
}

WITH AOP — concern defined once, applied everywhere:

@Aspect
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Entering {}", pjp.getSignature());
        Object result = pjp.proceed();
        log.info("Exiting {}", pjp.getSignature());
        return result;
    }
}
// OrderService and UserService remain clean — no logging code
```

**Classic cross-cutting concerns:**

```
Logging            → record method entry/exit, parameters, return values
Security           → check authorization before method execution
Transaction management → begin/commit/rollback around business methods
Caching            → intercept calls, check cache, populate cache
Performance monitoring → measure execution time
Error handling     → centralized exception handling and translation
Auditing           → record who did what and when
Retry logic        → automatically retry failed operations
Rate limiting      → throttle calls to external services
```

---

### 4.7.2 — AOP Terminology — The Complete Vocabulary

Every AOP concept has a precise definition. Getting these right is essential for certification exams.

#### JoinPoint

A **JoinPoint** is a point during program execution where an aspect can be applied. In Spring AOP, joinpoints are limited to **method executions on Spring-managed beans**.

```
Spring AOP JoinPoints = method execution on Spring beans ONLY

AspectJ JoinPoints (full AOP framework) include:
    - Method execution
    - Method call
    - Constructor execution
    - Constructor call
    - Field get/set
    - Static initializer execution
    - Exception handler execution
    - Object initialization

Spring AOP is a SUBSET of AspectJ — method execution only
```

```java
// JoinPoint examples:
orderService.placeOrder(order);        // method execution → joinpoint ✓
userRepository.save(user);             // method execution → joinpoint ✓
SomeClass.staticMethod();              // STATIC method → NOT a Spring AOP joinpoint ✗
new MyObject();                        // constructor → NOT a Spring AOP joinpoint ✗
myObject.field = value;               // field set → NOT a Spring AOP joinpoint ✗
```

#### Pointcut

A **Pointcut** is a predicate that matches JoinPoints. It defines WHICH joinpoints an advice should apply to. A pointcut is expressed using pointcut expressions.

```
Pointcut = "the set of joinpoints I care about"

"All methods in com.example.service package"
"All methods named 'save' on any class"
"All methods annotated with @Transactional"
"All methods that return void"
"All methods on classes annotated with @Repository"
```

```java
// Pointcut declaration:
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}  // pointcut method — empty body, serves as reference

// Pointcut expression: "execution(* com.example.service.*.*(..))"
//   execution     → joinpoint kind (method execution)
//   *             → any return type
//   com.example.service.*.* → any class in service package, any method
//   (..)          → any parameters
```

#### Advice

**Advice** is the ACTION taken at a joinpoint. It is the actual code that runs when a matched joinpoint is reached.

```
Five advice types:

@Before       → runs BEFORE the method executes
@After        → runs AFTER the method (regardless of success/exception)
@AfterReturning → runs AFTER successful method completion
@AfterThrowing  → runs AFTER the method throws an exception
@Around       → runs AROUND the method (before AND after) — most powerful
```

```
Execution timeline:

Method execution →
    @Before advice
    → [target method executes]
    → [method succeeds]    → @AfterReturning advice
    → [method throws]      → @AfterThrowing advice
    → [always]             → @After advice

@Around advice wraps the ENTIRE sequence (before + proceed + after)
```

#### Aspect

An **Aspect** is the module that combines Pointcuts + Advice. It is the complete unit of cross-cutting concern.

```java
@Aspect  // This is an Aspect
@Component
public class AuditAspect {

    // Pointcut — defines WHERE
    @Pointcut("@annotation(com.example.Auditable)")
    public void auditableOperation() {}

    // Advice — defines WHAT happens
    @Around("auditableOperation()")
    public Object audit(ProceedingJoinPoint pjp) throws Throwable {
        AuditEntry entry = AuditEntry.start(pjp);
        try {
            Object result = pjp.proceed();
            entry.success(result);
            return result;
        } catch (Exception e) {
            entry.failure(e);
            throw e;
        } finally {
            auditRepository.save(entry);
        }
    }
}
```

#### Target Object

The **Target Object** is the original bean that the advice is applied to — the object BEING advised. Spring AOP wraps the target in a proxy.

```java
// Target: the actual OrderService bean
public class OrderService {
    public void placeOrder(Order order) { /* business logic */ }
}

// When AOP applies:
OrderService target = new OrderService();              // target object
OrderService proxy = createProxy(target, advice);     // proxy wraps target
ctx.getBean(OrderService.class) → returns PROXY, not target
```

#### Proxy

The **Proxy** is the object that wraps the target object. In Spring AOP, proxies implement the same interface as the target (JDK dynamic proxy) or subclass the target (CGLIB proxy).

```
Client → [Proxy] → [Target]
             ↑
         Advice runs here
         (before/after delegating to target)
```

The proxy is what gets stored in the Spring container and what `@Autowired` injects.

#### Weaving

**Weaving** is the process of linking aspects with application types or objects to create an advised object.

```
Three types of weaving:

1. Compile-time weaving (AspectJ compiler / ajc)
   → Aspect code woven into .class files at compile time
   → Fastest — no runtime overhead
   → Requires AspectJ compiler

2. Load-time weaving (LTW)
   → Aspect code woven when .class files are loaded by ClassLoader
   → Uses Java agent: -javaagent:aspectjweaver.jar
   → Works on all joinpoints (including constructors, fields)
   → Medium overhead

3. Runtime weaving (Spring AOP default)
   → Proxy created at runtime wrapping the target
   → Only method execution joinpoints
   → No special compiler or agent needed
   → Small overhead per method call
```

**Spring AOP uses RUNTIME WEAVING exclusively** — it creates JDK dynamic proxies or CGLIB proxies at runtime (in `postProcessAfterInitialization()`).

#### Introduction (Inter-type Declaration)

An **Introduction** allows adding new methods or fields to existing classes without modifying them. In Spring AOP, introductions add new interface implementations to existing beans.

```java
// Add Auditable interface to ALL service beans without touching them
@Aspect
@Component
public class AuditableIntroductionAspect {

    @DeclareParents(
        value = "com.example.service.*+",  // all classes in service package
        defaultImpl = DefaultAuditable.class
    )
    public static Auditable mixin; // introduce Auditable to matched beans

    // Now ALL service beans implement Auditable!
}

// Usage:
OrderService orderService = ctx.getBean(OrderService.class);
((Auditable) orderService).getLastModified(); // works — via mixin!
```

---

### 5.1.3 — Pointcut Expression Language — Complete Syntax

Spring AOP uses AspectJ's pointcut expression language. The most important designators:

#### execution() — Method Execution

The most commonly used designator:

```
execution(modifiers? return-type declaring-type? name-signature throws?)

Pattern symbols:
*     → wildcard: matches any single element
..    → matches zero or more elements (in package or param list)
+     → matches subclasses (in type patterns)
```

```java
// Exact method:
execution(public void com.example.OrderService.placeOrder(Order))

// Any return type, any method name, any params in OrderService:
execution(* com.example.OrderService.*(..))

// Any method starting with "save" in any class:
execution(* *.save*(..))

// Any method with exactly one parameter of type String:
execution(* *.*(*) && args(java.lang.String))

// Any public method in any class:
execution(public * *(..))

// Any method returning String:
execution(String *(..))

// Any method in com.example.service or sub-packages:
execution(* com.example.service..*(..))
//                              ↑↑ double-dot = zero or more packages

// Any method in classes implementing OrderRepository:
execution(* com.example.repository.OrderRepository+.*(..))
//                                              ↑ + = subclasses/implementations

// Any method with no parameters:
execution(* *.*())

// Any method with first param String, rest anything:
execution(* *.*(String, ..))
```

#### within() — Type Scope

Matches joinpoints WITHIN certain types:

```java
// All methods in the service package
within(com.example.service.*)

// All methods in service package AND sub-packages
within(com.example.service..*)

// All methods in OrderService and its subclasses
within(com.example.OrderService+)

// Equivalent to execution(* com.example.service.*.*(..))
within(com.example.service.*)
```

**`within()` vs `execution()`:**

```
within(com.example.service.*) → matches ANY joinpoint in the type
execution(* com.example.service.*.*(..)) → matches only method execution joinpoints

For Spring AOP (method execution only): they are effectively equivalent
For AspectJ (all joinpoint types): within() catches all joinpoints including field access
```

#### @annotation() — Annotation on Method

Matches methods annotated with a specific annotation:

```java
// Methods annotated with @Transactional
@annotation(org.springframework.transaction.annotation.Transactional)

// Methods annotated with custom @Auditable
@annotation(com.example.Auditable)

// Combined: @Transactional AND in service layer
@annotation(Transactional) && within(com.example.service.*)
```

#### @within() — Annotation on Type

Matches all methods in classes annotated with a specific annotation:

```java
// All methods in classes annotated with @Service
@within(org.springframework.stereotype.Service)

// All methods in classes annotated with @Repository
@within(org.springframework.stereotype.Repository)
```

**`@annotation()` vs `@within()`:**

```
@annotation(Transactional) → method itself has @Transactional
@within(Transactional)     → class has @Transactional
                             (but individual method-level @Transactional is ignored)
```

#### args() — Method Parameters

Matches methods with specific parameter types at runtime:

```java
// Methods with a single Long parameter
args(Long)

// Methods with first param Long, rest anything
args(Long, ..)

// Bind parameter to advice variable (see below)
args(orderId, ..)  // binds first arg to 'orderId' variable in advice
```

**`args()` vs `execution()` parameter matching:**

```
execution(* *.*(Long)) → matches at COMPILE TIME based on declared param type
                          subtypes do NOT match

args(Long) → matches at RUNTIME based on actual argument type
              subtypes DO match (Long, or any subtype at runtime)
```

#### @args() — Annotation on Parameters

Matches methods where the actual argument at runtime is annotated:

```java
// Methods where first argument's type is annotated with @Entity
@args(javax.persistence.Entity)

// Any method receiving an object of an @Entity-annotated class
```

#### target() and this() — Object Type Matching

```java
// target() — matches when the TARGET object is an instance of the type
target(com.example.repository.Repository)

// this() — matches when the PROXY is an instance of the type
this(com.example.repository.Repository)
```

**`target()` vs `this()`:**

```
For JDK dynamic proxy (implements interface):
    proxy   implements: OrderRepository (interface)
    target  is:         OrderRepositoryImpl (concrete class)

    this(OrderRepository)       → matches (proxy implements interface)
    this(OrderRepositoryImpl)   → does NOT match (proxy does NOT extend impl)
    target(OrderRepository)     → matches (target implements interface)
    target(OrderRepositoryImpl) → matches (target IS the impl)

In practice for Spring AOP:
    target() → use when matching the actual implementation
    this()   → use when matching the interface/proxy type
```

#### bean() — Spring Bean Name

Spring AOP extension (not in AspectJ):

```java
// Match specific bean by name
bean(orderService)

// Match beans by name pattern
bean(*Service)           // all beans ending in "Service"
bean(order*)             // all beans starting with "order"
bean(*Repository)        // all repository beans
```

#### Combining Pointcut Expressions

```java
// AND: both conditions must match
execution(* com.example.service.*.*(..)) && @annotation(Transactional)

// OR: either condition matches
execution(* com.example.service.*.*(..)) || execution(* com.example.web.*.*(..))

// NOT: condition must not match
execution(* com.example.service.*.*(..)) && !execution(* com.example.service.*.get*(..))

// Symbols also work (same as keywords):
&&  → and
||  → or
!   → not
```

---

### 5.1.4 — Advice Types — Deep Dive

#### @Before

```java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void checkSecurity(JoinPoint jp) {
        // JoinPoint provides context about the intercepted method
        String methodName = jp.getSignature().getName();
        Object[] args = jp.getArgs();
        Object target = jp.getTarget();

        // Security check
        if (!SecurityContextHolder.getContext().isAuthenticated()) {
            throw new SecurityException("Not authenticated: " + methodName);
        }
    }
}
```

**`@Before` cannot prevent method execution** — it can only throw an exception. If `@Before` throws, the target method never runs. If it completes normally, the target method runs.

#### @After (Finally Advice)

```java
@Aspect
@Component
public class CleanupAspect {

    @After("execution(* com.example.service.*.*(..))")
    public void cleanup(JoinPoint jp) {
        // Runs ALWAYS — whether method succeeded or threw
        // Like a finally block
        // Cannot influence return value or exception propagation
        MDC.clear(); // clear logging context
    }
}
```

**`@After` has no access to return value or exception.** It is equivalent to a `finally` block.

#### @AfterReturning

```java
@Aspect
@Component
public class CachingAspect {

    @AfterReturning(
        pointcut = "execution(* com.example.service.*.find*(Long))",
        returning = "result"  // bind return value to this parameter name
    )
    public void cacheResult(JoinPoint jp, Object result) {
        // 'result' is the actual return value of the method
        // Runs ONLY if method completed successfully (no exception)
        String key = jp.getSignature().getName() + ":" + jp.getArgs()[0];
        cache.put(key, result);
    }
}
```

**`@AfterReturning` cannot change the return value** — it receives the value but cannot substitute a different one. For that, use `@Around`.

#### @AfterThrowing

```java
@Aspect
@Component
public class ExceptionLoggingAspect {

    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex"  // bind exception to this parameter name
    )
    public void logException(JoinPoint jp, Exception ex) {
        // 'ex' is the actual thrown exception
        // Runs ONLY if method threw an exception
        log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage(), ex);
        // CANNOT suppress the exception — it continues propagating
        // Can throw a DIFFERENT exception to replace the original
    }
}
```

**`@AfterThrowing` cannot suppress exceptions.** The exception always propagates after this advice runs. To suppress or translate exceptions, use `@Around`.

#### @Around — The Most Powerful

```java
@Aspect
@Component
public class PerformanceAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object measurePerformance(ProceedingJoinPoint pjp) throws Throwable {
        // ProceedingJoinPoint — extends JoinPoint with proceed()
        long start = System.nanoTime();
        try {
            // Call pjp.proceed() to invoke the target method
            Object result = pjp.proceed();
            // CAN modify the return value:
            return result; // or: return transformResult(result);
        } catch (Exception e) {
            // CAN suppress the exception:
            // return defaultValue; // suppress — return default instead
            // CAN translate the exception:
            // throw new CustomException("wrapped", e);
            throw e; // re-throw original
        } finally {
            long elapsed = System.nanoTime() - start;
            metrics.record(pjp.getSignature().toShortString(), elapsed);
        }
    }
}
```

**`@Around` uniquely can:**
- Prevent the target method from running (don't call `pjp.proceed()`)
- Call `pjp.proceed()` multiple times (retry logic)
- Modify arguments before passing to target: `pjp.proceed(newArgs)`
- Replace the return value
- Suppress exceptions (return a value instead of re-throwing)
- Translate exceptions

**`@Around` MUST call `pjp.proceed()` or return a value.** If you forget `proceed()`, the target method never runs.

---

### 5.1.5 — Advice Execution Order

When multiple advice applies to the same joinpoint:

```
WITHIN the same @Aspect (same aspect, multiple advice):
    @Around before → @Before → [target method] → @Around after → @AfterReturning → @After

ACROSS different @Aspects:
    Order is UNDEFINED unless explicitly specified
    Use @Order on the @Aspect class or implement Ordered

@Order(1) Aspect (lower = higher priority = outer):
    @Around before
    → @Order(2) Aspect:
        @Around before
        → [target method]
        → @Around after
    → @Around after
```

**Order visualization:**

```
Aspects ordered by @Order (lower value = outermost wrapper):

@Order(1) Aspect (outermost):
├── @Around before
│   @Order(2) Aspect:
│   ├── @Around before
│   │   @Order(3) Aspect:
│   │   ├── @Around before
│   │   │   @Before advices (innermost)
│   │   │   [TARGET METHOD]
│   │   │   @AfterReturning / @AfterThrowing
│   │   │   @After
│   │   └── @Around after
│   └── @Around after
└── @Around after
```

**Within a single aspect, advice ORDER for the same joinpoint:**

```
@Around (before proceed)
@Before
[target method executes]
@AfterReturning (or @AfterThrowing)
@After
@Around (after proceed)
```

---

### 5.1.6 — JoinPoint and ProceedingJoinPoint API

```java
public interface JoinPoint {

    // Signature of the intercepted method
    Signature getSignature();
    // → MethodSignature for method execution joinpoints

    // Arguments passed to the method
    Object[] getArgs();

    // The TARGET object (NOT the proxy)
    Object getTarget();

    // The PROXY object (THIS)
    Object getThis();

    // String representation
    String toString();
    String toShortString();
    String toLongString();

    // Static part of the joinpoint (compile-time info)
    JoinPoint.StaticPart getStaticPart();
}

public interface ProceedingJoinPoint extends JoinPoint {

    // Proceed with the target method using ORIGINAL args
    Object proceed() throws Throwable;

    // Proceed with MODIFIED args
    Object proceed(Object[] args) throws Throwable;
}

// MethodSignature (subinterface of Signature):
public interface MethodSignature extends Signature {
    Class<?> getReturnType();
    Method getMethod();
    Class<?>[] getParameterTypes();
    String[] getParameterNames();
    Class<?>[] getExceptionTypes();
}
```

**Using the API:**

```java
@Around("execution(* com.example.service.*.*(..))")
public Object adviceWithFullContext(ProceedingJoinPoint pjp) throws Throwable {

    // Method information
    MethodSignature sig = (MethodSignature) pjp.getSignature();
    Method method = sig.getMethod();
    String methodName = sig.getName();
    Class<?> returnType = sig.getReturnType();
    Class<?>[] paramTypes = sig.getParameterTypes();

    // Arguments
    Object[] args = pjp.getArgs();

    // Target and proxy
    Object target = pjp.getTarget();  // actual OrderServiceImpl instance
    Object proxy  = pjp.getThis();    // the CGLIB/JDK proxy

    // Check for annotations on the method
    Transactional tx = method.getAnnotation(Transactional.class);
    boolean isReadOnly = tx != null && tx.readOnly();

    // Modify arguments before proceeding
    Object[] newArgs = Arrays.copyOf(args, args.length);
    newArgs[0] = sanitize(newArgs[0]); // sanitize first arg
    Object result = pjp.proceed(newArgs);

    return result;
}
```

---

### 5.1.7 — Argument Binding in Advice

Advice methods can bind JoinPoint context to named parameters:

```java
@Aspect
@Component
public class BindingAspect {

    // Bind the target object
    @Before("execution(* com.example.service.*.*(..)) && target(service)")
    public void beforeWithTarget(JoinPoint jp, Object service) {
        System.out.println("Target: " + service.getClass().getSimpleName());
    }

    // Bind specific argument
    @Before("execution(* com.example.service.OrderService.placeOrder(..)) && args(order)")
    public void beforeWithArg(Order order) {
        // 'order' is the first (and only) argument
        System.out.println("Placing order: " + order.getId());
    }

    // Bind return value
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.find*(..))",
        returning = "result")
    public void afterWithResult(Object result) {
        System.out.println("Result: " + result);
    }

    // Bind thrown exception
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex")
    public void afterWithException(RuntimeException ex) {
        // Only binds RuntimeException or subtypes
        // Other exception types are NOT intercepted by this advice
        log.error("Runtime exception: ", ex);
    }

    // Bind annotation from the method
    @Before("@annotation(transactional)")
    public void beforeTransactional(Transactional transactional) {
        // 'transactional' is the actual @Transactional annotation instance
        System.out.println("Propagation: " + transactional.propagation());
        System.out.println("ReadOnly: " + transactional.readOnly());
    }
}
```

---

### 5.1.8 — Proxy Mechanism — JDK vs CGLIB

Spring AOP creates proxies in `postProcessAfterInitialization()` (via `AbstractAutoProxyCreator`).

**JDK Dynamic Proxy:**

```
Requirements: Target must implement at least one interface
Mechanism:    java.lang.reflect.Proxy creates a proxy class implementing the same interfaces
Limitation:   Proxy only intercepts calls through the INTERFACE
              Direct calls to non-interface methods bypass the proxy

Proxy structure:
    [Client] → [JDK Proxy implements OrderRepository] → [OrderRepositoryImpl]

interface OrderRepository { Order findById(Long id); }
class OrderRepositoryImpl implements OrderRepository { ... }

// JDK proxy intercepts:
orderRepository.findById(1L)  → intercepted ✓ (interface method)

// JDK proxy does NOT intercept:
((OrderRepositoryImpl)orderRepository).internalMethod()  → ClassCastException
//  (proxy IS a Proxy$$... not OrderRepositoryImpl — cast fails)
```

**CGLIB Proxy:**

```
Requirements: Target must NOT be final
              Method must NOT be final
              Target must have no-arg constructor (or usable constructor)
Mechanism:    CGLIB (Code Generation Library) generates a SUBCLASS of the target
              Subclass overrides all non-final methods to add advice
Limitation:   Cannot intercept final methods (cannot override)
              Cannot proxy final classes

Proxy structure:
    [Client] → [CGLIB Proxy extends OrderService] → [OrderService (via super())]

class OrderService { void placeOrder(Order o) {...} }
// CGLIB generates:
class OrderService$$SpringCGLIB$$0 extends OrderService {
    @Override
    void placeOrder(Order o) {
        // advice logic
        super.placeOrder(o);
        // after advice
    }
}
```

**Proxy selection rules:**

```
proxyTargetClass = false (default for @EnableAspectJAutoProxy):
    → Has interface? → JDK dynamic proxy
    → No interface?  → CGLIB proxy

proxyTargetClass = true:
    → Always CGLIB proxy (even for beans with interfaces)

Spring Boot default:
    → proxyTargetClass = true (since Spring Boot 2.0)
    → Always CGLIB
```

```java
// Force CGLIB globally:
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class AopConfig { }

// Or in Spring Boot application.properties:
// spring.aop.proxy-target-class=true  (default in Spring Boot 2+)
```

---

### 5.1.9 — Self-Invocation Problem

The most critical and misunderstood Spring AOP limitation:

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        validate(order);     // SELF-INVOCATION — proxy bypassed!
        // ...
    }

    @Transactional  // This will NOT be applied when called via placeOrder()!
    public void validate(Order order) {
        // ...
    }
}
```

**Why self-invocation bypasses the proxy:**

```
External call:
    Client → [Proxy] → validate()  ← advice RUNS ✓

Self-invocation:
    Client → [Proxy] → placeOrder() → validate()
                                  ↑
                        'this' reference inside placeOrder()
                        points to the TARGET object (not proxy)
                        → advice does NOT run ✗
```

**Solutions:**

```java
// Solution 1: Inject self via @Autowired (gets the proxy)
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // proxy injected

    public void placeOrder(Order order) {
        self.validate(order);   // goes through proxy → @Transactional applies ✓
    }

    @Transactional
    public void validate(Order order) { ... }
}

// Solution 2: ApplicationContext.getBean()
@Service
public class OrderService implements ApplicationContextAware {
    private ApplicationContext ctx;

    public void placeOrder(Order order) {
        ctx.getBean(OrderService.class).validate(order); // through proxy ✓
    }
}

// Solution 3: AopContext.currentProxy() (requires exposeProxy=true)
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)  // MUST enable
public class AopConfig { }

@Service
public class OrderService {
    public void placeOrder(Order order) {
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.validate(order); // through proxy ✓
    }
}

// Solution 4: Refactor — move validate() to a DIFFERENT bean
@Service
public class OrderValidator {
    @Transactional
    public void validate(Order order) { ... }
}

@Service
public class OrderService {
    @Autowired OrderValidator validator;
    public void placeOrder(Order order) {
        validator.validate(order); // different bean → through proxy ✓
    }
}
// Solution 4 is the CLEANEST approach — preferred
```

---

### 5.1.10 — @EnableAspectJAutoProxy — The Configuration Entry Point

```java
@Configuration
@EnableAspectJAutoProxy
// Equivalent to XML: <aop:aspectj-autoproxy/>
public class AopConfig { }
```

**What `@EnableAspectJAutoProxy` does:**

Registers `AnnotationAwareAspectJAutoProxyCreator` (AAAPC) as a `BeanDefinitionRegistryPostProcessor` which eventually registers the BPP that creates AOP proxies.

`AAAPC` is a `SmartInstantiationAwareBeanPostProcessor` that:

1. `postProcessBeforeInstantiation()` — handles special cases (AOP advisor targets)
2. `getEarlyBeanReference()` — wraps in proxy for circular dependency scenarios
3. `postProcessAfterInitialization()` — **creates AOP proxy for each bean**

```java
// AAAPC in postProcessAfterInitialization():
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

// wrapIfNecessary():
// 1. Find all Advisors (from @Aspect beans + programmatic advisors)
// 2. Filter advisors applicable to this bean's class
// 3. If any advisors found → create proxy
// 4. Return proxy (or original bean if no advisors)
```

**`@EnableAspectJAutoProxy` attributes:**

```java
@EnableAspectJAutoProxy(
    proxyTargetClass = false,  // default: JDK proxy when interface available
    exposeProxy = false        // default: don't expose proxy via AopContext
)
```

---

### 5.1.11 — @Aspect Bean Registration

`@Aspect` classes must be registered as Spring beans to be detected:

```java
// Method 1: @Component on @Aspect
@Aspect
@Component  // ← REQUIRED for component scanning to pick it up
public class LoggingAspect { ... }

// Method 2: @Bean in @Configuration
@Configuration
public class AopConfig {
    @Bean
    public LoggingAspect loggingAspect() {
        return new LoggingAspect();
    }
}

// Method 3: XML
// <bean id="loggingAspect" class="com.example.LoggingAspect"/>

// WRONG: @Aspect alone is NOT enough
@Aspect  // not @Component — won't be found
public class LoggingAspect { ... }
```

**`@Aspect` classes themselves are NOT proxied by AOP.** If a method in `@Aspect` is `@Transactional`, the transaction will NOT be applied. Aspect beans are instantiated before the AOP infrastructure is fully set up. This is similar to the BFPP/BPP limitation.

---

### Common Misconceptions

**Misconception 1:** "Spring AOP can intercept all joinpoint types — field access, constructors, etc."
Wrong. Spring AOP ONLY intercepts method executions on Spring-managed bean methods. For other joinpoint types, AspectJ compile-time or load-time weaving is required.

**Misconception 2:** "`@After` advice has access to the return value."
Wrong. `@After` is "finally advice" — it runs regardless of success or failure but has NO access to return value or exception. For return value access, use `@AfterReturning`. For exception access, use `@AfterThrowing`.

**Misconception 3:** "Self-invocation within a bean goes through the proxy."
Wrong. This is the most critical AOP limitation. When method A calls method B on the same bean (`this.methodB()`), the call goes directly on the target object, bypassing the proxy entirely. No advice runs.

**Misconception 4:** "JDK proxy works for classes without interfaces."
Wrong. JDK dynamic proxy REQUIRES the target to implement at least one interface. Without an interface, Spring automatically switches to CGLIB.

**Misconception 5:** "`@AfterThrowing` can suppress exceptions."
Wrong. `@AfterThrowing` advice CANNOT suppress exceptions — the exception always continues propagating after the advice completes. Only `@Around` can suppress (catch without re-throwing) an exception.

**Misconception 6:** "`@AfterReturning` can modify the return value."
Wrong. `@AfterReturning` receives the return value for inspection but cannot change what the caller receives. Only `@Around` can substitute a different return value.

**Misconception 7:** "The proxy and target are the same object."
Wrong. The proxy WRAPS the target. `pjp.getTarget()` returns the target. `pjp.getThis()` returns the proxy. They are different objects in memory.

**Misconception 8:** "Spring AOP always uses JDK dynamic proxies."
Wrong since Spring Boot 2.0. Spring Boot defaults to `proxyTargetClass = true` — always CGLIB. Pure Spring (without Spring Boot) defaults to JDK proxy when interface available, CGLIB otherwise.

**Misconception 9:** "`@Aspect` annotation alone registers the aspect as a Spring bean."
Wrong. `@Aspect` marks the class as an aspect for PROCESSING by `AnnotationAwareAspectJAutoProxyCreator`. But the class must ALSO be a Spring bean (via `@Component`, `@Bean`, or XML). Without bean registration, the aspect is never discovered.

**Misconception 10:** "final methods can be intercepted by CGLIB proxy."
Wrong. CGLIB creates a subclass and overrides methods. `final` methods cannot be overridden → they bypass the CGLIB proxy. Final methods are always executed directly on the target, regardless of any advice defined.

---

## ② CODE EXAMPLES

### Example 1 — Complete Working Aspect

```java
@Aspect
@Component
public class ComprehensiveLoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(ComprehensiveLoggingAspect.class);

    // Pointcut: all methods in service package
    @Pointcut("within(com.example.service..*)")
    public void serviceLayer() {}

    // Pointcut: only public methods
    @Pointcut("execution(public * *(..))")
    public void publicMethods() {}

    // Combined pointcut: public methods in service layer
    @Pointcut("serviceLayer() && publicMethods()")
    public void publicServiceMethods() {}

    // @Before: log entry
    @Before("publicServiceMethods()")
    public void logEntry(JoinPoint jp) {
        log.debug("→ {}.{}({})",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            Arrays.toString(jp.getArgs()));
    }

    // @AfterReturning: log result
    @AfterReturning(pointcut = "publicServiceMethods()", returning = "result")
    public void logResult(JoinPoint jp, Object result) {
        log.debug("← {}.{}() = {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            result);
    }

    // @AfterThrowing: log exception
    @AfterThrowing(pointcut = "publicServiceMethods()", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) {
        log.error("✗ {}.{}() threw: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            ex.getMessage());
    }

    // @After: always runs (like finally)
    @After("publicServiceMethods()")
    public void logExit(JoinPoint jp) {
        log.trace("done: {}", jp.getSignature().getName());
    }
}
```

---

### Example 2 — @Around for Performance + Retry

```java
@Aspect
@Component
public class ResilienceAspect {

    @Around("@annotation(com.example.annotation.Retryable)")
    public Object retry(ProceedingJoinPoint pjp) throws Throwable {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        Retryable retryable = method.getAnnotation(Retryable.class);

        int maxAttempts = retryable.maxAttempts();
        long delay = retryable.delayMs();
        Class<? extends Exception>[] retryOn = retryable.retryOn();

        Exception lastException = null;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                Object result = pjp.proceed();
                if (attempt > 1) {
                    log.info("Succeeded on attempt {} for {}",
                        attempt, pjp.getSignature().toShortString());
                }
                return result;
            } catch (Exception e) {
                boolean shouldRetry = Arrays.stream(retryOn)
                    .anyMatch(retryType -> retryType.isInstance(e));

                if (!shouldRetry || attempt == maxAttempts) {
                    throw e;
                }

                lastException = e;
                log.warn("Attempt {}/{} failed for {}: {}. Retrying in {}ms...",
                    attempt, maxAttempts, pjp.getSignature().toShortString(),
                    e.getMessage(), delay);

                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw e;
                }
            }
        }
        throw lastException;
    }
}

// Custom annotation:
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retryable {
    int maxAttempts() default 3;
    long delayMs() default 1000;
    Class<? extends Exception>[] retryOn() default {Exception.class};
}

// Usage:
@Service
public class PaymentService {
    @Retryable(maxAttempts = 3, delayMs = 500,
               retryOn = {TimeoutException.class, ConnectionException.class})
    public PaymentResult processPayment(PaymentRequest request) {
        return externalPaymentGateway.process(request);
    }
}
```

---

### Example 3 — Pointcut Expression Showcase

```java
@Aspect
@Component
public class PointcutShowcase {

    // execution: all getters
    @Pointcut("execution(* com.example..*.get*())")
    public void allGetters() {}

    // execution: all setters
    @Pointcut("execution(void com.example..*.set*(*))") // void return, single arg
    public void allSetters() {}

    // within: all beans in specific package
    @Pointcut("within(com.example.repository..*)")
    public void repositoryLayer() {}

    // @annotation: methods with specific annotation
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}

    // @within: all methods in @Service-annotated classes
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void serviceAnnotatedClasses() {}

    // args: methods with exactly one Long parameter
    @Pointcut("execution(* *.*(..)) && args(java.lang.Long)")
    public void methodsWithLongArg() {}

    // bean: by bean name pattern
    @Pointcut("bean(*ServiceImpl)")
    public void serviceImplementations() {}

    // Complex: public methods in service OR repository, not getters
    @Pointcut("(within(com.example.service..*) || within(com.example.repository..*)) "
            + "&& execution(public * *(..)) "
            + "&& !execution(* *.get*(..))")
    public void publicNonGetterServiceOrRepo() {}

    // Advice using named pointcuts
    @Before("publicNonGetterServiceOrRepo()")
    public void auditModifyingOperations(JoinPoint jp) {
        log.info("Modifying operation: {}", jp.getSignature());
    }
}
```

---

### Example 4 — Argument Binding and Annotation Access

```java
@Aspect
@Component
public class ArgumentBindingAspect {

    // Bind specific argument by type
    @Before("execution(* com.example.service.OrderService.updateOrder(..)) && args(order, ..)")
    public void beforeOrderUpdate(Order order) {
        // 'order' is bound to the first argument matching Order type
        System.out.println("Updating order: " + order.getId());
    }

    // Bind annotation instance
    @Around("@annotation(cacheable)")
    public Object handleCacheable(ProceedingJoinPoint pjp,
                                   Cacheable cacheable) throws Throwable {
        // 'cacheable' is the actual @Cacheable annotation instance
        String[] cacheNames = cacheable.value();
        String keyExpression = cacheable.key();
        boolean cacheNull = cacheable.cacheNull();

        System.out.println("Cache names: " + Arrays.toString(cacheNames));
        System.out.println("Key expression: " + keyExpression);
        return pjp.proceed();
    }

    // Bind target object typed to specific interface
    @Before("execution(* com.example..*(..)) && target(repository)")
    public void beforeRepository(JoinPoint jp, Repository repository) {
        // 'repository' bound to target typed as Repository
        System.out.println("Repository call: " + repository.getClass().getSimpleName());
    }
}
```

---

### Example 5 — Introduction (@DeclareParents)

```java
// Add Auditable interface to all service beans
public interface Auditable {
    String getCreatedBy();
    void setCreatedBy(String user);
    LocalDateTime getCreatedAt();
    void setCreatedAt(LocalDateTime time);
}

public class DefaultAuditable implements Auditable {
    private String createdBy;
    private LocalDateTime createdAt = LocalDateTime.now();

    @Override public String getCreatedBy() { return createdBy; }
    @Override public void setCreatedBy(String user) { this.createdBy = user; }
    @Override public LocalDateTime getCreatedAt() { return createdAt; }
    @Override public void setCreatedAt(LocalDateTime time) { this.createdAt = time; }
}

@Aspect
@Component
public class AuditMixinAspect {

    @DeclareParents(
        value = "com.example.service..*+",    // all classes in service package (including subclasses)
        defaultImpl = DefaultAuditable.class   // default implementation
    )
    public static Auditable auditable;         // the introduced interface

    // Also add advice to set audit info on every service method call
    @Before("within(com.example.service..*) && this(auditable)")
    public void setAuditInfo(JoinPoint jp, Auditable auditable) {
        if (auditable.getCreatedBy() == null) {
            auditable.setCreatedBy(SecurityContextHolder.getContext()
                .getAuthentication().getName());
        }
    }
}

// Now ALL service beans implement Auditable:
@Service
public class OrderService {
    public void placeOrder(Order order) { ... }
    // No Auditable code — but at runtime, the proxy adds it!
}

// Usage:
OrderService os = ctx.getBean(OrderService.class);
Auditable auditable = (Auditable) os; // works — introduced via mixin
System.out.println("Created by: " + auditable.getCreatedBy());
```

---

### Example 6 — Self-Invocation Solutions

```java
@Service
public class PaymentService {

    // Solution 1: Self-injection (gets proxy)
    @Autowired
    @Lazy // Break potential circular dependency
    private PaymentService self;

    public void processPayment(PaymentRequest req) {
        validateRequest(req);
        self.executePayment(req); // through proxy → @Transactional applies
    }

    @Transactional
    public void executePayment(PaymentRequest req) {
        // transaction is active
        paymentRepository.save(req);
        ledgerRepository.credit(req);
    }
}

// Better Solution: Refactor into separate bean
@Service
public class PaymentExecutor {
    @Transactional
    public void execute(PaymentRequest req) {
        paymentRepository.save(req);
        ledgerRepository.credit(req);
    }
}

@Service
public class PaymentService {
    @Autowired PaymentExecutor executor; // separate bean → proxy ✓

    public void processPayment(PaymentRequest req) {
        validateRequest(req);
        executor.execute(req); // goes through PaymentExecutor's proxy ✓
    }
}
```

---

### Example 7 — Proxy Type Detection

```java
@Service
public class InspectionService {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @PostConstruct
    public void inspect() {
        // Check if proxy was applied
        System.out.println("OrderService is AOP proxy: " +
            AopUtils.isAopProxy(orderService));

        // Check proxy type
        System.out.println("OrderService is JDK proxy: " +
            AopUtils.isJdkDynamicProxy(orderService));
        System.out.println("OrderService is CGLIB proxy: " +
            AopUtils.isCglibProxy(orderService));

        // Get the target class (unwrap proxy)
        Class<?> targetClass = AopUtils.getTargetClass(orderService);
        System.out.println("Target class: " + targetClass.getSimpleName());
        // prints: OrderServiceImpl (not the proxy class name)

        // Get the actual target object (unwrap proxy)
        try {
            Object target = ((Advised) orderService).getTargetSource().getTarget();
            System.out.println("Target object: " + target);
        } catch (Exception e) {
            System.err.println("Cannot unwrap: " + e.getMessage());
        }

        // Check proxy class name
        System.out.println("Proxy class: " + orderService.getClass().getName());
        // JDK: com.sun.proxy.$Proxy42
        // CGLIB: com.example.OrderServiceImpl$$SpringCGLIB$$0
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which of the following are Spring AOP joinpoint types? (Select all that apply)

A) Method execution on a Spring bean
B) Field write on a Spring bean
C) Constructor execution
D) Static method execution
E) Method call (as opposed to execution)

**Answer: A only**
Spring AOP ONLY supports method execution joinpoints on Spring-managed beans. Field access, constructors, static methods, and method calls (as distinct from executions) are NOT supported in Spring AOP. They require AspectJ compile-time or load-time weaving.

---

**Q2 — MCQ:**
What is the correct advice execution order for a single joinpoint in a single aspect?

A) `@Before` → `@Around` → `@AfterReturning` → `@After`
B) `@Around (before proceed)` → `@Before` → `[method]` → `@AfterReturning` → `@After` → `@Around (after proceed)`
C) `@Before` → `[method]` → `@AfterReturning` → `@After` → `@Around`
D) `@Around` → `@Before` → `@After` → `@AfterReturning` → `[method]`

**Answer: B**
The correct order: `@Around before proceed` → `@Before` → `[method executes]` → `@AfterReturning` (or `@AfterThrowing`) → `@After` → `@Around after proceed`.

---

**Q3 — Select all that apply:**
Which advice types can PREVENT the target method from executing? (Select ALL that apply)

A) `@Before` — by completing normally (not throwing)
B) `@Before` — by throwing an exception
C) `@Around` — by not calling `pjp.proceed()`
D) `@AfterReturning` — by returning a different value
E) `@After` — by throwing an exception
F) `@Around` — by catching exception and returning a default value

**Answer: B, C**
Only `@Before` (by throwing) and `@Around` (by not calling `proceed()`) can prevent the target method from executing. `@AfterReturning` cannot change return values to prevent execution (execution already happened). `@After` throws after the method has already run.

---

**Q4 — Code Output Prediction:**

```java
@Aspect
@Component
@Order(1)
public class FirstAspect {
    @Around("execution(* com.example.TargetBean.doWork())")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("First BEFORE");
        Object result = pjp.proceed();
        System.out.println("First AFTER");
        return result;
    }
}

@Aspect
@Component
@Order(2)
public class SecondAspect {
    @Around("execution(* com.example.TargetBean.doWork())")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Second BEFORE");
        Object result = pjp.proceed();
        System.out.println("Second AFTER");
        return result;
    }
}

@Component
public class TargetBean {
    public void doWork() { System.out.println("TARGET"); }
}
```

What is the output when `targetBean.doWork()` is called?

A)
```
First BEFORE
Second BEFORE
TARGET
Second AFTER
First AFTER
```
B)
```
Second BEFORE
First BEFORE
TARGET
First AFTER
Second AFTER
```
C)
```
First BEFORE
First AFTER
Second BEFORE
TARGET
Second AFTER
```
D)
```
TARGET
First BEFORE
Second BEFORE
Second AFTER
First AFTER
```

**Answer: A**
`@Order(1)` = lower value = HIGHER priority = OUTERMOST aspect. Like nested function calls: First wraps Second wraps Target. `First BEFORE` → `Second BEFORE` → `TARGET` → `Second AFTER` → `First AFTER`.

---

**Q5 — Trick Question:**

```java
@Service
public class EmailService {
    public void sendWelcome(User user) {
        sendEmail(user.getEmail(), "Welcome!"); // self-invocation
    }

    @Transactional
    public void sendEmail(String to, String subject) {
        emailRepository.save(new EmailLog(to, subject));
    }
}
```

Is the `@Transactional` on `sendEmail()` active when called from `sendWelcome()`?

A) Yes — `@Transactional` is always applied when the method is annotated
B) No — self-invocation bypasses the proxy; transaction is NOT active
C) Yes — but only if CGLIB proxy is used
D) Yes — Spring detects self-invocation and routes through the proxy

**Answer: B**
Self-invocation (`this.sendEmail(...)` inside `sendWelcome()`) bypasses the proxy. The `@Transactional` advice never runs. The `emailRepository.save()` executes WITHOUT a transaction. No exception — the code runs — but no transaction management.

---

**Q6 — Select all that apply:**
Which pointcut expressions correctly match ALL public methods in `com.example.service` package AND its sub-packages? (Select ALL that apply)

A) `execution(public * com.example.service.*.*(..))`
B) `execution(public * com.example.service..*.*(..))`
C) `within(com.example.service..*)` (for Spring AOP method execution)
D) `execution(* com.example.service.*(..))` — matches only top-level package methods
E) `execution(public * com.example.service..*.*(..))` — sub-packages with double dot

**Answer: B, C, E**
A matches only classes in `com.example.service` (single `*` after package = one level only). B matches service AND all sub-packages (`..` = zero or more packages). C with `within()` matches all joinpoints (method executions in Spring AOP) in service and sub-packages. D is missing the `*` for class name — malformed. E is equivalent to B.

---

**Q7 — MCQ:**
When does Spring create the AOP proxy for a singleton bean?

A) At the first `getBean()` call — lazy proxy creation
B) During `postProcessBeforeInitialization()` — before initialization callbacks
C) During `postProcessAfterInitialization()` — after all initialization callbacks
D) During `populateBean()` — alongside dependency injection

**Answer: C**
AOP proxy creation happens in `AbstractAutoProxyCreator.postProcessAfterInitialization()` — which runs AFTER `@PostConstruct`, `afterPropertiesSet()`, and custom `init-method`. The raw bean is fully initialized before the proxy wraps it.

---

**Q8 — Scenario:**

```java
public interface OrderRepository {}

@Repository
public class JpaOrderRepository implements OrderRepository { ... }

@Service
public class OrderService {
    @Autowired OrderRepository repo; // JDK or CGLIB proxy?
}
```

With default Spring AOP configuration (`proxyTargetClass = false`):
What type of proxy does `repo` receive?

A) No proxy — `OrderRepository` is not an aspect target
B) JDK dynamic proxy — target implements `OrderRepository` interface
C) CGLIB proxy — Spring always uses CGLIB
D) JDK dynamic proxy — only if `@Transactional` or other AOP is applied to the repository

**Answer: D**
A proxy is ONLY created if AOP advice actually applies to the bean. If `JpaOrderRepository` has `@Transactional` methods or other aspect advice, a proxy is created. With `proxyTargetClass = false` (default): since `JpaOrderRepository` implements `OrderRepository` interface → JDK dynamic proxy. If NO advice applies → no proxy — the raw `JpaOrderRepository` instance is injected.

---

**Q9 — MCQ:**
What does `pjp.getTarget()` return in an `@Around` advice?

A) The proxy object (what clients call methods on)
B) The target object (the original bean before proxying)
C) The aspect itself
D) The `JoinPoint` static part

**Answer: B**
`pjp.getTarget()` returns the TARGET — the original bean object, not the proxy. `pjp.getThis()` returns the PROXY. This distinction matters when the same advice needs to check if it is dealing with a proxy or a raw object.

---

**Q10 — Drag-and-Drop: Match advice type to capability:**

Advice types: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`

Capabilities:
- Can prevent method execution by not calling proceed()
- Can access return value but cannot change it
- Runs regardless of success or failure (finally-like)
- Can suppress exceptions
- Can modify method arguments before proceeding
- Can access exception but cannot suppress it
- Runs before method but cannot see return value
- Can call the target method multiple times

**Answer:**
- `@Around` → Can prevent method execution by not calling proceed()
- `@AfterReturning` → Can access return value but cannot change it
- `@After` → Runs regardless of success or failure (finally-like)
- `@Around` → Can suppress exceptions
- `@Around` → Can modify method arguments before proceeding
- `@AfterThrowing` → Can access exception but cannot suppress it
- `@Before` → Runs before method but cannot see return value
- `@Around` → Can call the target method multiple times

---

## ④ TRICK ANALYSIS

**Trap 1: @After has access to return value**
Dead wrong. `@After` is finally advice — it has NO access to the return value or exception. It is equivalent to a Java `finally` block. For return value access: `@AfterReturning`. For exception access: `@AfterThrowing`.

**Trap 2: @AfterReturning can change the return value**
Wrong. `@AfterReturning` can READ the return value via the `returning` binding, but CANNOT change what the caller receives. The method has already returned. Only `@Around` can substitute a different return value.

**Trap 3: @AfterThrowing suppresses the exception**
Wrong. `@AfterThrowing` CANNOT suppress exceptions — the exception always continues propagating after the advice completes. To suppress: `@Around` with try-catch that doesn't re-throw.

**Trap 4: Spring AOP intercepts static methods**
Wrong. Static methods are NOT joinpoints in Spring AOP. They bypass the proxy entirely (proxy is a subclass/proxy of the bean — static methods belong to the class, not instances). For static method interception: AspectJ CTW/LTW.

**Trap 5: final methods are intercepted by CGLIB**
Wrong. CGLIB creates a subclass and OVERRIDES methods. `final` methods cannot be overridden. CGLIB cannot intercept final methods. They execute directly on the target, bypassing all advice.

**Trap 6: Self-invocation routes through the proxy with CGLIB**
Wrong. Self-invocation bypasses the proxy regardless of whether JDK or CGLIB proxy is used. The `this` reference inside a method always points to the TARGET object, not the proxy. Neither proxy type changes this JVM behavior.

**Trap 7: @Aspect annotation automatically registers the bean**
Wrong. `@Aspect` marks the class for AOP processing, but the class must ALSO be a Spring bean. `@Component` (or `@Bean`) is required separately. Without Spring bean registration, the aspect is invisible to `AnnotationAwareAspectJAutoProxyCreator`.

**Trap 8: @Order on @Aspect controls advice execution within ONE aspect**
Wrong. `@Order` on an `@Aspect` class controls the ordering of DIFFERENT aspects when multiple aspects apply to the same joinpoint. Within a single aspect, the advice ordering follows: `@Around before` → `@Before` → method → `@AfterReturning` → `@After` → `@Around after`.

**Trap 9: JDK proxy can proxy any class**
Wrong. JDK dynamic proxy requires the target to implement at least ONE interface. If the target class implements no interfaces, Spring automatically uses CGLIB (when available). If `proxyTargetClass = true` is set, CGLIB is always used.

**Trap 10: @Transactional works on self-invoked methods with @Configuration classes**
Subtly wrong. Even in `@Configuration` (CGLIB-proxied) classes, `@Transactional` self-invocation does NOT work because the `@Transactional` proxy is the Spring AOP proxy wrapping the bean — separate from the CGLIB proxy that handles `@Bean` method singleton semantics. They are different proxy layers for different purposes.

---

## ⑤ SUMMARY SHEET

### AOP Vocabulary Quick Reference

```
JoinPoint    → A point in execution (method execution in Spring AOP)
Pointcut     → Predicate selecting JoinPoints (execution expression)
Advice       → Action at a JoinPoint (@Before, @After, @Around, etc.)
Aspect       → Module combining Pointcuts + Advice (@Aspect class)
Target       → The original bean being advised
Proxy        → Wrapper around Target that applies advice
Weaving      → Process of linking aspect to target (Spring: runtime)
Introduction → Adding new interface to existing class (@DeclareParents)
```

---

### Advice Type Capabilities

```
┌─────────────────┬──────────┬──────────┬──────────┬───────────┬──────────┐
│ Capability      │@Before   │@After    │@After    │@After     │@Around   │
│                 │          │          │Returning │Throwing   │          │
├─────────────────┼──────────┼──────────┼──────────┼───────────┼──────────┤
│ Access args     │ YES      │ YES      │ YES      │ YES       │ YES      │
│ Access return   │ NO       │ NO       │ YES(read)│ NO        │ YES(rw)  │
│ Modify return   │ NO       │ NO       │ NO       │ NO        │ YES      │
│ Access exception│ NO       │ NO       │ NO       │ YES(read) │ YES      │
│ Suppress except │ NO       │ NO       │ NO       │ NO        │ YES      │
│ Prevent method  │ throw ✗  │ NO*      │ NO*      │ NO*       │ YES      │
│ Call method 2x  │ NO       │ NO       │ NO       │ NO        │ YES      │
│ Modify args     │ NO       │ NO       │ NO       │ NO        │ YES      │
│ Runs on success │ YES      │ YES      │ YES      │ NO        │ YES      │
│ Runs on throw   │ YES(pre) │ YES      │ NO       │ YES       │ YES      │
└─────────────────┴──────────┴──────────┴──────────┴───────────┴──────────┘
* = method already executed / never had chance
```

---

### Key Pointcut Expression Patterns

```
All methods in package:        execution(* com.example.service.*.*(..))
All methods in pkg + subpkg:   execution(* com.example.service..*.*(..))
                               or: within(com.example.service..*)
All public methods:            execution(public * *(..))
All methods returning String:  execution(String *(..))
Method by exact name:          execution(* *.placeOrder(..))
With specific annotation:      @annotation(com.example.Transactional)
Class has annotation:          @within(org.springframework.stereotype.Service)
By bean name:                  bean(*ServiceImpl)
By parameter type:             args(java.lang.Long)
Subclass matching:             execution(* com.example.OrderRepo+.*(..))
```

---

### Proxy Type Selection Rules

```
Spring Boot 2.0+ default: proxyTargetClass=true → Always CGLIB

Pure Spring default (proxyTargetClass=false):
    Bean implements interface → JDK Dynamic Proxy
    Bean has NO interface     → CGLIB

CGLIB cannot proxy: final classes, cannot intercept final methods
JDK proxy cannot proxy: classes without interfaces
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Spring AOP joinpoints | Method execution ONLY on Spring beans |
| Static methods | NOT intercepted by Spring AOP |
| final methods | NOT intercepted by CGLIB proxy |
| Self-invocation | Bypasses proxy — advice NOT applied |
| `@After` = finally | No access to return value or exception |
| `@AfterReturning` | Read-only access to return value |
| `@AfterThrowing` | Cannot suppress exception |
| `@Around` | Only advice that can prevent execution, modify args/return, suppress |
| `@Aspect` alone | Does NOT register as Spring bean — needs `@Component` |
| AOP proxy creation | `postProcessAfterInitialization()` — after all init callbacks |
| `pjp.getTarget()` | TARGET object (raw bean) |
| `pjp.getThis()` | PROXY object |
| `@Order` on @Aspect | Lower value = outermost = runs first (outer in the chain) |

---

### Interview One-Liners

- "Spring AOP supports only method execution joinpoints on Spring-managed beans — for field access, constructors, or static methods, AspectJ compile-time or load-time weaving is required."
- "Self-invocation (calling `this.method()` within the same bean) bypasses the AOP proxy — the advice never runs. Solutions: refactor to a separate bean, inject self, or use `AopContext.currentProxy()`."
- "`@After` is finally advice — it has no access to return value or exception. `@AfterReturning` provides read-only return value access. Only `@Around` can modify return values or suppress exceptions."
- "AOP proxies are created in `postProcessAfterInitialization()` by `AbstractAutoProxyCreator` — after all initialization callbacks (`@PostConstruct`, `afterPropertiesSet()`) run on the RAW bean."
- "`@Aspect` marks a class for AOP processing, but `@Component` (or `@Bean`) is separately required to register it as a Spring bean — without bean registration, the aspect is never discovered."
- "CGLIB proxies cannot intercept `final` methods — the subclass cannot override them. JDK dynamic proxies cannot proxy classes without interfaces."
- "`pjp.getTarget()` returns the raw target bean; `pjp.getThis()` returns the proxy. They are different objects in memory."
- "With `@Order(1)` and `@Order(2)` aspects, `@Order(1)` is the OUTERMOST wrapper — its `@Around before` runs first and its `@Around after` runs last, like nested function calls."

---
