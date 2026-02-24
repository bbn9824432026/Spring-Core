# TOPIC 1.2 — BeanFactory: Deep Dive

---

## ① CONCEPTUAL EXPLANATION

### What is BeanFactory?

`BeanFactory` is the **root interface of the Spring IoC container**. It is defined in the package `org.springframework.beans.factory` and represents the most fundamental contract a Spring container must fulfill — the ability to hold bean definitions and return bean instances on demand.

Every Spring container, including `ApplicationContext`, is ultimately a `BeanFactory`. The entire Spring DI infrastructure is built on top of this interface.

```
org.springframework.beans.factory.BeanFactory  ← root interface
```

The interface itself is intentionally minimal. It defines only the essential operations for retrieving beans and querying bean metadata.

---

### BeanFactory Interface Contract

The full method signature of `BeanFactory`:

```java
public interface BeanFactory {

    String FACTORY_BEAN_PREFIX = "&"; // Used to dereference FactoryBean itself

    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

    String[] getAliases(String name);
}
```

Every `getBean()` call is the fundamental operation. When you call `getBean("orderService")`, the container must:
1. Look up the `BeanDefinition` by name
2. Determine if an instance already exists (singleton cache)
3. If not, instantiate it, inject dependencies, run lifecycle callbacks
4. Return the fully initialized instance

---

### DefaultListableBeanFactory — The Real Implementation

In practice, you never work with the `BeanFactory` interface directly at the implementation level. The concrete class that does all the actual work is:

```
org.springframework.beans.factory.support.DefaultListableBeanFactory
```

This class implements an enormous number of interfaces:

```
BeanFactory
    └── HierarchicalBeanFactory       ← supports parent-child container hierarchy
    └── ListableBeanFactory           ← adds bulk query methods (getBeansOfType, getBeanDefinitionNames)
            └── ConfigurableBeanFactory       ← adds configuration methods (addBPP, setParentBeanFactory, etc.)
                    └── ConfigurableListableBeanFactory  ← combined
                            └── BeanDefinitionRegistry   ← registers/removes BeanDefinitions
                                    └── DefaultListableBeanFactory  ← THE REAL IMPLEMENTATION
```

`DefaultListableBeanFactory` is the workhorse. It is used internally by every `ApplicationContext` implementation. When you configure an `ApplicationContext`, you are ultimately populating a `DefaultListableBeanFactory` instance that sits inside it.

---

### Internal Data Structures of DefaultListableBeanFactory

Understanding the internal maps is critical for architect-level comprehension:

```java
// BeanDefinition storage
private final Map<String, BeanDefinition> beanDefinitionMap
    = new ConcurrentHashMap<>(256);

// Ordered list of bean definition names
private volatile List<String> beanDefinitionNames
    = new ArrayList<>(256);

// Manually registered singleton instances (bypasses BeanDefinition)
private final Map<String, Object> manualSingletonNames
    = new LinkedHashSet<>(16);

// Singleton object cache (Level 1 cache — fully initialized)
private final Map<String, Object> singletonObjects
    = new ConcurrentHashMap<>(256);      // in DefaultSingletonBeanRegistry (parent class)

// Early singleton references (Level 2 cache — instantiated but not fully initialized)
private final Map<String, Object> earlySingletonObjects
    = new ConcurrentHashMap<>(16);

// Singleton factories (Level 3 cache — ObjectFactory for early exposure)
private final Map<String, ObjectFactory<?>> singletonFactories
    = new HashMap<>(16);

// Track beans currently being created (circular dependency detection)
private final Set<String> singletonsCurrentlyInCreation
    = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

// Type to bean name mapping for type-based lookup
private final Map<Class<?>, String[]> allBeanNamesByType
    = new ConcurrentHashMap<>(64);
private final Map<Class<?>, Object> resolvableDependencies
    = new ConcurrentHashMap<>(16);
```

This is the **three-level singleton cache** that Spring uses. We will go extremely deep on this in Topic 3.3 (Circular Dependencies), but you must know these structures exist and why right from the beginning.

---

### BeanFactory Interface Hierarchy — All Sub-interfaces

```
BeanFactory
├── HierarchicalBeanFactory
│       getParentBeanFactory()
│       containsLocalBean(String name)
│
├── ListableBeanFactory
│       getBeanDefinitionNames()
│       getBeanNamesForType(Class<?> type)
│       getBeansOfType(Class<T> type)
│       getBeanNamesForAnnotation(Class<? extends Annotation>)
│       getBeansWithAnnotation(Class<? extends Annotation>)
│       findAnnotationOnBean(String beanName, Class<A> annotationType)
│
└── AutowireCapableBeanFactory
        createBean(Class<T> beanClass)
        autowireBean(Object existingBean)
        configureBean(Object existingBean, String beanName)
        autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
        resolveDependency(DependencyDescriptor, String requestingBeanName)
```

`AutowireCapableBeanFactory` is particularly important — it enables Spring to autowire beans that were NOT created by the Spring container (e.g., JPA entities, Struts actions in legacy apps). This is how `@Configurable` works with AspectJ LTW.

---

### Lazy Initialization — BeanFactory Default Behavior

In a plain `BeanFactory`, beans are **instantiated lazily by default**. The bean definition is loaded and registered immediately when the factory is set up, but the actual object is not created until `getBean()` is called.

This has a critical implication: **configuration errors are not detected at startup**. If a bean has a missing dependency or a misconfigured constructor, you only find out at runtime when that bean is first requested. This is a significant operational risk in production systems.

This is one of the primary reasons `ApplicationContext` is preferred — it eagerly instantiates all singletons at startup, causing all wiring errors to be surfaced immediately at application boot time, not in production at arbitrary runtime.

```
BeanFactory behavior:
startup          getBean("A")      getBean("B")
    |                |                  |
  [register]     [instantiate A]    [instantiate B]
                 [inject deps A]    [inject deps B]
                 [lifecycle A]      [lifecycle B]

ApplicationContext behavior:
startup                              getBean("A")
    |                                    |
  [register]                         [return from cache]
  [instantiate ALL singletons]
  [inject all deps]
  [all lifecycle callbacks]
```

---

### When Would You Use BeanFactory Directly?

Almost never in modern application development. But legitimate use cases include:

1. **OSGi environments** — where memory is extremely constrained and lazy loading is a hard requirement
2. **Embedded systems** — very limited JVM environments
3. **Framework internals** — when building a framework on top of Spring where you control the container lifecycle precisely
4. **Testing infrastructure** — `DefaultListableBeanFactory` is sometimes used directly in unit tests for very fast, lightweight container setup without the full ApplicationContext overhead

In enterprise production Spring applications: **always use ApplicationContext**.

---

### BeanFactory vs ApplicationContext — Feature Comparison

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Basic DI / bean lifecycle | ✅ | ✅ |
| Eager singleton initialization | ❌ (lazy) | ✅ |
| Auto-detect BeanPostProcessors | ❌ Manual only | ✅ |
| Auto-detect BeanFactoryPostProcessors | ❌ Manual only | ✅ |
| MessageSource (i18n) | ❌ | ✅ |
| ApplicationEvent publishing | ❌ | ✅ |
| ResourceLoader | ❌ (basic) | ✅ |
| Environment / Profiles | ❌ | ✅ |
| `@PostConstruct` / `@PreDestroy` processing | ❌ Manual | ✅ Auto |
| AOP auto-proxy creation | ❌ Manual | ✅ |
| `@Scheduled`, `@Async` support | ❌ | ✅ |

**The BPP auto-detection distinction is critical for exams.** With plain `BeanFactory`, if you define a `BeanPostProcessor` as a bean in the factory, it will NOT be applied automatically. You must call:

```java
factory.addBeanPostProcessor(new MyBeanPostProcessor());
```

With `ApplicationContext`, all `BeanPostProcessor` beans are detected, ordered, and registered automatically during `refresh()`.

---

### ObjectProvider — Modern BeanFactory Addition

Spring 4.3 introduced `ObjectProvider<T>` as a lazy, safer alternative to `getBean()`:

```java
ObjectProvider<PaymentService> provider =
    context.getBeanProvider(PaymentService.class);

// Safe retrieval — returns null if not found (no exception)
PaymentService ps = provider.getIfAvailable();

// With default
PaymentService ps = provider.getIfAvailable(PaymentService::new);

// Iterate all matching beans
provider.forEach(service -> service.initialize());

// Stream all matching beans
provider.stream().filter(...).findFirst();
```

`ObjectProvider` is used heavily in framework code and auto-configuration to safely express optional dependencies. It avoids `NoSuchBeanDefinitionException` when a bean may or may not exist.

---

### The `&` Prefix — FactoryBean Dereference

The `BeanFactory.FACTORY_BEAN_PREFIX = "&"` constant reveals a core Spring concept. When a bean is a `FactoryBean<T>`, calling `getBean("myBean")` returns the *product* of the factory (type T), not the factory itself. To get the `FactoryBean` instance itself, you prefix the name with `&`:

```java
// Returns the product (e.g., DataSource)
DataSource ds = context.getBean("dataSource", DataSource.class);

// Returns the FactoryBean itself
FactoryBean<?> fb = context.getBean("&dataSource", FactoryBean.class);
```

This is covered in full in Topic 6.3. Mentioning it here because `BeanFactory` is where this prefix is defined.

---

### Parent-Child BeanFactory Hierarchy

`HierarchicalBeanFactory` enables parent-child container hierarchies. A child factory delegates to its parent when a bean is not found locally:

```java
DefaultListableBeanFactory parent = new DefaultListableBeanFactory();
// ... register beans in parent

DefaultListableBeanFactory child = new DefaultListableBeanFactory(parent);
// ... register beans in child
// child can see parent beans; parent cannot see child beans
```

This is the mechanism behind Spring MVC's dual-context architecture (root ApplicationContext as parent, DispatcherServlet's WebApplicationContext as child). Beans like `@Service` and `@Repository` go in the root context; `@Controller` beans go in the child context.

---

### Common Misconceptions

**Misconception 1:** "BeanFactory is just a Map of objects."
Wrong. BeanFactory is a sophisticated registry of *BeanDefinitions* (metadata) plus instantiation and lifecycle management logic. The map of object instances is just the singleton cache at the end of the process.

**Misconception 2:** "I can register a BeanPostProcessor as a `@Bean` in a plain BeanFactory and it will be applied."
Wrong. In a plain `BeanFactory`, you must add BPPs programmatically. The auto-detection magic happens in `ApplicationContext.refresh()`.

**Misconception 3:** "`containsBean(name)` tells me if a BeanDefinition exists."
`containsBean()` returns true if the name resolves to a bean — either from a BeanDefinition OR from a registered singleton object OR from a parent factory. Use `containsBeanDefinition()` (on `ListableBeanFactory`) if you want to check for an actual BeanDefinition.

---

## ② CODE EXAMPLES

### Example 1 — Direct DefaultListableBeanFactory Usage

```java
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.xml.XmlBeanDefinitionReader;
import org.springframework.core.io.ClassPathResource;

DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("beans.xml"));

// At this point: BeanDefinitions are registered, but NO beans instantiated

System.out.println("Bean count: " + factory.getBeanDefinitionCount());
System.out.println("Bean names: " +
    Arrays.toString(factory.getBeanDefinitionNames()));

// Only NOW is the bean instantiated:
OrderService service = factory.getBean("orderService", OrderService.class);
```

---

### Example 2 — Manually Registering a BeanPostProcessor with BeanFactory

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// Register BPP manually — required for plain BeanFactory
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());

XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("beans.xml"));

OrderService service = factory.getBean("orderService", OrderService.class);
// @Autowired now works because we manually added the BPP
```

If you forget `addBeanPostProcessor()`, `@Autowired` fields will NOT be injected in a plain BeanFactory — they will be null. This is a very common pitfall when writing test harnesses against a raw BeanFactory.

---

### Example 3 — Programmatic BeanDefinition Registration

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// Build a BeanDefinition programmatically
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(PaymentService.class);
bd.setScope(BeanDefinition.SCOPE_SINGLETON);
bd.getPropertyValues().add("timeout", 5000);

factory.registerBeanDefinition("paymentService", bd);

PaymentService ps = factory.getBean("paymentService", PaymentService.class);
```

This is how Spring Boot Auto-Configuration and various framework integrations dynamically register beans at runtime.

---

### Example 4 — ListableBeanFactory Operations

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);

// Get all bean names
String[] names = context.getBeanDefinitionNames();

// Get all beans of a specific type
Map<String, PaymentService> paymentBeans =
    context.getBeansOfType(PaymentService.class);

// Get all beans with a specific annotation
Map<String, Object> controllers =
    context.getBeansWithAnnotation(Controller.class);

// Get all bean names for a type (without instantiating)
String[] serviceNames =
    context.getBeanNamesForType(PaymentService.class);
```

---

### Example 5 — ObjectProvider for Optional/Lazy Dependencies

```java
@Service
public class OrderService {

    private final ObjectProvider<AuditService> auditServiceProvider;

    public OrderService(ObjectProvider<AuditService> auditServiceProvider) {
        this.auditServiceProvider = auditServiceProvider;
    }

    public void placeOrder(Order order) {
        // Only call if AuditService bean exists in context
        auditServiceProvider.ifAvailable(audit -> audit.log(order));
        // ... business logic
    }
}
```

This pattern allows `AuditService` to be entirely optional — the app works with or without it. No `@Autowired(required=false)` needed.

---

### Example 6 — Parent-Child BeanFactory Hierarchy

```java
// Parent context — shared services
AnnotationConfigApplicationContext parent =
    new AnnotationConfigApplicationContext(RootConfig.class);

// Child context — inherits from parent
AnnotationConfigApplicationContext child =
    new AnnotationConfigApplicationContext();
child.setParent(parent);
child.register(ChildConfig.class);
child.refresh();

// Child can resolve beans from parent:
OrderService os = child.getBean(OrderService.class); // works even if only in parent

// Parent CANNOT resolve beans only in child:
// parent.getBean(ChildOnlyBean.class); // throws NoSuchBeanDefinitionException
```

---

### Example 7 — `containsBean()` vs `containsBeanDefinition()` — Subtle Difference

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// Manually register a singleton INSTANCE (not a BeanDefinition)
factory.registerSingleton("manualBean", new SomeService());

System.out.println(factory.containsBean("manualBean"));           // true
System.out.println(factory.containsBeanDefinition("manualBean")); // false (!)
// Because no BeanDefinition was registered — just the instance directly
```

This distinction trips up experienced developers. `containsBeanDefinition()` only returns true if a proper `BeanDefinition` was registered, not a raw singleton instance.

---

### Example 8 — Edge Case: `getBean()` with Runtime Arguments (Prototype)

```java
// Bean definition
@Bean
@Scope("prototype")
public ReportGenerator reportGenerator(String reportType) {
    return new ReportGenerator(reportType);
}

// Runtime: passing constructor args to getBean
ReportGenerator gen = context.getBean("reportGenerator", "PDF");
```

The `getBean(name, args...)` variant allows passing constructor arguments at bean-retrieval time. This only works meaningfully for **prototype-scoped beans**, as singleton beans are already created. For singletons, the args are ignored after the first creation. This is a trap question on exams.

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which class is the primary concrete implementation of the Spring IoC container used internally by all ApplicationContext implementations?

A) `AbstractApplicationContext`
B) `DefaultBeanFactory`
C) `DefaultListableBeanFactory`
D) `XmlBeanFactory`

**Answer: C**
`XmlBeanFactory` is deprecated. `DefaultListableBeanFactory` is the real workhorse.

---

**Q2 — MCQ:**
In a plain `DefaultListableBeanFactory` (not ApplicationContext), a developer defines a `BeanPostProcessor` as a regular bean definition. What happens?

A) The BPP is automatically detected and applied to all beans
B) The BPP is applied only to beans created after it
C) The BPP is NOT automatically applied; it must be registered programmatically
D) Spring throws an exception

**Answer: C**
Auto-detection of BPPs is a feature of ApplicationContext, not plain BeanFactory.

---

**Q3 — Select all that apply:**
Which of the following are valid sub-interfaces of `BeanFactory`? (Select ALL that apply)

A) `ListableBeanFactory`
B) `HierarchicalBeanFactory`
C) `AutowireCapableBeanFactory`
D) `ConfigurableApplicationContext`
E) `BeanDefinitionRegistry`

**Answer: A, B, C**
`ConfigurableApplicationContext` extends `ApplicationContext`, not `BeanFactory` directly.
`BeanDefinitionRegistry` is a separate interface implemented by `DefaultListableBeanFactory` but not a sub-interface of `BeanFactory`.

---

**Q4 — Code Output Prediction:**

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("beans.xml"));

System.out.println("A");
MyService service = factory.getBean("myService", MyService.class);
System.out.println("B");
```

Assuming `MyService` prints "Creating MyService" in its constructor. What is the output?

A) Creating MyService → A → B
B) A → Creating MyService → B
C) A → B → Creating MyService
D) Creating MyService → A → B → Creating MyService

**Answer: B**
`BeanFactory` is lazy. Beans are not created until `getBean()` is called, which happens after printing "A".

---

**Q5 — Trick Question:**

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);

// AppConfig defines two @Bean methods both returning DataSource
DataSource ds = context.getBean("&primaryDataSource", DataSource.class);
```

What happens?

A) Returns the `DataSource` bean named `primaryDataSource`
B) Returns the `FactoryBean` that produces `primaryDataSource`
C) Throws `NoSuchBeanDefinitionException` because `primaryDataSource` is not a `FactoryBean`
D) Compile error

**Answer: C**
The `&` prefix is used to retrieve the `FactoryBean` *itself*. If `primaryDataSource` is a regular `@Bean` method (not a `FactoryBean`), Spring cannot find a `FactoryBean` for it and throws `NoSuchBeanDefinitionException`. The `&` prefix is only meaningful when the bean IS a `FactoryBean`.

---

**Q6 — Scenario-based:**
A developer calls `context.getBeanNamesForType(PaymentService.class)`. What does this method do?

A) Instantiates all beans and checks `instanceof`
B) Returns bean names whose BeanDefinition type matches, WITHOUT instantiating prototype beans unnecessarily
C) Throws exception if no matching bean exists
D) Only works for singleton-scoped beans

**Answer: B**
`getBeanNamesForType()` is designed to minimize unnecessary instantiation. Spring can often determine the type from the `BeanDefinition` metadata without instantiating the bean. However, for `FactoryBean` types, Spring may need to call `getObjectType()` which might require instantiation — this is a subtle detail.

---

**Q7 — Select all that apply:**
Which statements about `ObjectProvider<T>` are correct? (Select ALL that apply)

A) `getIfAvailable()` returns null if no bean of type T exists (no exception)
B) `getObject()` throws `NoSuchBeanDefinitionException` if no bean found
C) `ObjectProvider` supports streaming all matching beans
D) `ObjectProvider` was introduced in Spring 3.0
E) `ObjectProvider` can be used for lazy resolution of dependencies

**Answer: A, B, C, E**
`ObjectProvider` was introduced in Spring 4.3, not 3.0.

---

**Q8 — Compilation Error Identification:**

```java
BeanFactory factory = new DefaultListableBeanFactory();
String[] names = factory.getBeanDefinitionNames(); // Line X
```

What happens at Line X?

A) Returns all registered bean definition names
B) Compile error — `getBeanDefinitionNames()` is not on `BeanFactory`
C) Returns an empty array
D) Runtime exception

**Answer: B**
`getBeanDefinitionNames()` is defined on `ListableBeanFactory`, not the base `BeanFactory` interface. You must cast to `ListableBeanFactory` or `DefaultListableBeanFactory` to call it.

---

**Q9 — Scenario: `getBean(name, args)` on Singleton**

```java
@Bean
public PaymentService paymentService() {
    return new PaymentService("STRIPE");
}

// Later:
PaymentService ps = context.getBean("paymentService", "PAYPAL");
```

What is the gateway value of `ps.getProvider()`?

A) "PAYPAL" — the runtime arg overrides the bean definition
B) "STRIPE" — the singleton was already created; args are ignored
C) `IllegalArgumentException` is thrown
D) `UnsupportedOperationException` is thrown

**Answer: B**
For singletons, the bean was already created during context initialization. Runtime args to `getBean()` are ignored — you get the pre-existing singleton. Args only have effect for prototype beans.

---

## ④ TRICK ANALYSIS

**Trap 1: `XmlBeanFactory` still appears in old exam questions**
`XmlBeanFactory` was deprecated in Spring 3.1 and removed in Spring 6. If it appears as an answer choice for "which is the primary BeanFactory implementation," it is always wrong. The answer is `DefaultListableBeanFactory`.

**Trap 2: BPP registration in BeanFactory**
A very common trap is code that defines a `BeanPostProcessor` as a `<bean>` in an XML file loaded into a plain `DefaultListableBeanFactory`. The BPP will NOT be applied. You must call `factory.addBeanPostProcessor(...)`. Any answer claiming "it works automatically" is wrong for plain BeanFactory.

**Trap 3: `containsBean()` vs `containsBeanDefinition()`**
`containsBean()` returns true for:
- Beans with a BeanDefinition
- Manually registered singleton instances
- Beans resolvable from parent factory

`containsBeanDefinition()` returns true ONLY for beans with a registered BeanDefinition in the local factory. Parent factory beans do NOT count. Manually registered singleton instances do NOT count. Exam questions that conflate these two methods are traps.

**Trap 4: `getBeanNamesForType()` vs `getBeansOfType()`**
- `getBeanNamesForType()` — returns names, **avoids unnecessary instantiation**, preferred for checking existence
- `getBeansOfType()` — returns actual instances, **MAY trigger instantiation** of all matching beans
Questions asking which is more "efficient for checking if a bean type exists" → `getBeanNamesForType()`

**Trap 5: `getBean(name, args)` on singletons**
Runtime arguments to `getBean()` are silently ignored for singleton beans. The pre-created instance is returned. No error, no warning. This is surprising behavior that exams love.

**Trap 6: `&` prefix type mismatch**
`getBean("&myBean", DataSource.class)` when `myBean` is NOT a FactoryBean → `NoSuchBeanDefinitionException`, not `ClassCastException`. The factory doesn't find a FactoryBean entry for that name.

**Trap 7: `ListableBeanFactory` vs `BeanFactory` method availability**
`getBeanDefinitionNames()`, `getBeansOfType()`, `getBeansWithAnnotation()` — all on `ListableBeanFactory`. They do NOT exist on the base `BeanFactory`. Code trying to call them on a `BeanFactory`-typed reference will fail at compile time.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Primary concrete implementation | `DefaultListableBeanFactory` |
| BeanFactory instantiation | **Lazy** — only on `getBean()` |
| BPP in plain BeanFactory | Must be added **programmatically** via `addBeanPostProcessor()` |
| `&` prefix | Retrieves the `FactoryBean` instance, not its product |
| `containsBean()` | Checks local + parent + manual singletons |
| `containsBeanDefinition()` | Checks **only** local BeanDefinition registry |
| `getBean(name, args)` on singletons | Args **ignored** — pre-created instance returned |
| `getBeanNamesForType()` | No instantiation; `getBeansOfType()` may instantiate |
| `ObjectProvider` | Spring 4.3+, safe optional/lazy dependency resolution |

---

### BeanFactory Interface Hierarchy (Memorize)

```
BeanFactory
├── HierarchicalBeanFactory        → getParentBeanFactory(), containsLocalBean()
├── ListableBeanFactory            → getBeanDefinitionNames(), getBeansOfType()
│       └── ConfigurableListableBeanFactory
│                └── DefaultListableBeanFactory  ← THE IMPLEMENTATION
│                        also implements BeanDefinitionRegistry
│
└── AutowireCapableBeanFactory     → createBean(), autowireBean(), resolveDependency()
```

---

### Three-Level Cache (Preview — Deep dive in Topic 3.3)

```
Level 1: singletonObjects        → fully initialized singletons
Level 2: earlySingletonObjects   → instantiated but not fully initialized (for circular deps)
Level 3: singletonFactories      → ObjectFactory lambdas for early exposure
```

---

### BeanFactory vs ApplicationContext — Critical Exam Table

| Aspect | BeanFactory | ApplicationContext |
|---|---|---|
| Instantiation timing | Lazy | Eager (singletons) |
| BPP auto-detection | ❌ | ✅ |
| BFPP auto-detection | ❌ | ✅ |
| i18n / MessageSource | ❌ | ✅ |
| Events | ❌ | ✅ |
| `@PostConstruct` auto-processing | ❌ | ✅ |
| Configuration error detection | At runtime | At startup |
| Resource loading | Basic | Full |

---

### Common Interview One-Liners

- "`DefaultListableBeanFactory` is the core implementation that all ApplicationContext implementations delegate to internally."
- "Plain `BeanFactory` does not auto-detect `BeanPostProcessor` beans — you must register them manually."
- "The `&` prefix in `getBean()` retrieves the `FactoryBean` instance itself, not the product it creates."
- "`getBeanNamesForType()` avoids unnecessary bean instantiation; `getBeansOfType()` may trigger it."
- "`ObjectProvider<T>` provides safe, lazy, optional access to a dependency without throwing exceptions."
- "`containsBeanDefinition()` only checks the local registry; `containsBean()` also checks parent factories and manually registered singletons."

---
