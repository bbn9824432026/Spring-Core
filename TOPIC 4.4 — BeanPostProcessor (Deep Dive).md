# TOPIC 4.4 — BeanPostProcessor (Deep Dive)

---

## ① CONCEPTUAL EXPLANATION

### What is a BeanPostProcessor?

A `BeanPostProcessor` (BPP) is a Spring extension point that intercepts every bean's initialization lifecycle — immediately before and after initialization callbacks run. It is the most powerful hook Spring provides for customizing bean creation.

```java
public interface BeanPostProcessor {

    // Called BEFORE initialization (before @PostConstruct, afterPropertiesSet, init-method)
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;
    }

    // Called AFTER initialization (after @PostConstruct, afterPropertiesSet, init-method)
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;
    }
}
```

**The contract is deceptively simple but architecturally profound:**
- Receives the bean instance and its name
- Can return the SAME bean (most common), a WRAPPER, a PROXY, or a completely DIFFERENT object
- The returned value becomes the bean used going forward
- If both methods apply transformations, they chain: before-result → after-result

---

### 4.4.1 — BPP Architecture and Internal Mechanics

#### How Spring Discovers and Registers BPPs

BPPs are registered in `registerBeanPostProcessors()` — Step 6 of `refresh()`, which runs BEFORE any ordinary singleton beans are created (Step 11). This means BPPs are always ready before they are needed.

The registration algorithm:

```java
// In PostProcessorRegistrationDelegate.registerBeanPostProcessors():

// Step 1: Get all BPP bean names from BeanDefinitionRegistry
String[] postProcessorNames =
    beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

// Step 2: Categorize by ordering interface
List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();

for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        // Instantiate NOW — before other BPPs
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        priorityOrderedPostProcessors.add(pp);
    } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    } else {
        nonOrderedPostProcessorNames.add(ppName);
    }
}

// Step 3: Register in order
// 1. PriorityOrdered BPPs
sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

// 2. Ordered BPPs (instantiated now)
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
for (String ppName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(ppName, BeanPostProcessor.class));
}
sortPostProcessors(orderedPostProcessors, beanFactory);
registerBeanPostProcessors(beanFactory, orderedPostProcessors);

// 3. Non-ordered BPPs
// 4. MergedBeanDefinitionPostProcessors (always last)
```

#### BPP Storage in BeanFactory

BPPs are stored in an ordered list:

```java
// In AbstractBeanFactory:
private final List<BeanPostProcessor> beanPostProcessors =
    new BeanPostProcessorCache(); // thread-safe implementation

// BeanPostProcessorCache categorizes BPPs by sub-interface:
class BeanPostProcessorCache {
    final List<InstantiationAwareBeanPostProcessor> instantiationAware = ...;
    final List<SmartInstantiationAwareBeanPostProcessor> smartInstantiationAware = ...;
    final List<DestructionAwareBeanPostProcessor> destructionAware = ...;
    final List<MergedBeanDefinitionPostProcessor> mergedDefinition = ...;
}
```

#### BPP Execution Loop

For every bean initialization, Spring iterates the BPP list:

```java
// In AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization():
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) {
    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result; // null means "use previous result"
        }
        result = current; // chain: each BPP receives result of previous BPP
    }
    return result;
}
```

**Chaining behavior:**
BPPs form a pipeline. The output of BPP-1 becomes the input of BPP-2. If BPP-2 returns a proxy that wraps BPP-1's output, both transformations are applied.

---

### 4.4.2 — BPP Sub-interfaces — The Complete Hierarchy

```
BeanPostProcessor
├── InstantiationAwareBeanPostProcessor (IABPP)
│       ├── postProcessBeforeInstantiation()   ← before constructor!
│       ├── postProcessAfterInstantiation()    ← after constructor, before DI
│       └── postProcessProperties()           ← DI hook (replaces postProcessPropertyValues)
│
├── SmartInstantiationAwareBeanPostProcessor (SIABPP extends IABPP)
│       ├── predictBeanType()                 ← type prediction before creation
│       ├── determineCandidateConstructors()  ← constructor selection
│       └── getEarlyBeanReference()           ← circular dependency early reference
│
├── MergedBeanDefinitionPostProcessor (MBDPP)
│       └── postProcessMergedBeanDefinition() ← scans metadata after creation, before DI
│
└── DestructionAwareBeanPostProcessor (DABPP)
        └── postProcessBeforeDestruction()   ← before destroy callbacks
        └── requiresDestruction()           ← whether bean needs destroy processing
```

---

### 4.4.3 — InstantiationAwareBeanPostProcessor — Deep Dive

`IABPP` extends `BPP` with hooks that fire around INSTANTIATION (not initialization):

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    // Called BEFORE constructor — can return a replacement object to skip normal instantiation
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
            throws BeansException {
        return null; // null = proceed with normal instantiation
    }

    // Called AFTER constructor, BEFORE DI
    // Return false to SKIP all property injection
    default boolean postProcessAfterInstantiation(Object bean, String beanName)
            throws BeansException {
        return true; // true = proceed with property injection
    }

    // Called DURING DI phase — can modify/replace PropertyValues
    @Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {
        return pvs; // null = use original pvs
    }
}
```

#### `postProcessBeforeInstantiation()` — The Short-Circuit Hook

This is the EARLIEST Spring can intercept a bean. It runs BEFORE the constructor:

```java
// In AbstractAutowireCapableBeanFactory.createBean():
// Try to return a proxy instead of the bean itself
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
if (bean != null) {
    // SHORT-CIRCUIT: return the proxy directly
    // Normal instantiation (constructor, DI, init callbacks) NEVER runs
    return bean;
}
// ... normal creation flow
```

If any IABPP returns non-null from `postProcessBeforeInstantiation()`:
1. Normal instantiation is SKIPPED (constructor never called)
2. `postProcessAfterInitialization()` IS still called on the returned object
3. This is how Spring's AOP infrastructure can return fully-configured proxies without bean creation

```java
@Component
public class CustomInstantiationBPP implements InstantiationAwareBeanPostProcessor {

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        if (beanClass == SlowService.class) {
            // Return mock/stub — constructor never runs
            System.out.println("Short-circuiting SlowService instantiation");
            return new SlowServiceStub();
        }
        return null; // null = normal instantiation
    }
}
```

**Primary usage:** `AbstractAutoProxyCreator.postProcessBeforeInstantiation()` — checks if an infrastructure bean (like an AOP advisor target) needs special handling.

#### `postProcessAfterInstantiation()` — Skip DI Hook

Called after the constructor, before any property injection. Returning `false` skips ALL property injection for this bean:

```java
@Component
public class LegacySupportBPP implements InstantiationAwareBeanPostProcessor {

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) {
        if (bean instanceof LegacyBean) {
            // Legacy beans handle their own initialization
            // Skip Spring's property injection entirely
            ((LegacyBean) bean).initializeFromLegacySystem();
            return false; // false = skip DI
        }
        return true; // true = normal DI
    }
}
```

**Real Spring usage:** Spring Data's `@NoRepositoryBean` uses this to prevent Spring from injecting properties into repository marker beans.

#### `postProcessProperties()` — DI Hook

Called DURING property injection, after `postProcessAfterInstantiation()` returns `true`. Can inspect and modify `PropertyValues` before they are applied:

```java
// This is where @Autowired injection happens!
// AutowiredAnnotationBeanPostProcessor.postProcessProperties():
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

---

### 4.4.4 — SmartInstantiationAwareBeanPostProcessor — Deep Dive

`SIABPP` extends `IABPP` with three additional methods:

#### `predictBeanType()` — Type Prediction

Called when Spring needs to know a bean's type BEFORE creating it (for type-based dependency resolution):

```java
default Class<?> predictBeanType(Class<?> beanClass, String beanName) {
    return null; // null = use beanClass as-is
}
```

**Real usage:** `AbstractAutoProxyCreator.predictBeanType()` — if a bean will be proxied, the predicted type may be the proxy type or interface type. This affects `getBeanNamesForType()` results.

#### `determineCandidateConstructors()` — Constructor Selection

Called to determine which constructors to consider for autowiring:

```java
default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) {
    return null; // null = no specific candidates (use default strategy)
}
```

**Real usage:** `AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors()`:
- Scans `@Autowired` on constructors
- For single non-default constructor (Spring 4.3+), returns it as the candidate
- For `@Autowired(required=false)` on multiple constructors, returns all as candidates

```java
// Inside AutowiredAnnotationBPP:
@Override
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) {
    // 1. Scan for @Autowired constructors
    // 2. If single non-default constructor → return it
    // 3. If multiple @Autowired(required=false) → return all
    // 4. Otherwise → return null (use default no-arg constructor strategy)
}
```

#### `getEarlyBeanReference()` — Circular Dependency Support

This is the third level cache's ObjectFactory mechanism:

```java
default Object getEarlyBeanReference(Object bean, String beanName) {
    return bean; // default: return as-is
}
```

**Real usage:** `AbstractAutoProxyCreator.getEarlyBeanReference()`:

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    // Wrap in proxy if AOP applies — ensures circular dependency partner
    // gets the PROXY, not the raw bean
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

This ensures that even when a bean is injected as an early reference (before its initialization completes), the injected object is the AOP proxy — not the raw bean.

---

### 4.4.5 — MergedBeanDefinitionPostProcessor — Deep Dive

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

    // Called after bean instantiation, before populateBean (DI)
    // Used to scan bean metadata and cache injection points
    void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition,
                                          Class<?> beanType, String beanName);

    // Called when bean definition is reset/modified
    default void resetBeanDefinition(String beanName) {}
}
```

**When it runs:**

```java
// In AbstractAutowireCapableBeanFactory.doCreateBean():
// After createBeanInstance(), before addSingletonFactory(), before populateBean():
applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
```

**Real usage:**
- `AutowiredAnnotationBeanPostProcessor` — scans `@Autowired`, `@Value`, `@Inject` injection points on the bean class, caches result in `InjectionMetadata`
- `CommonAnnotationBeanPostProcessor` — scans `@PostConstruct`, `@PreDestroy`, `@Resource` on the bean class
- Scanning happens ONCE per class (cached) — not repeated for every prototype instance

```java
// Inside AutowiredAnnotationBPP:
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition,
                                              Class<?> beanType, String beanName) {
    // Scan the class hierarchy for @Autowired annotations
    // Cache result in InjectionMetadata — avoids repeated reflection
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}
```

**Why caching matters:**
For prototype beans, the SAME class may be instantiated thousands of times. Scanning the class hierarchy via reflection on every instantiation would be expensive. `MergedBeanDefinitionPostProcessor` caches the injection point metadata so subsequent instantiations just use the cache.

---

### 4.4.6 — DestructionAwareBeanPostProcessor — Deep Dive

```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {

    // Called before bean destruction (before @PreDestroy, DisposableBean.destroy(), etc.)
    void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;

    // Can opt-out of destruction processing for specific beans (performance optimization)
    default boolean requiresDestruction(Object bean) {
        return true; // process all beans by default
    }
}
```

**Real usage:**
`CommonAnnotationBeanPostProcessor` extends `DestructionAwareBeanPostProcessor`:

```java
// Calls @PreDestroy methods
@Override
public void postProcessBeforeDestruction(Object bean, String beanName) {
    LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
    try {
        metadata.invokeDestroyMethods(bean, beanName);
    } catch (InvocationTargetException ex) {
        // handle
    }
}

@Override
public boolean requiresDestruction(Object bean) {
    // Only true if the bean has @PreDestroy methods
    return findLifecycleMetadata(bean.getClass()).hasDestroyMethods();
}
```

The `requiresDestruction()` method is an optimization — beans with no destroy methods skip the destruction processing entirely.

---

### 4.4.7 — Key Built-in BPPs and Their Responsibilities

```
┌──────────────────────────────────────────────┬───────────────────────────────────────┐
│ BPP Class                                    │ Responsibility                        │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ AutowiredAnnotationBeanPostProcessor         │ @Autowired, @Value, @Inject           │
│   implements: SIABPP, MBDPP                  │ determineCandidateConstructors()      │
│   priority: PriorityOrdered (0)              │ postProcessProperties()               │
│                                              │ postProcessMergedBeanDefinition()     │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ CommonAnnotationBeanPostProcessor            │ @Resource, @PostConstruct, @PreDestroy│
│   implements: IABPP, MBDPP, DABPP           │ JSR-250 support                       │
│   priority: PriorityOrdered (Ordered.LOWEST) │ postProcessBeforeInitialization():    │
│                                              │   → calls @PostConstruct              │
│                                              │ postProcessBeforeDestruction():       │
│                                              │   → calls @PreDestroy                 │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ ApplicationContextAwareProcessor             │ EnvironmentAware, ResourceLoaderAware │
│   implements: BPP                            │ ApplicationContextAware, etc.         │
│   registered: in prepareBeanFactory()        │ postProcessBeforeInitialization()     │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ AbstractAutoProxyCreator                     │ AOP proxy creation                    │
│   implements: SIABPP, MBDPP                  │ postProcessAfterInitialization():     │
│     ↑ subclasses:                            │   wraps beans in JDK/CGLIB proxy      │
│   AnnotationAwareAspectJAutoProxyCreator     │ getEarlyBeanReference():              │
│   InfrastructureAdvisorAutoProxyCreator      │   early circular dep proxy            │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ AsyncAnnotationBeanPostProcessor             │ @Async method wrapping                │
│   implements: BPP                            │ postProcessAfterInitialization():     │
│                                              │   → wraps in async proxy              │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ ScheduledAnnotationBeanPostProcessor         │ @Scheduled method registration        │
│   implements: MBDPP, DABPP                   │ postProcessMergedBeanDefinition():    │
│                                              │   → scans @Scheduled methods          │
│                                              │ postProcessBeforeDestruction():       │
│                                              │   → unregisters scheduled tasks       │
├──────────────────────────────────────────────┼───────────────────────────────────────┤
│ PersistenceAnnotationBeanPostProcessor       │ @PersistenceContext, @PersistenceUnit │
│   implements: IABPP, MBDPP, DABPP           │ JPA EntityManager injection           │
└──────────────────────────────────────────────┴───────────────────────────────────────┘
```

---

### 4.4.8 — BPP Registration vs Bean Registration

**Critical timing distinction:**

BPPs that are defined as `@Component` or `@Bean` go through a DIFFERENT registration path than ordinary beans:

```
refresh() step 6: registerBeanPostProcessors()
    → getBeanNamesForType(BeanPostProcessor.class)
    → instantiate BPP beans via getBean()
    → add to beanPostProcessors list

refresh() step 11: finishBeanFactoryInitialization()
    → preInstantiateSingletons()
    → instantiate ALL remaining ordinary beans
    → these beans ARE processed by the BPPs registered in step 6
```

**Consequence:** BPPs are instantiated in Step 6 BEFORE ordinary beans. When a BPP bean is instantiated in Step 6, the BPPs registered SO FAR process it. BPPs registered LATER cannot process earlier BPPs.

**The "BPP registration problem":**

```java
@Configuration
public class Config {

    // BPP bean
    @Bean
    public MyBPP myBPP() { return new MyBPP(); }

    // Ordinary bean with @Autowired
    @Bean
    public ServiceA serviceA() { return new ServiceA(); }
}

// MyBPP is instantiated in step 6 (registerBeanPostProcessors)
// At step 6, AutowiredAnnotationBPP IS already registered (PriorityOrdered)
// So @Autowired on MyBPP fields DOES work
// BUT: @Transactional on MyBPP will NOT work — AnnotationAwareAspectJAutoProxyCreator
// may not yet have all advisors ready to create a proxy for MyBPP
```

**Warning message:**
Spring logs a warning when a bean that creates BPPs (`@Configuration` class with `@Bean` BPP methods) depends on other beans that should have been processed by BPPs:

```
Bean 'someBean' of type [...] is not eligible for getting processed by all BeanPostProcessors
(for example: not eligible for auto-proxying)
```

This warning means a bean was instantiated BEFORE all BPPs were registered — it won't receive processing from the BPPs registered after it.

---

### 4.4.9 — Writing a Production-Quality BPP

Implementing a BPP requires careful consideration of:

1. **Performance:** BPPs run for EVERY bean — keep checks fast
2. **Type checking first:** Check type before doing expensive reflection
3. **Return contract:** Always return the bean (or replacement), never null from `postProcessAfterInitialization`
4. **Thread safety:** BPP instances are singletons — must be thread-safe
5. **Idempotency:** Do not double-process beans

```java
@Component
public class ValidationAnnotationBPP
        implements BeanPostProcessor, PriorityOrdered {

    // Cache: class → list of @Validate-annotated methods
    private final Map<Class<?>, List<Method>> methodCache = new ConcurrentHashMap<>();

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 100; // after AABPP but before most custom BPPs
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> beanClass = AopUtils.getTargetClass(bean); // unwrap proxy if present

        // Fast path: skip beans without @Validate methods (cached)
        List<Method> validateMethods = methodCache.computeIfAbsent(beanClass, clazz -> {
            List<Method> methods = new ArrayList<>();
            ReflectionUtils.doWithMethods(clazz, methods::add,
                method -> method.isAnnotationPresent(Validate.class));
            return methods.isEmpty() ? Collections.emptyList() : methods;
        });

        if (validateMethods.isEmpty()) {
            return bean; // fast return — no processing needed
        }

        // Validate bean state via annotated methods
        for (Method method : validateMethods) {
            if (method.getParameterCount() == 0) {
                try {
                    ReflectionUtils.makeAccessible(method);
                    method.invoke(bean);
                } catch (InvocationTargetException e) {
                    throw new BeanInitializationException(
                        "Validation failed for bean '" + beanName + "'", e.getCause());
                } catch (IllegalAccessException e) {
                    throw new BeanInitializationException(
                        "Cannot invoke @Validate method on '" + beanName + "'", e);
                }
            }
        }
        return bean; // return same bean — no replacement needed
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // no-op for before-init
    }
}
```

---

### 4.4.10 — BPP Execution Order — Complete Picture

The EXACT sequence of BPP calls during a single bean's creation:

```
Bean creation (doCreateBean):
│
├── createBeanInstance()                     (Step 3: constructor)
│
├── IABPP.postProcessBeforeInstantiation()   (Step 3a: BEFORE constructor — can short-circuit)
│
├── [If not short-circuited]:
│   ├── constructor runs
│   ├── MBDPP.postProcessMergedBeanDefinition()  (Step 3b: scan injection metadata)
│   ├── addSingletonFactory()                    (Step 3c: L3 cache)
│   │
│   ├── IABPP.postProcessAfterInstantiation()    (Step 4a: can skip DI)
│   │
│   ├── [If DI not skipped]:
│   │   ├── XML PropertyValues applied
│   │   └── IABPP.postProcessProperties()       (Step 4b: @Autowired injection here)
│   │
│   ├── invokeAwareMethods()                     (Step 5: BeanName/Factory aware)
│   │
│   ├── BPP.postProcessBeforeInitialization()    (Step 6: ALL BPPs)
│   │   ├── ApplicationContextAwareProcessor    (→ ctx-aware callbacks)
│   │   ├── CommonAnnotationBPP                 (→ @PostConstruct)
│   │   └── [custom BPPs]
│   │
│   ├── invokeInitMethods()                      (Step 7: afterPropertiesSet + init-method)
│   │
│   └── BPP.postProcessAfterInitialization()    (Step 8: ALL BPPs)
│       ├── AbstractAutoProxyCreator            (→ AOP proxy creation)
│       ├── AsyncAnnotationBPP                  (→ @Async proxy)
│       └── [custom BPPs]
│
└── [If short-circuited]:
    └── BPP.postProcessAfterInitialization()    (Step 8: still runs)
```

---

### 4.4.11 — BPP Ordering — PriorityOrdered vs Ordered

For BPPs, ordering determines which BPP processes beans first:

```
Registration order:
1. PriorityOrdered BPPs (processed first batch)
   → AutowiredAnnotationBPP        (order: Ordered.LOWEST_PRECEDENCE - 2)
   → CommonAnnotationBPP           (order: Ordered.LOWEST_PRECEDENCE - 3)
   → RequiredAnnotationBPP         (deprecated)

2. Ordered BPPs (processed second batch)
   → (custom BPPs implementing Ordered)

3. Non-ordered BPPs (processed third batch)
   → (custom BPPs with no ordering)

4. MergedBeanDefinitionPostProcessors (always last)
   → AbstractAutoProxyCreator (implements SIABPP which extends IABPP)
```

**Important nuance:** `AbstractAutoProxyCreator` is registered LATER in the list (as a non-PriorityOrdered by default for most configurations). This means AOP proxy creation happens AFTER `@Autowired` injection, `@PostConstruct`, etc.

**But `AbstractAutoProxyCreator` is `SIABPP` — so for `postProcessBeforeInstantiation()`** it runs in the IABPP slot (earlier), while for `postProcessAfterInitialization()` it runs in its registered position (later).

---

### Common Misconceptions

**Misconception 1:** "BPPs run after all beans are created."
Wrong. BPPs run for EACH bean as it is being created. BPPs are not batch post-processors — they are per-bean hooks.

**Misconception 2:** "`postProcessBeforeInitialization` returning `null` means destroy the bean."
Wrong. Returning `null` from `postProcessBeforeInitialization()` means "use the previous result" (pass-through, no change). Returning `null` from `postProcessAfterInitialization()` also means "use the previous result." This is different from returning `null` to indicate "no change" — it's the same thing.

**Misconception 3:** "BPPs process themselves."
Partially wrong. A BPP IS processed by BPPs registered BEFORE it. But it is NOT processed by itself or BPPs registered AFTER it. This means a BPP cannot apply AOP to itself if the AOP BPP is registered after it.

**Misconception 4:** "`postProcessBeforeInstantiation` returning non-null replaces the bean permanently."
Nuanced. If IABPP returns non-null from `postProcessBeforeInstantiation()`, normal instantiation is skipped. But `postProcessAfterInitialization()` IS still called on the returned object. The returned object can be transformed further.

**Misconception 5:** "MBDPP.postProcessMergedBeanDefinition is called before the constructor."
Wrong. `postProcessMergedBeanDefinition()` is called AFTER the constructor runs — it receives the newly created instance. It runs between `createBeanInstance()` and `addSingletonFactory()`.

**Misconception 6:** "Custom BPPs always run after built-in BPPs."
Wrong. BPP ordering is controlled by `PriorityOrdered`, `Ordered`, and registration order. A custom BPP implementing `PriorityOrdered` with `getOrder() = Integer.MIN_VALUE` runs BEFORE any `Ordered` built-in BPP.

---

## ② CODE EXAMPLES

### Example 1 — Simplest Custom BPP — Logging Lifecycle

```java
@Component
public class LifecycleLoggingBPP implements BeanPostProcessor, Ordered {

    private static final Logger log = LoggerFactory.getLogger(LifecycleLoggingBPP.class);

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE; // run last — see final state
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (isApplicationBean(beanName)) {
            log.debug("BEFORE init: {}", beanName);
        }
        return bean; // return same bean — no modification
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (isApplicationBean(beanName)) {
            log.debug("AFTER init: {} → class: {}", beanName,
                AopUtils.getTargetClass(bean).getSimpleName());
            // After init, AOP proxy may have been applied by earlier BPPs
            // AopUtils.getTargetClass() unwraps the proxy to get the real class
        }
        return bean;
    }

    private boolean isApplicationBean(String beanName) {
        // Skip Spring infrastructure beans
        return !beanName.startsWith("org.springframework");
    }
}
```

---

### Example 2 — BPP Replacing Beans with Decorators

```java
@Component
public class MetricsDecoratorBPP implements BeanPostProcessor {

    @Autowired
    private MeterRegistry meterRegistry;

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {

        // If bean implements Service and has @Timed annotation — wrap it
        if (bean instanceof Service) {
            Class<?> targetClass = AopUtils.getTargetClass(bean);
            if (targetClass.isAnnotationPresent(Timed.class)) {
                Timed timed = targetClass.getAnnotation(Timed.class);
                // Return a DIFFERENT object — the decorator
                return new TimedServiceDecorator<>((Service) bean,
                    meterRegistry, timed.value());
                // The original bean is now wrapped
                // ctx.getBean(Service.class) returns TimedServiceDecorator
            }
        }
        return bean; // no replacement for other beans
    }
}
```

---

### Example 3 — InstantiationAwareBPP Preventing DI

```java
@Component
public class ExternalConfigBPP implements InstantiationAwareBeanPostProcessor {

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) {
        if (bean instanceof ExternallyConfigured) {
            // This bean loads its own config from an external system
            // We must not let Spring inject properties — it would override external config
            ((ExternallyConfigured) bean).loadFromExternalSystem();
            return false; // false = skip ALL Spring property injection for this bean
        }
        return true; // true = normal DI for all other beans
    }
}

public interface ExternallyConfigured {
    void loadFromExternalSystem();
}

@Component
public class LegacyConfigBean implements ExternallyConfigured {
    private String host;
    private int port;

    @Override
    public void loadFromExternalSystem() {
        // Load from legacy config store — Spring @Value/@Autowired not used
        Map<String, String> config = LegacyConfigStore.load("legacyConfigBean");
        this.host = config.get("host");
        this.port = Integer.parseInt(config.get("port"));
    }
}
```

---

### Example 4 — MBDPP Scanning Custom Annotations

```java
// Custom annotation for validation
@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NotEmpty {
    String message() default "Field must not be empty";
}

// Scan and cache @NotEmpty injection points
@Component
public class NotEmptyValidationBPP
        implements MergedBeanDefinitionPostProcessor, BeanPostProcessor {

    // Cache: class → fields/methods with @NotEmpty
    private final Map<Class<?>, List<Field>> fieldCache = new ConcurrentHashMap<>();

    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition,
                                                 Class<?> beanType, String beanName) {
        // Scan class ONCE — cache for subsequent prototype instantiations
        fieldCache.computeIfAbsent(beanType, clazz -> {
            List<Field> fields = new ArrayList<>();
            ReflectionUtils.doWithFields(clazz, fields::add,
                field -> field.isAnnotationPresent(NotEmpty.class));
            return fields;
        });
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> beanClass = AopUtils.getTargetClass(bean);
        List<Field> fields = fieldCache.getOrDefault(beanClass, List.of());

        for (Field field : fields) {
            ReflectionUtils.makeAccessible(field);
            Object value;
            try {
                value = field.get(bean);
            } catch (IllegalAccessException e) {
                throw new BeanInitializationException("Cannot validate field", e);
            }

            if (value == null || (value instanceof String && ((String) value).isEmpty())) {
                NotEmpty annotation = field.getAnnotation(NotEmpty.class);
                throw new BeanInitializationException(
                    "Bean '" + beanName + "', field '" + field.getName() +
                    "': " + annotation.message());
            }
        }
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // no-op
    }
}
```

---

### Example 5 — DestructionAwareBPP Custom Cleanup

```java
@Component
public class ResourceCleanupBPP implements DestructionAwareBeanPostProcessor {

    private final Map<String, List<AutoCloseable>> beanResources = new ConcurrentHashMap<>();

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof ResourceHolder) {
            // Track resources held by this bean for cleanup
            List<AutoCloseable> resources = ((ResourceHolder) bean).getResources();
            if (!resources.isEmpty()) {
                beanResources.put(beanName, resources);
            }
        }
        return bean;
    }

    @Override
    public void postProcessBeforeDestruction(Object bean, String beanName) {
        List<AutoCloseable> resources = beanResources.remove(beanName);
        if (resources != null) {
            for (AutoCloseable resource : resources) {
                try {
                    resource.close();
                    System.out.println("Closed resource for bean: " + beanName);
                } catch (Exception e) {
                    System.err.println("Error closing resource for " + beanName + ": " + e.getMessage());
                }
            }
        }
    }

    @Override
    public boolean requiresDestruction(Object bean) {
        // Only process beans that implement ResourceHolder and have resources
        return bean instanceof ResourceHolder &&
               !((ResourceHolder) bean).getResources().isEmpty();
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

---

### Example 6 — SmartInstantiationAwareBPP for Constructor Selection

```java
@Component
public class AnnotatedConstructorBPP
        implements SmartInstantiationAwareBeanPostProcessor {

    @Override
    public Constructor<?>[] determineCandidateConstructors(
            Class<?> beanClass, String beanName) throws BeansException {

        // Find constructors with @Preferred custom annotation
        List<Constructor<?>> preferred = Arrays.stream(beanClass.getDeclaredConstructors())
            .filter(c -> c.isAnnotationPresent(Preferred.class))
            .collect(Collectors.toList());

        if (!preferred.isEmpty()) {
            return preferred.toArray(new Constructor[0]);
        }
        return null; // null = use default Spring strategy
    }

    @Override
    public Object getEarlyBeanReference(Object bean, String beanName) {
        // For circular dependency resolution — return as-is (no wrapping needed)
        return bean;
    }
}

// Usage:
public class MultiConstructorService {

    @Preferred  // This constructor is selected by our custom BPP
    public MultiConstructorService(DataSource ds, CacheManager cache) {
        System.out.println("Preferred constructor used");
    }

    public MultiConstructorService(DataSource ds) {
        System.out.println("Fallback constructor used");
    }
}
```

---

### Example 7 — BPP Order Demonstration

```java
@Component
public class FirstBPP implements BeanPostProcessor, PriorityOrdered {
    @Override public int getOrder() { return 0; }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderTracker)
            ((OrderTracker) bean).addStep("FirstBPP");
        return bean;
    }
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
}

@Component
public class SecondBPP implements BeanPostProcessor, Ordered {
    @Override public int getOrder() { return 100; }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderTracker)
            ((OrderTracker) bean).addStep("SecondBPP");
        return bean;
    }
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
}

@Component
public class ThirdBPP implements BeanPostProcessor { // no ordering

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderTracker)
            ((OrderTracker) bean).addStep("ThirdBPP");
        return bean;
    }
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
}

@Component
public class OrderTracker {
    private final List<String> steps = new ArrayList<>();
    public void addStep(String step) { steps.add(step); }
    public List<String> getSteps() { return steps; }

    @PostConstruct
    public void init() { addStep("@PostConstruct"); }
}

// Result:
// orderTracker.getSteps() = ["@PostConstruct", "FirstBPP", "SecondBPP", "ThirdBPP"]
// PriorityOrdered(0) → Ordered(100) → non-ordered → registration order within each tier
```

---

### Example 8 — BPP Warning: Early Instantiation Issue

```java
@Configuration
public class ProblematicConfig {

    // This @Bean depends on another bean with @Value
    @Autowired
    private Environment env; // OK — Environment is a resolvable dependency

    @Bean
    public static MyBeanPostProcessor myBPP() { // STATIC — good for BFPP-like concerns
        return new MyBeanPostProcessor();
    }

    @Bean
    public SomeService someService() {
        return new SomeService();
    }
}

// Problem scenario:
@Component
public class EarlyBean {

    // EarlyBean is instantiated DURING registerBeanPostProcessors (step 6)
    // because some BPP depends on it

    @Transactional // AOP applied by AbstractAutoProxyCreator
    public void doWork() { }

    // IF EarlyBean is instantiated before AbstractAutoProxyCreator is registered,
    // it will NOT get @Transactional proxy applied
    // Spring logs: "Bean 'earlyBean' not eligible for auto-proxying"
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
In what order does Spring call the following for a singleton bean?

A) `MBDPP.postProcessMergedBeanDefinition()` → `IABPP.postProcessAfterInstantiation()` → `BPP.postProcessBeforeInitialization()` → `BPP.postProcessAfterInitialization()`

B) `BPP.postProcessBeforeInitialization()` → `MBDPP.postProcessMergedBeanDefinition()` → `IABPP.postProcessAfterInstantiation()` → `BPP.postProcessAfterInitialization()`

C) `IABPP.postProcessAfterInstantiation()` → `MBDPP.postProcessMergedBeanDefinition()` → `BPP.postProcessBeforeInitialization()` → `BPP.postProcessAfterInitialization()`

D) `MBDPP.postProcessMergedBeanDefinition()` → `BPP.postProcessBeforeInitialization()` → `IABPP.postProcessAfterInstantiation()` → `BPP.postProcessAfterInitialization()`

**Answer: A**
The exact order: `postProcessMergedBeanDefinition()` (after constructor, before addSingletonFactory) → `postProcessAfterInstantiation()` (after constructor, before DI) → `postProcessBeforeInitialization()` (before init callbacks) → `postProcessAfterInitialization()` (after init callbacks).

---

**Q2 — MCQ:**
A `BeanPostProcessor` returns `null` from `postProcessAfterInitialization()`. What happens?

A) The bean is removed from the container
B) The bean is set to null — NullPointerException on next getBean()
C) The previous result is used unchanged — null means "no change"
D) BeanCreationException thrown — null is not a valid return

**Answer: C**
The BPP execution loop checks for null and uses the previous result if null is returned. Returning null is equivalent to returning the original bean — it means "I don't want to modify this bean."

---

**Q3 — Select all that apply:**
Which BPP sub-interface methods are called BEFORE dependency injection? (Select ALL that apply)

A) `BPP.postProcessBeforeInitialization()`
B) `IABPP.postProcessBeforeInstantiation()`
C) `IABPP.postProcessAfterInstantiation()`
D) `MBDPP.postProcessMergedBeanDefinition()`
E) `BPP.postProcessAfterInitialization()`
F) `IABPP.postProcessProperties()`

**Answer: B, C, D**
`postProcessBeforeInstantiation()` — before constructor (before DI). `postProcessAfterInstantiation()` — after constructor, before DI. `postProcessMergedBeanDefinition()` — after constructor, before DI. `postProcessProperties()` (F) IS the DI hook but it runs DURING DI, not before. `postProcessBeforeInitialization()` (A) runs AFTER DI. `postProcessAfterInitialization()` (E) runs after everything.

---

**Q4 — Code Output Prediction:**

```java
@Component
public class PrintingBPP implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if ("targetBean".equals(beanName))
            System.out.println("BEFORE: " + bean.getClass().getSimpleName());
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if ("targetBean".equals(beanName))
            System.out.println("AFTER: " + bean.getClass().getSimpleName());
        return bean;
    }
}

@Component("targetBean")
public class TargetBean {
    @PostConstruct
    void init() { System.out.println("INIT: TargetBean"); }
}
```

What is the output?

A)
```
BEFORE: TargetBean
INIT: TargetBean
AFTER: TargetBean
```
B)
```
INIT: TargetBean
BEFORE: TargetBean
AFTER: TargetBean
```
C)
```
BEFORE: TargetBean
AFTER: TargetBean
INIT: TargetBean
```
D)
```
AFTER: TargetBean
BEFORE: TargetBean
INIT: TargetBean
```

**Answer: A**
`postProcessBeforeInitialization()` runs, then `@PostConstruct` (which is itself called from within `CommonAnnotationBPP.postProcessBeforeInitialization()` — but since `PrintingBPP` is registered with lower priority, it runs before `CommonAnnotationBPP`). Wait — actually, BPP execution order matters here. `CommonAnnotationBPP` is `PriorityOrdered`. If `PrintingBPP` is `Ordered` or non-ordered, it runs AFTER `CommonAnnotationBPP`. So: `CommonAnnotationBPP.before` (calls `@PostConstruct`) → `PrintingBPP.before` → init-method → `BPPs.after`.

Actually: since `CommonAnnotationBPP` has `PriorityOrdered` and `PrintingBPP` has no ordering (non-ordered), `CommonAnnotationBPP` runs FIRST in `postProcessBeforeInitialization`. So `@PostConstruct` runs BEFORE `PrintingBPP.before`. The answer is:

```
INIT: TargetBean     (CommonAnnotationBPP runs first — @PostConstruct fires)
BEFORE: TargetBean   (PrintingBPP runs second)
AFTER: TargetBean
```

Closest to **B** — this is a trap showing BPP ordering matters.

---

**Q5 — Trick Question:**

```java
@Component
public class ShortCircuitBPP implements InstantiationAwareBeanPostProcessor {

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        if ("myService".equals(beanName)) {
            System.out.println("Short-circuiting MyService");
            return new MyServiceStub();
        }
        return null;
    }
}

@Service("myService")
public class MyService {
    @PostConstruct
    public void init() { System.out.println("MyService @PostConstruct"); }
}
```

What is the output?

A)
```
Short-circuiting MyService
MyService @PostConstruct
```
B)
```
Short-circuiting MyService
```
C)
```
MyService @PostConstruct
Short-circuiting MyService
```
D)
```
Short-circuiting MyService
MyServiceStub @PostConstruct (if MyServiceStub has @PostConstruct)
```

**Answer: B**
When `postProcessBeforeInstantiation()` returns non-null, the normal instantiation is SKIPPED — constructor never runs, DI never runs, `@PostConstruct` NEVER runs. Only `postProcessAfterInitialization()` is still called on the stub. Since `MyServiceStub` (presumably) has no `@PostConstruct`, nothing more is printed.

---

**Q6 — Select all that apply:**
Which built-in Spring BPPs implement `PriorityOrdered`? (Select ALL that apply)

A) `AutowiredAnnotationBeanPostProcessor`
B) `AbstractAutoProxyCreator`
C) `CommonAnnotationBeanPostProcessor`
D) `AsyncAnnotationBeanPostProcessor`
E) `ApplicationContextAwareProcessor`
F) `ScheduledAnnotationBeanPostProcessor`

**Answer: A, C**
`AutowiredAnnotationBeanPostProcessor` and `CommonAnnotationBeanPostProcessor` implement `PriorityOrdered`. `AbstractAutoProxyCreator` implements `Ordered` (not PriorityOrdered — by default). `AsyncAnnotationBeanPostProcessor` is `Ordered`. `ApplicationContextAwareProcessor` is not ordered. `ScheduledAnnotationBeanPostProcessor` is not PriorityOrdered.

---

**Q7 — Scenario-based:**
A custom BPP is annotated with `@Transactional` on one of its methods. Will the transaction proxy be applied to the BPP?

A) Yes — all Spring beans get AOP processing applied
B) No — BPPs are instantiated before `AbstractAutoProxyCreator` is fully set up
C) Yes — but only if the BPP is `PriorityOrdered`
D) Yes — `@Transactional` is processed separately from BPP registration

**Answer: B**
BPPs are instantiated in `registerBeanPostProcessors()` (Step 6). At this point, `AbstractAutoProxyCreator` may not be fully configured with all advisors. More critically, when a BPP is instantiated, only the BPPs registered BEFORE it can process it. `AbstractAutoProxyCreator` (AOP proxy creator) is typically registered LATER. Spring logs a warning for this case. AOP does NOT reliably apply to BPP beans.

---

**Q8 — Code Output Prediction:**

```java
@Component
public class CountingBPP implements BeanPostProcessor {
    private int beforeCount = 0;
    private int afterCount = 0;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        beforeCount++;
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        afterCount++;
        return bean;
    }

    @PostConstruct
    public void report() {
        System.out.println("At @PostConstruct: before=" + beforeCount
            + " after=" + afterCount);
    }
}
```

What does `report()` print approximately?

A) `before=0, after=0` — BPP's own `@PostConstruct` runs before it processes other beans
B) `before=N, after=N-1` — BPP processes N beans before init, N-1 after (not itself after)
C) `before=N, after=0` — after-init hasn't run for any beans yet when @PostConstruct fires
D) `before=N+1, after=N` — counts include the BPP itself

**Answer: A is closest in spirit but the real answer is nuanced.**
When the BPP's `@PostConstruct` runs, it has already processed several beans in `postProcessBeforeInitialization` (beans instantiated before it) but NOT in `postProcessAfterInitialization` (those same beans are still mid-lifecycle). The actual counts depend on how many beans were instantiated before `CountingBPP` during the BPP registration phase. For the BPP itself: `before` is called (by other BPPs), then `@PostConstruct` runs, then `after` is called. At `@PostConstruct` time: `afterCount` for the CountingBPP itself hasn't been called yet. This is a deep question about timing.

---

**Q9 — MCQ:**
`IABPP.postProcessAfterInstantiation()` returns `false` for a bean. What is skipped?

A) Only setter injection — constructor injection already completed
B) ALL property injection — both XML `<property>` values AND `@Autowired` fields
C) Only `@Autowired` injection — XML properties are still applied
D) All initialization callbacks including `@PostConstruct`

**Answer: B**
Returning `false` from `postProcessAfterInstantiation()` causes `populateBean()` to SKIP ALL property injection — both XML-defined `<property>` values and annotation-based `@Autowired`/`@Resource` injection. The bean's constructor has already run, but no further injection occurs.

---

**Q10 — Drag-and-Drop: Order these BPP-related events for a SINGLE bean from first to last:**

- `IABPP.postProcessProperties()` (DI happens here)
- `MBDPP.postProcessMergedBeanDefinition()` (metadata scan)
- `BPP.postProcessAfterInitialization()` (AOP proxy created here)
- `IABPP.postProcessBeforeInstantiation()` (before constructor)
- `BPP.postProcessBeforeInitialization()` (includes @PostConstruct)
- `IABPP.postProcessAfterInstantiation()` (can skip DI)
- `Constructor runs`
- `invokeInitMethods()` (afterPropertiesSet + init-method)

**Answer:**
1. `IABPP.postProcessBeforeInstantiation()` — before constructor
2. `Constructor runs`
3. `MBDPP.postProcessMergedBeanDefinition()` — scan metadata
4. `IABPP.postProcessAfterInstantiation()` — can skip DI
5. `IABPP.postProcessProperties()` — DI injection
6. `BPP.postProcessBeforeInitialization()` — includes @PostConstruct
7. `invokeInitMethods()` — afterPropertiesSet + init-method
8. `BPP.postProcessAfterInitialization()` — AOP proxy

---

## ④ TRICK ANALYSIS

**Trap 1: BPP runs once for all beans — batch processing**
Wrong mental model. BPPs run for EACH individual bean as it is created. They are per-bean hooks called in sequence during `initializeBean()`. They are NOT called once after all beans are ready.

**Trap 2: null return from postProcessAfterInitialization destroys the bean**
Wrong. Returning `null` from EITHER `postProcessBeforeInitialization()` or `postProcessAfterInitialization()` means "no change — use previous result." Spring's loop: `if (current == null) return result;` — null is pass-through, not destruction.

**Trap 3: MBDPP runs before the constructor**
Wrong. `postProcessMergedBeanDefinition()` runs AFTER the constructor. It receives the newly created `bean` object. It runs between `createBeanInstance()` and `addSingletonFactory()` — after instantiation, before DI.

**Trap 4: @Transactional works on BPP beans**
Unreliable. BPPs are instantiated in Step 6 (registerBeanPostProcessors). AOP proxy creator (`AbstractAutoProxyCreator`) may not be fully configured when BPPs are instantiated. AOP does NOT reliably apply to BPP beans. Spring logs a warning when this situation is detected.

**Trap 5: postProcessBeforeInstantiation replaces the bean and skips everything**
Partially wrong. If `postProcessBeforeInstantiation()` returns non-null, normal instantiation (constructor, DI, all init callbacks including `@PostConstruct`) IS skipped. BUT `postProcessAfterInitialization()` IS still called on the returned object. This is commonly missed.

**Trap 6: PriorityOrdered BPPs process ALL beans before Ordered BPPs**
Wrong mental model. BPPs are REGISTERED in priority order. But during bean CREATION, the full list of registered BPPs runs for each bean. The ordering controls registration order (which BPP is registered first), which then determines which BPP's callbacks run first for each bean.

**Trap 7: postProcessAfterInstantiation returning false skips @Autowired only**
Wrong. Returning `false` from `postProcessAfterInstantiation()` skips ALL `populateBean()` logic — both XML `<property>` application AND annotation-based injection (`@Autowired`, `@Resource`, `@Value`). It is a complete skip of the DI phase.

**Trap 8: BPPs are singletons and must be thread-safe**
True and often overlooked. BPPs are Spring singletons. Multiple threads creating beans (e.g., during lazy initialization under concurrent request load) may call BPP methods concurrently. The BPP's internal state (like the `fieldCache` in the example) must be thread-safe.

**Trap 9: MergedBeanDefinitionPostProcessor is just another BPP**
MBDPP has different timing — `postProcessMergedBeanDefinition()` runs BEFORE `populateBean()` AND before `addSingletonFactory()` (the L3 cache registration). Regular `BPP.postProcessBeforeInitialization()` runs AFTER DI. Conflating them leads to wrong answers about when metadata scanning happens.

**Trap 10: Custom BPP automatically gets lowest priority**
Wrong. Without implementing `Ordered` or `PriorityOrdered`, a custom BPP gets DEFAULT priority — which is non-ordered. Non-ordered BPPs run in registration order after ALL `PriorityOrdered` and ALL `Ordered` BPPs. If the exam shows a custom BPP with no ordering, it will run AFTER `AutowiredAnnotationBPP` and `CommonAnnotationBPP`.

---

## ⑤ SUMMARY SHEET

### BPP Sub-interface Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│ INTERFACE              │ KEY METHOD                  │ TIMING           │
├────────────────────────┼─────────────────────────────┼──────────────────┤
│ BeanPostProcessor      │ postProcessBeforeInit()     │ After DI         │
│                        │ postProcessAfterInit()      │ After init calls │
├────────────────────────┼─────────────────────────────┼──────────────────┤
│ InstantiationAware BPP │ postProcessBeforeInst()     │ BEFORE constructor│
│                        │ postProcessAfterInst()      │ After ctor, pre-DI│
│                        │ postProcessProperties()     │ During DI         │
├────────────────────────┼─────────────────────────────┼──────────────────┤
│ SmartInstantiation BPP │ determineCandidateCtors()   │ Constructor select│
│   extends IABPP        │ getEarlyBeanReference()     │ Circular dep proxy│
│                        │ predictBeanType()           │ Type resolution   │
├────────────────────────┼─────────────────────────────┼──────────────────┤
│ MergedBeanDefinition   │ postProcessMergedBeanDef()  │ After ctor, pre-DI│
│   BPP                  │                             │ (metadata scan)   │
├────────────────────────┼─────────────────────────────┼──────────────────┤
│ DestructionAware BPP   │ postProcessBeforeDestruct() │ Before @PreDestroy│
│                        │ requiresDestruction()       │ Opt-out check     │
└────────────────────────┴─────────────────────────────┴──────────────────┘
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| BPP runs per-bean | Each bean creation triggers all BPP callbacks |
| null return | No change — previous result used |
| Non-null from postProcessBeforeInstantiation | Skip constructor/DI/init-callbacks; postProcessAfterInit still runs |
| false from postProcessAfterInstantiation | Skip ALL DI (XML + annotation-based) |
| AOP proxy creation timing | `postProcessAfterInitialization()` |
| @PostConstruct timing | `postProcessBeforeInitialization()` via CommonAnnotationBPP |
| MBDPP timing | After constructor, before addSingletonFactory, before DI |
| BPP ordering tiers | PriorityOrdered → Ordered → non-ordered → MBDPP |
| @Transactional on BPP beans | Unreliable — AOP may not apply |
| BPPs are singletons | Must be thread-safe |

---

### Built-in BPP Responsibilities

```
AutowiredAnnotationBPP (PriorityOrdered):
    → determineCandidateConstructors()  ← constructor selection
    → postProcessMergedBeanDefinition() ← scan @Autowired metadata
    → postProcessProperties()           ← inject @Autowired, @Value, @Inject

CommonAnnotationBPP (PriorityOrdered):
    → postProcessMergedBeanDefinition() ← scan @PostConstruct, @PreDestroy, @Resource
    → postProcessBeforeInitialization() ← CALL @PostConstruct
    → postProcessProperties()           ← inject @Resource
    → postProcessBeforeDestruction()    ← CALL @PreDestroy

ApplicationContextAwareProcessor (no ordering):
    → postProcessBeforeInitialization() ← call Aware callbacks 4-9

AbstractAutoProxyCreator (Ordered):
    → postProcessBeforeInstantiation()  ← infrastructure bean handling
    → getEarlyBeanReference()           ← AOP proxy for circular deps
    → postProcessAfterInitialization()  ← CREATE AOP PROXY
```

---

### Complete Per-Bean BPP Call Sequence

```
IABPP.postProcessBeforeInstantiation()   ← EARLIEST — before constructor
    ↓
[Constructor runs]
    ↓
MBDPP.postProcessMergedBeanDefinition()  ← scan injection metadata (cached)
    ↓
[addSingletonFactory — L3 cache]
    ↓
IABPP.postProcessAfterInstantiation()    ← return false = skip DI
    ↓
IABPP.postProcessProperties()           ← @Autowired, @Resource injected here
    ↓
[invokeAwareMethods: BeanName/ClassLoader/Factory aware]
    ↓
BPP.postProcessBeforeInitialization()   ← ApplicationContextAware, @PostConstruct
    ↓
[invokeInitMethods: afterPropertiesSet + init-method]
    ↓
BPP.postProcessAfterInitialization()    ← AOP PROXY CREATED HERE
```

---

### Interview One-Liners

- "BeanPostProcessor is called for EVERY bean as it is created — it is a per-bean hook, not a batch post-processor."
- "Returning `null` from `postProcessBeforeInitialization()` or `postProcessAfterInitialization()` means no change — the previous bean reference is used."
- "If `IABPP.postProcessBeforeInstantiation()` returns non-null, the constructor is never called, DI is never done, and init callbacks are skipped — but `postProcessAfterInitialization()` IS still called."
- "AOP proxies are created in `postProcessAfterInitialization()` by `AbstractAutoProxyCreator` — after ALL initialization callbacks complete on the raw bean."
- "`@PostConstruct` is invoked by `CommonAnnotationBeanPostProcessor.postProcessBeforeInitialization()` — before `afterPropertiesSet()` and custom init-method."
- "`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()` runs AFTER the constructor but BEFORE DI — it scans and caches annotation metadata for the injection phase."
- "BPP beans are instantiated in `registerBeanPostProcessors()` before ordinary beans — `@Transactional` and AOP do not reliably apply to BPP beans."
- "`DestructionAwareBPP.requiresDestruction()` is a performance optimization — beans without destroy callbacks return `false` and skip the destruction processing loop."

---
