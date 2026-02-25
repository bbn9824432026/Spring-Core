# TOPIC 1.3 — ApplicationContext: Deep Dive

---

## ① CONCEPTUAL EXPLANATION

### What is ApplicationContext?

`ApplicationContext` is the **central interface of the Spring framework** for providing configuration to an application. It extends `BeanFactory` and adds enterprise-specific functionality that makes it the container you use in virtually every real Spring application.

The interface is defined at:
```
org.springframework.context.ApplicationContext
```

But the critical insight is this: `ApplicationContext` is not *just* an enhanced BeanFactory. It is a **composition of multiple capabilities** fused into a single interface:

```java
public interface ApplicationContext extends
    EnvironmentCapable,        // access to Environment (profiles, properties)
    ListableBeanFactory,       // bulk bean queries
    HierarchicalBeanFactory,   // parent-child hierarchy
    MessageSource,             // i18n message resolution
    ApplicationEventPublisher, // publishing events
    ResourcePatternResolver    // resolving resource patterns (classpath*, etc.)
{
    String getId();
    String getApplicationName();
    String getDisplayName();
    long getStartupDate();
    ApplicationContext getParent();
    AutowireCapableBeanFactory getAutowireCapableBeanFactory();
}
```

Every ApplicationContext is simultaneously a BeanFactory, a MessageSource, an EventPublisher, and a ResourceLoader. This is achieved through a combination of **delegation** (to an internal `DefaultListableBeanFactory`) and **direct implementation**.

---

### ApplicationContext Interface Hierarchy — Complete

```
BeanFactory
└── ListableBeanFactory
└── HierarchicalBeanFactory
        └── ApplicationContext
                ├── ConfigurableApplicationContext       ← adds lifecycle control
                │       └── AbstractApplicationContext   ← template method implementation
                │               ├── AbstractRefreshableApplicationContext
                │               │       ├── AbstractXmlApplicationContext
                │               │       │       ├── ClassPathXmlApplicationContext
                │               │       │       └── FileSystemXmlApplicationContext
                │               │       └── AbstractRefreshableWebApplicationContext
                │               │               ├── XmlWebApplicationContext
                │               │               └── AnnotationConfigWebApplicationContext
                │               └── GenericApplicationContext
                │                       └── AnnotationConfigApplicationContext
                │
                └── WebApplicationContext                ← adds getServletContext()
                        └── ConfigurableWebApplicationContext
```

The key design is the **Template Method Pattern** in `AbstractApplicationContext`. The `refresh()` method is defined there as a fixed algorithm, and subclasses provide implementations of abstract methods like `loadBeanDefinitions()`.

---

### ConfigurableApplicationContext — The Lifecycle Interface

`ConfigurableApplicationContext` is what you need when you want to *control* the container, not just *use* it:

```java
public interface ConfigurableApplicationContext
        extends ApplicationContext, Lifecycle, Closeable {

    void setParent(ApplicationContext parent);
    void setEnvironment(ConfigurableEnvironment environment);
    ConfigurableEnvironment getEnvironment();

    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);
    void addApplicationListener(ApplicationListener<?> listener);

    void refresh() throws BeansException, IllegalStateException;
    void registerShutdownHook();

    void close();
    boolean isActive();
    ConfigurableListableBeanFactory getBeanFactory();
}
```

`ConfigurableApplicationContext` also extends `Lifecycle`:
```java
public interface Lifecycle {
    void start();
    void stop();
    boolean isRunning();
}
```

This means you can `start()` and `stop()` the context, which triggers `LifecycleProcessor` callbacks on beans that implement `Lifecycle` or `SmartLifecycle`. This is distinct from `refresh()` and `close()`.

---

### The `refresh()` Method — Heart of the Container

This is the single most important method in the entire Spring framework. Every time an `ApplicationContext` is initialized, `refresh()` is called. Understanding its internal steps at the source code level is essential.

Defined in `AbstractApplicationContext.refresh()`:

```
refresh()
│
├── 1. prepareRefresh()
│       - Set start date and active flag
│       - Initialize property sources (subclass hook)
│       - Validate required properties
│       - Initialize early ApplicationListeners set
│
├── 2. obtainFreshBeanFactory()
│       - For refreshable contexts: destroy existing BF, create new one, load BeanDefinitions
│       - For GenericApplicationContext: just returns the existing internal BF
│       - Returns: ConfigurableListableBeanFactory
│
├── 3. prepareBeanFactory(beanFactory)
│       - Set ClassLoader
│       - Set BeanExpressionResolver (SpEL support)
│       - Add ResourceEditorRegistrar (PropertyEditor support)
│       - Add ApplicationContextAwareProcessor (BPP for Aware interfaces)
│       - Register ignorable dependency interfaces (BeanNameAware, etc.)
│       - Register resolvable dependencies (BeanFactory, ApplicationContext, etc.)
│       - Add ApplicationListenerDetector (BPP)
│       - Register default environment beans
│
├── 4. postProcessBeanFactory(beanFactory)
│       - Empty hook for subclasses
│       - Web contexts use this to register web-specific scopes (request, session)
│       - Register ServletContextAwareProcessor
│
├── 5. invokeBeanFactoryPostProcessors(beanFactory)
│       ← THIS IS WHERE BeanDefinitionRegistryPostProcessors run first
│       ← THEN BeanFactoryPostProcessors run
│       ← ConfigurationClassPostProcessor runs here (processes @Configuration classes)
│       ← PropertySourcesPlaceholderConfigurer runs here
│
├── 6. registerBeanPostProcessors(beanFactory)
│       - Find all BPP bean definitions
│       - Instantiate them (in order: PriorityOrdered → Ordered → rest)
│       - Register them in the BeanFactory
│       ← AutowiredAnnotationBeanPostProcessor registered here
│       ← CommonAnnotationBeanPostProcessor registered here
│
├── 7. initMessageSource()
│       - Look for bean named "messageSource"
│       - If not found, create DelegatingMessageSource
│
├── 8. initApplicationEventMulticaster()
│       - Look for bean named "applicationEventMulticaster"
│       - If not found, create SimpleApplicationEventMulticaster
│
├── 9. onRefresh()
│       - Empty hook for subclasses
│       - Web contexts create the embedded web server here (Spring Boot)
│
├── 10. registerListeners()
│        - Register statically specified ApplicationListeners
│        - Register ApplicationListener beans
│        - Publish early application events
│
├── 11. finishBeanFactoryInitialization(beanFactory)
│        ← ALL NON-LAZY SINGLETON BEANS ARE INSTANTIATED HERE
│        - Set ConversionService
│        - Freeze configuration
│        - beanFactory.preInstantiateSingletons()
│              For each bean name:
│              - Is it a FactoryBean? → instantiate FactoryBean, then getObject() if not lazy
│              - Is it non-abstract, singleton, non-lazy? → getBean(name)
│              - SmartInitializingSingleton.afterSingletonsInstantiated() called after all
│
└── 12. finishRefresh()
         - Clear resource caches
         - Initialize LifecycleProcessor
         - Call lifecycleProcessor.onRefresh() → starts Lifecycle beans
         - Publish ContextRefreshedEvent
         - Register with JMX if needed
```

**Step 5 (invokeBeanFactoryPostProcessors) and Step 11 (finishBeanFactoryInitialization) are the most critical to memorize for exams and interviews.**

---

### Eager Initialization — What Actually Happens in Step 11

In `preInstantiateSingletons()`, Spring iterates `beanDefinitionNames` in registration order and for each bean checks:

```
Is abstract?       → skip
Is singleton?      → continue
Is lazy (@Lazy)?   → skip (will be created on first getBean())
Is FactoryBean?    → instantiate FactoryBean; if SmartFactoryBean.isEagerInit() → getObject()
Otherwise          → getBean(beanName)  → triggers full instantiation + wiring + lifecycle
```

After ALL singletons are created, Spring iterates again looking for beans that implement `SmartInitializingSingleton` and calls `afterSingletonsInstantiated()` on them. This is the hook used by framework components that need to do post-processing after the entire singleton graph is ready (e.g., `@EventListener` method registration happens here via `EventListenerMethodProcessor`).

---

### The `close()` Method — Destruction Sequence

When `context.close()` is called (or the JVM shutdown hook fires):

```
close()
│
├── doClose()
│       ├── Publish ContextClosedEvent
│       ├── lifecycleProcessor.onClose()  → stops Lifecycle beans
│       └── destroyBeans()
│               For each singleton (in reverse registration order):
│               ├── @PreDestroy methods
│               ├── DisposableBean.destroy()
│               └── custom destroy-method
│
└── closeBeanFactory()  → clears internal BF
```

**Reverse order is critical:** beans are destroyed in the reverse of their creation order. If bean A depends on bean B, A was created after B, and A is destroyed before B. This ensures that a bean's dependencies are still alive during its own destruction.

---

### ApplicationContext Implementations — Deep Comparison

#### `ClassPathXmlApplicationContext`

```java
ApplicationContext ctx =
    new ClassPathXmlApplicationContext("spring/beans.xml");

// Multiple files:
ApplicationContext ctx =
    new ClassPathXmlApplicationContext(
        "spring/services.xml",
        "spring/repositories.xml"
    );
```

- Loads XML from classpath
- Extends `AbstractRefreshableApplicationContext` — supports `refresh()` that creates a new BeanFactory
- BeanDefinitions loaded via `XmlBeanDefinitionReader`

#### `FileSystemXmlApplicationContext`

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("/opt/app/config/beans.xml");
```

- Same as above but resolves paths from the filesystem, not classpath
- Relative paths resolved from current working directory

#### `AnnotationConfigApplicationContext`

```java
// From @Configuration class(es):
ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// From package scan:
ApplicationContext ctx =
    new AnnotationConfigApplicationContext("com.example");

// Programmatic (no-arg constructor):
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext();
ctx.register(AppConfig.class, DataConfig.class);
ctx.scan("com.example.services");
ctx.refresh();
```

- Extends `GenericApplicationContext` — BeanFactory is created once and NOT recreated on refresh (unlike refreshable contexts)
- Uses `AnnotatedBeanDefinitionReader` for `@Configuration` classes
- Uses `ClassPathBeanDefinitionScanner` for package scanning
- **Most important: cannot be refreshed multiple times** (GenericApplicationContext throws exception on second refresh)

#### `AbstractRefreshableApplicationContext` vs `GenericApplicationContext`

| | `AbstractRefreshableApplicationContext` | `GenericApplicationContext` |
|---|---|---|
| Used by | `ClassPathXmlApplicationContext` | `AnnotationConfigApplicationContext` |
| BeanFactory lifecycle | New BF created on each `refresh()` | Single BF, never recreated |
| Multiple `refresh()` | Supported | Throws `IllegalStateException` |
| Use case | Legacy/XML, test contexts needing re-fresh | Modern annotation-based |

---

### `prepareBeanFactory()` — What Gets Registered Automatically

This step explains why certain injections work "for free" in an ApplicationContext:

**Resolvable dependencies registered** (injectable without a bean definition):
```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

This is why you can `@Autowired ApplicationContext context` in any bean — the ApplicationContext itself is registered as a resolvable dependency, not as a regular bean definition.

**BPPs registered at this stage:**
- `ApplicationContextAwareProcessor` — handles `EnvironmentAware`, `EmbeddedValueResolverAware`, `ResourceLoaderAware`, `ApplicationEventPublisherAware`, `MessageSourceAware`, `ApplicationContextAware`
- `ApplicationListenerDetector` — detects beans that implement `ApplicationListener` and auto-registers them

**Interfaces ignored for autowiring** (because they're handled by Aware callbacks instead):
```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

This prevents Spring from trying to autowire these interfaces via normal DI — instead they're satisfied via the Aware callback mechanism.

---

### WebApplicationContext

`WebApplicationContext` adds web-specific capabilities:

```java
public interface WebApplicationContext extends ApplicationContext {
    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE =
        WebApplicationContext.class.getName() + ".ROOT";

    ServletContext getServletContext();
}
```

In a Spring MVC application, there are typically **two ApplicationContext instances**:

```
Root WebApplicationContext (parent)
    └── Created by ContextLoaderListener
    └── Contains: @Service, @Repository, DataSource, TransactionManager, etc.
    └── Stored in ServletContext as attribute

Child WebApplicationContext (per DispatcherServlet)
    └── Created by DispatcherServlet
    └── Contains: @Controller, ViewResolver, HandlerMapping, etc.
    └── Can see all beans from Root context (parent)
    └── Root context CANNOT see child beans
```

This hierarchy exists so that multiple DispatcherServlets can share the same root context (shared services) while each having their own web-layer beans.

---

### MessageSource Integration

Because `ApplicationContext` implements `MessageSource`, you can resolve internationalized messages directly from the context:

```java
String msg = context.getMessage("greeting", new Object[]{"World"}, Locale.ENGLISH);
```

Spring looks for a bean named exactly `"messageSource"` of type `MessageSource`. If not found, a `DelegatingMessageSource` is used that delegates to the parent context (if any). We cover this fully in Topic 16.

---

### ApplicationEvent Publishing

Because `ApplicationContext` implements `ApplicationEventPublisher`:

```java
context.publishEvent(new OrderPlacedEvent(this, order));
```

Internally, events are dispatched via `SimpleApplicationEventMulticaster` (unless you configure an async one). All beans implementing `ApplicationListener<E>` receive matching events. We cover this fully in Topic 15.

---

### Resource Loading

Because `ApplicationContext` implements `ResourcePatternResolver`:

```java
Resource resource = context.getResource("classpath:config/settings.properties");
Resource[] resources = context.getResources("classpath*:config/*.xml");
```

The `classpath*:` prefix is special — it searches ALL JARs on the classpath, not just the first match. This is critical for modular applications. We cover this fully in Topic 10.

---

### Memory Implications

Because `ApplicationContext` eagerly instantiates all singletons:
- **More memory used at startup** vs BeanFactory
- **Startup time is longer** (all beans created, all dependencies wired, all lifecycle callbacks run)
- **Runtime requests are faster** — no instantiation cost on `getBean()`
- **Configuration errors surface at startup** — fail-fast behavior

In Spring Boot applications with hundreds of auto-configurations, startup time and memory pressure are active engineering concerns. This is why Spring Boot 3.x introduced **AOT (Ahead-of-Time) compilation** — to pre-compute bean definitions and proxy classes at build time, reducing the reflection-heavy startup overhead.

---

### Common Misconceptions

**Misconception 1:** "You can call `refresh()` multiple times on `AnnotationConfigApplicationContext`."
Wrong. `AnnotationConfigApplicationContext` extends `GenericApplicationContext`, which throws `IllegalStateException` on a second `refresh()`.

**Misconception 2:** "The ApplicationContext IS the BeanFactory."
Technically incorrect at the object level. ApplicationContext *wraps* / *delegates to* a `DefaultListableBeanFactory`. They are separate objects. `context.getBeanFactory()` returns the internal factory.

**Misconception 3:** "`@Autowired ApplicationContext ctx` requires a bean definition for ApplicationContext."
Wrong. `ApplicationContext` is registered as a **resolvable dependency** in `prepareBeanFactory()`, not as a bean definition. `context.containsBeanDefinition("applicationContext")` returns false, but `@Autowired ApplicationContext` still works.

**Misconception 4:** "All beans in the context are destroyed in registration order."
Wrong. Beans are destroyed in **reverse** registration order (reverse creation order). This ensures dependency-correct teardown.

---

## ② CODE EXAMPLES

### Example 1 — Full ApplicationContext Bootstrap Styles

```java
// Style 1: XML
ApplicationContext ctx1 =
    new ClassPathXmlApplicationContext("beans.xml");

// Style 2: Java Config
ApplicationContext ctx2 =
    new AnnotationConfigApplicationContext(AppConfig.class);

// Style 3: Package scan
ApplicationContext ctx3 =
    new AnnotationConfigApplicationContext("com.example");

// Style 4: Programmatic with multiple sources
AnnotationConfigApplicationContext ctx4 =
    new AnnotationConfigApplicationContext();
ctx4.getEnvironment().setActiveProfiles("production");  // must be BEFORE refresh
ctx4.register(AppConfig.class, InfraConfig.class);
ctx4.scan("com.example.plugins");
ctx4.refresh();
```

**Critical:** Profile activation MUST happen before `refresh()`. After `refresh()`, profiles are locked.

---

### Example 2 — Shutdown Hook and Lifecycle

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// Register JVM shutdown hook — ensures destroy callbacks fire on Ctrl+C or System.exit()
ctx.registerShutdownHook();

// Or manually:
Runtime.getRuntime().addShutdownHook(new Thread(ctx::close));

// Explicit close (blocks until destruction complete):
ctx.close();

// Check if context is still active:
boolean active = ctx.isActive(); // false after close()
```

---

### Example 3 — Accessing the Internal BeanFactory

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// Get the internal DefaultListableBeanFactory
ConfigurableListableBeanFactory beanFactory = ctx.getBeanFactory();

// Programmatically register a BPP after context creation (unusual but possible):
beanFactory.addBeanPostProcessor(new CustomAuditBeanPostProcessor());

// Get raw BeanDefinition metadata:
BeanDefinition bd = beanFactory.getBeanDefinition("orderService");
System.out.println(bd.getScope());        // "singleton"
System.out.println(bd.getBeanClassName()); // "com.example.OrderService"
System.out.println(bd.isLazyInit());       // false
```

---

### Example 4 — Parent-Child ApplicationContext

```java
// Root context (shared services)
AnnotationConfigApplicationContext parent =
    new AnnotationConfigApplicationContext(ServiceConfig.class);

// Child context (web layer)
AnnotationConfigApplicationContext child =
    new AnnotationConfigApplicationContext();
child.setParent(parent);
child.register(WebConfig.class);
child.refresh();

// Child can see parent beans:
OrderService os = child.getBean(OrderService.class); // defined in parent

// Parent cannot see child beans:
try {
    child.getBean(OrderController.class); // works — defined in child
    parent.getBean(OrderController.class); // throws NoSuchBeanDefinitionException
} catch (NoSuchBeanDefinitionException e) {
    System.out.println("Parent cannot see child beans");
}
```

---

### Example 5 — `@Autowired ApplicationContext` (Resolvable Dependency)

```java
@Service
public class DynamicBeanService {

    @Autowired
    private ApplicationContext context; // Works — registered as resolvable dependency

    @Autowired
    private BeanFactory beanFactory; // Also works

    @Autowired
    private ResourceLoader resourceLoader; // Also works

    @Autowired
    private ApplicationEventPublisher eventPublisher; // Also works

    public void doWork() {
        // Get a prototype bean dynamically:
        ReportGenerator gen = context.getBean(ReportGenerator.class);
    }
}
```

All four injections above work without any explicit `@Bean` definition for them. They are registered in `prepareBeanFactory()`.

---

### Example 6 — SmartInitializingSingleton (Post-initialization Hook)

```java
@Component
public class CacheWarmer implements SmartInitializingSingleton {

    @Autowired
    private ProductRepository productRepository;

    @Override
    public void afterSingletonsInstantiated() {
        // Called AFTER all singletons are fully initialized
        // Safe to use all autowired dependencies here
        System.out.println("Warming up cache...");
        productRepository.findAll().forEach(CacheManager::put);
    }
}
```

This is called at the very end of `preInstantiateSingletons()`, after every singleton is fully initialized. This is different from `@PostConstruct`, which is called during each individual bean's initialization, before other beans are guaranteed to be ready.

---

### Example 7 — Context Refresh on Refreshable Contexts

```java
// ClassPathXmlApplicationContext IS refreshable
ClassPathXmlApplicationContext ctx =
    new ClassPathXmlApplicationContext("beans.xml");

// Modify beans.xml externally...

ctx.refresh(); // Destroys all beans, reloads XML, recreates everything
// This is valid for ClassPathXmlApplicationContext
```

```java
// AnnotationConfigApplicationContext is NOT refreshable
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

ctx.refresh(); // throws IllegalStateException — GenericApplicationContext cannot be refreshed twice
```

---

### Example 8 — Detecting Configuration Errors Early (ApplicationContext advantage)

```java
// Bad bean definition — missing required dependency
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// PaymentService NOT defined in context

// With ApplicationContext:
// → Throws NoSuchBeanDefinitionException AT STARTUP (during refresh())
// → Fail-fast behavior

// With plain BeanFactory:
// → No error at startup
// → Throws exception only when getBean("orderService") is called at runtime
// → Could be in production, under specific user flow
```

---

### Example 9 — Multiple XML Files with `<import>`

```xml
<!-- main-context.xml -->
<beans>
    <import resource="services.xml"/>
    <import resource="repositories.xml"/>
    <import resource="classpath:infra/datasource.xml"/>

    <bean id="orderFacade" class="com.example.OrderFacade"/>
</beans>
```

```java
ApplicationContext ctx =
    new ClassPathXmlApplicationContext("main-context.xml");
// All imported files are processed as one unified context
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which interface must you cast `ApplicationContext` to in order to call `refresh()` and `registerShutdownHook()`?

A) `ListableBeanFactory`
B) `ConfigurableApplicationContext`
C) `HierarchicalBeanFactory`
D) `AutowireCapableBeanFactory`

**Answer: B**

---

**Q2 — MCQ:**
At which step in `AbstractApplicationContext.refresh()` are all non-lazy singleton beans instantiated?

A) `prepareBeanFactory()`
B) `invokeBeanFactoryPostProcessors()`
C) `registerBeanPostProcessors()`
D) `finishBeanFactoryInitialization()`

**Answer: D**

---

**Q3 — Select all that apply:**
Which of the following can be injected via `@Autowired` in any Spring-managed bean WITHOUT defining a `@Bean` method for them? (Select ALL that apply)

A) `ApplicationContext`
B) `BeanFactory`
C) `ResourceLoader`
D) `ApplicationEventPublisher`
E) `TransactionManager`
F) `Environment`

**Answer: A, B, C, D**
`TransactionManager` needs a bean definition. `Environment` requires `@Autowired` + bean definition or `EnvironmentAware`. (Note: `Environment` IS accessible via `@Autowired` because it's also registered — so F could be argued, but the cleanest exam answer is A, B, C, D as these are explicitly registered in `prepareBeanFactory()`.)

---

**Q4 — Code Output Prediction:**

```java
@Configuration
public class AppConfig {
    @Bean
    public BeanA beanA() {
        System.out.println("Creating BeanA");
        return new BeanA();
    }

    @Bean
    public BeanB beanB() {
        System.out.println("Creating BeanB");
        return new BeanB();
    }
}

System.out.println("Before context");
ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
System.out.println("After context");
ctx.getBean(BeanA.class);
System.out.println("After getBean");
```

What is the output (assuming BeanA registered before BeanB)?

A)
```
Before context
After context
Creating BeanA
After getBean
```
B)
```
Before context
Creating BeanA
Creating BeanB
After context
After getBean
```
C)
```
Creating BeanA
Creating BeanB
Before context
After context
After getBean
```
D) Depends on JVM

**Answer: B**
Both singletons are created eagerly inside the `AnnotationConfigApplicationContext` constructor (which calls `refresh()`), before control returns to the calling code. `getBean()` on a singleton just returns the cached instance — no "Creating" output.

---

**Q5 — Trick Question:**

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
ctx.refresh(); // second refresh
```

What happens?

A) Context is re-initialized with fresh beans
B) `IllegalStateException` is thrown
C) Nothing — it's a no-op
D) Context publishes `ContextRefreshedEvent` again

**Answer: B**
`GenericApplicationContext` (parent of `AnnotationConfigApplicationContext`) throws `IllegalStateException` on a second call to `refresh()`.

---

**Q6 — Scenario-based:**
A developer defines a bean named `"messageSource"` of type `ResourceBundleMessageSource` and also calls `context.getMessage(...)`. What happens?

A) Spring ignores the custom bean and uses its default `DelegatingMessageSource`
B) Spring uses the custom `messageSource` bean for message resolution
C) Spring throws `NoUniqueBeanDefinitionException` because there are now two MessageSources
D) `UnsupportedOperationException` is thrown

**Answer: B**
In `initMessageSource()`, Spring checks for a bean named exactly `"messageSource"`. If found, it uses it. If not found, it creates a `DelegatingMessageSource`. The name `"messageSource"` is a **magic bean name** in Spring.

---

**Q7 — Select all that apply:**
Which of the following are "magic bean names" that Spring's ApplicationContext looks for by name during initialization? (Select ALL that apply)

A) `messageSource`
B) `applicationEventMulticaster`
C) `lifecycleProcessor`
D) `beanFactory`
E) `conversionService`

**Answer: A, B, C, E**
`"beanFactory"` is not a magic name — the internal BeanFactory is not looked up by name.

---

**Q8 — Compilation/Runtime Error Identification:**

```java
ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// Attempt to set active profile:
ctx.getEnvironment().setActiveProfiles("prod"); // Line X

ctx.refresh(); // This won't be called — ctx is already refreshed
```

What is the problem with this code?

A) Line X causes a compile error — `getEnvironment()` not on `ApplicationContext`
B) Line X throws `IllegalStateException` — context is already active, profiles cannot be changed
C) The profile is set but silently ignored
D) Works correctly — profiles can be changed at any time

**Answer: B**
After `refresh()` completes, the context is active (`isActive() == true`). Attempting to set profiles on an active context throws `IllegalStateException`. Profiles must be configured BEFORE `refresh()`.

---

**Q9 — Scenario: SmartInitializingSingleton vs @PostConstruct**

A bean `CacheWarmer` implements `SmartInitializingSingleton` and its `afterSingletonsInstantiated()` tries to use a bean `ProductService`. Another bean `ReportService` has a `@PostConstruct` method. Which executes first?

A) `CacheWarmer.afterSingletonsInstantiated()` then `ReportService @PostConstruct`
B) `ReportService @PostConstruct` then `CacheWarmer.afterSingletonsInstantiated()`
C) Both run concurrently
D) Depends on bean registration order

**Answer: B**
`@PostConstruct` runs during each individual bean's initialization phase (step 11 per-bean). `SmartInitializingSingleton.afterSingletonsInstantiated()` runs after ALL singletons are fully initialized. So `@PostConstruct` on ALL beans runs before any `afterSingletonsInstantiated()`.

---

**Q10 — Drag-and-Drop Style:**
Order these `refresh()` steps correctly (1 = first):

- `finishBeanFactoryInitialization()` — singletons instantiated
- `registerBeanPostProcessors()`
- `invokeBeanFactoryPostProcessors()`
- `prepareRefresh()`
- `finishRefresh()` — ContextRefreshedEvent published
- `prepareBeanFactory()`

**Answer:**
1. `prepareRefresh()`
2. `prepareBeanFactory()`
3. `invokeBeanFactoryPostProcessors()`
4. `registerBeanPostProcessors()`
5. `finishBeanFactoryInitialization()`
6. `finishRefresh()`

---

## ④ TRICK ANALYSIS

**Trap 1: `AnnotationConfigApplicationContext` cannot be refreshed twice**
Any question showing a second `ctx.refresh()` call on an `AnnotationConfigApplicationContext` is a trap. It always throws `IllegalStateException`. `ClassPathXmlApplicationContext` CAN be refreshed. Knowing WHICH contexts support multiple refreshes is the differentiator.

**Trap 2: Profile must be set BEFORE refresh()**
A very common exam scenario: developer creates context, then sets profile, then tries to use it. The profile set after `refresh()` has no effect (or throws exception). Any "set profile after context creation" code → wrong answer.

**Trap 3: Magic bean names**
`"messageSource"`, `"applicationEventMulticaster"`, `"lifecycleProcessor"`, `"conversionService"` are looked up by name. If a question asks what happens when you define a bean with one of these names, the answer is always "Spring uses it." This is a magic name contract in Spring.

**Trap 4: `@Autowired ApplicationContext` vs bean definition**
`containsBeanDefinition("applicationContext")` returns FALSE. But `@Autowired ApplicationContext` WORKS. These are resolvable dependencies, not bean definitions. Exam questions may ask "how many beans of type ApplicationContext exist in the context?" — the answer is 0 (from bean definitions), but injection still works.

**Trap 5: `SmartInitializingSingleton` vs `@PostConstruct` ordering**
`@PostConstruct` runs per-bean during initialization. `afterSingletonsInstantiated()` runs once after ALL singletons are ready. If code in `afterSingletonsInstantiated()` needs other beans to be ready — it's safe because they all are. `@PostConstruct` does NOT have this guarantee for OTHER beans.

**Trap 6: Destroy order**
Destroy callbacks run in **reverse** creation order. If a question asks "bean A depends on bean B — which is destroyed first?" — A is destroyed first (because A was created after B, so it's destroyed before B). This ensures A's destroy callback can still use B safely.

**Trap 7: `getBeanFactory()` vs `context` itself**
`context.getBeanFactory()` returns the internal `ConfigurableListableBeanFactory`. Operations like `getBeanDefinition()`, `addBeanPostProcessor()` are on the factory, not on the context directly. Exam code calling `context.getBeanDefinition()` → compile error.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `refresh()` sequence | prepareRefresh → prepareBF → invokeBFPPs → registerBPPs → initMsgSrc → initEventMulticaster → onRefresh → registerListeners → finishBFInit → finishRefresh |
| Singleton instantiation step | `finishBeanFactoryInitialization()` → `preInstantiateSingletons()` |
| `SmartInitializingSingleton` | Called AFTER all singletons ready — safest post-init hook |
| Profile activation | MUST happen before `refresh()` |
| Second `refresh()` | OK for `ClassPathXmlApplicationContext`; throws `IllegalStateException` for `AnnotationConfigApplicationContext` |
| Magic bean names | `messageSource`, `applicationEventMulticaster`, `lifecycleProcessor`, `conversionService` |
| Resolvable dependencies | `ApplicationContext`, `BeanFactory`, `ResourceLoader`, `ApplicationEventPublisher` — injectable without bean definitions |
| Destroy order | **Reverse** of creation order |
| `ContextRefreshedEvent` | Published in `finishRefresh()` — last step of `refresh()` |

---

### refresh() Step Order (Memorize the Numbers)

```
1.  prepareRefresh()                    ← flags, early events
2.  obtainFreshBeanFactory()            ← get/create BeanFactory
3.  prepareBeanFactory()                ← ClassLoader, BPPs, resolvable deps
4.  postProcessBeanFactory()            ← subclass hook (web scopes here)
5.  invokeBeanFactoryPostProcessors()   ← BFPPs + ConfigurationClassPostProcessor
6.  registerBeanPostProcessors()        ← register BPPs (but don't run them yet)
7.  initMessageSource()                 ← "messageSource" magic name
8.  initApplicationEventMulticaster()   ← "applicationEventMulticaster" magic name
9.  onRefresh()                         ← subclass hook (web server starts in Boot)
10. registerListeners()                 ← ApplicationListener beans
11. finishBeanFactoryInitialization()   ← ALL SINGLETONS CREATED HERE
12. finishRefresh()                     ← ContextRefreshedEvent published
```

---

### ApplicationContext vs BeanFactory — Extended Exam Table

| Aspect | BeanFactory | ApplicationContext |
|---|---|---|
| Instantiation | Lazy | Eager (singletons) |
| BPP auto-detection | ❌ | ✅ |
| BFPP auto-detection | ❌ | ✅ |
| MessageSource | ❌ | ✅ |
| Events | ❌ | ✅ |
| ResourcePatternResolver | ❌ | ✅ |
| Environment/Profiles | ❌ | ✅ |
| Fail-fast config validation | ❌ | ✅ |
| Multiple refresh() | N/A | Depends on impl |
| Magic bean names | ❌ | ✅ |
| Resolvable deps injection | ❌ | ✅ |

---

### Interview One-Liners

- "`AbstractApplicationContext.refresh()` is the single most important method in Spring — it orchestrates the entire container initialization in 12 ordered steps."
- "`finishBeanFactoryInitialization()` is where all non-lazy singleton beans are created — this is the critical step that makes ApplicationContext fail-fast."
- "`SmartInitializingSingleton.afterSingletonsInstantiated()` is the only hook guaranteed to run after the ENTIRE singleton graph is ready."
- "Profile activation must happen before `refresh()` — setting profiles after refresh has no effect."
- "`AnnotationConfigApplicationContext` cannot be refreshed twice — it extends `GenericApplicationContext`."
- "Beans are destroyed in reverse creation order to ensure dependency-correct teardown."
- "`ApplicationContext`, `BeanFactory`, `ResourceLoader`, and `ApplicationEventPublisher` are injectable via `@Autowired` because they are resolvable dependencies, not bean definitions."
- "Magic bean names: `messageSource`, `applicationEventMulticaster`, `lifecycleProcessor`, `conversionService` — Spring looks for these by name during `refresh()`."

---

