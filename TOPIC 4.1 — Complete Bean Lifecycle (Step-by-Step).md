# TOPIC 4.1 — Complete Bean Lifecycle (Step-by-Step)

---

## ① CONCEPTUAL EXPLANATION

### Why Bean Lifecycle Mastery is Critical

The bean lifecycle is the most tested topic in Spring certification exams. Every other topic — BPPs, BFPPs, AOP, transactions, circular dependencies — intersects here. Understanding the precise ORDER, the MECHANISM, and the EDGE CASES of every lifecycle phase separates architects from developers.

The complete lifecycle has two phases:

```
CREATION PHASE (steps 1–10)   → bean goes from definition to fully initialized
DESTRUCTION PHASE (steps 11–13) → bean goes from active to destroyed
```

---

### The Complete Bean Lifecycle — All Steps

```
┌─────────────────────────────────────────────────────────────────┐
│                   BEAN CREATION PHASE                           │
├─────────────────────────────────────────────────────────────────┤
│  Step 1:  BeanDefinition loaded & registered                    │
│  Step 2:  BeanFactoryPostProcessor runs (modifies BDs)          │
│  Step 3:  Bean instantiation (constructor invoked)              │
│  Step 4:  Dependency injection (populateBean)                   │
│  Step 5:  Aware interface callbacks                             │
│  Step 6:  BPP.postProcessBeforeInitialization()                 │
│  Step 7:  Initialization callbacks (3 mechanisms)               │
│  Step 8:  BPP.postProcessAfterInitialization()                  │
│  Step 9:  Bean ready — stored in singleton cache                │
├─────────────────────────────────────────────────────────────────┤
│                  BEAN DESTRUCTION PHASE                         │
├─────────────────────────────────────────────────────────────────┤
│  Step 10: @PreDestroy                                           │
│  Step 11: DisposableBean.destroy()                              │
│  Step 12: Custom destroy-method                                 │
└─────────────────────────────────────────────────────────────────┘
```

Now let's go extremely deep on every single step.

---

### Step 1 — BeanDefinition Loading and Registration

Before any bean is created, the container reads all configuration and converts it into `BeanDefinition` objects stored in the `BeanDefinitionRegistry`.

**Sources of BeanDefinitions:**
- XML `<bean>` elements → `XmlBeanDefinitionReader`
- `@Component`/stereotype annotations → `ClassPathBeanDefinitionScanner`
- `@Bean` methods → `ConfigurationClassBeanDefinitionReader` (invoked by `ConfigurationClassPostProcessor`)
- Programmatic `registerBeanDefinition()`

**What is registered:**
```
beanDefinitionMap: ConcurrentHashMap<String, BeanDefinition>
beanDefinitionNames: List<String>  (ordered — registration order)
```

At this point: NO objects created. Pure metadata only.

---

### Step 2 — BeanFactoryPostProcessor Execution

BFPPs run AFTER all BeanDefinitions are registered but BEFORE any bean instantiation. They can:
- Modify existing BeanDefinitions (change scope, class, property values)
- Add new BeanDefinitions (`BeanDefinitionRegistryPostProcessor`)
- Resolve property placeholders (`PropertySourcesPlaceholderConfigurer`)

```
invokeBeanFactoryPostProcessors() order:
1. BeanDefinitionRegistryPostProcessor (PriorityOrdered)
   → ConfigurationClassPostProcessor runs here
2. BeanDefinitionRegistryPostProcessor (Ordered)
3. BeanDefinitionRegistryPostProcessor (no order)
4. BeanFactoryPostProcessor (PriorityOrdered)
   → PropertySourcesPlaceholderConfigurer runs here
5. BeanFactoryPostProcessor (Ordered)
6. BeanFactoryPostProcessor (no order)
```

**Critical:** BFPPs themselves must be instantiated to run. When Spring instantiates a BFPP, it does NOT run the full lifecycle for it — specifically, other BPPs do NOT process BFPP beans (to avoid chicken-and-egg problems). BFPPs get: constructor → Aware callbacks → init callbacks. But they do NOT get BPP processing.

---

### Step 3 — Bean Instantiation (Constructor Invoked)

This is where the Java object is created. Spring calls:

```java
// Inside AbstractAutowireCapableBeanFactory.createBeanInstance():
Constructor.newInstance(resolvedArgs)    // constructor injection
// OR
Method.invoke(factoryBean, args)         // @Bean factory method
// OR
Class.newInstance()                      // no-arg constructor (deprecated in Java 9+)
```

**What happens internally:**

```
AbstractAutowireCapableBeanFactory.doCreateBean()
    │
    ├── createBeanInstance()
    │       ├── If @Bean method → obtainFromSupplier() or instantiateUsingFactoryMethod()
    │       ├── If constructor autowiring → autowireConstructor() via ConstructorResolver
    │       └── Otherwise → instantiateBean() → BeanUtils.instantiateClass()
    │
    ├── applyMergedBeanDefinitionPostProcessors()
    │       → MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
    │       → AutowiredAnnotationBeanPostProcessor scans @Autowired metadata here
    │       → CommonAnnotationBeanPostProcessor scans @PostConstruct/@PreDestroy here
    │       → Result cached in InjectionMetadata
    │
    └── [If singleton] addSingletonFactory()
            → Registers ObjectFactory in Level 3 cache
            → Enables early exposure for circular dependency resolution
```

**At the end of Step 3:**
- The object EXISTS in memory
- All fields are at Java default values (null, 0, false)
- It is registered in the Level 3 singleton cache (ObjectFactory)
- NO dependencies injected yet

---

### Step 4 — Dependency Injection (populateBean)

Spring injects all dependencies into the instantiated bean:

```java
AbstractAutowireCapableBeanFactory.populateBean(beanName, mbd, bw):
    │
    ├── InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
    │       → Returns false? → SKIP all property injection entirely
    │       → Used by: Spring Data's @NoRepositoryBean, custom frameworks
    │
    ├── Apply XML/programmatic PropertyValues (autowireByName / autowireByType)
    │       → XML autowire="byName" or autowire="byType" resolved here
    │       → Explicit <property> values applied
    │
    └── InstantiationAwareBeanPostProcessor.postProcessProperties()
            → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
                    → InjectionMetadata.inject()
                    → For each @Autowired field/method:
                            resolveDependency() → getBean() → set via reflection
            → CommonAnnotationBeanPostProcessor.postProcessProperties()
                    → @Resource injection resolved here
```

**At the end of Step 4:**
- All `@Autowired`, `@Resource`, `@Value`, `@Inject` fields/methods injected
- XML `<property>` values applied
- Constructor args already satisfied from Step 3
- Aware interfaces NOT yet called

---

### Step 5 — Aware Interface Callbacks

After dependency injection, Spring calls Aware interface callbacks. These give beans access to Spring infrastructure objects.

**The Aware callback order is fixed and critical:**

```
Order of Aware callbacks:
1. BeanNameAware.setBeanName(String name)
2. BeanClassLoaderAware.setBeanClassLoader(ClassLoader classLoader)
3. BeanFactoryAware.setBeanFactory(BeanFactory beanFactory)
   ─── Above three are called by AbstractAutowireCapableBeanFactory directly ───
4. EnvironmentAware.setEnvironment(Environment environment)
5. EmbeddedValueResolverAware.setEmbeddedValueResolver(StringValueResolver resolver)
6. ResourceLoaderAware.setResourceLoader(ResourceLoader resourceLoader)
7. ApplicationEventPublisherAware.setApplicationEventPublisher(ApplicationEventPublisher publisher)
8. MessageSourceAware.setMessageSource(MessageSource messageSource)
9. ApplicationContextAware.setApplicationContext(ApplicationContext applicationContext)
   ─── Above six are called by ApplicationContextAwareProcessor (a BPP) ───
10. ServletContextAware (web only)
```

**Internal mechanism:**
- Callbacks 1-3 are called directly in `AbstractAutowireCapableBeanFactory.invokeAwareMethods()`
- Callbacks 4-9 are called by `ApplicationContextAwareProcessor.postProcessBeforeInitialization()`
  - This is a BPP registered in `prepareBeanFactory()` during `refresh()`
  - It runs as part of BPP processing (Step 6), but because it handles Aware interfaces, it's conceptually Step 5

**Why not just use `@Autowired` instead of Aware?**
- Aware callbacks are called BEFORE `@PostConstruct` — earlier than `@Autowired` initialization
- They are guaranteed to run even in BeanFactory (not just ApplicationContext)
- They work without annotation processing infrastructure
- Framework-internal beans use Aware to avoid circular dependencies with the container itself

---

### Step 6 — BPP.postProcessBeforeInitialization()

All registered BPPs run their `postProcessBeforeInitialization()` method before any init callbacks:

```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        return bean; // return modified or replacement bean, or original
    }
}
```

**Critical contract:** The return value IS the bean that continues through the lifecycle. If a BPP returns a different object, that object replaces the original. If it returns `null`, Spring uses the original (null return is treated as "no change" for this callback).

**Key BPPs running here:**
- `ApplicationContextAwareProcessor` — handles Aware callbacks 4-9 (conceptually Step 5)
- `ConfigurationPropertiesBindingPostProcessor` — binds `@ConfigurationProperties` (Spring Boot)
- `InitDestroyAnnotationBeanPostProcessor` — actually calls `@PostConstruct` here (via CommonAnnotationBPP)

**Wait — @PostConstruct is in Step 6?**
Yes. `CommonAnnotationBeanPostProcessor` extends `InitDestroyAnnotationBeanPostProcessor`. Its `postProcessBeforeInitialization()` method calls all `@PostConstruct` methods. So `@PostConstruct` runs as part of "before initialization" BPP processing — which runs BEFORE `InitializingBean.afterPropertiesSet()`.

This is a source of confusion. The conceptual order is:
```
BPP.postProcessBeforeInitialization (which includes @PostConstruct)
    → InitializingBean.afterPropertiesSet()
    → custom init-method
```

---

### Step 7 — Initialization Callbacks (Three Mechanisms)

Spring supports three initialization mechanisms, called in strict order:

```
Order:
1. @PostConstruct           (JSR-250 — via CommonAnnotationBeanPostProcessor in Step 6)
2. InitializingBean.afterPropertiesSet()  (Spring interface)
3. Custom init-method       (XML init-method / @Bean(initMethod=...))
```

**Why this order?**

`@PostConstruct` runs in Step 6 (as part of BPP before-init). Then `afterPropertiesSet()` and `init-method` run in Step 7 proper, called directly by `AbstractAutowireCapableBeanFactory.invokeInitMethods()`:

```java
protected void invokeInitMethods(String beanName, Object bean, RootBeanDefinition mbd) {

    // Step 7a: InitializingBean
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null ||
            !mbd.hasAnyExternallyManagedInitMethod("afterPropertiesSet"))) {
        ((InitializingBean) bean).afterPropertiesSet();
    }

    // Step 7b: Custom init-method
    if (mbd != null) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName))) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

**What each is used for:**

| Mechanism | Use Case |
|---|---|
| `@PostConstruct` | JSR-250 standard, recommended for application code |
| `InitializingBean` | Framework/library code (couples to Spring API) |
| `init-method` | Third-party classes you cannot annotate |

**All three can coexist on the same bean — they all run, in the above order.**

```java
@Component
public class FullLifecycleBean implements InitializingBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct"); // FIRST
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. afterPropertiesSet"); // SECOND
    }

    public void customInit() {
        System.out.println("3. custom init-method"); // THIRD
    }
}
```

---

### Step 8 — BPP.postProcessAfterInitialization()

After all initialization callbacks complete, BPPs run their `postProcessAfterInitialization()`:

```java
public interface BeanPostProcessor {
    default Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;
    }
}
```

**This is where AOP proxies are created.**

`AnnotationAwareAspectJAutoProxyCreator` (or `AbstractAutoProxyCreator`) runs here. It inspects the bean, determines if any advice applies, and if so, wraps the bean in a JDK dynamic proxy or CGLIB proxy:

```
postProcessAfterInitialization(bean, beanName):
    → Does any Advisor/Aspect match this bean?
    → YES: create proxy wrapping the bean
           return proxy (replaces original bean in container)
    → NO:  return bean unchanged
```

**Critical implication:** The object stored in the singleton cache after Step 8 might be a PROXY, not the original bean. When you `@Autowired` a bean that has AOP applied, you get the proxy. `AopUtils.isAopProxy(bean)` will return `true`.

**Other important BPPs running here:**
- `AsyncAnnotationBeanPostProcessor` — wraps beans with `@Async` in async proxy
- `CachingOperationsSource` — wraps `@Cacheable` beans in caching proxy
- `ScheduledAnnotationBeanPostProcessor` — registers `@Scheduled` methods

---

### Step 9 — Bean Ready for Use

After Step 8 completes:

```java
// In DefaultSingletonBeanRegistry.getSingleton():
// If singleton: add to Level 1 cache, remove from Level 2 and Level 3
addSingleton(beanName, singletonObject);

private void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);      // L1
        this.singletonFactories.remove(beanName);                   // remove L3
        this.earlySingletonObjects.remove(beanName);                // remove L2
        this.registeredSingletons.add(beanName);
    }
}
```

The bean (or its proxy) is now:
- In the singleton cache (Level 1)
- Available for `getBean()` calls
- Available for injection into other beans

---

### Step 10 — @PreDestroy

When the container shuts down (`close()` called or JVM shutdown hook fires):

`CommonAnnotationBeanPostProcessor` (which also extends `DestructionAwareBeanPostProcessor`) calls `@PreDestroy` methods:

```java
// In DestructionAwareBeanPostProcessor:
void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
```

`@PreDestroy` is the first destruction callback — called before `DisposableBean.destroy()` or custom destroy-method.

---

### Step 11 — DisposableBean.destroy()

```java
public interface DisposableBean {
    void destroy() throws Exception;
}
```

Called after `@PreDestroy`. If a bean implements `DisposableBean`, `destroy()` is invoked.

---

### Step 12 — Custom destroy-method

The custom destroy method (specified via `@Bean(destroyMethod="...")`, XML `destroy-method`, or `@PreDestroy`-equivalent) is called last.

**Default destroy-method detection (Spring's "inferred" destroy method):**

Spring has an `AbstractBeanDefinition.INFER_METHOD = "(inferred)"` constant. When `destroyMethod` is set to `"(inferred)"`, Spring automatically detects and calls a `public void close()` or `public void shutdown()` method on the bean, if present. This is the default for `@Bean` methods — they automatically detect `close()` and `shutdown()`.

```java
@Bean  // destroyMethod defaults to "(inferred)" — close() will be called automatically
public DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    return ds; // HikariDataSource.close() called at shutdown automatically
}

@Bean(destroyMethod = "")  // Explicitly disable inferred destroy method
public DataSource externalDataSource() {
    return sharedDataSource; // Don't close — managed externally
}
```

---

### Destruction Order Rules

**Destruction is the REVERSE of initialization order:**
- Beans created last are destroyed first
- If A depends on B (B created first), then A is destroyed first, B is destroyed last
- This ensures a bean's dependencies are still alive during its own destruction

**Prototype beans are NOT destroyed by Spring:**
- Spring creates prototype beans but does NOT track them
- `@PreDestroy`, `DisposableBean.destroy()`, and custom destroy-methods are NEVER called for prototype beans
- The developer is responsible for prototype bean cleanup

**Lazy beans destruction:**
- If a lazy bean was never created (never requested), its destroy callbacks are never called
- If it was created (first requested), it IS tracked and destroyed

---

### Complete Lifecycle — The Exact Call Stack

```java
// AbstractApplicationContext.refresh() → finishBeanFactoryInitialization()
//  → DefaultListableBeanFactory.preInstantiateSingletons()
//   → AbstractBeanFactory.getBean()
//    → AbstractBeanFactory.doGetBean()
//     → DefaultSingletonBeanRegistry.getSingleton() [with ObjectFactory]
//      → AbstractAutowireCapableBeanFactory.createBean()
//       → AbstractAutowireCapableBeanFactory.doCreateBean()

doCreateBean(beanName, mbd, args):
    │
    ├── createBeanInstance()                           ← Step 3: Constructor
    ├── applyMergedBeanDefinitionPostProcessors()      ← Step 3: Scan @Autowired metadata
    ├── addSingletonFactory()                          ← Step 3: Early exposure (L3 cache)
    ├── populateBean()                                 ← Step 4: DI
    │       ├── postProcessAfterInstantiation()        ← IABPP hook
    │       └── postProcessProperties()               ← @Autowired, @Resource injection
    └── initializeBean()                              ← Steps 5-8
            ├── invokeAwareMethods()                  ← Step 5a: BeanNameAware, BeanFactoryAware
            ├── applyBeanPostProcessorsBeforeInit()   ← Step 6: BPP.before (includes @PostConstruct)
            ├── invokeInitMethods()                   ← Step 7: afterPropertiesSet + init-method
            └── applyBeanPostProcessorsAfterInit()    ← Step 8: BPP.after (AOP proxy created here)
```

---

### The initializeBean() Call — Source Level

```java
protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {

    // Step 5a: BeanNameAware, BeanClassLoaderAware, BeanFactoryAware
    invokeAwareMethods(beanName, bean);

    // Step 6: BPP.postProcessBeforeInitialization
    // (includes ApplicationContextAwareProcessor → env/ctx aware callbacks)
    // (includes CommonAnnotationBPP → @PostConstruct called HERE)
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // Step 7: afterPropertiesSet() + init-method
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Invocation of init method failed", ex);
    }

    // Step 8: BPP.postProcessAfterInitialization
    // (AOP proxy created HERE by AbstractAutoProxyCreator)
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean; // May be a proxy!
}
```

---

### Why @PostConstruct Runs Before afterPropertiesSet()

This surprises many developers. The reason:

`CommonAnnotationBeanPostProcessor` extends `InitDestroyAnnotationBeanPostProcessor` which implements `BeanPostProcessor`. Its `postProcessBeforeInitialization()` method finds and invokes `@PostConstruct` methods. Since ALL BPPs run in Step 6 (before Step 7), `@PostConstruct` runs before `afterPropertiesSet()`.

This was a deliberate design decision — `@PostConstruct` is the "outermost" (first to run) init mechanism, allowing framework code using `InitializingBean` to build upon what `@PostConstruct` established.

---

### Lifecycle of a BFPP (Different from Regular Beans)

BFPPs have an ABBREVIATED lifecycle because they must exist before other BPPs are registered:

```
BFPP Lifecycle:
1. Instantiation (constructor)
2. Dependency injection (populateBean)
3. Aware callbacks (BeanNameAware, BeanFactoryAware only — NOT ApplicationContextAware)
4. @PostConstruct / afterPropertiesSet() / init-method
   ↑ NO BPP processing — BPPs not registered yet when BFPPs run
5. BFPP.postProcessBeanFactory() — the BFPP's own work
6. [At shutdown] Destruction callbacks
```

**This explains why:**
- You cannot use `@Autowired` in a BFPP if the dependency is another BFPP (circular BFPP creation)
- Static `@Bean` methods for BFPPs avoid this by not requiring CGLIB proxy
- BFPPs should use constructor injection where possible

---

### Lifecycle of a BPP (Also Different)

BPPs have a SLIGHTLY abbreviated lifecycle:

```
BPP Lifecycle:
1. Instantiation (constructor)
2. Dependency injection
3. Aware callbacks
4. Merging BeanDefinition post-processing (MergedBeanDefinitionPostProcessor)
5. @PostConstruct / afterPropertiesSet() / init-method
   ↑ BPPs registered before this BPP run — but THIS BPP is not yet registered
   → So this BPP's initialization IS processed by already-registered BPPs
6. BPP ready — registered in BPP list
7. [At shutdown] Destruction callbacks
```

---

### Prototype Bean Lifecycle

Prototype beans follow the CREATION lifecycle identically to singletons, with two critical differences:

```
Prototype creation lifecycle: SAME as singleton (steps 1-9)

Differences:
1. NOT stored in singleton cache
   → Each getBean() creates a NEW instance

2. Spring does NOT track prototype beans after creation
   → @PreDestroy is NEVER called
   → DisposableBean.destroy() is NEVER called
   → Custom destroy-method is NEVER called
   → Developer is responsible for cleanup

3. BPP.postProcessAfterInitialization() IS called for prototypes
   → AOP proxies ARE applied to prototype beans
   → Each prototype may get its own proxy instance
```

---

### Lifecycle with AOP — When the Proxy Appears

```
Step 3: Raw bean instantiated → MyService@1a2b3c
Step 4: Dependencies injected into raw bean
Step 5: Aware callbacks on raw bean
Step 6: @PostConstruct called on RAW bean
Step 7: afterPropertiesSet() called on RAW bean
Step 8: BPP.postProcessAfterInitialization()
        → AbstractAutoProxyCreator checks: does any aspect apply?
        → YES: create proxy wrapping MyService@1a2b3c
        → return MyService$$SpringCGLIB$$0@4d5e6f (PROXY)

Result stored in singleton cache: MyService$$SpringCGLIB$$0@4d5e6f (PROXY)

When you @Autowired MyService service:
    → you get the PROXY
    → calling service.doWork() → proxy intercepts → advice runs → delegates to raw bean
```

**Critical:** `@PostConstruct`, `afterPropertiesSet()`, and `init-method` are called on the RAW bean, NOT the proxy. The proxy does not exist yet when these callbacks run.

---

### Common Misconceptions

**Misconception 1:** "AOP proxy is created before initialization callbacks."
Wrong. AOP proxy is created in `postProcessAfterInitialization()` — AFTER `@PostConstruct`, `afterPropertiesSet()`, and `init-method` all complete on the RAW bean.

**Misconception 2:** "`@PostConstruct` runs after `afterPropertiesSet()`."
Wrong. `@PostConstruct` runs BEFORE `afterPropertiesSet()`. `@PostConstruct` is processed by `CommonAnnotationBPP` in `postProcessBeforeInitialization()` (Step 6). `afterPropertiesSet()` runs in Step 7 (invokeInitMethods).

**Misconception 3:** "Prototype beans get `@PreDestroy` called by Spring."
Wrong. Spring does NOT track prototype beans after creation. No destruction callbacks are called for prototypes.

**Misconception 4:** "The bean in the singleton cache is always the original bean instance."
Wrong. If AOP applies, the singleton cache holds the PROXY. The original bean is held by the proxy as its target. `context.getBean(MyService.class)` returns the proxy.

**Misconception 5:** "`ApplicationContextAware.setApplicationContext()` is called before BPP processing."
Wrong. `ApplicationContextAwareProcessor` is a BPP. It runs in Step 6 (`postProcessBeforeInitialization`), NOT before it. Callbacks 1-3 (`BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`) ARE called before BPP processing. Callbacks 4-9 are handled by a BPP.

**Misconception 6:** "All BPPs process all beans."
Wrong. BPPs themselves have an abbreviated lifecycle. When a BPP bean is being initialized, other BPPs that were registered before it DO process it. BPPs registered after it do NOT.

**Misconception 7:** "The default `destroyMethod` is empty (no destroy method)."
Wrong. For `@Bean` methods, the default `destroyMethod` is `"(inferred)"` — Spring automatically detects and calls `close()` or `shutdown()` methods. You must explicitly set `destroyMethod=""` to suppress this.

---

## ② CODE EXAMPLES

### Example 1 — Complete Lifecycle Demonstration

```java
@Component
public class FullLifecycleBean
        implements InitializingBean, DisposableBean,
                   BeanNameAware, BeanFactoryAware, ApplicationContextAware {

    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;

    @Autowired
    private SomeDependency dependency;

    // ── Step 3: Constructor ──────────────────────────────────────────
    public FullLifecycleBean() {
        System.out.println("STEP 3: Constructor called");
        // dependency is NULL here — DI not done yet
    }

    // ── Step 4: DI (handled by BPP, not shown here explicitly) ──────

    // ── Step 5a: BeanNameAware ──────────────────────────────────────
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("STEP 5a: BeanNameAware.setBeanName(" + name + ")");
    }

    // ── Step 5b: BeanFactoryAware ────────────────────────────────────
    @Override
    public void setBeanFactory(BeanFactory factory) {
        this.beanFactory = factory;
        System.out.println("STEP 5b: BeanFactoryAware.setBeanFactory()");
    }

    // ── Step 5c: ApplicationContextAware (via BPP in Step 6) ─────────
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
        System.out.println("STEP 5c/6: ApplicationContextAware.setApplicationContext()");
    }

    // ── Step 6: @PostConstruct (via CommonAnnotationBPP) ────────────
    @PostConstruct
    public void postConstruct() {
        System.out.println("STEP 6: @PostConstruct — dependency: " + dependency);
        // dependency IS injected by now ✓
    }

    // ── Step 7a: InitializingBean ────────────────────────────────────
    @Override
    public void afterPropertiesSet() {
        System.out.println("STEP 7a: InitializingBean.afterPropertiesSet()");
    }

    // ── Step 7b: Custom init-method (declared via @Bean or XML) ──────
    public void customInit() {
        System.out.println("STEP 7b: Custom init-method");
    }

    // ── Runtime usage ────────────────────────────────────────────────
    public void doWork() {
        System.out.println("Bean in use: " + beanName);
    }

    // ── Step 10: @PreDestroy ──────────────────────────────────────────
    @PreDestroy
    public void preDestroy() {
        System.out.println("STEP 10: @PreDestroy");
    }

    // ── Step 11: DisposableBean ──────────────────────────────────────
    @Override
    public void destroy() {
        System.out.println("STEP 11: DisposableBean.destroy()");
    }

    // ── Step 12: Custom destroy-method (declared via @Bean or XML) ───
    public void customDestroy() {
        System.out.println("STEP 12: Custom destroy-method");
    }
}
```

```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public FullLifecycleBean fullLifecycleBean() {
        return new FullLifecycleBean();
    }
}
```

**Output:**
```
STEP 3:    Constructor called
STEP 5a:   BeanNameAware.setBeanName(fullLifecycleBean)
STEP 5b:   BeanFactoryAware.setBeanFactory()
STEP 5c/6: ApplicationContextAware.setApplicationContext()
STEP 6:    @PostConstruct — dependency: SomeDependency@...
STEP 7a:   InitializingBean.afterPropertiesSet()
STEP 7b:   Custom init-method
--- Bean in use ---
STEP 10:   @PreDestroy
STEP 11:   DisposableBean.destroy()
STEP 12:   Custom destroy-method
```

---

### Example 2 — AOP Proxy Creation Timing

```java
@Aspect
@Component
public class TimingAspect {
    @Around("execution(* com.example.MyService.*(..))")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        System.out.println("Elapsed: " + (System.currentTimeMillis() - start));
        return result;
    }
}

@Service
public class MyService {

    @PostConstruct
    public void init() {
        // 'this' here is the RAW bean — NOT the proxy
        System.out.println("@PostConstruct: this.getClass() = " +
            this.getClass().getSimpleName());
        // prints: MyService (NOT MyService$$SpringCGLIB$$0)
    }

    public void doWork() { /* ... */ }
}
```

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyService service = ctx.getBean(MyService.class);
System.out.println("From ctx: " + service.getClass().getSimpleName());
// prints: MyService$$SpringCGLIB$$0 (or JDK proxy)
// The proxy was created in postProcessAfterInitialization, AFTER @PostConstruct ran
```

---

### Example 3 — All Three Init Mechanisms Together

```java
@Component
public class TripleInitBean implements InitializingBean {

    @PostConstruct
    public void step1() {
        System.out.println("Init 1: @PostConstruct"); // ALWAYS FIRST
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("Init 2: afterPropertiesSet"); // ALWAYS SECOND
    }

    public void step3() {
        System.out.println("Init 3: custom init-method"); // ALWAYS THIRD
    }
}

@Configuration
public class Config {
    @Bean(initMethod = "step3")
    public TripleInitBean tripleInitBean() {
        return new TripleInitBean();
    }
}
```

Output:
```
Init 1: @PostConstruct
Init 2: afterPropertiesSet
Init 3: custom init-method
```

---

### Example 4 — All Three Destroy Mechanisms Together

```java
@Component
public class TripleDestroyBean implements DisposableBean {

    @PreDestroy
    public void step1() {
        System.out.println("Destroy 1: @PreDestroy"); // ALWAYS FIRST
    }

    @Override
    public void destroy() {
        System.out.println("Destroy 2: DisposableBean.destroy()"); // ALWAYS SECOND
    }

    public void step3() {
        System.out.println("Destroy 3: custom destroy-method"); // ALWAYS THIRD
    }
}

@Configuration
public class Config {
    @Bean(destroyMethod = "step3")
    public TripleDestroyBean tripleDestroyBean() {
        return new TripleDestroyBean();
    }
}
```

---

### Example 5 — Prototype Bean Lifecycle Proof

```java
@Component
@Scope("prototype")
public class PrototypeBean {

    @PostConstruct
    public void init() {
        System.out.println("Prototype @PostConstruct: " + this.hashCode());
    }

    @PreDestroy
    public void destroy() {
        // This NEVER runs — Spring doesn't track prototypes
        System.out.println("Prototype @PreDestroy: " + this.hashCode());
    }
}

@Service
public class PrototypeConsumer {
    @Autowired
    private ApplicationContext ctx;

    public void demonstrate() {
        PrototypeBean p1 = ctx.getBean(PrototypeBean.class);
        // prints: Prototype @PostConstruct: 12345 ← @PostConstruct DOES run
        PrototypeBean p2 = ctx.getBean(PrototypeBean.class);
        // prints: Prototype @PostConstruct: 67890 ← new instance, @PostConstruct runs again
        System.out.println(p1 == p2); // false — different instances
    }
}

// At context.close():
// Nothing printed for PrototypeBean — @PreDestroy never called
```

---

### Example 6 — BeanPostProcessor Intercepting Lifecycle

```java
@Component
public class LifecycleTrackerBPP implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof TrackedBean) {
            System.out.println("BPP before-init for: " + beanName
                + " | Class: " + bean.getClass().getSimpleName());
            // bean here is the RAW bean (before @PostConstruct in some BPP orderings)
            // Actually @PostConstruct already ran in CommonAnnotationBPP
        }
        return bean; // Return same bean — no replacement
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof TrackedBean) {
            System.out.println("BPP after-init for: " + beanName
                + " | Class: " + bean.getClass().getSimpleName());
            // If AOP applied, this BPP receives the PROXY from AbstractAutoProxyCreator
            // (depending on BPP order)
        }
        return bean;
    }
}
```

---

### Example 7 — Default destroyMethod Inference

```java
@Configuration
public class DataSourceConfig {

    // Case 1: Default inferred destroyMethod — close() will be called automatically
    @Bean
    public HikariDataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost/mydb");
        return ds;
        // At shutdown: HikariDataSource.close() called automatically
        // because @Bean destroyMethod defaults to "(inferred)"
    }

    // Case 2: Explicitly disable destroy method inference
    @Bean(destroyMethod = "")
    public DataSource sharedDataSource() {
        return externallyManagedDataSource; // managed externally — don't close
    }

    // Case 3: Explicit destroy method name
    @Bean(destroyMethod = "shutdown")
    public ExecutorService executorService() {
        return Executors.newFixedThreadPool(10);
        // At shutdown: executorService.shutdown() called
    }
}
```

---

### Example 8 — Aware Interfaces Order Proof

```java
@Component
public class AwareOrderDemo
        implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware,
                   EnvironmentAware, ResourceLoaderAware,
                   ApplicationEventPublisherAware, ApplicationContextAware {

    @Override
    public void setBeanName(String name) {
        System.out.println("1. BeanNameAware");
    }

    @Override
    public void setBeanClassLoader(ClassLoader cl) {
        System.out.println("2. BeanClassLoaderAware");
    }

    @Override
    public void setBeanFactory(BeanFactory bf) {
        System.out.println("3. BeanFactoryAware");
    }

    // Steps 4-9 handled by ApplicationContextAwareProcessor BPP

    @Override
    public void setEnvironment(Environment env) {
        System.out.println("4. EnvironmentAware");
    }

    @Override
    public void setResourceLoader(ResourceLoader rl) {
        System.out.println("5. ResourceLoaderAware");
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher pub) {
        System.out.println("6. ApplicationEventPublisherAware");
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("7. ApplicationContextAware");
    }

    @PostConstruct
    public void init() {
        System.out.println("8. @PostConstruct");
        // All Aware callbacks already completed by this point
    }
}
```

Output:
```
1. BeanNameAware
2. BeanClassLoaderAware
3. BeanFactoryAware
4. EnvironmentAware
5. ResourceLoaderAware
6. ApplicationEventPublisherAware
7. ApplicationContextAware
8. @PostConstruct
```

---

### Example 9 — Edge Case: BPP Returning Replacement Object

```java
@Component
public class WrappingBPP implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof PaymentService) {
            // Return a DIFFERENT object — wraps the original
            return new AuditingPaymentServiceDecorator((PaymentService) bean);
        }
        return bean;
    }
}

// Now ctx.getBean(PaymentService.class) returns AuditingPaymentServiceDecorator
// NOT the original PaymentService
// This is how Spring's AOP proxy creation works internally
```

---

### Example 10 — Inferred vs Explicit destroyMethod

```java
public class ConnectionPool {
    public void close() {
        System.out.println("ConnectionPool.close() called");
    }

    public void shutdown() {
        System.out.println("ConnectionPool.shutdown() called");
    }
}

@Configuration
public class Config {

    // Default: destroyMethod = "(inferred)" → close() called (close takes priority over shutdown)
    @Bean
    public ConnectionPool pool1() { return new ConnectionPool(); }
    // At shutdown: close() called ✓

    // Explicit: only shutdown() called
    @Bean(destroyMethod = "shutdown")
    public ConnectionPool pool2() { return new ConnectionPool(); }
    // At shutdown: shutdown() called ✓

    // Disabled: no destroy method called
    @Bean(destroyMethod = "")
    public ConnectionPool pool3() { return new ConnectionPool(); }
    // At shutdown: nothing called ✓
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
In what order are the following initialization mechanisms called for a Spring bean?

A) `afterPropertiesSet()` → `@PostConstruct` → custom `init-method`
B) `@PostConstruct` → custom `init-method` → `afterPropertiesSet()`
C) `@PostConstruct` → `afterPropertiesSet()` → custom `init-method`
D) custom `init-method` → `@PostConstruct` → `afterPropertiesSet()`

**Answer: C**
`@PostConstruct` runs in `postProcessBeforeInitialization()` (BPP Step 6). `afterPropertiesSet()` runs first in `invokeInitMethods()` (Step 7). Custom `init-method` runs second in `invokeInitMethods()` (Step 7).

---

**Q2 — MCQ:**
When is the AOP proxy for a Spring bean created?

A) Before constructor injection, so all injected objects go through the proxy
B) After `@PostConstruct` but before `afterPropertiesSet()`
C) In `postProcessAfterInitialization()` — after all initialization callbacks
D) During `populateBean()` alongside dependency injection

**Answer: C**
AOP proxy creation happens in `postProcessAfterInitialization()` via `AbstractAutoProxyCreator`. This runs AFTER `@PostConstruct`, `afterPropertiesSet()`, and `init-method` all complete on the raw bean.

---

**Q3 — Select all that apply:**
Which statements about prototype bean lifecycle are correct? (Select ALL that apply)

A) `@PostConstruct` IS called for prototype beans
B) `@PreDestroy` IS called for prototype beans when the container closes
C) Spring creates a new prototype instance for each `getBean()` call
D) AOP proxies ARE applied to prototype beans
E) `DisposableBean.destroy()` is called for prototype beans
F) Spring stores prototype bean references in the singleton cache after creation

**Answer: A, C, D**
B and E are wrong — Spring does NOT call destroy callbacks for prototype beans. F is wrong — prototype beans are NOT stored in the singleton cache (that would make them singletons).

---

**Q4 — Code Output Prediction:**

```java
@Component
public class LifecycleBean implements InitializingBean {

    @PostConstruct
    public void init1() { System.out.println("A"); }

    @Override
    public void afterPropertiesSet() { System.out.println("B"); }

    @PreDestroy
    public void destroy1() { System.out.println("C"); }
}
```

```java
ConfigurableApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
ctx.close();
```

What is the output?

A) A → B → C
B) B → A → C
C) A → C → B
D) C → A → B

**Answer: A**
Init: `@PostConstruct` (A) runs before `afterPropertiesSet()` (B). No custom destroy-method. Destroy: `@PreDestroy` (C). Full output: `A → B → C`.

---

**Q5 — Trick Question:**

```java
@Service
public class MyService {
    @PostConstruct
    public void init() {
        System.out.println("Class in @PostConstruct: " +
            this.getClass().getSimpleName());
    }
}

// MyService has @Transactional methods — so AOP proxy is created
```

What does the `@PostConstruct` print?

A) `MyService$$SpringCGLIB$$0` — the proxy is already created
B) `MyService` — the raw bean, proxy not yet created
C) `Proxy$$MyService$$0` — JDK proxy class name
D) Depends on whether JDK or CGLIB proxy is used

**Answer: B**
`@PostConstruct` runs during `postProcessBeforeInitialization()` (Step 6). AOP proxy is created in `postProcessAfterInitialization()` (Step 8). At `@PostConstruct` time, `this` refers to the raw `MyService` — no proxy exists yet.

---

**Q6 — Select all that apply:**
Which Aware interface callbacks are handled directly by `AbstractAutowireCapableBeanFactory.invokeAwareMethods()` (NOT by a BPP)? (Select ALL that apply)

A) `BeanNameAware`
B) `BeanClassLoaderAware`
C) `BeanFactoryAware`
D) `ApplicationContextAware`
E) `EnvironmentAware`
F) `ResourceLoaderAware`

**Answer: A, B, C**
`BeanNameAware`, `BeanClassLoaderAware`, and `BeanFactoryAware` are handled directly by `invokeAwareMethods()`. The others (D, E, F) are handled by `ApplicationContextAwareProcessor` (a BPP) in `postProcessBeforeInitialization()`.

---

**Q7 — Scenario-based:**

```java
@Configuration
public class Config {

    @Bean  // No explicit destroyMethod
    public HikariDataSource dataSource() {
        return new HikariDataSource();
    }
}
```

When the ApplicationContext is closed, what happens to the `HikariDataSource`?

A) Nothing — no destroy-method is configured
B) `close()` is called — Spring infers it from the `(inferred)` default
C) `destroy()` is called — Spring always calls destroy() on shutdown
D) `finalize()` is called — JVM GC handles it

**Answer: B**
For `@Bean` methods, the default `destroyMethod` is `"(inferred)"`. Spring checks if the bean has a `public void close()` or `public void shutdown()` method and calls it. `HikariDataSource` has a `close()` method → it is called automatically at shutdown.

---

**Q8 — Code Output Prediction:**

```java
@Component
public class BeanA {
    @PostConstruct void init() { System.out.println("A init"); }
    @PreDestroy   void dest() { System.out.println("A destroy"); }
}

@Component
public class BeanB {
    @Autowired BeanA beanA; // B depends on A
    @PostConstruct void init() { System.out.println("B init"); }
    @PreDestroy   void dest() { System.out.println("B destroy"); }
}
```

```java
ConfigurableApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
ctx.close();
```

What is the output?

A)
```
A init
B init
A destroy
B destroy
```
B)
```
A init
B init
B destroy
A destroy
```
C)
```
B init
A init
B destroy
A destroy
```
D)
```
A init
B init
A destroy
B destroy
```

**Answer: B**
A is created first (B depends on A). A init → B init. Destruction is REVERSE of creation: B destroyed first → A destroyed last. Output: `A init → B init → B destroy → A destroy`.

---

**Q9 — Compilation/Runtime Error:**

```java
@Component
public class EarlyBean implements BeanFactoryAware {

    private DataSource dataSource;

    @Override
    public void setBeanFactory(BeanFactory bf) throws BeansException {
        // Trying to access another bean inside BeanFactoryAware callback
        this.dataSource = bf.getBean(DataSource.class);
    }
}
```

What happens?

A) Works perfectly — all beans are available by the time `BeanFactoryAware` is called
B) `NoSuchBeanDefinitionException` — DataSource not registered yet
C) Works but is unsafe — may trigger premature initialization of DataSource, bypassing some BPP processing
D) `NullPointerException` — BeanFactory is null

**Answer: C**
`BeanFactoryAware.setBeanFactory()` is called during `initializeBean()` — DURING the singleton creation loop. Calling `bf.getBean(DataSource.class)` at this point forces `DataSource` to be created NOW, potentially before its BPP processing is complete, bypassing proper lifecycle ordering. It "works" but is a dangerous anti-pattern. Using `@Autowired` is always safer.

---

**Q10 — Drag-and-Drop: Order these lifecycle events from first to last:**

- `BPP.postProcessAfterInitialization()`
- `Constructor called`
- `@PreDestroy`
- `BeanFactoryAware.setBeanFactory()`
- `@PostConstruct`
- `Dependency injection (populateBean)`
- `ApplicationContextAware.setApplicationContext()`
- `InitializingBean.afterPropertiesSet()`
- `DisposableBean.destroy()`
- `Custom init-method`

**Answer:**
1. `Constructor called`
2. `Dependency injection (populateBean)`
3. `BeanFactoryAware.setBeanFactory()`
4. `ApplicationContextAware.setApplicationContext()`
5. `@PostConstruct`
6. `InitializingBean.afterPropertiesSet()`
7. `Custom init-method`
8. `BPP.postProcessAfterInitialization()`
9. `@PreDestroy`
10. `DisposableBean.destroy()`

---

## ④ TRICK ANALYSIS

**Trap 1: @PostConstruct vs afterPropertiesSet() order**
The single most tested lifecycle ordering question. `@PostConstruct` always runs BEFORE `afterPropertiesSet()`. `@PostConstruct` is a BPP-driven mechanism (CommonAnnotationBPP runs in `postProcessBeforeInitialization`). `afterPropertiesSet()` is called in `invokeInitMethods()` which is Step 7. Any answer saying `afterPropertiesSet()` runs before `@PostConstruct` is wrong.

**Trap 2: AOP proxy creation timing**
AOP proxy is created in `postProcessAfterInitialization()` — AFTER ALL initialization callbacks. Any answer suggesting the proxy exists during `@PostConstruct` or `afterPropertiesSet()` is wrong. When `this` is used inside `@PostConstruct`, it refers to the RAW bean.

**Trap 3: Prototype @PreDestroy**
Spring does NOT call `@PreDestroy` on prototype beans. EVER. Any question asking "when does @PreDestroy run for a prototype bean?" — the answer is NEVER. The lifecycle ends at Step 9 for prototypes (after `postProcessAfterInitialization()`). No tracking, no destruction.

**Trap 4: Default destroyMethod for @Bean**
For `@Bean` methods without explicit `destroyMethod`, the default is `"(inferred)"` — Spring auto-detects `close()` or `shutdown()`. This is NOT an empty string. Many candidates assume no destroyMethod = no cleanup. Wrong. If your bean has `close()`, it WILL be called unless you explicitly set `destroyMethod=""`.

**Trap 5: ApplicationContextAware runs before or after @PostConstruct?**
`ApplicationContextAware.setApplicationContext()` is handled by `ApplicationContextAwareProcessor` which is a BPP running in `postProcessBeforeInitialization()`. `@PostConstruct` is ALSO handled by a BPP in `postProcessBeforeInitialization()`. The order depends on BPP registration order. `ApplicationContextAwareProcessor` is registered in `prepareBeanFactory()` BEFORE `CommonAnnotationBPP` is registered in `registerBeanPostProcessors()`. So `ApplicationContextAware` runs BEFORE `@PostConstruct`.

**Trap 6: BeanFactoryAware vs ApplicationContextAware invocation mechanism**
`BeanFactoryAware` (and `BeanNameAware`, `BeanClassLoaderAware`) are called directly in `invokeAwareMethods()` — NOT via BPP. `ApplicationContextAware` (and `EnvironmentAware`, etc.) ARE called via `ApplicationContextAwareProcessor` BPP. This distinction matters because `invokeAwareMethods()` runs BEFORE any BPP processing.

**Trap 7: BPP returning null from postProcessBeforeInitialization**
If a BPP returns `null` from `postProcessBeforeInitialization()`, Spring uses the ORIGINAL bean (null return = no change). If a BPP returns `null` from `postProcessAfterInitialization()`, Spring also uses the original bean. Returning null is treated as "pass through." Returning a different object REPLACES the bean.

**Trap 8: When is the bean in the singleton cache?**
The bean is placed in the Level 1 singleton cache AFTER `postProcessAfterInitialization()` completes (after Step 8). During Steps 3-8, the bean is tracked via Level 3 (ObjectFactory) and potentially Level 2 (early reference), but NOT in Level 1. A `getBean()` call during initialization of another bean may get an early reference from L2/L3.

**Trap 9: BFPP beans and BPP processing**
BFPP beans do NOT get processed by `BeanPostProcessors` in the normal way because BFPPs are instantiated before BPPs are registered. This means `@Autowired` on a BFPP field that requires another BFPP may fail. It also means `@Async`, `@Transactional`, and AOP do NOT apply to BFPP beans.

**Trap 10: Destruction of lazy beans**
If a lazy bean was NEVER requested (never created), its destroy callbacks are NEVER called (because it was never created — nothing to destroy). If it WAS requested (and created), it IS tracked as a singleton and WILL have its destroy callbacks called. Laziness only affects creation timing, not destruction (for beans that were actually created).

---

## ⑤ SUMMARY SHEET

### Complete Lifecycle Order (Memorize This)

```
CREATION:
 1.  BeanDefinition registered
 2.  BFPP runs (modifies BDs)
 3.  Constructor called
     → MergedBeanDefinition BPPs scan metadata
     → ObjectFactory registered in L3 cache (singleton)
 4.  populateBean()
     → IABPP.postProcessAfterInstantiation()
     → XML PropertyValues applied
     → IABPP.postProcessProperties() [@Autowired, @Resource via BPPs]
 5.  invokeAwareMethods()  ← DIRECTLY (not via BPP)
     → BeanNameAware
     → BeanClassLoaderAware
     → BeanFactoryAware
 6.  BPP.postProcessBeforeInitialization()  ← BPPs run
     → ApplicationContextAwareProcessor:
         EnvironmentAware, ResourceLoaderAware,
         ApplicationEventPublisherAware, MessageSourceAware,
         ApplicationContextAware
     → CommonAnnotationBPP: @PostConstruct called HERE
 7.  invokeInitMethods()
     → InitializingBean.afterPropertiesSet()
     → Custom init-method
 8.  BPP.postProcessAfterInitialization()
     → AOP PROXY CREATED HERE
     → @Async proxy, @Cacheable proxy, etc.
 9.  Bean (or proxy) stored in L1 singleton cache

DESTRUCTION (reverse creation order):
10. @PreDestroy  (via DestructionAwareBPP)
11. DisposableBean.destroy()
12. Custom destroy-method
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Init order | `@PostConstruct` → `afterPropertiesSet()` → `init-method` |
| Destroy order | `@PreDestroy` → `DisposableBean.destroy()` → custom `destroy-method` |
| AOP proxy creation | In `postProcessAfterInitialization()` — AFTER all init callbacks |
| `@PostConstruct` on raw bean | YES — proxy doesn't exist yet during `@PostConstruct` |
| Prototype `@PreDestroy` | NEVER called — Spring doesn't track prototypes |
| Default `@Bean` destroyMethod | `"(inferred)"` — auto-detects `close()` or `shutdown()` |
| `BeanFactoryAware` mechanism | Direct call in `invokeAwareMethods()` — NOT via BPP |
| `ApplicationContextAware` mechanism | Via `ApplicationContextAwareProcessor` BPP in Step 6 |
| BPP return null | No change — original bean used |
| BFPP beans + BPPs | BFPPs do NOT get full BPP processing (AOP, @Async etc. don't apply) |
| L1 cache population | AFTER Step 8 (`postProcessAfterInitialization`) completes |
| Destruction order | REVERSE of creation order |

---

### Initialization Mechanisms Comparison

| Mechanism | When | Processed By | Spring-specific? | Recommended For |
|---|---|---|---|---|
| `@PostConstruct` | Step 6 (BPP-before) | `CommonAnnotationBPP` | No (JSR-250) | Application code |
| `afterPropertiesSet()` | Step 7 | Direct invocation | Yes | Framework/library code |
| `init-method` | Step 7 | Direct invocation | Yes | Third-party classes |

### Destruction Mechanisms Comparison

| Mechanism | When | Processed By | Prototype? |
|---|---|---|---|
| `@PreDestroy` | Step 10 | `DestructionAwareBPP` | NEVER |
| `DisposableBean.destroy()` | Step 11 | Direct invocation | NEVER |
| `destroy-method` | Step 12 | Direct invocation | NEVER |

---

### Aware Interface Invocation Mechanism

```
Handled by invokeAwareMethods() DIRECTLY (before BPP step):
    BeanNameAware              → setBeanName(String)
    BeanClassLoaderAware       → setBeanClassLoader(ClassLoader)
    BeanFactoryAware           → setBeanFactory(BeanFactory)

Handled by ApplicationContextAwareProcessor BPP (in postProcessBeforeInitialization):
    EnvironmentAware           → setEnvironment(Environment)
    EmbeddedValueResolverAware → setEmbeddedValueResolver(StringValueResolver)
    ResourceLoaderAware        → setResourceLoader(ResourceLoader)
    ApplicationEventPublisherAware → setApplicationEventPublisher(AEP)
    MessageSourceAware         → setMessageSource(MessageSource)
    ApplicationContextAware    → setApplicationContext(ApplicationContext)
```

---

### Interview One-Liners

- "The complete initialization order is: constructor → DI → BeanNameAware/BeanFactoryAware → ApplicationContextAware → `@PostConstruct` → `afterPropertiesSet()` → `init-method` → AOP proxy creation."
- "AOP proxies are created in `postProcessAfterInitialization()` — AFTER all initialization callbacks complete on the RAW bean. `this` in `@PostConstruct` is always the raw bean."
- "`@PostConstruct` runs BEFORE `afterPropertiesSet()` — it is processed by `CommonAnnotationBeanPostProcessor` in `postProcessBeforeInitialization()`, which runs before `invokeInitMethods()`."
- "Spring never calls destroy callbacks on prototype beans — it doesn't track them after creation. Only the developer can manage prototype lifecycle."
- "The default `destroyMethod` for `@Bean` methods is `'(inferred)'` — Spring auto-detects and calls `close()` or `shutdown()`. Set `destroyMethod=\"\"` to suppress this."
- "`BeanFactoryAware` is invoked directly by `invokeAwareMethods()`; `ApplicationContextAware` is invoked by `ApplicationContextAwareProcessor` BPP — they use different mechanisms."
- "BFPPs themselves do NOT receive full BPP processing — AOP, `@Async`, and `@Transactional` do not apply to BFPP beans."
- "Destruction order is the exact reverse of creation order — the last bean created is the first bean destroyed, ensuring dependencies outlive their dependents."

---
