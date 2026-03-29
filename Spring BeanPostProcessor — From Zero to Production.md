# Spring BeanPostProcessor — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

Your platform team has a new set of tickets — all variations of the same underlying need:

```
TICKET-441: Add metrics to every service method automatically
- We have 60 @Service beans. Every public method needs timing metrics.
- We cannot ask every team to manually add timer code to their methods.
- Adding @Timed to each method is acceptable but the WRAPPING must be automatic.

TICKET-442: Validate bean state after injection — before the bean goes live
- Some beans have fields that MUST not be null after @Autowired runs.
- If a required field is null, the app should fail at startup, not at 3am.
- Currently: NullPointerException in production at runtime.

TICKET-443: Custom cleanup for beans holding native resources
- Some beans hold file handles, socket connections, native memory.
- @PreDestroy isn't enough — we need to intercept destruction ourselves.
- We need to track which beans hold which resources.

TICKET-444: Inject the current user's locale into beans that implement LocaleAware
- Without making every bean implement ApplicationContextAware.
- Without adding boilerplate to 40+ beans.
```

You look at what you know and try each:

```java
// For TICKET-441: try AOP @Around on all @Service beans
@Aspect
@Component
public class MetricsAspect {
    @Around("@within(org.springframework.stereotype.Service)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try { return pjp.proceed(); }
        finally { recordMetric(pjp.getSignature().getName(),
                              System.currentTimeMillis() - start); }
    }
}
// This actually works... but AOP itself IS implemented via a BeanPostProcessor.
// The moment you need to do something AOP can't do (wrap a non-interface, modify
// the bean object itself, trigger on instantiation, not just method calls) — you're stuck.

// For TICKET-442: try @PostConstruct validation in each bean
@Service
public class OrderService {
    @Autowired private PaymentGateway paymentGateway;

    @PostConstruct
    public void validate() {
        if (paymentGateway == null)
            throw new IllegalStateException("paymentGateway must not be null");
    }
}
// Problem: every bean needs this boilerplate. 60 beans = 60 @PostConstruct methods.
// And you can't enforce it — a developer forgets to add it. Validation is optional.

// For TICKET-443: try @PreDestroy in each bean
@Service
public class FileProcessorService {
    private FileChannel channel;

    @PreDestroy
    public void close() { channel.close(); }  // but what if channel is null?
    // And you can't track which beans are holding resources from outside.
    // You can't audit the cleanup from a central place.
}

// For TICKET-444: try ApplicationContextAware on each bean
@Service
public class ReportService implements ApplicationContextAware {
    private Locale locale;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        // ctx has no Locale — you'd need to do something else entirely
        // And you're adding ApplicationContextAware to 40 beans.
    }
}
```

Everything runs TOO LATE (at method call time) or TOO EARLY (at definition time) or requires TOO MUCH PER-BEAN BOILERPLATE.

**What you actually need:**

```
A hook that runs for EVERY bean, individually, after it's created and injected,
where you can:
  → inspect its final state
  → wrap it in something else (a proxy, a decorator)
  → call callbacks on it (setLocale(), validate())
  → register it somewhere (resource tracker)
  → replace it with a completely different object
→ And all of this happens ONCE PER BEAN, automatically, with zero boilerplate in the bean itself.
```

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Loop over all beans in a `@PostConstruct` somewhere**

```java
@Component
public class StartupHook {
    @Autowired ApplicationContext ctx;

    @PostConstruct
    public void processAllBeans() {
        ctx.getBeanDefinitionNames()
           .forEach(name -> {
               Object bean = ctx.getBean(name);
               applyValidation(bean);
               applyMetrics(bean);
           });
    }
}
```

**Why this fails:** By the time `@PostConstruct` on `StartupHook` runs, ALL other beans are ALREADY CREATED and ALREADY in the container. If `OrderService` had a null field, it's been sitting in the container for seconds already — other beans may have already been injected with it. You can't wrap it in a proxy at this point (the reference to the raw bean is already held by other beans that autowired it). And calling `ctx.getBean()` in a loop forces ALL beans to be created eagerly, which may itself cause failures or ordering issues.

---

**Naive Approach 2: Subclass every bean to add validation**

```java
public class ValidatedOrderService extends OrderService {
    @Override
    public void placeOrder(Order o) {
        Objects.requireNonNull(paymentGateway, "paymentGateway must not be null");
        super.placeOrder(o);
    }
}
```

**Why this fails:** You're maintaining 60 validation subclasses. Every time the original class changes, the subclass may need updating. It doesn't compose — if you want BOTH metrics AND validation, you need `MetricsValidatedOrderService extends ValidatedMetricsOrderService extends OrderService`. The class hierarchy explodes. And you still need to manually register the subclass instead of the original.

---

**Naive Approach 3: Use `@EventListener(ContextRefreshedEvent.class)`**

```java
@EventListener(ContextRefreshedEvent.class)
public void onRefresh(ContextRefreshedEvent event) {
    ApplicationContext ctx = event.getApplicationContext();
    // same problem as approach 1 — too late to replace beans
}
```

**Why this fails:** Same as approach 1 — by `ContextRefreshedEvent`, all beans are fully created AND all dependencies are already injected with references to those beans. Replacing a bean now would orphan all the existing references. The container holds the mapping `"orderService" → OrderService instance`, but `paymentService` already holds a direct reference to that same instance — changing the container mapping doesn't update `paymentService`'s field.

---

**The correct solution emerges:**

There's a specific phase in each bean's lifecycle — after it's fully built but BEFORE it's handed to anyone else — where you can intercept. At that moment, the bean exists as a complete object but no one holds a reference to it yet. If you return a replacement object (a proxy, a wrapper), THAT replacement is what gets stored in the container and handed to everyone who autowires this bean. Spring calls components that run in this phase `BeanPostProcessors`.

```
Bean is created + dependencies injected
    ↓
[ YOUR CODE RUNS HERE — BPP phase ]
  → you see the complete bean
  → you can validate it, modify it, or return a DIFFERENT object entirely
  → if you return a proxy, the PROXY is what other beans get when they @Autowire this
    ↓
The bean (or your replacement) is stored in the container
Other beans @Autowired this — they get whatever YOU returned
```

---

## 🧠 3. Concept Introduction — Plain English First

### The Assembly Line Quality Inspector Analogy

Imagine a car factory assembly line. At the end of the line, before a car rolls off to the parking lot (the Spring container), there are several **inspection stations**:

- Station 1: Checks for missing parts (validates required fields)
- Station 2: Installs a GPS tracker (registers the bean in a resource tracker)
- Station 3: Wraps the car in protective film for transport (wraps in a proxy)
- Station 4: Attaches a monitoring device that logs every time the car moves (metrics wrapper)

Each station looks at the car, can modify it, can attach things to it, and can even say "throw this one out and substitute this better model." The car that leaves the last station is what gets delivered to the customer (what other beans get when they `@Autowire`).

```
Bean created (car rolls off initial assembly)
    ↓ 
BPP-1.postProcessBeforeInitialization()  ← before quality control tests run
    ↓
@PostConstruct / afterPropertiesSet()    ← quality control tests (bean self-checks)
    ↓
BPP-1.postProcessAfterInitialization()  ← WRAP IN PROXY (AOP happens here)
BPP-2.postProcessAfterInitialization()  ← add metrics wrapper
BPP-3.postProcessAfterInitialization()  ← validate and register
    ↓
Final object stored in container — this is what everyone gets
```

Each BPP receives what the PREVIOUS BPP returned. They form a pipeline. If BPP-2 returns a proxy wrapping the original bean, BPP-3 receives the proxy (not the original bean).

### What BPPs See and Can Do

```
BPP receives:
  Object bean    ← the actual bean instance (or whatever the previous BPP returned)
  String beanName ← "orderService", "paymentGateway", etc.

BPP can return:
  → The SAME bean (no change — just inspection)
  → A DECORATED version (wraps the bean, adds behavior)
  → A PROXY (intercepts method calls)
  → A COMPLETELY DIFFERENT object (rare — must implement same interface)
  → null (means "no change" — previous result kept)

The returned object is what gets stored in the container.
```

### The Two Callback Points

```
postProcessBeforeInitialization():
  Called AFTER dependency injection but BEFORE @PostConstruct/@afterPropertiesSet
  Bean has all its fields set (autowiring complete)
  Good for: calling callbacks on the bean (setLocale, setContext), logging

postProcessAfterInitialization():
  Called AFTER @PostConstruct and afterPropertiesSet/init-method
  Bean is completely initialized
  Good for: wrapping in proxy, validation, registering, replacing
  AOP PROXY CREATION HAPPENS HERE
```

### The Full Sub-interface Family

```
BeanPostProcessor (base)
  → postProcessBeforeInitialization()   [after DI, before init callbacks]
  → postProcessAfterInitialization()    [after ALL init — AOP happens here]

InstantiationAwareBeanPostProcessor (extends BPP)
  → postProcessBeforeInstantiation()    [BEFORE constructor — can short-circuit entirely]
  → postProcessAfterInstantiation()     [after constructor, before DI — can skip DI]
  → postProcessProperties()             [DURING DI — @Autowired injection happens here]

MergedBeanDefinitionPostProcessor (extends BPP)
  → postProcessMergedBeanDefinition()   [after constructor, before DI — scan metadata, cache it]

DestructionAwareBeanPostProcessor (extends BPP)
  → postProcessBeforeDestruction()      [before @PreDestroy]
  → requiresDestruction()               [optimization: skip if this bean needs no cleanup]

SmartInstantiationAwareBeanPostProcessor (extends IABPP)
  → determineCandidateConstructors()    [which constructor to use]
  → getEarlyBeanReference()             [circular dependency support]
  → predictBeanType()                   [type resolution before creation]
```

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TODO 1: Write the simplest BPP — learn the API, understand the pipeline
TODO 2: Solve TICKET-442: validate bean state after initialization (postProcessAfterInitialization)
TODO 3: Solve TICKET-441: wrap service beans in timing decorators (postProcessAfterInitialization, returns replacement)
TODO 4: Solve TICKET-444: inject locale into LocaleAware beans (postProcessBeforeInitialization)
TODO 5: Understand MergedBeanDefinitionPostProcessor — how @Autowired metadata scanning works
TODO 6: Solve TICKET-443: custom destruction tracking (DestructionAwareBeanPostProcessor)
TODO 7: Understand InstantiationAwareBPP — the hooks before the constructor
TODO 8: Understand ordering — why @PostConstruct runs before your custom BPP
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: The simplest BPP — learn the pipeline

```java
// BeanPostProcessor: the interface with two methods.
// Spring calls BOTH methods for EVERY bean in the container — one by one, as each bean is created.
// This is NOT a batch operation. It runs once per bean, mid-creation.
@Component
public class LifecycleTrackerBPP implements BeanPostProcessor {

    // postProcessBeforeInitialization:
    //   Called AFTER dependency injection (all @Autowired fields are set)
    //   Called BEFORE @PostConstruct, afterPropertiesSet(), and init-method
    //   The bean is "ready but not yet validated by itself"
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (isApplicationBean(beanName)) {
            System.out.println("BEFORE init: " + beanName
                + " [" + bean.getClass().getSimpleName() + "]");
        }
        // MUST return a non-null object — return the same bean to leave it unchanged.
        // Returning null is treated as "no change" (use previous result).
        // This is the most common source of bugs: forgetting to return the bean.
        return bean;
    }

    // postProcessAfterInitialization:
    //   Called AFTER @PostConstruct, afterPropertiesSet(), and init-method
    //   The bean is FULLY initialized — this is where AOP proxies are created
    //   Whatever you return becomes the bean stored in the container
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (isApplicationBean(beanName)) {
            // AopUtils.getTargetClass(): if the bean is already wrapped in an AOP proxy
            // (by a BPP that ran before this one), this unwraps it to get the real class.
            // Without this: bean.getClass() might be OrderService$$EnhancerByCGLIB$$...
            System.out.println("AFTER init:  " + beanName
                + " [" + AopUtils.getTargetClass(bean).getSimpleName() + "]"
                + (AopUtils.isAopProxy(bean) ? " (PROXY)" : ""));
        }
        return bean;
    }

    private boolean isApplicationBean(String name) {
        return !name.startsWith("org.springframework") && !name.contains("$$");
    }
}
```

**What you'll see when this runs — the ordering revelation:**

```
BEFORE init: orderService [OrderService]          ← your BEFORE BPP runs
BEFORE init: paymentGateway [StripeGateway]       ← @PostConstruct hasn't fired yet for these beans
                                                     WAIT — why?
                                                     
Actual output depends on BPP registration order:
  CommonAnnotationBPP (PriorityOrdered) runs FIRST → calls @PostConstruct
  Your BPP (non-ordered) runs SECOND → sees bean after @PostConstruct already ran

So the real output is:
  [OrderService @PostConstruct]     ← CommonAnnotationBPP called @PostConstruct before your BEFORE
  BEFORE init: orderService         ← your postProcessBeforeInitialization  
  AFTER init:  orderService (PROXY) ← your postProcessAfterInitialization (after AOP proxy created)
```

**The BPP ordering trap — explained:**

```
BPPs are registered in tiers:
  Tier 1 (PriorityOrdered): AutowiredAnnotationBPP, CommonAnnotationBPP
  Tier 2 (Ordered): custom BPPs implementing Ordered
  Tier 3 (no ordering): your BPP above

For EACH bean, ALL registered BPPs run in registration order.
CommonAnnotationBPP is PriorityOrdered → it processes BEFORE your non-ordered BPP.
CommonAnnotationBPP.postProcessBeforeInitialization() → calls @PostConstruct.

So when YOUR postProcessBeforeInitialization() runs, @PostConstruct HAS ALREADY FIRED.
"Before initialization" means before afterPropertiesSet() and init-method,
but @PostConstruct is itself called by CommonAnnotationBPP.postProcessBeforeInitialization()
which already ran in Tier 1.
```

---

### TODO 2: Solve TICKET-442 — validate bean state after initialization

```java
// Custom annotation: put this on fields that MUST be non-null after all injection is done.
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Required {
    String message() default "Field must not be null after injection";
}

// The BPP that enforces it:
@Component
public class RequiredFieldValidatorBPP implements BeanPostProcessor, Ordered {

    @Override
    public int getOrder() {
        // Ordered: we want to run AFTER @Autowired injection (which is handled by
        // AutowiredAnnotationBPP, PriorityOrdered) and AFTER @PostConstruct.
        // Ordered.LOWEST_PRECEDENCE = run as late as possible in the after-init phase.
        return Ordered.LOWEST_PRECEDENCE;
    }

    // Cache: class → list of @Required fields.
    // IMPORTANT: BPP is a singleton. For prototype beans, the same class can be
    // instantiated thousands of times. We don't want to scan via reflection every time.
    // ConcurrentHashMap because multiple threads may be creating beans simultaneously.
    private final Map<Class<?>, List<Field>> fieldCache = new ConcurrentHashMap<>();

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // AopUtils.getTargetClass(): if a previous BPP already wrapped this bean in a proxy,
        // we need the ORIGINAL class to find its @Required annotations.
        // bean.getClass() would give us the CGLIB proxy class — annotations aren't there.
        Class<?> targetClass = AopUtils.getTargetClass(bean);

        // computeIfAbsent: scan once per class, cache forever.
        // This is the critical performance pattern for any BPP that does reflection.
        List<Field> requiredFields = fieldCache.computeIfAbsent(targetClass, clazz -> {
            List<Field> fields = new ArrayList<>();
            // ReflectionUtils.doWithFields: walks the entire class hierarchy
            // (class, superclass, superclass's superclass...) — finds inherited fields too.
            ReflectionUtils.doWithFields(clazz, fields::add,
                field -> field.isAnnotationPresent(Required.class));
            return Collections.unmodifiableList(fields);
        });

        if (requiredFields.isEmpty()) return bean;  // fast path: no @Required fields

        for (Field field : requiredFields) {
            ReflectionUtils.makeAccessible(field);  // bypass private modifier
            Object value;
            try {
                value = field.get(bean);
            } catch (IllegalAccessException e) {
                throw new BeanInitializationException(
                    "Cannot read field '" + field.getName() + "' on '" + beanName + "'", e);
            }

            if (value == null) {
                Required annotation = field.getAnnotation(Required.class);
                // BeanInitializationException: the correct exception to throw from a BPP.
                // Spring treats this as a fatal startup error — app refuses to start.
                // Much better than NullPointerException at 3am in production.
                throw new BeanInitializationException(
                    "Bean '" + beanName + "', field '" + field.getName() + "': "
                    + annotation.message());
            }
        }
        return bean;  // bean passes validation — return unchanged
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // nothing to do before init
    }
}

// Usage — completely transparent to the service developer:
@Service
public class OrderService {

    @Required  // ← if this is null after @Autowired, app refuses to start with clear error
    @Autowired
    private PaymentGateway paymentGateway;

    @Required
    @Autowired
    private InventoryService inventoryService;

    // No @PostConstruct validation method needed.
    // No boilerplate. The BPP handles it for every bean in the app.
}
```

**The AopUtils.getTargetClass() trap:**

```java
// WRONG: bean.getClass() when bean might be a proxy
Class<?> clazz = bean.getClass();
// For an AOP-proxied bean: clazz = OrderService$$EnhancerByCGLIB$$0
// Scanning this class for @Required finds nothing — annotations are on OrderService, not CGLIB class

// CORRECT: unwrap the proxy to get the real class
Class<?> clazz = AopUtils.getTargetClass(bean);
// For an AOP-proxied bean: clazz = OrderService (the real target)
// Now @Required is found on the actual fields

// When does this matter?
// Our RequiredFieldValidatorBPP is Ordered (runs late).
// AbstractAutoProxyCreator (AOP) is also registered — it might run before us.
// If AOP runs first, the bean we receive is already a proxy.
// Always use AopUtils.getTargetClass() in BPPs for safety.
```

---

### TODO 3: Solve TICKET-441 — wrap beans in timing decorators

This is where BPPs show their real power: returning a DIFFERENT object entirely.

```java
// When postProcessAfterInitialization returns a different object,
// THAT object is what gets stored in the container.
// Every bean that @Autowires OrderService gets the TimingWrapper, not the raw OrderService.
@Component
public class TimingDecoratorBPP implements BeanPostProcessor {

    @Autowired
    private MeterRegistry meterRegistry;

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> targetClass = AopUtils.getTargetClass(bean);

        // Only wrap @Service beans (not repositories, controllers, BPPs themselves)
        if (!targetClass.isAnnotationPresent(Service.class)) {
            return bean;  // no-op — return unchanged
        }

        // Find interfaces this bean implements — needed for JDK dynamic proxy
        Class<?>[] interfaces = targetClass.getInterfaces();
        if (interfaces.length == 0) {
            // No interfaces → can't use JDK proxy.
            // CGLIB would work but let's keep this simple for demonstration.
            return bean;
        }

        System.out.println("Wrapping " + beanName + " in timing decorator");

        // Proxy.newProxyInstance: creates a JDK dynamic proxy.
        // The proxy intercepts ALL method calls on the bean.
        // This is a REPLACEMENT — the returned proxy IS what the container stores.
        // Every bean that @Autowires OrderService gets this proxy, not the raw OrderService.
        Object proxy = Proxy.newProxyInstance(
            targetClass.getClassLoader(),
            interfaces,
            (proxyObj, method, args) -> {
                // This lambda runs on EVERY method call to the proxy
                Timer.Sample sample = Timer.start(meterRegistry);
                try {
                    // pjp.proceed() equivalent: call the real method on the real bean
                    return method.invoke(bean, args);
                } catch (InvocationTargetException e) {
                    throw e.getCause();  // unwrap reflection wrapper
                } finally {
                    sample.stop(Timer.builder("service.method.duration")
                        .tag("bean", beanName)
                        .tag("method", method.getName())
                        .register(meterRegistry));
                }
            }
        );

        return proxy;  // ← the container stores this proxy, not `bean`
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

**The reference chain after this BPP runs:**

```
Container registry:
  "orderService" → TimingProxy (wraps OrderService)

PaymentService has:
  @Autowired OrderService orderService;
  → paymentService.orderService points to TimingProxy

When PaymentService calls orderService.placeOrder():
  → goes through TimingProxy.invoke()
  → records start time
  → calls real OrderService.placeOrder()
  → records end time, registers metric
  → PaymentService never knows the difference
```

**The "returning a different object" pipeline trap:**

```java
// BPP-A runs first, returns ProxyA wrapping the original bean
// BPP-B runs second, receives ProxyA as "bean"

@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // bean here is ProxyA — NOT the original OrderService
    Class<?> targetClass = bean.getClass();  // ← WRONG: gives ProxyA$$CGLIB class
    // Scanning ProxyA for @Service annotation → not found → returns bean unchanged
    // Your wrapping is SKIPPED for all AOP-proxied beans!
    
    // FIX: always use AopUtils.getTargetClass()
    Class<?> targetClass2 = AopUtils.getTargetClass(bean);  // ← OrderService (correct)
}
```

---

### TODO 4: Solve TICKET-444 — inject locale into LocaleAware beans

`postProcessBeforeInitialization` is perfect for "call a callback on the bean to give it something" — without making the bean implement Spring infrastructure interfaces.

```java
// Simple marker interface — teams implement this, BPP provides the value
public interface LocaleAware {
    void setLocale(Locale locale);
}

// BPP that injects locale into any bean implementing LocaleAware
// This is the same pattern Spring uses internally for:
//   ApplicationContextAware → setApplicationContext()
//   BeanNameAware → setBeanName()
//   EnvironmentAware → setEnvironment()
// They're all implemented in ApplicationContextAwareProcessor — a BPP.
@Component
public class LocaleAwareBPP implements BeanPostProcessor, PriorityOrdered {

    @Autowired
    private LocaleConfiguration localeConfig;

    // PriorityOrdered: inject locale BEFORE @PostConstruct fires.
    // If the bean's @PostConstruct needs the locale (e.g., to format a welcome message),
    // we must set it in postProcessBeforeInitialization(), before @PostConstruct runs.
    // But we need to be careful: PriorityOrdered runs before CommonAnnotationBPP...
    // Actually no — within PriorityOrdered, order values determine the sequence.
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 50;
        // Run early among PriorityOrdered BPPs — before @PostConstruct fires
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // instanceof check: fast, no reflection needed
        if (bean instanceof LocaleAware) {
            Locale locale = localeConfig.getDefaultLocale();
            ((LocaleAware) bean).setLocale(locale);
            System.out.println("Injected locale " + locale + " into " + beanName);
        }
        return bean;  // same bean — just called a method on it
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;  // nothing needed after init
    }
}

// Usage — zero boilerplate in the service:
@Service
public class ReportService implements LocaleAware {

    private Locale locale;

    @Override
    public void setLocale(Locale locale) {
        this.locale = locale;  // BPP calls this before @PostConstruct
    }

    @PostConstruct
    public void init() {
        // locale is already set by the time @PostConstruct runs — we can use it here
        System.out.println("ReportService ready with locale: " + locale);
    }

    public String generateReport(Report data) {
        // Uses this.locale — set automatically without any boilerplate
        return formatReport(data, locale);
    }
}
```

**This is EXACTLY how Spring's own Aware interfaces work:**

```java
// ApplicationContextAwareProcessor (a built-in BPP):
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    // Spring's version of what we just did for Locale:
    if (bean instanceof ApplicationContextAware aware) {
        aware.setApplicationContext(this.applicationContext);
    }
    if (bean instanceof EnvironmentAware aware) {
        aware.setEnvironment(this.environment);
    }
    if (bean instanceof BeanNameAware aware) {
        aware.setBeanName(beanName);
    }
    // ... etc
    return bean;
}
// Spring didn't need any special mechanism for these Aware interfaces.
// They're just BPP callbacks calling setter methods.
// Your LocaleAware follows the exact same pattern.
```

---

### TODO 5: MergedBeanDefinitionPostProcessor — how @Autowired scanning works

You never write this interface directly for application code, but understanding it explains why `@Autowired` is fast for prototype beans.

```java
// MergedBeanDefinitionPostProcessor: a BPP sub-interface with a DIFFERENT callback point.
// postProcessMergedBeanDefinition() fires AFTER the constructor runs,
// but BEFORE dependency injection and BEFORE addSingletonFactory (L3 cache).
//
// Its purpose: SCAN the class for injection points ONCE and CACHE the result.
// For prototype beans created 1000 times, this avoids 1000 reflection scans.
//
// Built-in example: AutowiredAnnotationBPP scans for @Autowired fields/methods here.
// Built-in example: CommonAnnotationBPP scans for @PostConstruct/@PreDestroy here.

@Component
public class AnnotationMetadataCacheBPP
        implements MergedBeanDefinitionPostProcessor, BeanPostProcessor {

    // Cache maps class → injection metadata (scanned once per class, used N times)
    private final Map<Class<?>, InjectionMetadata> metadataCache = new ConcurrentHashMap<>();

    // This fires AFTER constructor, BEFORE DI — use it to scan and cache
    @Override
    public void postProcessMergedBeanDefinition(
            RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {

        // Scan the class once:
        List<Field> configFields = new ArrayList<>();
        ReflectionUtils.doWithFields(beanType, configFields::add,
            f -> f.isAnnotationPresent(ConfigValue.class));

        // Cache it indexed by the class:
        metadataCache.put(beanType, new InjectionMetadata(configFields));

        System.out.println("Cached " + configFields.size()
            + " @ConfigValue fields for " + beanType.getSimpleName());
    }

    // Now use the cached data during the after-init phase:
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> targetClass = AopUtils.getTargetClass(bean);
        InjectionMetadata metadata = metadataCache.get(targetClass);

        if (metadata == null || metadata.fields().isEmpty()) return bean;

        for (Field field : metadata.fields()) {
            ConfigValue annotation = field.getAnnotation(ConfigValue.class);
            String value = ConfigStore.getValue(annotation.key());
            if (value != null) {
                ReflectionUtils.makeAccessible(field);
                try {
                    field.set(bean, value);
                } catch (IllegalAccessException e) {
                    throw new BeanInitializationException("Cannot inject @ConfigValue", e);
                }
            }
        }
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    // Simple cache record:
    record InjectionMetadata(List<Field> fields) {}
}
```

**Why the timing matters for MergedBeanDefinitionPostProcessor:**

```
doCreateBean():
  1. createBeanInstance() ← constructor runs
  2. MBDPP.postProcessMergedBeanDefinition() ← scan now, before L3 cache
  3. addSingletonFactory() ← add to L3 cache (for circular dependency resolution)
  4. populateBean() ← DI runs (uses cached metadata from step 2)
  5. initializeBean() ← BPP.before, @PostConstruct, BPP.after

If MBDPP scanned AFTER addSingletonFactory:
  → L3 cache holds early reference without injection metadata
  → Circular dependency resolution might give out a bean with incomplete metadata

If we used regular BPP.postProcessBeforeInitialization() for scanning instead:
  → That fires after DI (step 4 already happened)
  → Too late to affect DI
  → Must re-scan on every prototype instantiation (no shared MBDPP cache timing)
```

---

### TODO 6: Solve TICKET-443 — DestructionAwareBeanPostProcessor

```java
// DestructionAwareBeanPostProcessor: extends BPP with destruction callbacks.
// postProcessBeforeDestruction(): called before @PreDestroy and DisposableBean.destroy()
// requiresDestruction(): optimization — return false if this bean needs no cleanup
@Component
public class NativeResourceTrackerBPP
        implements DestructionAwareBeanPostProcessor, BeanPostProcessor {

    // Track which beans hold which resources:
    // beanName → list of closeable resources
    private final Map<String, List<AutoCloseable>> trackedResources = new ConcurrentHashMap<>();

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // ResourceHolder: a marker interface teams implement on beans that hold native resources
        if (bean instanceof ResourceHolder holder) {
            List<AutoCloseable> resources = holder.getNativeResources();
            if (resources != null && !resources.isEmpty()) {
                trackedResources.put(beanName, new ArrayList<>(resources));
                System.out.println("Tracking " + resources.size()
                    + " native resources for bean: " + beanName);
            }
        }
        return bean;
    }

    // requiresDestruction(): called by Spring when deciding whether to register
    // a bean for destruction processing. Return false = skip this bean entirely.
    // This is a PERFORMANCE OPTIMIZATION — beans with nothing to clean up
    // don't get added to the destruction tracking list.
    @Override
    public boolean requiresDestruction(Object bean) {
        // Only register for destruction if:
        //   a) bean implements ResourceHolder AND
        //   b) it's in our tracked map (meaning it had resources at init time)
        return bean instanceof ResourceHolder
            && trackedResources.containsKey(
                ((ResourceHolder) bean).getBeanName());
    }

    // postProcessBeforeDestruction(): called before @PreDestroy
    // This is the RIGHT time to release resources — before the bean cleans itself up
    @Override
    public void postProcessBeforeDestruction(Object bean, String beanName) {
        List<AutoCloseable> resources = trackedResources.remove(beanName);
        if (resources == null) return;

        System.out.println("Releasing " + resources.size()
            + " native resources for bean: " + beanName);

        for (AutoCloseable resource : resources) {
            try {
                resource.close();
            } catch (Exception e) {
                // Log but don't rethrow — continue closing other resources
                System.err.println("Failed to close resource for " + beanName
                    + ": " + e.getMessage());
            }
        }

        // Audit log: central place to track all resource releases
        auditLog.record("Released resources for " + beanName
            + " at " + LocalDateTime.now());
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // nothing needed before init
    }
}

// Interface for beans with native resources:
public interface ResourceHolder {
    List<AutoCloseable> getNativeResources();
    String getBeanName();  // or inject this via BeanNameAware
}

// A team's service — just implements the interface, tracker does the rest:
@Service
public class FileProcessorService implements ResourceHolder, BeanNameAware {

    private FileChannel inputChannel;
    private FileChannel outputChannel;
    private String beanName;

    @Override
    public void setBeanName(String name) { this.beanName = name; }

    @PostConstruct
    public void openFiles() throws IOException {
        inputChannel  = FileChannel.open(Path.of("/data/input.bin"), READ);
        outputChannel = FileChannel.open(Path.of("/data/output.bin"), WRITE);
    }

    @Override
    public List<AutoCloseable> getNativeResources() {
        return List.of(inputChannel, outputChannel);
    }

    @Override
    public String getBeanName() { return beanName; }
    // @PreDestroy if needed, but NativeResourceTrackerBPP handles it first
}
```

---

### TODO 7: InstantiationAwareBPP — hooks before the constructor

```java
// InstantiationAwareBeanPostProcessor adds three more hooks:
//   postProcessBeforeInstantiation: BEFORE constructor — can skip the whole bean
//   postProcessAfterInstantiation:  after constructor, before DI — can skip DI
//   postProcessProperties:          DURING DI — can modify/add property values

@Component
public class ExternalConfigBPP implements InstantiationAwareBeanPostProcessor {

    @Autowired
    private ExternalConfigStore externalConfig;

    // postProcessBeforeInstantiation: the EARLIEST interception point.
    // Return non-null → constructor NEVER runs, DI NEVER runs, @PostConstruct NEVER runs.
    // Only postProcessAfterInitialization still fires on your returned object.
    // Use case: return a stub or mock for certain beans (useful in tests).
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        // Only short-circuit if this is a "simulated" service in a test environment:
        if (beanClass.isAnnotationPresent(SimulatedService.class)
                && "test".equals(System.getProperty("app.env"))) {
            System.out.println("Short-circuiting " + beanName + " — using simulation");
            // Returning an instance HERE means:
            // → Constructor of the real class: NEVER called
            // → @Autowired injection: NEVER done
            // → @PostConstruct: NEVER called
            // → postProcessAfterInitialization: IS called on this returned object
            return SimulationFactory.create(beanClass);
        }
        return null;  // null = proceed with normal instantiation
    }

    // postProcessAfterInstantiation: after constructor, before ANY dependency injection.
    // Return false → Spring skips ALL property injection for this bean.
    // Use case: beans that manage their own configuration from an external source.
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) {
        if (bean instanceof SelfConfiguringBean selfConf) {
            // This bean loads its own config — Spring must NOT inject properties
            // (doing so would overwrite the externally loaded values)
            selfConf.loadFromExternalConfig(externalConfig);
            return false;  // false = SKIP ALL DI for this bean
        }
        return true;  // true = proceed with normal DI for all other beans
    }

    // postProcessProperties: called DURING DI, after postProcessAfterInstantiation returned true.
    // pvs = the PropertyValues Spring is about to apply to this bean.
    // You can add/modify/remove values here.
    // Note: @Autowired injection itself is done in AutowiredAnnotationBPP.postProcessProperties()
    @Override
    public PropertyValues postProcessProperties(
            PropertyValues pvs, Object bean, String beanName) {
        // Example: inject a dynamic property based on external config
        if (bean instanceof DynamicConfigBean) {
            MutablePropertyValues mpvs = new MutablePropertyValues(pvs);
            mpvs.add("apiEndpoint", externalConfig.get("api.endpoint"));
            return mpvs;
        }
        return pvs;  // return unchanged pvs for all other beans
    }
}
```

**The short-circuit trap:**

```java
// TRAP: returning non-null from postProcessBeforeInstantiation and expecting
// @PostConstruct to run on your returned object.

@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    if (beanClass == SlowService.class) {
        SlowServiceStub stub = new SlowServiceStub();
        // Does stub's @PostConstruct run? → NO
        // Does stub get @Autowired injection? → NO
        // Does stub.postProcessAfterInitialization() run? → YES
        return stub;
    }
    return null;
}

// If you need @PostConstruct-like initialization on the stub:
// Put it in the stub's constructor, or call it explicitly here, or
// use postProcessAfterInitialization to detect and initialize it.
```

---

### TODO 8: The ordering rules — making all BPPs work together

```java
// The ordering decision — when does YOUR BPP need to run?

// Case A: You need to run BEFORE @PostConstruct (e.g., inject locale before bean validates itself)
// → Use PriorityOrdered with a LOW order value
// → Because CommonAnnotationBPP (which calls @PostConstruct) is PriorityOrdered
// → To beat CommonAnnotationBPP, be PriorityOrdered with getOrder() < its value
@Component
public class LocaleInjectorBPP implements BeanPostProcessor, PriorityOrdered {
    @Override
    public int getOrder() { return Ordered.HIGHEST_PRECEDENCE + 50; }
    // Runs before CommonAnnotationBPP → setLocale() called before @PostConstruct
}

// Case B: You need to run AFTER AOP proxies are created (e.g., validate the proxy too)
// → Use Ordered with a high value, or non-ordered
// → AbstractAutoProxyCreator is Ordered — you run after it if you're non-ordered
@Component
public class PostProxyValidatorBPP implements BeanPostProcessor {
    // non-ordered = runs after all Ordered BPPs (including AOP proxy creator)
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // bean here might already be a proxy — AOP ran before us
        // AopUtils.getTargetClass(bean) to get the real class
        return bean;
    }
}

// Case C: You need to run BEFORE AOP so your modification gets proxied
// → Use PriorityOrdered or Ordered with value lower than AbstractAutoProxyCreator
// → Less common — usually you want to wrap AFTER AOP, not before
@Component
public class PreAopModifierBPP implements BeanPostProcessor, PriorityOrdered {
    @Override
    public int getOrder() { return Ordered.LOWEST_PRECEDENCE - 100; }
    // Still PriorityOrdered → runs before Ordered BPPs including AOP
}
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: Which callback to use — Before vs After initialization**

```
postProcessBeforeInitialization():
  ✓ Bean has all @Autowired fields set
  ✓ Called before @PostConstruct and afterPropertiesSet()
  ✓ Good for: setting callbacks (setLocale, setContext), logging pre-init state
  ✗ @PostConstruct may have already run if a PriorityOrdered BPP calls it before you
  Use for: Aware-style injection (setX() callbacks)

postProcessAfterInitialization():
  ✓ Bean is FULLY initialized — all callbacks have run
  ✓ Best time to REPLACE the bean with a proxy/decorator
  ✓ AOP proxies are created here — whatever you return becomes "the bean"
  Use for: wrapping, validation, registration, AOP
```

**Decision 2: When to implement the sub-interfaces**

```
Plain BPP → 99% of use cases
  → Inspect/validate/wrap beans after they're ready

InstantiationAwareBPP → rare, advanced
  → Skip instantiation entirely (testing, mocking infrastructure)
  → Skip DI for externally configured beans
  → Modify property values during injection
  → Risk: skipping DI can leave beans in unexpected states

MergedBeanDefinitionPostProcessor → library/framework authors only
  → Scan annotation metadata once, cache for prototype performance
  → If you write a library that adds injection points, use this

DestructionAwareBPP → replace complex @PreDestroy patterns
  → Central resource tracking
  → requiresDestruction() for performance (skip beans without cleanup)

SmartInstantiationAwareBPP → Spring internal use / very advanced
  → Custom constructor selection
  → Circular dependency proxy support
  → Rarely needed in application code
```

**Decision 3: Thread safety**

```
BPPs are SINGLETONS — instantiated once, called for every bean.
In a concurrent application, multiple threads may be creating beans simultaneously.
Any mutable state in a BPP MUST be thread-safe.

BAD — race condition:
private Map<Class<?>, List<Field>> fieldCache = new HashMap<>();  // NOT thread-safe

GOOD:
private Map<Class<?>, List<Field>> fieldCache = new ConcurrentHashMap<>();  // thread-safe
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
@SpringBootTest
class BPPEdgeCaseTest {

    // TRAP 1: Forgetting to return bean from postProcessAfterInitialization
    @Component
    static class BrokenBPP implements BeanPostProcessor {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            if (bean instanceof Configurable c) {
                c.configure();
                // BUG: forgot return statement — returns null implicitly
                // null → treated as "no change" → previous result used
                // Except if this is the FIRST BPP in the chain → previous result = null
                // Then: the bean stored in container IS null
                // Every @Autowired point gets null injected
                // NullPointerException at first method call — impossible to trace
            }
            return bean;  // ← ALWAYS return something non-null (or null explicitly for no-change)
        }
    }

    // TRAP 2: @Transactional on a BPP — silently ignored
    @Component
    static class TransactionalBPP implements BeanPostProcessor {
        @Transactional  // ← this annotation is silently ignored
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            // This method runs WITHOUT any transaction.
            // AOP proxy (which implements @Transactional) is not applied to BPP beans.
            // BPPs are instantiated during registerBeanPostProcessors() — before AOP is configured.
            // Spring logs: "Bean 'transactionalBPP' not eligible for auto-proxying"
            return bean;
        }
    }

    // TRAP 3: Calling ctx.getBean() inside a BPP — forces premature instantiation
    @Component
    static class LazyDependentBPP implements BeanPostProcessor {
        @Autowired
        ApplicationContext ctx;

        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            // DANGER: getBean() forces OtherService to be created NOW
            // OtherService is created before all BPPs are registered
            // OtherService might miss processing from BPPs registered after LazyDependentBPP
            // Spring logs a warning about this
            OtherService other = ctx.getBean(OtherService.class);  // ← DON'T DO THIS
            return bean;
        }
    }

    // TRAP 4: Bean.getClass() instead of AopUtils.getTargetClass()
    @Component
    static class WrongClassBPP implements BeanPostProcessor {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            Class<?> clazz = bean.getClass();  // ← might be OrderService$$CGLIB$$0
            boolean hasAnnotation = clazz.isAnnotationPresent(Service.class);
            // hasAnnotation = FALSE for CGLIB proxies — @Service is on OrderService, not the proxy

            // Fix:
            Class<?> realClass = AopUtils.getTargetClass(bean);  // ← always OrderService
            boolean correct = realClass.isAnnotationPresent(Service.class);
            return bean;
        }
    }

    // TRAP 5: BPP that creates beans during postProcessAfterInitialization
    @Component
    static class BeanCreatingBPP implements BeanPostProcessor {
        @Autowired BeanFactory factory;

        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            // Creating a new bean here via factory.getBean() forces it to be
            // instantiated mid-BPP-chain. That bean goes through the BPP chain too
            // — but only the BPPs registered so far. BPPs registered later skip it.
            // This can cause @Async, @Transactional, etc. to not apply to that bean.
            return bean;
        }
    }

    // TRAP 6: Non-thread-safe BPP state
    @Component
    static class RaceConditionBPP implements BeanPostProcessor {
        private final List<String> processedBeans = new ArrayList<>();  // ← NOT thread-safe!
        // Under concurrent bean creation (e.g., async, lazy beans under concurrent load):
        // ConcurrentModificationException or data loss

        // Fix:
        private final List<String> safe = Collections.synchronizedList(new ArrayList<>());
        // Or use ConcurrentHashMap if you need bean → data mapping
    }

    @Test
    void missingReturnCausesNullBean() {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        ctx.register(BrokenBPP.class, SomeConfigurableBean.class);
        assertThatThrownBy(ctx::refresh)
            .isInstanceOf(Exception.class);  // NullPointerException or BeanCreationException
        ctx.close();
    }
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version problem:** Our `RequiredFieldValidatorBPP`, `TimingDecoratorBPP`, and `LocaleAwareBPP` all independently scan classes with reflection, each maintaining their own `ConcurrentHashMap` cache. For an app with 200 beans and 3 BPPs, each class is scanned 3 times, each cache stores 200 entries separately. They also have duplicate "skip infrastructure beans" logic.

**Senior dev refactoring:** extract annotation scanning into a shared `BeanMetadataScanner`, make it the single cache, and implement `MergedBeanDefinitionPostProcessor` in the right BPPs to scan at the optimal time:

```java
// Shared metadata cache — one place for all annotation scanning results.
// Populated by MergedBeanDefinitionPostProcessor callbacks (optimal timing).
// Read by BeanPostProcessor callbacks (zero re-scanning).
@Component
public class BeanAnnotationMetadataCache {

    // class → @Required fields (used by RequiredFieldValidatorBPP)
    private final Map<Class<?>, List<Field>> requiredFields = new ConcurrentHashMap<>();

    // class → has @Service (used by TimingDecoratorBPP)
    private final Map<Class<?>, Boolean> serviceAnnotated = new ConcurrentHashMap<>();

    // class → implements LocaleAware (used by LocaleAwareBPP)
    private final Map<Class<?>, Boolean> localeAware = new ConcurrentHashMap<>();

    // Called once per class (by MBDPP) — caches everything for subsequent reads:
    public void scan(Class<?> clazz) {
        List<Field> required = new ArrayList<>();
        ReflectionUtils.doWithFields(clazz, required::add,
            f -> f.isAnnotationPresent(Required.class));
        requiredFields.put(clazz, Collections.unmodifiableList(required));

        serviceAnnotated.put(clazz, clazz.isAnnotationPresent(Service.class));

        localeAware.put(clazz, LocaleAware.class.isAssignableFrom(clazz));
    }

    public List<Field> getRequiredFields(Class<?> clazz) {
        return requiredFields.getOrDefault(clazz, List.of());
    }

    public boolean isServiceAnnotated(Class<?> clazz) {
        return serviceAnnotated.getOrDefault(clazz, false);
    }

    public boolean isLocaleAware(Class<?> clazz) {
        return localeAware.getOrDefault(clazz, false);
    }
}

// One MBDPP that scans everything once — at the optimal moment (after constructor, before DI):
@Component
public class MetadataScannerBPP
        implements MergedBeanDefinitionPostProcessor, BeanPostProcessor {

    @Autowired
    private BeanAnnotationMetadataCache cache;

    // Scans once per class — result is shared by all other BPPs:
    @Override
    public void postProcessMergedBeanDefinition(
            RootBeanDefinition beanDef, Class<?> beanType, String beanName) {
        cache.scan(beanType);
    }

    // All other BPPs now read from cache — zero additional reflection:
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) { return bean; }
}

// Now all other BPPs are clean — just cache reads:
@Component
public class RequiredFieldValidatorBPP implements BeanPostProcessor, Ordered {
    @Autowired private BeanAnnotationMetadataCache cache;

    @Override
    public int getOrder() { return Ordered.LOWEST_PRECEDENCE; }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        List<Field> fields = cache.getRequiredFields(AopUtils.getTargetClass(bean));
        // no scanning here — just a map lookup
        for (Field field : fields) {
            ReflectionUtils.makeAccessible(field);
            try {
                if (field.get(bean) == null) {
                    throw new BeanInitializationException(
                        "Bean '" + beanName + "' field '" + field.getName()
                        + "': " + field.getAnnotation(Required.class).message());
                }
            } catch (IllegalAccessException e) {
                throw new BeanInitializationException("Cannot read field", e);
            }
        }
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
}
```

**What improved:**

- Each class scanned EXACTLY ONCE regardless of how many BPPs use the result
- Scanning happens at the optimal time (`postProcessMergedBeanDefinition`) — before DI, before L3 cache
- For prototype beans (instantiated N times), each BPP operation is a map lookup — no reflection at all after the first instance
- "Skip infrastructure beans" logic is in one place (`scan()` receives the class, knows to skip non-application classes)
- Adding a new BPP = add a new `getBeans()` method to the cache + read from it — zero change to scanning logic

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **BeanPostProcessor** | A component Spring calls for every bean, at two points during its creation. You inspect the bean, optionally replace it, and return whatever should be stored in the container. |
| **postProcessBeforeInitialization** | Called after `@Autowired` injection but before `@PostConstruct` and `afterPropertiesSet`. The bean is injected but not yet self-validated. Good for setting callbacks on the bean (setLocale, etc.). |
| **postProcessAfterInitialization** | Called after all init callbacks complete. The bean is fully ready. Whatever you return is what the container stores and what other beans receive when they `@Autowire`. AOP proxies are created here. |
| **InstantiationAwareBPP** | Extends BPP with hooks BEFORE and DURING construction. Can skip instantiation entirely (return a stub), skip DI, or modify property values being injected. |
| **postProcessBeforeInstantiation** | Fires before the constructor. Return non-null → entire bean creation (constructor, DI, `@PostConstruct`) is skipped. `postProcessAfterInitialization` still runs on your returned object. |
| **postProcessAfterInstantiation** | Fires after the constructor, before DI. Return false → ALL property injection (both XML and @Autowired) is skipped for this bean. |
| **postProcessProperties** | Fires DURING DI. This is where `@Autowired` injection actually happens (AutowiredAnnotationBPP implements this). You can add/modify property values here. |
| **MergedBeanDefinitionPostProcessor** | Has `postProcessMergedBeanDefinition()` that fires after constructor, before DI. Used to scan and cache annotation metadata — the right time to do reflection that will be needed during DI. |
| **DestructionAwareBPP** | Adds `postProcessBeforeDestruction()` (called before `@PreDestroy`) and `requiresDestruction()` (return false to skip destruction tracking for beans that don't need it). |
| **SmartInstantiationAwareBPP** | Adds constructor selection (`determineCandidateConstructors`), circular dependency support (`getEarlyBeanReference`), and type prediction (`predictBeanType`). Spring internal use. |
| **AopUtils.getTargetClass(bean)** | Always use this instead of `bean.getClass()` in BPPs. If the bean is already an AOP proxy (created by a BPP that ran before yours), this unwraps it to give you the real class with real annotations. |
| **PriorityOrdered** | A tier above `Ordered`. BPPs implementing this are registered and run before ANY `Ordered` BPP, regardless of `getOrder()` values. `AutowiredAnnotationBPP` and `CommonAnnotationBPP` are here. |
| **BPP registration order** | BPPs are stored in an ordered list. For each bean, ALL registered BPPs run in list order. Each BPP receives the result of the previous BPP — they form a pipeline. |

---

### The decision flowchart:

```
I need to hook into bean creation. Which interface?

When does my logic need to run?
  ├── After the bean is FULLY ready (post-@PostConstruct)?
  │     → BeanPostProcessor.postProcessAfterInitialization()
  │       Use for: wrapping/proxying/decorating, validation, registration
  │
  ├── Before @PostConstruct but after @Autowired?
  │     → BeanPostProcessor.postProcessBeforeInitialization()
  │       (but be careful about ordering — PriorityOrdered BPPs run before you)
  │       Use for: Aware-style callbacks (setX() on the bean)
  │
  ├── After constructor but before DI (to scan metadata)?
  │     → MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
  │       Use for: scanning annotations, caching metadata for DI
  │
  ├── Before the constructor entirely?
  │     → InstantiationAwareBPP.postProcessBeforeInstantiation()
  │       Use for: replacing with a stub/mock, short-circuiting
  │
  ├── To skip DI entirely for certain beans?
  │     → InstantiationAwareBPP.postProcessAfterInstantiation() → return false
  │
  └── Before bean destruction?
        → DestructionAwareBPP.postProcessBeforeDestruction()
          (+ requiresDestruction() for optimization)

What does my BPP return?
  → Same bean: I just inspected/called something on it, no replacement
  → Different object: I wrapped it in a proxy/decorator
  → null: treated as "no change" — same as returning the input bean
  → NEVER forget to return something from postProcessAfterInitialization

When does my BPP need to run relative to others?
  → Must run before @PostConstruct: PriorityOrdered with low getOrder() value
  → Don't care / run after AOP: no ordering (non-ordered)
  → Must run before AOP proxy creation: PriorityOrdered or Ordered with low value
  → Must run after AOP: non-ordered (AOP creator is Ordered, so non-ordered comes after)
```

### The per-bean lifecycle timeline:

```
IABPP.postProcessBeforeInstantiation()      ← can return stub → skip everything below
    ↓
[Constructor runs — @Autowired constructor injection here if chosen]
    ↓
MBDPP.postProcessMergedBeanDefinition()     ← scan annotation metadata, cache it
    ↓
[addSingletonFactory — L3 circular dependency cache]
    ↓
IABPP.postProcessAfterInstantiation()       ← return false → skip all DI below
    ↓
IABPP.postProcessProperties()              ← @Autowired/@Value/@Resource injected HERE
    ↓
[invokeAwareMethods: BeanNameAware, BeanFactoryAware]
    ↓
BPP.postProcessBeforeInitialization()      ← Aware callbacks, THEN @PostConstruct
  ├── ApplicationContextAwareProcessor     ← setApplicationContext(), setEnvironment()...
  ├── CommonAnnotationBPP (PriorityOrdered)← CALLS @PostConstruct
  └── Your custom BPP (if PriorityOrdered) ← runs in your getOrder() position
    ↓
[invokeInitMethods: afterPropertiesSet() + init-method]
    ↓
BPP.postProcessAfterInitialization()       ← AOP PROXY CREATED HERE
  ├── AbstractAutoProxyCreator             ← wraps in JDK/CGLIB proxy for @Transactional etc.
  ├── AsyncAnnotationBPP                   ← wraps in async proxy for @Async
  └── Your custom BPP                      ← you see the (possibly already proxied) bean

Final object stored in container
Other beans that @Autowire this → get whatever was returned from the last BPP
```
