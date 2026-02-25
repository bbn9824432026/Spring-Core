# TOPIC 2.1 — Bean Definition Internals

---

## ① CONCEPTUAL EXPLANATION

### What is a BeanDefinition?

Before Spring creates a single object, it first builds a complete **metadata model** of every bean it will manage. This metadata model is encapsulated in the `BeanDefinition` interface. A `BeanDefinition` is Spring's internal representation of a bean — it is the blueprint, not the instance.

Think of the relationship this way:
```
BeanDefinition  →  is to  →  Bean Instance
   (blueprint)               (built object)

Java Class      →  is to  →  Object Instance
   (blueprint)               (new MyClass())
```

The container reads configuration (XML, annotations, Java config), converts everything into `BeanDefinition` objects, stores them in the `BeanDefinitionRegistry`, and only THEN begins instantiation. This two-phase approach is fundamental — it means the entire configuration can be inspected, modified, and validated BEFORE any object is created.

This separation is what makes `BeanFactoryPostProcessor` possible — it operates on `BeanDefinition` objects, modifying metadata before instantiation begins.

---

### BeanDefinition Interface — Full Contract

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    // Scope constants
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";

    // Role constants (framework internal classification)
    int ROLE_APPLICATION = 0;   // user-defined beans
    int ROLE_SUPPORT = 1;       // supporting beans (from @ImportResource, etc.)
    int ROLE_INFRASTRUCTURE = 2; // framework-internal beans (BPPs, etc.)

    // ── Class metadata ──
    void setBeanClassName(@Nullable String beanClassName);
    String getBeanClassName();

    // ── Scope ──
    void setScope(@Nullable String scope);
    String getScope();

    // ── Lazy initialization ──
    void setLazyInit(boolean lazyInit);
    boolean isLazyInit();

    // ── Dependencies ──
    void setDependsOn(@Nullable String... dependsOn);
    String[] getDependsOn();

    // ── Autowiring ──
    void setAutowireCandidate(boolean autowireCandidate);
    boolean isAutowireCandidate();

    // ── Primary ──
    void setPrimary(boolean primary);
    boolean isPrimary();

    // ── Factory ──
    void setFactoryBeanName(@Nullable String factoryBeanName);
    String getFactoryBeanName();
    void setFactoryMethodName(@Nullable String factoryMethodName);
    String getFactoryMethodName();

    // ── Constructor arguments ──
    ConstructorArgumentValues getConstructorArgumentValues();
    boolean hasConstructorArgumentValues();

    // ── Property values ──
    MutablePropertyValues getPropertyValues();
    boolean hasPropertyValues();

    // ── Lifecycle methods ──
    void setInitMethodName(@Nullable String initMethodName);
    String getInitMethodName();
    void setDestroyMethodName(@Nullable String destroyMethodName);
    String getDestroyMethodName();

    // ── Role ──
    void setRole(int role);
    int getRole();

    // ── Description ──
    void setDescription(@Nullable String description);
    String getDescription();

    // ── Derived state ──
    ResolvableType getResolvableType();
    boolean isSingleton();
    boolean isPrototype();
    boolean isAbstract();

    // ── Source ──
    String getResourceDescription();
    BeanDefinition getOriginatingBeanDefinition();
}
```

Every piece of configuration you write — XML attributes, annotation values, `@Bean` method signatures — ultimately maps to fields in a `BeanDefinition` object.

---

### BeanDefinition Implementations — Class Hierarchy

```
BeanDefinition (interface)
└── AbstractBeanDefinition  ← base class with all field storage
        ├── RootBeanDefinition       ← merged, standalone bean definition
        ├── ChildBeanDefinition      ← inherits from a parent BeanDefinition
        └── GenericBeanDefinition    ← general-purpose, used during scanning/config processing
                └── ScannedGenericBeanDefinition  ← created by ClassPathBeanDefinitionScanner
                └── AnnotatedGenericBeanDefinition ← created by AnnotatedBeanDefinitionReader
```

**GenericBeanDefinition** is the mutable working form. It's what gets created when Spring processes `@Component`, `@Bean`, or XML `<bean>` elements.

**RootBeanDefinition** is the final, merged form. Before instantiation, Spring calls `getMergedBeanDefinition()` which resolves parent-child inheritance and produces a `RootBeanDefinition`. The container works exclusively with `RootBeanDefinition` during instantiation. You will see this in stack traces when debugging Spring internals.

**ChildBeanDefinition** is for bean definition inheritance — a child bean inherits properties from a parent bean definition (not object inheritance). This is an XML-era feature.

---

### AbstractBeanDefinition — Internal Fields

`AbstractBeanDefinition` contains all the actual storage. Key fields:

```java
public abstract class AbstractBeanDefinition
        extends BeanMetadataAttributeAccessor
        implements BeanDefinition, Cloneable {

    // Core
    private volatile Object beanClass;          // Class or String class name
    private String scope = SCOPE_DEFAULT;       // "" = singleton effectively
    private boolean abstractFlag = false;
    private Boolean lazyInit;                   // null = use container default

    // Autowiring
    private int autowireMode = AUTOWIRE_NO;     // byName, byType, constructor, etc.
    private boolean autowireCandidate = true;   // eligible for autowiring?
    private boolean primary = false;            // tie-breaker for autowiring?

    // Dependencies
    private String[] dependsOn;

    // Constructor and property injection
    private ConstructorArgumentValues constructorArgumentValues;
    private MutablePropertyValues propertyValues;

    // Lifecycle
    private String initMethodName;
    private String destroyMethodName;
    private boolean enforceInitMethod = true;   // throw if init-method not found?
    private boolean enforceDestroyMethod = true;

    // Factory method
    private String factoryBeanName;
    private String factoryMethodName;

    // Metadata
    private int role = BeanDefinition.ROLE_APPLICATION;
    private String description;
    private Resource resource;

    // Qualifiers (for @Qualifier resolution)
    private final Map<String, AutowireCandidateQualifier> qualifiers =
        new LinkedHashMap<>();

    // Post-processing flags (internal tracking)
    private boolean postProcessed = false;
    volatile boolean beforeInstantiationResolved;
}
```

---

### BeanDefinitionRegistry

`BeanDefinitionRegistry` is the interface for registering and managing `BeanDefinition` objects:

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException;

    void removeBeanDefinition(String beanName)
        throws NoSuchBeanDefinitionException;

    BeanDefinition getBeanDefinition(String beanName)
        throws NoSuchBeanDefinitionException;

    boolean containsBeanDefinition(String beanName);

    String[] getBeanDefinitionNames();

    int getBeanDefinitionCount();

    boolean isBeanNameInUse(String beanName);
}
```

`DefaultListableBeanFactory` implements `BeanDefinitionRegistry`. So does `GenericApplicationContext`. This is why you can call `ctx.registerBeanDefinition(...)` directly on `AnnotationConfigApplicationContext` — it delegates to its internal factory.

---

### How BeanDefinitions Are Created From Each Configuration Style

**From XML:**
```xml
<bean id="orderService"
      class="com.example.OrderService"
      scope="singleton"
      lazy-init="false"
      init-method="init"
      destroy-method="cleanup"
      depends-on="dataSource">
    <property name="timeout" value="5000"/>
    <constructor-arg ref="paymentService"/>
</bean>
```
→ `XmlBeanDefinitionReader` parses this into a `GenericBeanDefinition`
→ `class` → `beanClassName`
→ `scope` → `scope`
→ `init-method` → `initMethodName`
→ `destroy-method` → `destroyMethodName`
→ `depends-on` → `dependsOn[]`
→ `<property>` → `propertyValues`
→ `<constructor-arg>` → `constructorArgumentValues`

**From `@Component` scanning:**
```java
@Component("orderService")
@Scope("prototype")
@Lazy
public class OrderService { }
```
→ `ClassPathBeanDefinitionScanner` creates `ScannedGenericBeanDefinition`
→ Reads class-level annotations to populate the BeanDefinition fields
→ `@Scope` → `scope`
→ `@Lazy` → `lazyInit = true`

**From `@Bean` method:**
```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "init", destroyMethod = "cleanup")
    @Scope("prototype")
    public OrderService orderService() {
        return new OrderService();
    }
}
```
→ `ConfigurationClassBeanDefinitionReader` creates `ConfigurationClassBeanDefinition` (a subtype of `RootBeanDefinition`)
→ `factoryBeanName` → `"appConfig"` (the `@Configuration` class bean name)
→ `factoryMethodName` → `"orderService"` (the `@Bean` method name)
→ `initMethodName` → `"init"`
→ `destroyMethodName` → `"cleanup"`

This explains why `@Bean` beans are created via factory method invocation — Spring calls `appConfig.orderService()` (the factory method) rather than calling `new OrderService()` directly.

---

### Bean Definition Merging — Parent-Child Inheritance

XML supports bean definition inheritance (not Java class inheritance):

```xml
<!-- Parent bean definition — can be abstract (no class instantiation) -->
<bean id="baseService"
      class="com.example.BaseService"
      abstract="true"
      init-method="init"
      scope="singleton">
    <property name="timeout" value="3000"/>
    <property name="retryCount" value="3"/>
</bean>

<!-- Child inherits from parent -->
<bean id="orderService"
      parent="baseService"
      class="com.example.OrderService">
    <!-- Inherits: init-method, scope, timeout, retryCount -->
    <!-- Overrides: -->
    <property name="timeout" value="5000"/>  <!-- overrides parent -->
    <property name="serviceName" value="OrderService"/>  <!-- new property -->
</bean>
```

When Spring processes `orderService`, it calls `getMergedLocalBeanDefinition("orderService")` which:
1. Retrieves `ChildBeanDefinition` for `orderService`
2. Retrieves `RootBeanDefinition` for `baseService`
3. Creates a new `RootBeanDefinition` that merges both — child overrides parent
4. The merged `RootBeanDefinition` is cached in `mergedBeanDefinitions` map

**What gets inherited:**
- `scope`
- `lazyInit`
- `autowireMode`
- `dependsOn`
- `constructorArgumentValues` (merged)
- `propertyValues` (merged — child overrides same-named properties)
- `initMethodName`
- `destroyMethodName`

**What does NOT get inherited:**
- `abstract` flag (child is always concrete)
- `beanClassName` — if child specifies its own class, it overrides parent's class

**Abstract parent beans** — you can define a parent bean without a class (pure template):
```xml
<bean id="baseConfig" abstract="true">
    <property name="maxRetries" value="3"/>
    <property name="timeout" value="5000"/>
</bean>

<bean id="serviceA" parent="baseConfig" class="com.example.ServiceA"/>
<bean id="serviceB" parent="baseConfig" class="com.example.ServiceB"/>
```
Both services inherit the property values. `baseConfig` itself is never instantiated (abstract=true).

---

### Bean Aliases

A bean can have multiple names. The primary name is the bean ID; additional names are aliases:

```xml
<bean id="transactionManager" class="...DataSourceTransactionManager">...</bean>
<alias name="transactionManager" alias="txManager"/>
<alias name="transactionManager" alias="platformTransactionManager"/>
```

All three names resolve to the same singleton instance. Spring's `SimpleAliasRegistry` (parent of `DefaultListableBeanFactory`) maintains an `aliasMap`:

```java
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
// key: alias → value: canonical name (or another alias that resolves to canonical)
```

Aliases can chain: `"txMgr"` → `"txManager"` → `"transactionManager"` (canonical).

In Java config, the `@Bean` annotation's `name` attribute accepts multiple names:
```java
@Bean(name = {"transactionManager", "txManager", "platformTransactionManager"})
public PlatformTransactionManager transactionManager() { ... }
```
The first name is the primary name; subsequent names are registered as aliases.

---

### Inner Beans

An inner bean is a bean definition nested inside another bean's `<property>` or `<constructor-arg>`. It has no ID (or the ID is ignored) and cannot be referenced elsewhere:

```xml
<bean id="orderService" class="com.example.OrderService">
    <property name="paymentGateway">
        <!-- Inner bean — anonymous, cannot be referenced by other beans -->
        <bean class="com.example.StripePaymentGateway">
            <property name="apiKey" value="${stripe.key}"/>
        </bean>
    </property>
</bean>
```

Inner beans are always prototype-like in their creation (always created fresh for each outer bean), even if you specify `scope="singleton"` — their scope is effectively meaningless because they cannot be accessed independently.

In Java config, the equivalent is simply creating an object inline:
```java
@Bean
public OrderService orderService() {
    return new OrderService(new StripePaymentGateway("sk_live_..."));
}
```

---

### BeanDefinition Roles

The `role` field classifies beans for tooling and framework use:

| Role Constant | Value | Meaning |
|---|---|---|
| `ROLE_APPLICATION` | 0 | User-defined application bean |
| `ROLE_SUPPORT` | 1 | Supporting bean (imported config) |
| `ROLE_INFRASTRUCTURE` | 2 | Internal framework infrastructure |

Spring Boot's auto-configuration beans are typically `ROLE_INFRASTRUCTURE`. When you look at your application in Spring Boot Actuator's bean endpoint, infrastructure beans are often filtered out from user-visible listings. Spring's own internal BPPs like `AutowiredAnnotationBeanPostProcessor` have role `ROLE_INFRASTRUCTURE`.

---

### The `autowireCandidate` Flag

The `autowireCandidate=false` flag on a BeanDefinition prevents a bean from being a candidate for autowiring by type. The bean still exists and can be retrieved by name via `getBean("name")`, but it will NOT be injected when Spring resolves `@Autowired` by type:

```xml
<bean id="legacyDataSource"
      class="com.example.LegacyDataSource"
      autowire-candidate="false"/>

<bean id="modernDataSource"
      class="com.example.HikariDataSource"/>
```

Now `@Autowired DataSource` will only find `modernDataSource` — `legacyDataSource` is excluded from type-based resolution, eliminating `NoUniqueBeanDefinitionException`.

This is different from `@Primary` (which still allows both but picks one as winner) — `autowire-candidate=false` completely removes the bean from type-based resolution.

---

### ConstructorArgumentValues

This internal class holds constructor arguments, which can be specified by index or by type:

```xml
<constructor-arg index="0" value="5000"/>
<constructor-arg index="1" ref="paymentService"/>
<!-- OR by type: -->
<constructor-arg type="int" value="5000"/>
<constructor-arg type="com.example.PaymentService" ref="paymentService"/>
<!-- OR by name: -->
<constructor-arg name="timeout" value="5000"/>
```

These are stored in `ConstructorArgumentValues` as indexed and generic (non-indexed) argument maps. During bean instantiation, `ConstructorResolver` uses this information to determine which constructor to invoke and what arguments to pass.

---

### MutablePropertyValues

Holds property injection values (setter injection). Each entry is a `PropertyValue` with a name and a value. Values can be:
- Literal strings (converted via `PropertyEditor` or `ConversionService`)
- Bean references (`RuntimeBeanReference`)
- Other `BeanDefinition` instances (inner beans)
- Collections (`List`, `Set`, `Map`, `Properties`)

```xml
<property name="timeout" value="5000"/>      <!-- string literal → int conversion -->
<property name="paymentService" ref="ps"/>   <!-- RuntimeBeanReference -->
<property name="tags">
    <list>
        <value>urgent</value>
        <value>premium</value>
    </list>
</property>
```

---

### Internal Processing Flow — From Configuration to BeanDefinition

```
Configuration Input
        │
        ▼
┌───────────────────────────────────────┐
│  Configuration Readers / Scanners     │
│  ├── XmlBeanDefinitionReader          │
│  ├── ClassPathBeanDefinitionScanner   │
│  └── AnnotatedBeanDefinitionReader    │
└───────────────────────────────────────┘
        │
        ▼ creates
┌───────────────────────────────────────┐
│  GenericBeanDefinition / Subtype      │
│  (mutable working form)               │
└───────────────────────────────────────┘
        │
        ▼ registered in
┌───────────────────────────────────────┐
│  BeanDefinitionRegistry               │
│  (DefaultListableBeanFactory)         │
│  beanDefinitionMap: ConcurrentHashMap │
└───────────────────────────────────────┘
        │
        ▼ BeanFactoryPostProcessors run (modify BDs)
        │
        ▼ before instantiation: merging
┌───────────────────────────────────────┐
│  getMergedLocalBeanDefinition()       │
│  produces: RootBeanDefinition         │
│  cached in: mergedBeanDefinitions     │
└───────────────────────────────────────┘
        │
        ▼ instantiation uses RootBeanDefinition
┌───────────────────────────────────────┐
│  AbstractAutowireCapableBeanFactory   │
│  .createBean(beanName, mbd, args)     │
└───────────────────────────────────────┘
```

---

### JVM and Reflection Implications

`BeanDefinition` stores the class as either a `Class<?>` object or a `String` class name. The string form is used when the class hasn't been loaded yet (lazy initialization scenario). When the bean is actually instantiated, Spring resolves the class name to a `Class<?>` via `ClassUtils.forName(className, classLoader)`, which triggers JVM class loading.

This has implications for:
- **ClassLoader issues** in OSGi or application server environments — if the wrong classloader is used, `ClassNotFoundException` results
- **Class resolution timing** — with lazy beans, the class might not be loaded until first access
- **Spring's `ClassLoader` caching** — Spring caches class resolution results in `CachedIntrospectionResults`

---

### Common Misconceptions

**Misconception 1:** "`BeanDefinition` IS the bean."
Wrong. `BeanDefinition` is metadata/blueprint. The actual bean instance is a completely separate Java object stored in the singleton cache after creation.

**Misconception 2:** "Modifying a `BeanDefinition` after `refresh()` changes the bean."
Wrong. After `refresh()`, singleton beans are already created. Modifying the `BeanDefinition` will not affect the existing singleton instance. Only for prototype beans would a subsequent `getBean()` use the modified definition — but this is very dangerous and unsupported.

**Misconception 3:** "Abstract bean definitions cannot have a class."
Partially wrong. Abstract bean definitions CAN have a class (they just won't be instantiated). They can also be WITHOUT a class — used purely as property templates.

**Misconception 4:** "Aliases are different beans."
Wrong. Multiple aliases resolve to the SAME singleton instance. `isSingleton("txManager")` and `isSingleton("transactionManager")` both return true, and `getBean("txManager") == getBean("transactionManager")` is true.

**Misconception 5:** "`autowire-candidate=false` makes a bean inaccessible."
Wrong. The bean is still accessible by name via `getBean("beanName")`. It just won't be a candidate for type-based autowiring resolution.

---

## ② CODE EXAMPLES

### Example 1 — Programmatic BeanDefinition Creation and Registration

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext();

// Build BeanDefinition programmatically
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(OrderService.class);
bd.setScope(BeanDefinition.SCOPE_SINGLETON);
bd.setLazyInit(false);
bd.setInitMethodName("initialize");
bd.setDestroyMethodName("cleanup");

// Constructor argument
ConstructorArgumentValues cav = new ConstructorArgumentValues();
cav.addIndexedArgumentValue(0, "ORDER_SERVICE");
bd.setConstructorArgumentValues(cav);

// Property value
MutablePropertyValues pvs = new MutablePropertyValues();
pvs.add("timeout", 5000);
pvs.add("retryCount", 3);
bd.setPropertyValues(pvs);

// Register
ctx.registerBeanDefinition("orderService", bd);
ctx.refresh();

OrderService os = ctx.getBean("orderService", OrderService.class);
```

---

### Example 2 — Inspecting BeanDefinitions at Runtime

```java
@Component
public class BeanDefinitionInspector
        implements ApplicationContextAware, SmartInitializingSingleton {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    @Override
    public void afterSingletonsInstantiated() {
        ConfigurableListableBeanFactory factory =
            ((ConfigurableApplicationContext) ctx).getBeanFactory();

        for (String name : factory.getBeanDefinitionNames()) {
            BeanDefinition bd = factory.getBeanDefinition(name);
            System.out.printf(
                "Bean: %-30s | Scope: %-10s | Lazy: %-5s | Role: %d%n",
                name,
                bd.getScope(),
                bd.isLazyInit(),
                bd.getRole()
            );
        }
    }
}
```

---

### Example 3 — Parent-Child Bean Definition (XML)

```xml
<beans>
    <!-- Abstract parent — never instantiated -->
    <bean id="baseRepository"
          abstract="true"
          init-method="init"
          destroy-method="destroy">
        <property name="dataSource" ref="dataSource"/>
        <property name="fetchSize" value="100"/>
    </bean>

    <!-- Child — inherits dataSource, fetchSize, init-method, destroy-method -->
    <bean id="orderRepository"
          parent="baseRepository"
          class="com.example.OrderRepository">
        <property name="fetchSize" value="200"/>  <!-- override -->
        <property name="tableName" value="orders"/>  <!-- new -->
    </bean>

    <!-- Child — inherits everything from baseRepository -->
    <bean id="productRepository"
          parent="baseRepository"
          class="com.example.ProductRepository"/>
</beans>
```

---

### Example 4 — Bean Aliases

```xml
<!-- XML aliases -->
<bean id="transactionManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<alias name="transactionManager" alias="txManager"/>
<alias name="transactionManager" alias="tx"/>
```

```java
// Java config aliases
@Bean(name = {"transactionManager", "txManager", "tx"})
public PlatformTransactionManager transactionManager(DataSource ds) {
    return new DataSourceTransactionManager(ds);
}
```

```java
// All three resolve to same instance:
PlatformTransactionManager tm1 = ctx.getBean("transactionManager", PlatformTransactionManager.class);
PlatformTransactionManager tm2 = ctx.getBean("txManager", PlatformTransactionManager.class);
PlatformTransactionManager tm3 = ctx.getBean("tx", PlatformTransactionManager.class);

System.out.println(tm1 == tm2); // true
System.out.println(tm2 == tm3); // true

// Get all aliases for a bean:
String[] aliases = ctx.getAliases("transactionManager");
// ["txManager", "tx"]
```

---

### Example 5 — `autowire-candidate` Flag

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource primaryDataSource() {
        return new HikariDataSource(primaryConfig());
    }

    @Bean(autowireCandidate = false)  // excluded from type-based autowiring
    public DataSource legacyDataSource() {
        return new BasicDataSource();
    }
}
```

```java
@Service
public class OrderService {
    @Autowired
    private DataSource dataSource;
    // Injects primaryDataSource — legacyDataSource is excluded
    // No NoUniqueBeanDefinitionException
}
```

```java
// But still accessible by name:
DataSource legacy = ctx.getBean("legacyDataSource", DataSource.class); // works
```

---

### Example 6 — Modifying BeanDefinition via BeanFactoryPostProcessor

```java
@Component
public class TimeoutOverrideBFPP implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(
            ConfigurableListableBeanFactory factory) throws BeansException {

        BeanDefinition bd = factory.getBeanDefinition("orderService");

        // Modify scope before instantiation
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);

        // Override a property value
        bd.getPropertyValues().add("timeout", 10000);

        // Change the init method
        bd.setInitMethodName("customInit");

        System.out.println("Modified orderService BeanDefinition");
    }
}
// This runs BEFORE any beans are instantiated
// The modification affects all subsequent bean creation
```

---

### Example 7 — Inner Bean (XML)

```xml
<bean id="orderProcessor" class="com.example.OrderProcessor">
    <property name="validator">
        <!-- Inner bean — no ID, not accessible externally -->
        <bean class="com.example.OrderValidator">
            <property name="strictMode" value="true"/>
        </bean>
    </property>
</bean>
```

```java
// Cannot do this — inner bean has no accessible ID:
ctx.getBean("???"); // no way to reference the inner OrderValidator
// Only accessible through orderProcessor.getValidator()
```

---

### Example 8 — RootBeanDefinition vs GenericBeanDefinition (Internal Perspective)

```java
ConfigurableListableBeanFactory factory =
    ((ConfigurableApplicationContext) ctx).getBeanFactory();

// getBeanDefinition() returns the REGISTERED definition
// (could be GenericBeanDefinition, ScannedGenericBeanDefinition, etc.)
BeanDefinition registered = factory.getBeanDefinition("orderService");
System.out.println(registered.getClass().getSimpleName());
// "ScannedGenericBeanDefinition" (if from @Component)
// "ConfigurationClassBeanDefinition" (if from @Bean)
// "GenericBeanDefinition" (if registered programmatically)

// getMergedBeanDefinition() returns the MERGED RootBeanDefinition
// (always RootBeanDefinition, always the final resolved form)
BeanDefinition merged = factory.getMergedBeanDefinition("orderService");
System.out.println(merged.getClass().getSimpleName());
// Always "RootBeanDefinition"
```

---

### Example 9 — Edge Case: `abstract=true` Without Class

```xml
<!-- Pure template — no class, just properties -->
<bean id="commonConfig" abstract="true">
    <property name="timeout" value="3000"/>
    <property name="maxRetries" value="3"/>
    <property name="environment" value="${app.env}"/>
</bean>

<bean id="emailService"
      parent="commonConfig"
      class="com.example.EmailService">
    <property name="smtpHost" value="smtp.example.com"/>
</bean>

<bean id="smsService"
      parent="commonConfig"
      class="com.example.SmsService">
    <property name="apiEndpoint" value="https://sms.example.com"/>
</bean>
```

Attempting to do `ctx.getBean("commonConfig")` throws `BeanIsAbstractException`.

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What is the concrete class always returned by `getMergedBeanDefinition()`, regardless of how the bean was originally defined?

A) `GenericBeanDefinition`
B) `ChildBeanDefinition`
C) `RootBeanDefinition`
D) `ScannedGenericBeanDefinition`

**Answer: C**
`getMergedBeanDefinition()` always returns a `RootBeanDefinition` — the final, fully resolved, merged form used for instantiation.

---

**Q2 — MCQ:**
A bean is defined with `autowire-candidate="false"`. Which statement is correct?

A) The bean cannot be retrieved by name via `getBean()`
B) The bean is destroyed immediately after creation
C) The bean is excluded from type-based autowiring but is still retrievable by name
D) The bean behaves as a prototype scope

**Answer: C**

---

**Q3 — Select all that apply:**
Which of the following are inherited by a child bean definition from its parent? (Select ALL that apply)

A) `scope`
B) `initMethodName`
C) `beanClassName` (when child specifies its own class)
D) `propertyValues` (merged, child overrides parent for same names)
E) `abstract` flag
F) `lazyInit`

**Answer: A, B, D, F**
When a child specifies its own class (C), it overrides — not inherits — the parent's class. The `abstract` flag (E) is never inherited — child beans are always concrete.

---

**Q4 — Code Output Prediction:**

```java
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(MyService.class);
bd.setScope("singleton");

AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext();
ctx.registerBeanDefinition("myService", bd);
ctx.refresh();

BeanDefinition retrieved = ctx.getBeanFactory()
                              .getMergedBeanDefinition("myService");
System.out.println(retrieved.getClass().getSimpleName());
System.out.println(retrieved.isSingleton());
```

What is the output?

A)
```
GenericBeanDefinition
true
```
B)
```
RootBeanDefinition
true
```
C)
```
AbstractBeanDefinition
true
```
D) Runtime exception

**Answer: B**
`getMergedBeanDefinition()` always returns `RootBeanDefinition`. `isSingleton()` returns true because scope is "singleton".

---

**Q5 — Trick Question:**

```java
@Bean(name = {"dataSource", "primaryDataSource", "mainDS"})
public DataSource dataSource() {
    return new HikariDataSource();
}
```

How many `BeanDefinition` objects are registered for this `@Bean` method?

A) 3 — one per name
B) 2 — one for the primary name and one for aliases combined
C) 1 — one BeanDefinition with 2 aliases registered
D) Depends on `@Primary`

**Answer: C**
One `BeanDefinition` is registered under the primary name `"dataSource"`. The additional names (`"primaryDataSource"`, `"mainDS"`) are registered as aliases in the `AliasRegistry`. All three resolve to the same definition and same singleton instance.

---

**Q6 — Scenario-based:**
A developer calls `ctx.getBean("baseService")` where `baseService` is defined as `abstract="true"` in XML. What happens?

A) Returns null
B) `BeanIsAbstractException` is thrown
C) Creates and returns an instance using the class specified in the definition
D) `NoSuchBeanDefinitionException` is thrown

**Answer: B**
Abstract bean definitions cannot be instantiated. Spring throws `BeanIsAbstractException` which is a subclass of `BeansException`.

---

**Q7 — Select all that apply:**
Which BeanDefinition implementations are created automatically by Spring's scanning infrastructure? (Select ALL that apply)

A) `RootBeanDefinition`
B) `ScannedGenericBeanDefinition`
C) `AnnotatedGenericBeanDefinition`
D) `ChildBeanDefinition`
E) `ConfigurationClassBeanDefinition` (internal subtype of RootBeanDefinition)

**Answer: B, C, E**
`ScannedGenericBeanDefinition` is created by `ClassPathBeanDefinitionScanner` for `@Component` classes.
`AnnotatedGenericBeanDefinition` is created by `AnnotatedBeanDefinitionReader` for explicitly registered `@Configuration` classes.
`ConfigurationClassBeanDefinition` is created internally for `@Bean` methods.

---

**Q8 — Compilation/Runtime Error:**

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// AppConfig has: @Bean public OrderService orderService() {...}

BeanDefinition bd = ctx.getBeanDefinition("orderService"); // Line X
```

What happens at Line X?

A) Returns the `BeanDefinition` for `orderService`
B) Compile error — `getBeanDefinition()` not on `ApplicationContext`
C) `NoSuchBeanDefinitionException`
D) Returns `null`

**Answer: B**
`getBeanDefinition()` is defined on `ConfigurableListableBeanFactory`, not on `ApplicationContext`. You must call `ctx.getBeanFactory().getBeanDefinition("orderService")`.

---

**Q9 — Scenario: BeanDefinition Modification After Refresh**

```java
ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

BeanDefinition bd = ctx.getBeanFactory().getBeanDefinition("orderService");
bd.setScope("prototype"); // Modify AFTER refresh

OrderService a = ctx.getBean("orderService", OrderService.class);
OrderService b = ctx.getBean("orderService", OrderService.class);

System.out.println(a == b);
```

What is the output?

A) `false` — scope was changed to prototype
B) `true` — singleton was already created; scope change has no effect
C) `IllegalStateException` — cannot modify BeanDefinition after refresh
D) Unpredictable

**Answer: B**
The singleton bean was created during `refresh()`. The BeanDefinition is a metadata store — modifying it after instantiation does not retroactively change the singleton cache. The existing singleton instance is returned from the cache. (Note: in a properly locked production context, modifying BDs post-refresh is unsupported and undefined behavior — but for exam purposes: the singleton is already there.)

---

**Q10 — Drag-and-Drop: Order the BeanDefinition processing stages:**

- `BeanFactoryPostProcessor` modifies BDs
- `getMergedBeanDefinition()` produces `RootBeanDefinition`
- XML/Annotation/Java config parsed
- `BeanDefinition` registered in `BeanDefinitionRegistry`
- Bean instantiated using `RootBeanDefinition`

**Answer:**
1. XML/Annotation/Java config parsed
2. `BeanDefinition` registered in `BeanDefinitionRegistry`
3. `BeanFactoryPostProcessor` modifies BDs
4. `getMergedBeanDefinition()` produces `RootBeanDefinition`
5. Bean instantiated using `RootBeanDefinition`

---

## ④ TRICK ANALYSIS

**Trap 1: `getBeanDefinition()` is NOT on ApplicationContext**
Many exam questions show `ctx.getBeanDefinition("name")` on an `ApplicationContext` reference. This is a compile error. It's on `ConfigurableListableBeanFactory`, accessed via `ctx.getBeanFactory().getBeanDefinition("name")`.

**Trap 2: `getMergedBeanDefinition()` always returns `RootBeanDefinition`**
Whether a bean was defined via XML, `@Component`, `@Bean`, or programmatically, `getMergedBeanDefinition()` ALWAYS returns `RootBeanDefinition`. Any question offering `GenericBeanDefinition` or `ChildBeanDefinition` as the return type of `getMergedBeanDefinition()` is wrong.

**Trap 3: Aliases ≠ Multiple BeanDefinitions**
Multiple names on `@Bean` or `<alias>` elements create ONE `BeanDefinition` + multiple alias registrations. Not multiple definitions. This affects `getBeanDefinitionCount()` — it counts BeanDefinitions only, not aliases.

**Trap 4: `abstract=true` beans throw `BeanIsAbstractException`, not `NoSuchBeanDefinitionException`**
The bean EXISTS (there's a BeanDefinition for it), but it's marked as not instantiatable. The exception is `BeanIsAbstractException`. Questions offering `NoSuchBeanDefinitionException` for abstract beans are wrong.

**Trap 5: `autowire-candidate=false` vs `@Primary`**
These are often offered as alternatives in questions. Key distinction:
- `autowire-candidate=false` → completely excluded from type-based resolution
- `@Primary` → still a candidate, but wins the tie-breaking competition
- Both can coexist: you can have `@Primary` on bean A and `autowire-candidate=false` on bean B — A wins by primary, B is excluded

**Trap 6: Child bean definition class override**
When a child bean specifies its own `class`, it overrides the parent's class. The child does NOT inherit the parent's class. When a child does NOT specify `class`, it inherits the parent's class. Questions may show parent with class X and child with class Y and ask "what class is instantiated?" — always Y when child specifies class.

**Trap 7: Inner bean scope**
Even if you write `scope="singleton"` on an inner bean, it behaves like a prototype — a new instance is created each time the outer bean requests it, because inner beans are not registered independently in the container.

**Trap 8: Modifying BeanDefinition post-refresh**
The container does NOT prevent you from modifying `BeanDefinition` objects after `refresh()` — but changes have no effect on already-created singleton instances. Only new `getBean()` calls for prototype beans would pick up the change. This is unsupported behavior. Exam questions asking "what is the effect of modifying a BD after refresh on singletons?" → no effect.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `BeanDefinition` = blueprint | Metadata only — not the instance |
| Working form | `GenericBeanDefinition` (and subtypes) |
| Final form for instantiation | `RootBeanDefinition` (from `getMergedBeanDefinition()`) |
| `getBeanDefinition()` location | `ConfigurableListableBeanFactory` — NOT `ApplicationContext` |
| Abstract bean instantiation | Throws `BeanIsAbstractException` |
| Alias count | 1 `BeanDefinition` + N aliases (not N definitions) |
| `autowire-candidate=false` | Excluded from type-based autowiring; accessible by name |
| Inner bean scope | Effectively prototype regardless of declared scope |
| BD modification post-refresh | No effect on existing singletons |
| Parent-child merging | `getMergedBeanDefinition()` → child overrides parent; result is always `RootBeanDefinition` |

---

### BeanDefinition Key Fields — Quick Reference

```
beanClassName      → what class to instantiate
scope              → singleton / prototype / custom
lazyInit           → defer until first request?
dependsOn          → force initialization order
autowireCandidate  → eligible for @Autowired resolution?
primary            → preferred when multiple candidates?
factoryBeanName    → for @Bean: the @Configuration class name
factoryMethodName  → for @Bean: the @Bean method name
constructorArgs    → ConstructorArgumentValues
propertyValues     → MutablePropertyValues (setter injection)
initMethodName     → called after properties set
destroyMethodName  → called on context close
role               → APPLICATION(0) / SUPPORT(1) / INFRASTRUCTURE(2)
```

---

### BeanDefinition Class Hierarchy

```
BeanDefinition (interface)
└── AbstractBeanDefinition (abstract — all field storage)
        ├── GenericBeanDefinition         ← programmatic / XML
        │       ├── ScannedGenericBeanDefinition   ← @Component scanning
        │       └── AnnotatedGenericBeanDefinition ← explicit @Configuration register
        ├── RootBeanDefinition            ← merged form / @Bean methods
        └── ChildBeanDefinition           ← XML parent-child inheritance
```

---

### Processing Pipeline (Memorize)

```
Config Input → BeanDefinitionReader/Scanner
            → GenericBeanDefinition registered in Registry
            → BeanFactoryPostProcessors modify BDs
            → getMergedBeanDefinition() → RootBeanDefinition
            → AbstractAutowireCapableBeanFactory.createBean()
            → Singleton stored in singletonObjects cache
```

---

### Interview One-Liners

- "`BeanDefinition` is the blueprint; the bean instance is the product — they are completely separate objects."
- "`getMergedBeanDefinition()` always returns `RootBeanDefinition` — the final resolved form ready for instantiation."
- "Modifying a `BeanDefinition` must be done in a `BeanFactoryPostProcessor` — before instantiation begins."
- "`autowire-candidate=false` excludes a bean from type-based resolution but still allows name-based `getBean()` access."
- "Bean aliases share one `BeanDefinition`; `getBeanDefinitionCount()` counts definitions, not aliases."
- "Inner beans behave as prototypes regardless of declared scope — they cannot be accessed independently."
- "Abstract bean definitions serve as property templates; instantiating them throws `BeanIsAbstractException`."
- "The `role` field classifies beans as APPLICATION, SUPPORT, or INFRASTRUCTURE — used by tooling and actuator endpoints."

---
