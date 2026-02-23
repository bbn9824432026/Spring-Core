# SPRING CORE FRAMEWORK — MASTER INDEX

---

## PART 1: THE IOC CONTAINER & CORE CONTAINER

### 1.1 Introduction to the Spring IoC Container
- 1.1.1 What is Inversion of Control (IoC)?
- 1.1.2 Dependency Injection as a form of IoC
- 1.1.3 The Container concept — BeanFactory vs ApplicationContext
- 1.1.4 Spring Container responsibilities
- 1.1.5 Container initialization lifecycle (high-level)
- 1.1.6 ClassPathXmlApplicationContext, FileSystemXmlApplicationContext, AnnotationConfigApplicationContext
- 1.1.7 WebApplicationContext overview

### 1.2 BeanFactory
- 1.2.1 BeanFactory interface and contract
- 1.2.2 DefaultListableBeanFactory internals
- 1.2.3 BeanFactory vs ApplicationContext — feature comparison
- 1.2.4 Lazy initialization by default in BeanFactory
- 1.2.5 When to use BeanFactory directly

### 1.3 ApplicationContext
- 1.3.1 ApplicationContext interface hierarchy
- 1.3.2 ConfigurableApplicationContext
- 1.3.3 ApplicationContext additional capabilities (i18n, events, resource loading)
- 1.3.4 Eager initialization of singleton beans
- 1.3.5 ApplicationContext implementations deep dive
- 1.3.6 Refreshing the context — refresh() internals
- 1.3.7 Closing the context — close() and shutdown hooks

---

## PART 2: BEAN DEFINITION & METADATA

### 2.1 Bean Definition Internals
- 2.1.1 BeanDefinition interface
- 2.1.2 RootBeanDefinition, ChildBeanDefinition, GenericBeanDefinition
- 2.1.3 BeanDefinitionRegistry
- 2.1.4 Bean metadata: class, scope, constructor args, properties, init/destroy methods
- 2.1.5 Bean definition merging (parent-child bean definitions)
- 2.1.6 Bean aliases
- 2.1.7 Inner beans

### 2.2 Configuration Styles
- 2.2.1 XML-based configuration (full reference)
  - namespace declarations
  - `<bean>`, `<alias>`, `<import>`
  - p-namespace, c-namespace
  - util namespace
- 2.2.2 Annotation-based configuration
  - `@Component`, `@Service`, `@Repository`, `@Controller`
  - Component scanning — `@ComponentScan`
  - `@Autowired`, `@Qualifier`, `@Primary`
  - `@Value`
  - JSR-250: `@Resource`, `@PostConstruct`, `@PreDestroy`
  - JSR-330: `@Inject`, `@Named`
- 2.2.3 Java-based configuration (`@Configuration`)
  - `@Bean` methods
  - `@Configuration` vs `@Component` — CGLIB proxy difference
  - `@Import`, `@ImportResource`
  - Programmatic bean registration
- 2.2.4 Mixed configuration (XML + Annotation + Java Config)
- 2.2.5 Configuration metadata processing order

---

## PART 3: DEPENDENCY INJECTION

### 3.1 Types of Dependency Injection
- 3.1.1 Constructor injection — internals and mechanics
- 3.1.2 Setter injection — internals and mechanics
- 3.1.3 Field injection — reflection-based, pitfalls
- 3.1.4 Method injection — Lookup Method injection
- 3.1.5 Arbitrary method replacement

### 3.2 Autowiring
- 3.2.1 Autowiring modes: no, byName, byType, constructor
- 3.2.2 `@Autowired` — resolution algorithm
- 3.2.3 `@Qualifier` — fine-grained selection
- 3.2.4 `@Primary` — tie-breaking
- 3.2.5 `@Autowired` on collections and maps
- 3.2.6 Optional dependencies — `required=false`, `Optional<T>`, `@Nullable`
- 3.2.7 Autowiring limitations and exclusions
- 3.2.8 `@Resource` vs `@Autowired` — resolution difference
- 3.2.9 JSR-330 `@Inject` behavior

### 3.3 Circular Dependencies
- 3.3.1 Constructor injection circular dependency — why it fails
- 3.3.2 Setter injection circular dependency — three-level cache solution
- 3.3.3 Three-level cache internals (singletonObjects, earlySingletonObjects, singletonFactories)
- 3.3.4 `@Lazy` to break circular dependency
- 3.3.5 Detecting and resolving circular dependency in enterprise apps

### 3.4 Dependency Resolution Order
- 3.4.1 Bean instantiation order
- 3.4.2 `@DependsOn`
- 3.4.3 Lazy beans — `@Lazy`
- 3.4.4 FactoryBean and dependency resolution

---

## PART 4: BEAN LIFECYCLE

### 4.1 Complete Bean Lifecycle (Step-by-Step)
- 4.1.1 BeanDefinition loading
- 4.1.2 BeanFactoryPostProcessor execution
- 4.1.3 Bean instantiation (constructor)
- 4.1.4 Dependency injection (populate properties)
- 4.1.5 BeanNameAware, BeanFactoryAware, ApplicationContextAware callbacks
- 4.1.6 BeanPostProcessor — postProcessBeforeInitialization
- 4.1.7 @PostConstruct / InitializingBean.afterPropertiesSet() / init-method
- 4.1.8 BeanPostProcessor — postProcessAfterInitialization
- 4.1.9 Bean ready for use
- 4.1.10 @PreDestroy / DisposableBean.destroy() / destroy-method
- 4.1.11 Destruction order and caveats

### 4.2 Aware Interfaces
- 4.2.1 BeanNameAware
- 4.2.2 BeanFactoryAware
- 4.2.3 ApplicationContextAware
- 4.2.4 EnvironmentAware
- 4.2.5 ResourceLoaderAware
- 4.2.6 MessageSourceAware
- 4.2.7 ApplicationEventPublisherAware
- 4.2.8 Order of Aware callbacks

### 4.3 Initialization Mechanisms (Priority Order)
- 4.3.1 @PostConstruct (JSR-250)
- 4.3.2 InitializingBean.afterPropertiesSet()
- 4.3.3 Custom init-method (XML / @Bean(initMethod=...))
- 4.3.4 Combining all three — execution order

### 4.4 Destruction Mechanisms (Priority Order)
- 4.4.1 @PreDestroy
- 4.4.2 DisposableBean.destroy()
- 4.4.3 Custom destroy-method
- 4.4.4 Prototype scope destruction — Spring does NOT manage

---

## PART 5: BEAN SCOPES

### 5.1 Singleton Scope
- 5.1.1 Container-level singleton (not JVM-level)
- 5.1.2 Eager vs lazy initialization
- 5.1.3 Shared mutable state — thread safety implications
- 5.1.4 Internal storage in DefaultListableBeanFactory

### 5.2 Prototype Scope
- 5.2.1 New instance per request
- 5.2.2 Spring does not manage prototype lifecycle after creation
- 5.2.3 Injecting prototype into singleton — the stale reference problem
- 5.2.4 Solutions: Lookup Method, ObjectFactory, ApplicationContext.getBean()

### 5.3 Web-Aware Scopes
- 5.3.1 Request scope
- 5.3.2 Session scope
- 5.3.3 Application scope
- 5.3.4 WebSocket scope
- 5.3.5 Scope proxy — ScopedProxyMode
- 5.3.6 `<aop:scoped-proxy>` in XML

### 5.4 Custom Scopes
- 5.4.1 Implementing the Scope interface
- 5.4.2 Registering custom scope with ConfigurableBeanFactory

---

## PART 6: CONTAINER EXTENSION POINTS

### 6.1 BeanFactoryPostProcessor
- 6.1.1 Purpose and contract
- 6.1.2 Execution timing (before bean instantiation)
- 6.1.3 PropertySourcesPlaceholderConfigurer
- 6.1.4 PropertyOverrideConfigurer
- 6.1.5 Custom BeanFactoryPostProcessor
- 6.1.6 `@Order` and `Ordered` interface with BFPPs
- 6.1.7 BeanDefinitionRegistryPostProcessor

### 6.2 BeanPostProcessor
- 6.2.1 Purpose and contract
- 6.2.2 postProcessBeforeInitialization
- 6.2.3 postProcessAfterInitialization
- 6.2.4 AutowiredAnnotationBeanPostProcessor (internal)
- 6.2.5 CommonAnnotationBeanPostProcessor (internal)
- 6.2.6 Returning a proxy from BPP — AOP implication
- 6.2.7 BPP registration and ordering

### 6.3 FactoryBean
- 6.3.1 FactoryBean interface
- 6.3.2 getObject(), getObjectType(), isSingleton()
- 6.3.3 `&` prefix to access FactoryBean itself vs the product
- 6.3.4 When Spring uses FactoryBean internally (e.g. ProxyFactoryBean)

---

## PART 7: ANNOTATION-BASED CONTAINER CONFIGURATION

### 7.1 Component Scanning Deep Dive
- 7.1.1 `@ComponentScan` — basePackages, basePackageClasses
- 7.1.2 Include/exclude filters — FilterType types
- 7.1.3 Default filter (stereotype annotations)
- 7.1.4 `useDefaultFilters=false`
- 7.1.5 `<context:component-scan>` XML equivalent
- 7.1.6 `<context:annotation-config>` — what it registers

### 7.2 Stereotype Annotations
- 7.2.1 `@Component` as meta-annotation
- 7.2.2 `@Service`, `@Repository`, `@Controller` — semantic differences
- 7.2.3 `@Repository` — exception translation postprocessor

### 7.3 `@Autowired` Internals
- 7.3.1 AutowiredAnnotationBeanPostProcessor — how it scans
- 7.3.2 Injection point resolution algorithm (type → qualifier → name)
- 7.3.3 Constructor selection with `@Autowired`
- 7.3.4 `@Autowired` on `List<T>`, `Set<T>`, `Map<String, T>`

---

## PART 8: JAVA-BASED CONFIGURATION DEEP DIVE

### 8.1 `@Configuration` and CGLIB
- 8.1.1 Why `@Configuration` classes are CGLIB-proxied
- 8.1.2 Inter-bean method calls — singleton guarantee
- 8.1.3 `@Configuration(proxyBeanMethods=false)` — lite mode
- 8.1.4 `@Bean` method visibility rules
- 8.1.5 Static `@Bean` methods — why and when

### 8.2 Composing Configurations
- 8.2.1 `@Import` — importing other `@Configuration` classes
- 8.2.2 `@ImportResource` — importing XML
- 8.2.3 `ImportSelector` and `ImportBeanDefinitionRegistrar`
- 8.2.4 Conditional configuration — `@Conditional`

---

## PART 9: ENVIRONMENT ABSTRACTION

### 9.1 Environment Interface
- 9.1.1 Environment and ConfigurableEnvironment
- 9.1.2 Profiles in Environment
- 9.1.3 Property resolution in Environment

### 9.2 Profiles
- 9.2.1 `@Profile` on beans and configuration classes
- 9.2.2 Activating profiles — `spring.profiles.active`
- 9.2.3 Default profile — `spring.profiles.default`
- 9.2.4 Profile expressions (Spring 5.1+): `!`, `&`, `|`
- 9.2.5 XML profile attribute on `<beans>`

### 9.3 PropertySources
- 9.3.1 PropertySource abstraction
- 9.3.2 PropertySource hierarchy and override order
- 9.3.3 `@PropertySource` annotation
- 9.3.4 StandardEnvironment default property sources
- 9.3.5 `@TestPropertySource`

### 9.4 `@Value` and Property Injection
- 9.4.1 `@Value` with `${}` (property placeholder)
- 9.4.2 `@Value` with `#{}` (SpEL expression)
- 9.4.3 Default values in `@Value`
- 9.4.4 PropertySourcesPlaceholderConfigurer requirement

---

## PART 10: RESOURCE ABSTRACTION

### 10.1 Resource Interface
- 10.1.1 Resource interface methods
- 10.1.2 ClassPathResource, FileSystemResource, UrlResource, ServletContextResource
- 10.1.3 InputStreamResource, ByteArrayResource

### 10.2 ResourceLoader
- 10.2.1 ResourceLoader interface
- 10.2.2 ResourceLoaderAware
- 10.2.3 ApplicationContext as ResourceLoader

### 10.3 Resource Paths and Prefixes
- 10.3.1 `classpath:`, `classpath*:`, `file:`, `http:`
- 10.3.2 Ant-style path patterns
- 10.3.3 ResourcePatternResolver

---

## PART 11: TYPE CONVERSION & DATA BINDING

### 11.1 PropertyEditors (Legacy)
- 11.1.1 JavaBeans PropertyEditor
- 11.1.2 Built-in PropertyEditors in Spring
- 11.1.3 Custom PropertyEditor registration

### 11.2 Spring ConversionService
- 11.2.1 Converter interface
- 11.2.2 ConverterFactory
- 11.2.3 GenericConverter
- 11.2.4 ConversionService setup and registration
- 11.2.5 DefaultConversionService, DefaultFormattingConversionService

### 11.3 Formatter SPI
- 11.3.1 Formatter interface (print + parse)
- 11.3.2 AnnotationFormatterFactory
- 11.3.3 FormatterRegistry

### 11.4 DataBinder
- 11.4.1 DataBinder and binding process
- 11.4.2 Binding result and type mismatch errors

---

## PART 12: VALIDATION

### 12.1 Spring Validator Interface
- 12.1.1 Validator interface — supports() and validate()
- 12.1.2 Errors and BindingResult
- 12.1.3 ValidationUtils

### 12.2 JSR-303/349/380 Bean Validation
- 12.2.1 `@Valid` and `@Validated`
- 12.2.2 Constraint annotations
- 12.2.3 LocalValidatorFactoryBean
- 12.2.4 Method validation with `@Validated`

### 12.3 Combining Spring Validation and Bean Validation

---

## PART 13: SPRING EXPRESSION LANGUAGE (SpEL)

### 13.1 SpEL Fundamentals
- 13.1.1 ExpressionParser, Expression, EvaluationContext
- 13.1.2 StandardEvaluationContext vs SimpleEvaluationContext
- 13.1.3 SpEL in bean definitions — XML and annotation

### 13.2 SpEL Syntax Reference
- 13.2.1 Literal expressions
- 13.2.2 Property access, array/list/map access
- 13.2.3 Method invocation
- 13.2.4 Operators (relational, logical, arithmetic, ternary, Elvis)
- 13.2.5 Type operator — `T()`
- 13.2.6 Constructor invocation
- 13.2.7 Variables — `#variable`
- 13.2.8 Bean references — `@beanName`
- 13.2.9 Collection projection and selection
- 13.2.10 Templated expressions

### 13.3 SpEL in Spring Security, Spring Data (forward reference only)

---

## PART 14: SPRING AOP (CORE FUNDAMENTALS)

### 14.1 AOP Concepts
- 14.1.1 Cross-cutting concerns
- 14.1.2 Aspect, Advice, Joinpoint, Pointcut, Weaving, Target, Proxy
- 14.1.3 AOP vs OOP

### 14.2 Spring AOP Architecture
- 14.2.1 Proxy-based AOP — not bytecode weaving
- 14.2.2 JDK Dynamic Proxy vs CGLIB Proxy
- 14.2.3 When each proxy type is chosen
- 14.2.4 Self-invocation problem — why it bypasses AOP
- 14.2.5 AopProxyFactory, ProxyFactory, ProxyFactoryBean

### 14.3 Advice Types
- 14.3.1 Before advice
- 14.3.2 After returning advice
- 14.3.3 After throwing advice
- 14.3.4 After (finally) advice
- 14.3.5 Around advice — ProceedingJoinPoint
- 14.3.6 Advice execution order

### 14.4 Pointcut Expressions (AspectJ)
- 14.4.1 `execution()` — full syntax
- 14.4.2 `within()`
- 14.4.3 `this()` vs `target()`
- 14.4.4 `@annotation()`
- 14.4.5 `args()` and `@args()`
- 14.4.6 `bean()` — Spring extension
- 14.4.7 Combining pointcuts — `&&`, `||`, `!`

### 14.5 `@AspectJ` Style
- 14.5.1 `@Aspect`, `@Pointcut`, advice annotations
- 14.5.2 Enabling AspectJ support — `@EnableAspectJAutoProxy`
- 14.5.3 `proxyTargetClass=true` implications

### 14.6 Schema-based AOP (XML)
- 14.6.1 `<aop:config>`, `<aop:aspect>`, `<aop:pointcut>`, `<aop:advisor>`

### 14.7 AOP Ordering
- 14.7.1 `@Order` on aspects
- 14.7.2 Advice precedence within same aspect

### 14.8 Load-Time Weaving Basics
- 14.8.1 LTW vs proxy-based AOP
- 14.8.2 `@EnableLoadTimeWeaving`
- 14.8.3 `aop.xml` configuration
- 14.8.4 Spring agent setup

---

## PART 15: EVENTS

### 15.1 Standard ApplicationEvents
- 15.1.1 ContextRefreshedEvent
- 15.1.2 ContextStartedEvent, ContextStoppedEvent, ContextClosedEvent
- 15.1.3 RequestHandledEvent

### 15.2 Custom Events
- 15.2.1 Extending ApplicationEvent (legacy)
- 15.2.2 POJO events (Spring 4.2+)
- 15.2.3 Publishing events — ApplicationEventPublisher
- 15.2.4 `@EventListener`
- 15.2.5 `@TransactionalEventListener`

### 15.3 Async Events
- 15.3.1 `@Async` on `@EventListener`
- 15.3.2 Event ordering — `@Order`

### 15.4 Generics-based Events
- 15.4.1 `ApplicationEvent<T>` — ResolvableTypeProvider

---

## PART 16: INTERNATIONALIZATION (i18n) & MessageSource

### 16.1 MessageSource Interface
- 16.1.1 getMessage() variants
- 16.1.2 ResourceBundleMessageSource
- 16.1.3 ReloadableResourceBundleMessageSource
- 16.1.4 StaticMessageSource

### 16.2 MessageSource Hierarchy
- 16.2.1 Parent context message source delegation
- 16.2.2 `basename` configuration

### 16.3 LocaleResolver and LocaleContextHolder (forward reference to Web)

---

## PART 17: TESTING SUPPORT

### 17.1 Spring TestContext Framework
- 17.1.1 `@RunWith(SpringRunner.class)` / `@ExtendWith(SpringExtension.class)`
- 17.1.2 `@ContextConfiguration`
- 17.1.3 `@SpringBootTest` (brief — belongs to Boot)
- 17.1.4 Context caching between tests

### 17.2 Test Annotations
- 17.2.1 `@TestPropertySource`
- 17.2.2 `@ActiveProfiles`
- 17.2.3 `@DirtiesContext`
- 17.2.4 `@Sql`, `@SqlConfig`
- 17.2.5 `@MockBean`, `@SpyBean` (forward reference)

### 17.3 Dependency Injection in Tests
- 17.3.1 `@Autowired` in test classes
- 17.3.2 `@Value` in test classes

### 17.4 Transaction Management in Tests
- 17.4.1 `@Transactional` in tests — rollback by default
- 17.4.2 `@Commit`, `@Rollback`
- 17.4.3 `@BeforeTransaction`, `@AfterTransaction`

### 17.5 MockEnvironment, MockPropertySource, MockMvc (reference only)

---

## HOW WE WILL PROCEED

Each of the above topics will be taught one at a time, following the strict 5-part format:

**① Conceptual Explanation → ② Code Examples → ③ Exam Questions → ④ Trick Analysis → ⑤ Summary Sheet**

---

# TOPIC 1.1 — Introduction to the Spring IoC Container

---

## ① CONCEPTUAL EXPLANATION

### What is Inversion of Control (IoC)?

IoC is an architectural principle, not a Spring-specific concept. In traditional procedural or tightly coupled object-oriented code, a class is responsible for satisfying its own dependencies. The control of dependency creation and lifecycle lies *with the class itself*.

```java
// Traditional — class controls its own dependencies
public class OrderService {
    private PaymentService paymentService = new PaymentService(); // hardcoded
}
```

The problem is obvious: `OrderService` is now tightly coupled to the concrete `PaymentService` implementation. Testing is hard, swapping implementations is hard, lifecycle management is manual.

**Inversion of Control** means you *invert* this. Instead of the class pulling its dependencies in, something external *pushes* dependencies into the class. The class declares what it needs; someone else satisfies that need.

"Don't call us, we'll call you." — The Hollywood Principle. This is IoC.

IoC is a *broad principle*. Dependency Injection is the *most common pattern* through which IoC is implemented. Other IoC patterns include Service Locator, Template Method pattern, and Event-driven callbacks — but Spring primarily uses DI.

---

### Dependency Injection as a Form of IoC

DI is the mechanism through which the IoC container provides (injects) dependencies into objects. The object doesn't know *how* its dependencies are resolved — it just declares them, and the container fulfills them.

Spring's IoC container is the *runtime* that:
1. Reads configuration metadata (XML / annotations / Java config)
2. Instantiates objects (beans)
3. Wires dependencies between objects
4. Manages the complete lifecycle of those objects
5. Provides enterprise services on top (transactions, AOP, events, etc.)

---

### The Container Concept — BeanFactory vs ApplicationContext

Spring has two primary container interfaces:

**`BeanFactory`** — `org.springframework.beans.factory.BeanFactory`
This is the foundational contract. It provides basic DI support. It is a lazy container by default — beans are instantiated only when requested via `getBean()`.

**`ApplicationContext`** — `org.springframework.context.ApplicationContext`
This extends `BeanFactory` and adds enterprise-grade features. It is an *eager* container by default — all singleton beans are pre-instantiated at container startup. It also adds:

- MessageSource (i18n)
- ApplicationEvent publishing
- Resource loading (ResourceLoader)
- Environment abstraction
- BeanPostProcessor auto-detection
- BeanFactoryPostProcessor auto-detection

In practice, **you almost always use ApplicationContext**. BeanFactory is used in extremely constrained environments (embedded systems, OSGi containers, very limited memory budgets).

**Critical internal distinction:** `ApplicationContext` *wraps* a `DefaultListableBeanFactory` internally. The `ApplicationContext` implementation delegates actual bean definition storage and instantiation to this internal BeanFactory. This is the Delegate pattern.

---

### Spring Container Responsibilities

The container has the following core responsibilities, in order:

1. **Loading configuration metadata** — reading XML files, scanning annotations, processing `@Configuration` classes
2. **Parsing and registering BeanDefinitions** — building an internal registry (`BeanDefinitionRegistry`) of all bean metadata
3. **Running BeanFactoryPostProcessors** — allowing modification of BeanDefinitions before any beans are instantiated
4. **Instantiating singleton beans** (eagerly, for ApplicationContext)
5. **Injecting dependencies** into each bean
6. **Running BeanPostProcessors** — including proxy creation (AOP), annotation processing, etc.
7. **Invoking lifecycle callbacks** — `@PostConstruct`, `InitializingBean`, custom init methods
8. **Serving beans** to application code via `getBean()` or injection
9. **Shutting down** — invoking destroy callbacks in reverse order

---

### Container Initialization Lifecycle (High-Level)

```
new AnnotationConfigApplicationContext(AppConfig.class)
        │
        ▼
  ┌─────────────────────────────────────────┐
  │ 1. Register configuration metadata      │
  │ 2. Call refresh()                       │
  │    ├─ prepareRefresh()                  │
  │    ├─ obtainFreshBeanFactory()          │
  │    ├─ prepareBeanFactory()              │
  │    ├─ postProcessBeanFactory()          │
  │    ├─ invokeBeanFactoryPostProcessors() │ ← BFPPs run here
  │    ├─ registerBeanPostProcessors()      │ ← BPPs registered
  │    ├─ initMessageSource()              │
  │    ├─ initApplicationEventMulticaster() │
  │    ├─ onRefresh()                       │
  │    ├─ registerListeners()              │
  │    ├─ finishBeanFactoryInitialization() │ ← singletons instantiated
  │    └─ finishRefresh()                  │ ← ContextRefreshedEvent
  └─────────────────────────────────────────┘
```

This `refresh()` method in `AbstractApplicationContext` is the single most important method in the Spring framework. Every ApplicationContext eventually calls it. Understanding this sequence is essential for understanding BeanFactoryPostProcessor and BeanPostProcessor ordering (covered in Part 6).

---

### ApplicationContext Implementations

| Implementation | Configuration Source |
|---|---|
| `ClassPathXmlApplicationContext` | XML files on classpath |
| `FileSystemXmlApplicationContext` | XML files from filesystem path |
| `AnnotationConfigApplicationContext` | `@Configuration` classes or packages |
| `GenericWebApplicationContext` | Web environments |
| `XmlWebApplicationContext` | XML in web environments |
| `AnnotationConfigWebApplicationContext` | `@Configuration` in web environments |
| `GenericGroovyApplicationContext` | Groovy bean definitions |

**`AnnotationConfigApplicationContext`** is the modern standard for non-web Spring applications.

---

### Internal Architecture — How the Container Actually Works

Internally, `AnnotationConfigApplicationContext` maintains:

- **`DefaultListableBeanFactory`** — stores all `BeanDefinition` objects in a `ConcurrentHashMap<String, BeanDefinition>` called `beanDefinitionMap`
- **`beanDefinitionNames`** — an ordered `List<String>` to maintain registration order
- **`singletonObjects`** — a `ConcurrentHashMap<String, Object>` holding fully initialized singleton instances (Level 1 cache)
- **`earlySingletonObjects`** — a `HashMap<String, Object>` for early references (Level 2 cache — used for circular dependency resolution)
- **`singletonFactories`** — a `HashMap<String, ObjectFactory<?>>` (Level 3 cache)

These three caches form the **three-level cache** that enables setter-injection circular dependency resolution. We will cover this in deep detail in Topic 3.3.

---

### JVM Behavior and Reflection

The Spring container uses **Java Reflection** (`java.lang.reflect`) extensively:
- To instantiate beans (`Constructor.newInstance()`)
- To inject field dependencies (`Field.set()`)
- To call setter methods (`Method.invoke()`)
- To call init/destroy lifecycle methods
- To read annotations at runtime (`Field.getAnnotations()`, `Method.getAnnotations()`)

This has implications:
- Reflection is **slower** than direct method calls — this is why Spring caches reflection metadata aggressively via `ReflectionUtils` and `CachedIntrospectionResults`
- Private fields and methods **can** be accessed because Spring calls `setAccessible(true)` — which is why field injection works even on private fields (though it's still discouraged for testability)
- JDK 16+ module system can restrict `setAccessible(true)` — `--add-opens` JVM flags may be needed in strict module configurations

---

### Common Misconceptions

**Misconception 1:** "Spring singleton is the same as the GoF Singleton pattern."
Wrong. Spring singleton means *one instance per container*. If you create two `ApplicationContext` instances with the same configuration, you get *two* instances of each singleton bean. GoF Singleton means one instance per JVM classloader.

**Misconception 2:** "BeanFactory is deprecated."
Wrong. BeanFactory is not deprecated. It is still the foundational interface. ApplicationContext simply extends it with additional capabilities.

**Misconception 3:** "The container creates beans when you call `getBean()`."
For `ApplicationContext`, singleton beans are pre-instantiated at `refresh()` time, not at `getBean()` time. `getBean()` on a singleton just returns the already-created instance from the cache.

**Misconception 4:** "You need `<context:annotation-config>` for component scanning."
`<context:component-scan>` *includes* the effect of `<context:annotation-config>`. You do not need both. But `<context:annotation-config>` alone does NOT enable scanning.

---

### Real Enterprise Pitfalls

**Pitfall 1 — Closing the context:** If you use `ApplicationContext` in a standalone app and never call `context.close()` or register a shutdown hook (`context.registerShutdownHook()`), `@PreDestroy` methods and destroy callbacks **will never execute**. Always register the shutdown hook.

**Pitfall 2 — Multiple context hierarchies:** In Spring MVC apps, there are typically two contexts — a root `ApplicationContext` and a child `WebApplicationContext`. Child contexts can see parent beans, but not vice versa. Beans defined in the wrong context cause `NoSuchBeanDefinitionException` or invisible beans.

**Pitfall 3 — Refreshing context:** Calling `refresh()` on an already-running context destroys all existing singletons and re-initializes them. This is rarely desirable in production and can cause serious issues if done unintentionally.

---

## ② CODE EXAMPLES

### Example 1 — Classic XML-based Container Bootstrap

```xml
<!-- applicationContext.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="paymentService" class="com.example.PaymentService"/>

    <bean id="orderService" class="com.example.OrderService">
        <property name="paymentService" ref="paymentService"/>
    </bean>
</beans>
```

```java
// Bootstrapping
ApplicationContext context =
    new ClassPathXmlApplicationContext("applicationContext.xml");

OrderService orderService = context.getBean("orderService", OrderService.class);
// OR
OrderService orderService = context.getBean(OrderService.class);
```

---

### Example 2 — Java Config (Modern Standard)

```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }

    @Bean
    public OrderService orderService() {
        return new OrderService(paymentService()); // inter-bean method call — CGLIB handles this
    }
}
```

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);

OrderService orderService = context.getBean(OrderService.class);
```

---

### Example 3 — Annotation-based (Component Scan)

```java
@Service
public class PaymentService { }

@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}
```

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig { }

ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);
```

---

### Example 4 — Programmatic Bean Registration (Rarely asked but architecturally important)

```java
AnnotationConfigApplicationContext context =
    new AnnotationConfigApplicationContext();
context.register(AppConfig.class);
context.scan("com.example.services");
context.refresh(); // MUST be called manually when using no-arg constructor
```

**Important:** When using the no-arg constructor, `refresh()` must be called explicitly. The single-arg constructors (taking `Class` or package name) call `refresh()` internally.

---

### Example 5 — Shutdown Hook (Critical for lifecycle callbacks)

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);
((ConfigurableApplicationContext) context).registerShutdownHook();
// Now @PreDestroy and destroy methods WILL fire on JVM shutdown
```

---

### Example 6 — BeanFactory Direct Usage (for contrast)

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("applicationContext.xml");

// Beans are NOT yet instantiated — lazy by default
OrderService orderService = factory.getBean("orderService", OrderService.class);
// ONLY NOW is orderService instantiated and wired
```

---

### Example 7 — Edge Case: Same Bean Type, Multiple Beans

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource primaryDataSource() { ... }

    @Bean
    public DataSource secondaryDataSource() { ... }
}

// This will throw NoUniqueBeanDefinitionException:
DataSource ds = context.getBean(DataSource.class);

// This works:
DataSource ds = context.getBean("primaryDataSource", DataSource.class);
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What is the primary difference between `BeanFactory` and `ApplicationContext`?

A) `BeanFactory` supports annotation-based configuration; `ApplicationContext` does not
B) `ApplicationContext` eagerly initializes singleton beans; `BeanFactory` is lazy by default
C) `BeanFactory` supports AOP; `ApplicationContext` does not
D) `ApplicationContext` uses XML only; `BeanFactory` supports Java Config

**Answer: B**

---

**Q2 — MCQ:**
Which of the following statements about Spring singletons is correct?

A) A Spring singleton bean guarantees one instance per JVM
B) A Spring singleton bean guarantees one instance per ClassLoader
C) A Spring singleton bean guarantees one instance per ApplicationContext
D) A Spring singleton bean guarantees one instance per thread

**Answer: C**

---

**Q3 — Select all that apply:**
Which features does `ApplicationContext` add over `BeanFactory`? (Select ALL that apply)

A) Bean lifecycle management
B) MessageSource for internationalization
C) ApplicationEvent publishing
D) BeanPostProcessor auto-detection
E) Dependency injection
F) ResourceLoader capabilities

**Answer: B, C, D, F**
(A and E are provided by BeanFactory itself)

---

**Q4 — Code Output Prediction:**

```java
@Configuration
public class AppConfig {
    @Bean
    public MyBean myBean() {
        System.out.println("Creating MyBean");
        return new MyBean();
    }
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
System.out.println("Context created");
ctx.getBean(MyBean.class);
System.out.println("Bean retrieved");
```

What is the output?

A)
```
Context created
Creating MyBean
Bean retrieved
```
B)
```
Creating MyBean
Context created
Bean retrieved
```
C)
```
Creating MyBean
Bean retrieved
Context created
```
D) Depends on JVM

**Answer: B**
Singleton beans are eagerly initialized during `refresh()`, which happens inside the `AnnotationConfigApplicationContext` constructor, before control returns to the line `System.out.println("Context created")`.

---

**Q5 — Scenario-based:**
You create two separate `AnnotationConfigApplicationContext` instances with the same `@Configuration` class. How many instances of a `@Bean`-annotated singleton will exist?

A) 1 — Spring enforces JVM-level singleton
B) 2 — one per ApplicationContext
C) 0 — you cannot create two contexts with the same config
D) Depends on the `@Scope` annotation

**Answer: B**
Spring singleton scope is *per-container*, not per-JVM.

---

**Q6 — Trick Question:**
A developer uses the no-arg `AnnotationConfigApplicationContext()` constructor, registers a class, and immediately calls `getBean()`. What happens?

A) The bean is returned successfully
B) `IllegalStateException` is thrown because `refresh()` was not called
C) `NullPointerException` is thrown
D) The bean is lazily created on `getBean()`

**Answer: B**
When using the no-arg constructor, you MUST call `refresh()` before calling `getBean()`. Forgetting `refresh()` is a common trap.

---

**Q7 — Select all that apply:**
Which ApplicationContext implementations can be used in a non-web standalone Java application? (Select ALL that apply)

A) `ClassPathXmlApplicationContext`
B) `AnnotationConfigApplicationContext`
C) `XmlWebApplicationContext`
D) `FileSystemXmlApplicationContext`
E) `AnnotationConfigWebApplicationContext`

**Answer: A, B, D**

---

**Q8 — Compilation Error Identification:**

```java
BeanFactory factory = new AnnotationConfigApplicationContext(AppConfig.class);
factory.registerShutdownHook(); // Line X
```

What happens at Line X?

A) Compiles and runs fine
B) Compile error — `registerShutdownHook()` is not on `BeanFactory`
C) Runtime exception
D) The method runs but does nothing

**Answer: B**
`registerShutdownHook()` is defined on `ConfigurableApplicationContext`, not on `BeanFactory`. You must cast to `ConfigurableApplicationContext` to call it.

---

## ④ TRICK ANALYSIS

**Trap 1: "BeanFactory is lazy" vs "ApplicationContext is eager"**
Exam questions love to ask about bean instantiation timing. BeanFactory instantiates beans lazily (on `getBean()`). ApplicationContext instantiates all singletons eagerly at context refresh. Prototype beans are ALWAYS instantiated lazily regardless of container type.

**Trap 2: Spring singleton ≠ JVM singleton**
This is probably the most tested misconception. Options like "one per JVM" or "one per ClassLoader" are always wrong. It's always "one per ApplicationContext instance."

**Trap 3: No-arg constructor + no refresh()**
Any question showing `new AnnotationConfigApplicationContext()` followed immediately by `getBean()` without `refresh()` is a trap. It will throw `IllegalStateException`.

**Trap 4: `<context:annotation-config>` vs `<context:component-scan>`**
A question may offer `<context:annotation-config>` as a way to enable component scanning. It does NOT scan. It only registers annotation processors (`AutowiredAnnotationBeanPostProcessor` etc.) to process annotations on beans already registered in the context.

**Trap 5: BeanFactory vs ApplicationContext BPP auto-detection**
`BeanFactory` does NOT auto-detect and auto-register `BeanPostProcessor` and `BeanFactoryPostProcessor` beans. You must manually register them. `ApplicationContext` does this automatically. This is a critical differentiator.

**Trap 6: `getBean(Class)` with multiple beans**
If you have two beans of the same type and call `context.getBean(SomeType.class)`, it throws `NoUniqueBeanDefinitionException`, NOT `NoSuchBeanDefinitionException`. The distinction matters.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Spring singleton scope | One instance **per ApplicationContext**, not per JVM |
| BeanFactory default | **Lazy** initialization |
| ApplicationContext default | **Eager** initialization of singletons |
| No-arg constructor | Must manually call `refresh()` |
| `registerShutdownHook()` | On `ConfigurableApplicationContext`, not `BeanFactory` |
| BPP auto-detection | `ApplicationContext` only, not plain `BeanFactory` |
| `getBean(Class)` — multiple candidates | `NoUniqueBeanDefinitionException` |
| Destroy callbacks | **Only fire** if context is closed or shutdown hook registered |

---

### ApplicationContext Hierarchy (simplified)

```
BeanFactory
    └── HierarchicalBeanFactory
            └── ListableBeanFactory
                    └── ApplicationContext
                            ├── ConfigurableApplicationContext  ← adds refresh(), close(), registerShutdownHook()
                            │        └── AbstractApplicationContext
                            │                 ├── ClassPathXmlApplicationContext
                            │                 ├── FileSystemXmlApplicationContext
                            │                 └── AnnotationConfigApplicationContext
                            └── WebApplicationContext
```

---

### Container Initialization Order (Memorize This)

```
1.  Read configuration metadata
2.  Register BeanDefinitions in BeanDefinitionRegistry
3.  Run BeanFactoryPostProcessors (modify BeanDefinitions)
4.  Register BeanPostProcessors
5.  Initialize MessageSource
6.  Initialize ApplicationEventMulticaster
7.  Instantiate singleton beans (finishBeanFactoryInitialization)
       Per bean:
       a. Instantiate (constructor)
       b. Populate properties (DI)
       c. Aware callbacks
       d. BPP.postProcessBeforeInitialization
       e. @PostConstruct / afterPropertiesSet() / init-method
       f. BPP.postProcessAfterInitialization
8.  Publish ContextRefreshedEvent
--- SHUTDOWN ---
9.  @PreDestroy / destroy() / destroy-method (reverse order)
10. Publish ContextClosedEvent
```

---

### Interview One-Liners

- "Spring IoC container implements the Dependency Injection pattern to achieve Inversion of Control."
- "ApplicationContext is a sub-interface of BeanFactory that adds enterprise features and eagerly instantiates singletons."
- "Spring singleton is per-container, not per-JVM."
- "The `refresh()` method in `AbstractApplicationContext` is the heart of container initialization."
- "BeanFactory requires manual registration of BPPs; ApplicationContext auto-detects them."
- "Always call `registerShutdownHook()` in standalone apps to ensure destroy callbacks fire."

---
