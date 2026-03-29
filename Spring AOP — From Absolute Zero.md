# 🎯 Spring AOP — From Absolute Zero

---

## 🧩 1. Requirement Understanding — Feel the Pain First

You join a team. Your ticket:

```
TICKET-156: Add logging to every service method — 
            who called it, with what args, how long it took.

TICKET-157: Every service method must check if the user is authenticated
            before executing.

TICKET-158: Every database-writing method needs transaction management.
```

You open `OrderService`:

```java
@Service
public class OrderService {

    public Order placeOrder(Order order) {
        orderRepository.save(order);
        return order;
    }

    public void cancelOrder(Long id) {
        orderRepository.delete(id);
    }
}
```

You implement the tickets the obvious way:

```java
@Service
public class OrderService {

    public Order placeOrder(Order order) {
        // TICKET-157: auth check
        if (!SecurityContext.isAuthenticated()) throw new SecurityException();
        // TICKET-156: logging
        log.info("→ placeOrder({})", order);
        long start = System.currentTimeMillis();
        // TICKET-158: transaction
        Transaction tx = db.beginTransaction();
        try {
            orderRepository.save(order);
            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        }
        // TICKET-156: more logging
        log.info("← placeOrder took {}ms", System.currentTimeMillis() - start);
        return order;
    }

    public void cancelOrder(Long id) {
        if (!SecurityContext.isAuthenticated()) throw new SecurityException(); // AGAIN
        log.info("→ cancelOrder({})", id);                                    // AGAIN
        long start = System.currentTimeMillis();                               // AGAIN
        Transaction tx = db.beginTransaction();                                // AGAIN
        try {
            orderRepository.delete(id);
            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        }
        log.info("← cancelOrder took {}ms", System.currentTimeMillis() - start); // AGAIN
    }
}
```

Now you have `UserService`, `PaymentService`, `InventoryService`. Each method looks like this. Every one.

**The business logic — one line — is buried under 15 lines of infrastructure noise.** Adding a new cross-cutting concern means opening every service. Forgetting one method means a security hole. Changing the logging format means editing 40 files.

---

## 🧠 2. Design Thinking — The Naive Approaches

### Attempt 1: Extract into utility methods

```java
public class ServiceHelper {
    public static void checkAuth() { ... }
    public static void logEntry(String method, Object... args) { ... }
    public static Transaction beginTx() { ... }
}

public class OrderService {
    public Order placeOrder(Order order) {
        ServiceHelper.checkAuth();
        ServiceHelper.logEntry("placeOrder", order);
        Transaction tx = ServiceHelper.beginTx();
        // ... business logic
    }
}
```

❌ **Rejected.** You still have to call these in every method. You still forget some. You still can't change the behavior without editing every caller. The coupling is still everywhere — you just moved where the code lives.

---

### Attempt 2: Base class with template methods

```java
public abstract class BaseService {
    public final Object execute(String method, Supplier<Object> logic) {
        checkAuth();
        logEntry(method);
        long start = System.currentTimeMillis();
        // ... wrap in transaction
        Object result = logic.get();
        logExit(method, start);
        return result;
    }
}

public class OrderService extends BaseService {
    public Order placeOrder(Order order) {
        return (Order) execute("placeOrder", () -> {
            return orderRepository.save(order); // buried inside a lambda
        });
    }
}
```

❌ **Rejected.** You've burned your single inheritance on a framework concern. The API is ugly. Every method needs to wrap its logic in a lambda. You can't easily add new behaviors to `execute()` for specific methods. And Java's single inheritance means `OrderService` can never extend anything else.

---

### Attempt 3: Wrapper/Decorator class

```java
public class LoggingOrderService implements OrderService {
    private final OrderService delegate;

    public Order placeOrder(Order order) {
        log.info("→ placeOrder");
        Order result = delegate.placeOrder(order);
        log.info("← placeOrder");
        return result;
    }
    // ... repeat for every method
}
```

❌ **Rejected.** You now maintain two classes per service. Adding a method means editing both. Adding a new concern (security) means a THIRD class. This doesn't scale. You've re-invented the proxy pattern manually — and Spring already automates it.

**The right answer:** Define the cross-cutting behavior ONCE, declare WHICH methods it applies to, and let Spring intercept those method calls automatically at runtime.

---

## 🧠 3. Concept Introduction — How It Works (No Code Yet)

Think of a **toll booth on a highway**.

```
Old way (no AOP):
    Every car (method call) handles its own tollpay.
    The car driver calculates the toll, pays it, gets a receipt.
    The logic of "paying toll" is inside every car.

AOP way:
    The highway authority installs a toll booth (aspect).
    Every car (method call) passes through it automatically.
    The car doesn't know the booth exists.
    The booth can log, charge, block — whatever it wants.
    New behavior? Replace the booth. Touch no cars.
```

**The four pieces of AOP:**

```
WHAT to intercept   → Pointcut
    "all methods in the service package"
    "all methods annotated with @Transactional"

WHEN to act         → Advice type
    "before the method runs" (@Before)
    "after it returns" (@AfterReturning)
    "around it — wrap before AND after" (@Around)

WHAT to do          → Advice code
    The actual logging / auth check / transaction logic

Package it all together → Aspect
    One class, one place
```

**How Spring intercepts method calls — the proxy:**

```
Without AOP:
    Your code         OrderService
    ─────────────→    placeOrder()
                      [runs directly]

With AOP:
    Your code         [PROXY]              OrderService
    ─────────────→    placeOrder()    →    placeOrder()
                      intercepts!          [real work]
                      ↑ runs advice
                      ↑ then calls real method
                      ↑ then runs more advice

The proxy is a WRAPPER Spring creates at startup.
ctx.getBean(OrderService.class) silently gives you the WRAPPER.
You never see it. Your code looks the same.
```

**The proxy is created at startup, stored in Spring's container, and injected everywhere.** Your `@Autowired OrderService` is actually the proxy — you're calling the toll booth every time, without knowing.

---

## 🏗 4. Task Breakdown

```
TODO 1: Create a basic @Aspect with @Before advice — add logging without touching services
TODO 2: Use @Around — the most powerful advice — wrap before AND after
TODO 3: Write pointcut expressions — target exactly the right methods
TODO 4: Add @AfterReturning and @AfterThrowing — react to outcomes
TODO 5: Understand the self-invocation trap — the silent failure that kills transactions
TODO 6: Order multiple aspects — who runs first?
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: First Aspect — @Before Advice

```java
// @Aspect tells Spring: "this class defines cross-cutting behavior"
// @Component tells Spring: "manage this class as a bean"
// BOTH are required. @Aspect alone doesn't register it. @Component alone doesn't activate AOP.
@Aspect
@Component
public class SecurityAspect {

    // @Before runs BEFORE the matched method executes.
    // The string inside is the pointcut expression — it defines WHICH methods to intercept.
    // "execution(* com.example.service.*.*(..))" means:
    //   execution  → intercept method execution (not field access, not constructors)
    //   *          → any return type
    //   com.example.service.* → any class in the service package
    //   .*         → any method name
    //   (..)       → any parameters (zero or more)
    @Before("execution(* com.example.service.*.*(..))")
    public void checkAuthentication(JoinPoint jp) {
        // JoinPoint gives you context about the intercepted method call
        // Think of it as: "who got stopped at the toll booth and why"

        String methodName = jp.getSignature().getName();  // "placeOrder"
        Object[] args = jp.getArgs();                     // [Order@1a2b3c]
        Object target = jp.getTarget();                   // OrderService (the real bean)

        if (!SecurityContext.isAuthenticated()) {
            throw new SecurityException("Not authenticated: " + methodName);
        }
    }
}
```

**What Spring does internally:**

```
At startup:
    ① Spring sees @EnableAspectJAutoProxy (or Spring Boot auto-configures it)
    ② Registers AnnotationAwareAspectJAutoProxyCreator as a BeanPostProcessor
    ③ After every bean is initialized:
        "Does any aspect's pointcut match this bean's methods?"
        OrderService.placeOrder() — matches execution(* com.example.service.*.*(..))?
        YES → wrap OrderService in a CGLIB proxy
    ④ CGLIB generates OrderService$$SpringCGLIB$$0
       This subclass overrides placeOrder() to:
           - run checkAuthentication()
           - call super.placeOrder()
    ⑤ Proxy stored in container. All @Autowired get the proxy.

At runtime, when someone calls orderService.placeOrder(order):
    → Actually calls OrderService$$SpringCGLIB$$0.placeOrder(order)
    → Proxy runs SecurityAspect.checkAuthentication()
    → If auth passes, proxy calls super.placeOrder(order) — the real method
```

**Enabling AOP — you need this once:**

```java
// In pure Spring: add this annotation to your config class
@Configuration
@EnableAspectJAutoProxy  // activates the AOP infrastructure
public class AppConfig { }

// In Spring Boot: already auto-configured. Nothing to add.
```

**The `@Before` silent failure — forgetting `@Component`:**

```java
// ❌ @Aspect without @Component — advice is NEVER applied
@Aspect
// missing @Component!
public class SecurityAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void checkAuthentication(JoinPoint jp) {
        // This code never runs.
        // No error. No warning. Methods execute without security checks.
    }
}

// ✅ Both annotations required
@Aspect
@Component
public class SecurityAspect { ... }
```

---

### TODO 2: @Around — The Most Powerful Advice

`@Before` can only run before. `@Around` wraps the ENTIRE call — you control what happens before, after, and whether the method runs at all.

```java
@Aspect
@Component
public class PerformanceAspect {

    // @Around intercepts the method call entirely.
    // The parameter type changes: ProceedingJoinPoint instead of JoinPoint.
    // ProceedingJoinPoint is JoinPoint + the ability to actually CALL the method.
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        // Everything BEFORE pjp.proceed() is "before the method"
        String method = pjp.getSignature().toShortString();
        long start = System.nanoTime();
        log.info("→ {}", method);

        try {
            // pjp.proceed() is the moment you actually call the real method.
            // If you never call this, the real method NEVER executes.
            // The return value is what the real method returned.
            Object result = pjp.proceed();

            // Everything AFTER pjp.proceed() is "after the method returned"
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("← {} took {}ms", method, elapsed);

            // You MUST return the result. If you return null here,
            // the caller gets null — even if the method returned something.
            return result;

        } catch (Exception e) {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.error("✗ {} failed after {}ms: {}", method, elapsed, e.getMessage());
            throw e;  // re-throw so the caller sees the exception
        }
    }
}
```

**The four superpowers of `@Around` that no other advice has:**

```java
@Around("execution(* com.example.service.*.*(..))")
public Object demonstrate(ProceedingJoinPoint pjp) throws Throwable {

    // Superpower 1: Prevent the method from running entirely
    if (isBlacklisted(pjp.getArgs())) {
        return null;  // method never called — caller gets null
    }

    // Superpower 2: Modify arguments before passing to the method
    Object[] args = pjp.getArgs();
    args[0] = sanitize(args[0]);  // clean up the first argument
    Object result = pjp.proceed(args);  // pass modified args

    // Superpower 3: Replace the return value
    return enrichResult(result);  // caller gets enriched version

    // Superpower 4 (in catch block): Suppress exceptions
    // catch (Exception e) { return defaultValue; }  // caller sees no exception
}
```

**The most dangerous silent failure — forgetting to return the result:**

```java
// ❌ Silent bug — all void methods work but all non-void methods return null
@Around("execution(* com.example.service.*.*(..))")
public Object loggingAround(ProceedingJoinPoint pjp) throws Throwable {
    log.info("before");
    pjp.proceed();  // result discarded!
    log.info("after");
    // return statement missing!
    // Java returns null implicitly for Object return type
    // Your controller now gets null from orderService.placeOrder()
}

// ✅ Always capture and return:
public Object loggingAround(ProceedingJoinPoint pjp) throws Throwable {
    log.info("before");
    Object result = pjp.proceed();
    log.info("after");
    return result;  // ← mandatory
}
```

---

### TODO 3: Pointcut Expressions — Target the Right Methods

A pointcut expression is a pattern that Spring evaluates against every method in the container to decide where advice applies.

```java
@Aspect
@Component
public class PointcutExamples {

    // The @Pointcut annotation defines a reusable named pointcut.
    // The method body is always empty — it's just a named anchor.
    // Think of it as a variable that holds a pattern.
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}   // ← empty body, just a label

    @Pointcut("execution(* com.example.repository.*.*(..))")
    public void repositoryLayer() {}

    // Combine named pointcuts with &&, ||, !
    @Pointcut("serviceLayer() || repositoryLayer()")
    public void dataAccessLayer() {}

    // Reference a named pointcut from advice:
    @Before("serviceLayer()")
    public void beforeService(JoinPoint jp) { ... }

    // Or use the expression inline:
    @Before("execution(* com.example.service.*.*(..))")
    public void alsoBeforeService(JoinPoint jp) { ... }
}
```

**Reading pointcut expressions — anatomy:**

```
execution(  *          com.example.service.  *  .  *  (..))
           ↑           ↑                     ↑     ↑   ↑
        return type  package             class  method params

execution(* com.example.service.*.*(..))
    → return type: any (*)
    → package: com.example.service
    → class: any (*) — one level deep
    → method: any (*)
    → params: any count and type (..)

execution(* com.example.service..*.*(..))
    → the double-dot (..) after "service" means:
       "service package AND all sub-packages"
    → com.example.service.order.OrderService — matches ✅
    → com.example.service.payment.PaymentService — matches ✅

execution(public void com.example.service.OrderService.placeOrder(..))
    → only the placeOrder method on OrderService returning void ← very specific

execution(* *.save*(..))
    → any method NAME STARTING WITH "save" in any class
    → OrderRepository.save() — matches ✅
    → UserRepository.saveAll() — matches ✅
    → OrderService.placeOrder() — does NOT match ❌
```

**Alternative pointcut designators — beyond `execution()`:**

```java
// @annotation: match methods that HAVE a specific annotation
@Before("@annotation(org.springframework.transaction.annotation.Transactional)")
public void beforeTransactional(JoinPoint jp) {
    // Runs before ANY method annotated with @Transactional
}

// @within: match all methods in classes that HAVE a specific annotation
@Before("@within(org.springframework.stereotype.Service)")
public void beforeServiceClass(JoinPoint jp) {
    // Runs before any method in any @Service-annotated class
}

// within: match all methods in a specific package (cleaner than execution for whole-package)
@Before("within(com.example.service..*)")
public void withinService(JoinPoint jp) {
    // All methods in service package and sub-packages
}

// bean: match by Spring bean NAME
@Before("bean(*ServiceImpl)")
public void beforeServiceImpls(JoinPoint jp) {
    // Runs for beans whose name ends in "ServiceImpl"
}
```

**Common silent failure — package typo in pointcut:**

```java
// ❌ Typo in package name — advice silently never runs
@Before("execution(* com.exmaple.service.*.*(..))")
//                        ↑ typo: "exmaple" instead of "example"
public void neverRuns(JoinPoint jp) {
    // This never executes. No error. Spring just finds no matching methods.
}

// ✅ Test your pointcuts by adding a log and checking it fires on startup:
@Before("execution(* com.example.service.*.*(..))")
public void shouldRun(JoinPoint jp) {
    log.debug("Aspect fired for: {}", jp.getSignature()); // verify this appears
}
```

---

### TODO 4: @AfterReturning and @AfterThrowing — React to Outcomes

```java
@Aspect
@Component
public class OutcomeAspect {

    // @AfterReturning runs ONLY when the method returns successfully (no exception).
    // The "returning" attribute binds the return value to a method parameter.
    // Parameter name must match exactly.
    @AfterReturning(
        pointcut = "execution(* com.example.service.OrderService.placeOrder(..))",
        returning = "result"  // ← the parameter name below
    )
    public void onOrderPlaced(JoinPoint jp, Order result) {
        // result is the actual Order object that placeOrder() returned
        // You CAN read it. You CANNOT change what the caller receives.
        log.info("Order placed successfully: {}", result.getId());
        analyticsService.trackOrderPlaced(result);
    }

    // @AfterThrowing runs ONLY when the method throws an exception.
    // The "throwing" attribute binds the exception to a parameter.
    // You CAN declare a specific exception type — only that type (and subtypes) triggers this.
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex"  // ← the parameter name below
    )
    public void onException(JoinPoint jp, Exception ex) {
        // ex is the actual exception thrown
        // This advice CANNOT suppress it — it always propagates after this runs
        log.error("Service exception in {}: {}", jp.getSignature().getName(), ex.getMessage());
        alertService.notifyOncall(ex);
    }

    // @After runs ALWAYS — success OR failure.
    // It's the equivalent of a finally block.
    // NO access to return value. NO access to exception. Just "it's done."
    @After("execution(* com.example.service.*.*(..))")
    public void always(JoinPoint jp) {
        // cleanup, metrics, MDC clearing — things that always need to happen
        MDC.clear();
    }
}
```

**The advice capability table — most important thing to memorize:**

```
                    @Before  @After  @AfterReturning  @AfterThrowing  @Around
─────────────────────────────────────────────────────────────────────────────
Read return value?    NO       NO        YES (read)         NO           YES
Change return value?  NO       NO        NO                 NO           YES
Read exception?       NO       NO        NO                 YES (read)   YES
Suppress exception?   NO       NO        NO                 NO           YES
Prevent method run?  throw    N/A*       N/A*               N/A*         YES
Call method 2x?       NO       NO        NO                 NO           YES
─────────────────────────────────────────────────────────────────────────────
N/A* = method already ran (or: @Before via throwing, which counts as preventing)
```

**Common silent failure — @After thinking it has access to return value:**

```java
// ❌ @After has no returning/throwing attributes — they don't exist on @After
@After(
    pointcut = "execution(* com.example.service.*.*(..))",
    returning = "result"  // ← COMPILE ERROR: unknown attribute on @After
)
public void after(JoinPoint jp, Object result) { }

// ✅ Use @AfterReturning for return value access:
@AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
public void afterReturning(JoinPoint jp, Object result) { }
```

---

### TODO 5: The Self-Invocation Trap — The Silent Failure That Kills Transactions

This is the most dangerous AOP concept. No error. No warning. Just silent failure.

```java
@Service
public class OrderService {

    @Transactional  // managed by Spring AOP via a proxy
    public void placeOrder(Order order) {
        validateOrder(order);  // ← THIS IS THE TRAP
        orderRepository.save(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) {
        // This should run in a NEW separate transaction.
        // But it doesn't. Why?
    }
}
```

**Why it fails — the proxy picture:**

```
External call path (advice RUNS ✅):
    Controller → [OrderService PROXY] → placeOrder()
                    ↑
                 proxy intercepts
                 @Transactional advice runs
                 transaction started

Internal call path (advice DOES NOT RUN ❌):
    [OrderService PROXY] → placeOrder() → validateOrder()
                                      ↑
                              "this.validateOrder()"
                              "this" = the RAW bean
                              NOT the proxy
                              proxy is bypassed entirely
                              @Transactional advice on validateOrder NEVER runs
```

**Visualized:**

```
The proxy is a shell around the raw bean:

    ┌─────────────────────────────────┐
    │  PROXY (what @Autowired gives)  │
    │  ┌───────────────────────────┐  │
    │  │  RAW OrderService         │  │
    │  │  placeOrder() {           │  │
    │  │    this.validateOrder()   │──┼──→ goes here (inside raw bean)
    │  │  }                        │  │    NOT to proxy → advice skipped
    │  │  validateOrder() { ... }  │  │
    │  └───────────────────────────┘  │
    └─────────────────────────────────┘

External call → hits proxy shell → advice runs → enters raw bean
Internal call → already inside raw bean → "this" = raw bean → advice skipped
```

**The four fixes:**

```java
// Fix 1 (BEST): Move to a separate bean — cleanest design
@Service
public class OrderValidator {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validate(Order order) { ... }  // own bean = own proxy ✅
}

@Service
public class OrderService {
    @Autowired OrderValidator validator;  // injected proxy

    @Transactional
    public void placeOrder(Order order) {
        validator.validate(order);  // → goes through OrderValidator's proxy ✅
    }
}

// Fix 2: Self-inject (inject yourself — you get the proxy)
@Service
public class OrderService {
    @Autowired
    @Lazy  // prevent circular dependency
    private OrderService self;  // Spring injects the PROXY of yourself

    @Transactional
    public void placeOrder(Order order) {
        self.validateOrder(order);  // through the proxy ✅
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) { ... }
}

// Fix 3: AopContext.currentProxy() — requires exposeProxy=true
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)  // must opt in
public class AppConfig { }

@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.validateOrder(order);  // through proxy ✅
    }
}
```

**Test that catches it:**

```java
// ❌ This test passes even though transactions are broken
@Test
void placeOrder_shouldValidateInNewTransaction() {
    orderService.placeOrder(testOrder);
    // How do you verify the transaction behavior? You don't. Silent.
}

// ✅ Test that catches the self-invocation bug:
@Test
@Transactional  // outer transaction
void validateOrder_shouldRunInNewTransaction() {
    // Call validateOrder THROUGH THE PROXY (not internally)
    orderService.validateOrder(testOrder);
    // If REQUIRES_NEW worked: new transaction was created
    // If self-invocation bug: REQUIRES_NEW ignored, uses outer tx
    // Verify with transaction listener or by checking TX status
}
```

---

### TODO 6: Multiple Aspects — Who Runs First?

```java
// Aspect 1: Security (should run OUTERMOST — reject before doing anything else)
@Aspect
@Component
@Order(1)  // lower number = outermost = runs first (wraps everything else)
public class SecurityAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object secure(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Security: BEFORE");
        Object result = pjp.proceed();
        System.out.println("Security: AFTER");
        return result;
    }
}

// Aspect 2: Transaction (should be closer to the method)
@Aspect
@Component
@Order(2)  // runs after Security in "before", before Security in "after"
public class TransactionAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object transact(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Transaction: BEFORE");
        Object result = pjp.proceed();
        System.out.println("Transaction: AFTER");
        return result;
    }
}
```

**Output when `orderService.placeOrder()` is called:**

```
Security: BEFORE        ← @Order(1) is outermost, runs first
Transaction: BEFORE     ← @Order(2) is inside Security
[placeOrder executes]
Transaction: AFTER      ← @Order(2) exits first (innermost)
Security: AFTER         ← @Order(1) exits last (outermost)
```

**Visualized like nested boxes:**

```
@Order(1) Security [
    @Order(2) Transaction [
        [actual placeOrder method]
    ]
]

Lower @Order number = outer box = first in, last out
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

### Decision 1: Which advice type?

```
Need to run something before and see the result?
    → @Around (only option that does both)

Need to prevent execution?
    → @Around (don't call pjp.proceed())
    → @Before (throw an exception)

Need to react to return value without modifying it?
    → @AfterReturning (cleaner intent)
    → @Around (more powerful but overkill)

Need to react to exceptions without suppressing them?
    → @AfterThrowing (clear intent)

Need cleanup that always runs?
    → @After (like finally)

Rule of thumb:
    @Around for anything non-trivial (logging, transactions, retries)
    @Before for pure "gate" logic (auth, validation)
    @After for cleanup
```

### Decision 2: JDK proxy vs CGLIB proxy

```
JDK Dynamic Proxy:
    Bean MUST implement an interface
    Proxy implements the same interface
    Cannot cast to the concrete class — ClassCastException
    spring.aop.proxy-target-class=false

CGLIB Proxy (Spring Boot default):
    No interface needed
    Proxy is a SUBCLASS of your class
    CAN cast to the concrete class
    Cannot intercept final methods (cannot subclass them)
    Cannot proxy final classes

Practical rule: Spring Boot uses CGLIB always. Don't fight it.
Just don't make your service classes or methods final.
```

### Decision 3: Named pointcuts vs inline expressions

```
Inline: quick, fine for one-off advice
Named (@Pointcut): reusable, composable, mandatory for complex pointcuts

// Always use named pointcuts when:
// - Same expression appears in multiple advice methods
// - Expression is long / complex
// - You want to document what the pointcut MEANS
@Pointcut("execution(* com.example.service..*.*(..))")
public void serviceLayer() {}  // clear name = clear intent

@Pointcut("@annotation(com.example.security.RequiresAdmin)")
public void adminOperations() {}

@Before("serviceLayer() && adminOperations()")
public void checkAdminAndLog(JoinPoint jp) { ... }
```

---

## 🧪 7. Edge Cases & Testing

### Silent Break #1: Static methods are never intercepted

```java
@Service
public class OrderService {
    public static Order createDraftOrder(String type) {
        return new Order(type); // static → NO proxy → NO advice
    }
}

// ❌ This advice never fires for static methods
@Before("execution(* com.example.service.*.*(..))")
public void beforeAll(JoinPoint jp) { }

// No error. The method runs. The advice doesn't.
// Test: verify your advice fires for static methods — it won't.
```

### Silent Break #2: Final methods with CGLIB

```java
@Service
public class OrderService {
    public final void placeOrder(Order order) {  // FINAL
        orderRepository.save(order);
    }
}

// CGLIB cannot override final methods.
// Spring creates the proxy. The proxy starts.
// But calling placeOrder() goes directly to the target — bypasses proxy.
// @Transactional, @Cacheable, your custom aspects — all silently skipped.
// No exception. No warning.

// ✅ Catch it: never make service methods final
// ✅ Catch it in test:
@Test
void transactionalFinalMethod_shouldHaveTransaction() {
    // Verify transaction is active inside placeOrder
    // If final: TransactionSynchronizationManager.isActualTransactionActive() = false
}
```

### Silent Break #3: Non-Spring-managed objects

```java
// ❌ You create the object yourself — no proxy
OrderService service = new OrderService();  // raw object, not from Spring
service.placeOrder(order);  // @Transactional? Not applied. @Cacheable? Not applied.

// ✅ Always get beans from Spring:
OrderService service = ctx.getBean(OrderService.class);  // gives you the proxy
service.placeOrder(order);  // all advice applied ✅
// OR: just use @Autowired — always gives you the proxy
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version — scattered advice, no reuse:**

```java
// ❌ Advice mixed, inline expressions repeated everywhere
@Aspect @Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void log(JoinPoint jp) { log.info("→ {}", jp.getSignature()); }
}

@Aspect @Component
public class SecurityAspect {
    @Before("execution(* com.example.service.*.*(..))")  // duplicated expression
    public void check(JoinPoint jp) { ... }
}
```

**Senior refactored version — centralized pointcuts, single comprehensive aspect:**

```java
// ✅ Centralized, reusable pointcut library
public class Pointcuts {

    @Pointcut("within(com.example.service..*)")
    public void serviceLayer() {}

    @Pointcut("within(com.example.repository..*)")
    public void repositoryLayer() {}

    @Pointcut("@annotation(com.example.annotation.AdminOnly)")
    public void adminOperations() {}

    @Pointcut("serviceLayer() && !@annotation(com.example.annotation.NoAudit)")
    public void auditableServiceOperations() {}
}

// ✅ Ordered, comprehensive aspect with single responsibility
@Aspect
@Component
@Order(1)  // outermost — first to check, last to exit
@Slf4j
public class OperationalAspect {

    // Reference pointcuts from the library
    @Around("com.example.aop.Pointcuts.serviceLayer()")
    public Object operationWrapper(ProceedingJoinPoint pjp) throws Throwable {
        String operation = pjp.getSignature().toShortString();

        // 1. Auth check
        if (!SecurityContext.isAuthenticated()) {
            throw new SecurityException("Unauthenticated: " + operation);
        }

        // 2. Timing
        long start = System.nanoTime();

        try {
            Object result = pjp.proceed();

            // 3. Success metrics
            metrics.recordSuccess(operation,
                (System.nanoTime() - start) / 1_000_000);
            return result;

        } catch (Exception e) {
            // 4. Failure metrics + alerting
            metrics.recordFailure(operation, e.getClass().getSimpleName());
            if (e instanceof BusinessException) {
                log.warn("Business rule: {} — {}", operation, e.getMessage());
            } else {
                log.error("Unexpected failure: {}", operation, e);
                alerting.notify(operation, e);
            }
            throw e;
        } finally {
            // 5. Cleanup always
            MDC.clear();
        }
    }
}
```

**What changed and why:**
- Pointcuts centralized in one class — change once, affects everywhere
- Single `@Around` handles auth + logging + metrics + alerting — one place to maintain
- `@Order` explicit — no mystery about execution order
- Business vs unexpected exceptions handled differently — better observability
- `finally` for MDC cleanup — guaranteed even on exception

---

## 🧠 9. Mental Model Summary

### The jargon, decoded

| Term | What it actually means |
|---|---|
| **JoinPoint** | A method call that Spring CAN intercept. In Spring AOP: only method executions on Spring beans. |
| **Pointcut** | A pattern/filter that selects WHICH joinpoints get intercepted. `execution(* com.example.service.*.*(..))` = "all methods in service package" |
| **Advice** | The code that runs at the matched joinpoints. `@Before`, `@After`, `@Around` etc. |
| **Aspect** | A class (`@Aspect @Component`) that packages pointcuts + advice together |
| **Proxy** | A wrapper Spring generates around your bean. `ctx.getBean(OrderService.class)` returns this wrapper, not the original. |
| **Target** | The real original bean object hidden inside the proxy. `pjp.getTarget()` gives you this. |
| **Weaving** | The process of "connecting" aspects to beans. Spring does this at runtime by creating proxies. |
| **`@Before`** | "Run this before the method. Can't see the return value. Can throw to prevent execution." |
| **`@After`** | "Run this after the method, always. Like finally. Can't see return value or exception." |
| **`@AfterReturning`** | "Run this only on success. Can read (but not change) the return value." |
| **`@AfterThrowing`** | "Run this only on exception. Can read (but not suppress) the exception." |
| **`@Around`** | "Wrap the entire call. Call `pjp.proceed()` to invoke the real method. Can modify args, return value, suppress exceptions." |
| **`JoinPoint`** | Parameter available in all advice — gives you method name, args, target |
| **`ProceedingJoinPoint`** | `@Around`-only parameter — extends JoinPoint with `proceed()` to call the real method |
| **Self-invocation** | When a bean calls its own method internally (`this.method()`). Bypasses the proxy. All AOP silently skipped. |
| **`proxyTargetClass`** | When `true` (Spring Boot default): always CGLIB. When `false`: JDK proxy if interface exists. |
| **CGLIB proxy** | Generated SUBCLASS of your bean. Can cast to concrete type. Cannot intercept `final` methods. |
| **JDK proxy** | Generated class that implements your INTERFACES. Cannot cast to concrete type. |

---

### The complete AOP picture

```
Your service class:                 Spring's view at runtime:

@Service                            ┌──────────────────────────────┐
public class OrderService {         │  OrderService$$CGLIB$$0      │ ← what @Autowired gives
    @Transactional                  │  (proxy — subclass of yours) │
    public void placeOrder(..) {    │                              │
        // business logic           │  overrides placeOrder():     │
    }                               │    run advice chain          │
}                                   │    call super.placeOrder()   │
                                    │    run more advice           │
                                    │                              │
                                    │  holds reference to:         │
                                    │  ┌──────────────────────┐   │
                                    │  │ raw OrderService     │   │ ← pjp.getTarget()
                                    │  │ (original bean)      │   │
                                    │  └──────────────────────┘   │
                                    └──────────────────────────────┘
```

---

### The pointcut cheat sheet

```
All methods in package:            execution(* com.example.service.*.*(..))
All methods in pkg + sub-pkgs:     execution(* com.example.service..*.*(..))
All public methods:                execution(public * *(..))
Methods annotated with X:          @annotation(com.example.X)
Classes annotated with @Service:   @within(org.springframework.stereotype.Service)
All in package (cleaner):          within(com.example.service..*)
By bean name:                      bean(*ServiceImpl)
Combine:                           serviceLayer() && !adminOperations()
```

---

### The one decision rule

```
Something needs to happen for many methods, but doesn't belong in any of them?
    → It's a cross-cutting concern → use AOP

Pick your advice type:
    Gate/check before?          → @Before (can throw to block)
    Wrap before + after?        → @Around (most common)
    React to success only?      → @AfterReturning
    React to failure only?      → @AfterThrowing
    Always cleanup?             → @After

Watch out for:
    Calling your own methods internally?  → self-invocation trap
    Making methods or classes final?      → CGLIB can't intercept
    Forgetting @Component on @Aspect?     → aspect never runs
    Forgetting to return result in @Around? → all methods return null
```
