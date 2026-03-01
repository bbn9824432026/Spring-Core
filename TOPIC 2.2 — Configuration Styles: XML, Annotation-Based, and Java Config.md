# TOPIC 2.2 — Configuration Styles: XML, Annotation-Based, and Java Config

---

## ① CONCEPTUAL EXPLANATION

### Overview — Three Configuration Paradigms

Spring supports three distinct configuration styles, each producing the same end result — `BeanDefinition` objects registered in a `BeanDefinitionRegistry`. The style is purely a matter of how you express configuration metadata. The container behavior is identical regardless of which style you use.

```
XML Configuration          → XmlBeanDefinitionReader
                                    │
Annotation Configuration   → ClassPathBeanDefinitionScanner        → BeanDefinitionRegistry
                                    │
Java Config (@Configuration)→ ConfigurationClassPostProcessor
```

All three styles can be freely mixed in a single application. Understanding the internal processing mechanism for each style is critical for architect-level mastery.

---

### 2.2.1 — XML-Based Configuration (Complete Reference)

XML was Spring's original configuration style. Despite being largely replaced by annotations and Java config in modern applications, it remains important for:
- Legacy enterprise codebases
- Externalizing configuration from code
- Frameworks that generate Spring config programmatically
- Certification exam coverage

#### Namespace Declarations

Spring's XML schema is extensible through namespaces. Each namespace provides shorthand for common configurations:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop.xsd
           http://www.springframework.org/schema/tx
           http://www.springframework.org/schema/tx/spring-tx.xsd
           http://www.springframework.org/schema/util
           http://www.springframework.org/schema/util/spring-util.xsd">
```

Each namespace has a handler (`NamespaceHandler`) that Spring discovers via `META-INF/spring.handlers` files on the classpath. When the XML parser encounters `<context:component-scan>`, it delegates to `ContextNamespaceHandler` which registers the appropriate bean definitions.

---

#### Core `<bean>` Element — All Attributes

```xml
<bean
    id="orderService"                    <!-- primary name -->
    name="orderSvc,os"                   <!-- additional aliases (comma/semicolon/space separated) -->
    class="com.example.OrderService"     <!-- fully qualified class name -->
    scope="singleton"                    <!-- singleton | prototype | request | session | ... -->
    lazy-init="false"                    <!-- true | false | default -->
    abstract="false"                     <!-- true = template only, not instantiated -->
    parent="baseService"                 <!-- inherit from parent bean definition -->
    init-method="init"                   <!-- called after properties set -->
    destroy-method="cleanup"             <!-- called on context close -->
    factory-bean="serviceFactory"        <!-- delegate instantiation to FactoryBean -->
    factory-method="createService"       <!-- static or instance factory method -->
    depends-on="dataSource,cacheManager" <!-- force init order -->
    autowire="no"                        <!-- no | byName | byType | constructor | autodetect -->
    autowire-candidate="true"            <!-- eligible for type-based autowiring? -->
    primary="false"                      <!-- preferred candidate for autowiring? -->
>
```

---

#### Constructor Injection in XML

```xml
<bean id="orderService" class="com.example.OrderService">
    <!-- By index (0-based) -->
    <constructor-arg index="0" value="5000"/>
    <constructor-arg index="1" ref="paymentService"/>

    <!-- By type -->
    <constructor-arg type="int" value="5000"/>
    <constructor-arg type="com.example.PaymentService" ref="paymentService"/>

    <!-- By name (requires -parameters compiler flag or @ConstructorProperties) -->
    <constructor-arg name="timeout" value="5000"/>
    <constructor-arg name="paymentService" ref="paymentService"/>
</bean>
```

---

#### Setter Injection in XML

```xml
<bean id="orderService" class="com.example.OrderService">
    <property name="timeout" value="5000"/>
    <property name="paymentService" ref="paymentService"/>
    <property name="tags">
        <list>
            <value>premium</value>
            <value>urgent</value>
        </list>
    </property>
    <property name="config">
        <map>
            <entry key="retries" value="3"/>
            <entry key="timeout" value="5000"/>
        </map>
    </property>
    <property name="dbProperties">
        <props>
            <prop key="url">jdbc:mysql://localhost/db</prop>
            <prop key="username">root</prop>
        </props>
    </property>
    <property name="allowedStatuses">
        <set>
            <value>PENDING</value>
            <value>CONFIRMED</value>
        </set>
    </property>
    <property name="nullProperty"><null/></property>  <!-- explicitly set null -->
</bean>
```

---

#### p-namespace and c-namespace (Shorthand)

The `p:` namespace is shorthand for `<property>` elements:

```xml
<!-- Traditional -->
<bean id="orderService" class="com.example.OrderService">
    <property name="timeout" value="5000"/>
    <property name="paymentService" ref="paymentService"/>
</bean>

<!-- p-namespace equivalent -->
<bean id="orderService"
      class="com.example.OrderService"
      p:timeout="5000"
      p:paymentService-ref="paymentService"/>
<!-- Note: -ref suffix for bean references -->
```

The `c:` namespace is shorthand for `<constructor-arg>` elements:

```xml
<!-- Traditional -->
<bean id="orderService" class="com.example.OrderService">
    <constructor-arg name="timeout" value="5000"/>
    <constructor-arg name="paymentService" ref="paymentService"/>
</bean>

<!-- c-namespace equivalent -->
<bean id="orderService"
      class="com.example.OrderService"
      c:timeout="5000"
      c:paymentService-ref="paymentService"/>
<!-- OR by index: -->
<bean id="orderService"
      class="com.example.OrderService"
      c:_0="5000"
      c:_1-ref="paymentService"/>
```

**Important:** `c-namespace` requires the parameter names to be available (compiled with `-parameters` flag or `@ConstructorProperties`). Using index (`_0`, `_1`) is always safe.

---

#### util namespace

```xml
<!-- Define a standalone List bean -->
<util:list id="adminEmails" value-type="java.lang.String">
    <value>admin@example.com</value>
    <value>ops@example.com</value>
</util:list>

<!-- Define a standalone Map bean -->
<util:map id="currencyRates">
    <entry key="USD" value="1.0"/>
    <entry key="EUR" value="0.85"/>
</util:map>

<!-- Define a standalone Properties bean -->
<util:properties id="appProps"
                 location="classpath:application.properties"/>

<!-- Define a constant value from a static field -->
<util:constant id="maxRetries"
               static-field="com.example.Constants.MAX_RETRIES"/>
```

---

#### `<import>` for Multi-file Configuration

```xml
<!-- main-context.xml -->
<beans>
    <import resource="classpath:config/services.xml"/>
    <import resource="classpath:config/repositories.xml"/>
    <import resource="file:/opt/app/config/datasource.xml"/>
    <import resource="${config.location}/custom.xml"/>  <!-- property placeholder -->
</beans>
```

`<import>` is resolved relative to the importing file's location (unless absolute path or classpath prefix is used). Circular imports are detected and prevented.

---

#### XML Autowiring Modes

The `autowire` attribute on `<bean>` enables automatic wiring without explicit `<property>` or `<constructor-arg>` elements:

```xml
<bean id="orderService"
      class="com.example.OrderService"
      autowire="byType"/>
<!-- Spring automatically injects all properties whose types match registered beans -->
```

| Mode | Behavior |
|---|---|
| `no` | Default — no autowiring, explicit wiring only |
| `byName` | Match property name to bean name |
| `byType` | Match property type to bean type (fails if multiple matches) |
| `constructor` | Match constructor parameter types to bean types |
| `autodetect` | Deprecated — tries constructor, falls back to byType |

XML-level autowiring is different from annotation-based `@Autowired`. It applies globally to all properties of the bean. `@Autowired` is more granular (field/method level). Modern code uses `@Autowired`, not XML `autowire` attribute.

---

### 2.2.2 — Annotation-Based Configuration

Annotation-based configuration uses class-level and member-level annotations to express the same metadata as XML. The key enabler is `<context:component-scan>` in XML or `@ComponentScan` in Java config.

#### How Component Scanning Works Internally

When `@ComponentScan` (or `<context:component-scan>`) is processed:

1. `ClassPathBeanDefinitionScanner` is created
2. For each package, it scans the classpath using `PathMatchingResourcePatternResolver`
3. Each `.class` file is read using ASM bytecode reader (`ClassReader`) — **without loading the class into the JVM**
4. The ASM-based `MetadataReader` checks for matching annotations
5. Only matching classes are loaded into the JVM and registered as `ScannedGenericBeanDefinition`

**Critical insight:** Spring uses ASM to read bytecode metadata at scan time, avoiding unnecessary class loading. This is why scanning thousands of classes doesn't cause memory explosion — most classes are never actually loaded into the JVM unless they match the filter.

---

#### Stereotype Annotations

```java
@Component   // Generic Spring-managed component
@Service     // Business logic layer (no technical difference — semantic only)
@Repository  // Data access layer (adds exception translation)
@Controller  // Web MVC controller (web-aware behavior)
@RestController // @Controller + @ResponseBody (not part of core Spring)
```

All are `@Component` meta-annotations. `@Repository` is special — it triggers `PersistenceExceptionTranslationPostProcessor` to translate persistence exceptions into Spring's `DataAccessException` hierarchy.

**Bean naming from stereotype annotations:**
```java
@Component              // bean name: "myService" (class name, first letter lowercase)
public class MyService { }

@Component("customName") // bean name: "customName"
public class MyService { }

@Service                // bean name: "paymentService"
public class PaymentService { }
```

The default name is generated by `AnnotationBeanNameGenerator`:
- Takes the class simple name
- Lowercases the first letter
- If the first two letters are both uppercase (e.g., `URLParser`), the name is unchanged: `URLParser`

---

#### `@Autowired` — Resolution Algorithm (Critical)

`@Autowired` is processed by `AutowiredAnnotationBeanPostProcessor`. Resolution follows a strict algorithm:

```
@Autowired DataSource dataSource;
          │
          ▼
Step 1: Find all beans of type DataSource
          │
          ├── 0 beans found → required=true? → NoSuchBeanDefinitionException
          │                 → required=false? → leave null / skip
          │
          ├── 1 bean found → inject it (done)
          │
          └── Multiple beans found →
                    │
                    ▼
              Step 2: Is there a @Primary bean? → inject it
                    │
                    ▼
              Step 3: Is there a @Qualifier matching the field/param name?
                    → inject matching bean
                    │
                    ▼
              Step 4: Does any bean name match the field/param name?
                    → inject by name fallback
                    │
                    ▼
              Step 5: → NoUniqueBeanDefinitionException
```

**Field name fallback (Step 4)** is a critical detail. If you have:
```java
@Autowired
private DataSource primaryDataSource;
```
And there's a bean named `"primaryDataSource"`, Spring will inject it even without `@Qualifier`, because the field name matches the bean name. This is the "name fallback" in type-based resolution.

---

#### `@Qualifier` — Fine-Grained Selection

```java
@Autowired
@Qualifier("primaryDataSource")
private DataSource dataSource;
```

`@Qualifier` narrows the candidate set AFTER type resolution. It matches against:
1. Bean names
2. Qualifier annotations on bean definitions

You can create custom qualifier annotations:
```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Primary { }

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Secondary { }
```

```java
@Component @Primary
public class HikariDataSource implements DataSource { }

@Component @Secondary
public class LegacyDataSource implements DataSource { }

@Service
public class OrderService {
    @Autowired @Primary
    private DataSource dataSource; // injects HikariDataSource
}
```

---

#### `@Primary` — Tie-Breaking

`@Primary` marks a bean as the preferred candidate when multiple beans of the same type exist. It is a tie-breaker — it only activates when multiple candidates are found:

```java
@Bean @Primary
public DataSource primaryDataSource() { return new HikariDataSource(); }

@Bean
public DataSource secondaryDataSource() { return new BasicDataSource(); }
```

```java
@Autowired
private DataSource dataSource; // always injects primaryDataSource
```

**`@Primary` vs `@Qualifier`:** `@Qualifier` is more precise and overrides `@Primary`. If a field has both `@Autowired @Qualifier("secondary")`, the secondary is injected despite another bean being `@Primary`.

---

#### `@Value` — Property Injection

```java
@Component
public class AppConfig {

    @Value("${server.port:8080}")       // property placeholder with default
    private int serverPort;

    @Value("${app.name}")               // property placeholder, no default
    private String appName;

    @Value("#{systemProperties['os.name']}") // SpEL expression
    private String osName;

    @Value("#{T(java.lang.Math).PI}")   // SpEL — static field access
    private double pi;

    @Value("#{orderService.timeout}")   // SpEL — bean property
    private int orderTimeout;
}
```

`@Value` with `${}` requires `PropertySourcesPlaceholderConfigurer` (registered automatically by `<context:property-placeholder>` or when using `@PropertySource`). `@Value` with `#{}` uses SpEL and is always available.

---

#### JSR-250 Annotations — `@Resource`, `@PostConstruct`, `@PreDestroy`

Processed by `CommonAnnotationBeanPostProcessor` (auto-registered by ApplicationContext):

```java
@Component
public class OrderService {

    @Resource(name = "primaryDataSource")  // inject by NAME first, then type
    private DataSource dataSource;

    @Resource  // name defaults to field name: looks for bean "paymentService"
    private PaymentService paymentService;

    @PostConstruct
    public void init() {
        // Called after all dependencies injected
        // Before BPP.postProcessAfterInitialization
    }

    @PreDestroy
    public void cleanup() {
        // Called before bean destruction
    }
}
```

**`@Resource` vs `@Autowired` — Resolution Strategy:**

| | `@Autowired` | `@Resource` |
|---|---|---|
| Primary strategy | **By type** | **By name** |
| Secondary strategy | Name fallback | Type fallback |
| Supports `@Qualifier` | Yes | No (has own `name` attribute) |
| JSR standard | No (Spring) | JSR-250 |
| Processed by | `AutowiredAnnotationBeanPostProcessor` | `CommonAnnotationBeanPostProcessor` |

---

#### JSR-330 — `@Inject` and `@Named`

```java
import javax.inject.Inject;
import javax.inject.Named;
import javax.inject.Singleton;

@Named("orderService")  // equivalent to @Component("orderService")
@Singleton              // equivalent to @Scope("singleton")
public class OrderService {

    @Inject              // equivalent to @Autowired (required=true always)
    private PaymentService paymentService;

    @Inject
    @Named("primaryDataSource")  // equivalent to @Qualifier
    private DataSource dataSource;
}
```

**Key difference:** `@Inject` has no `required` attribute — it always behaves as `@Autowired(required=true)`. If the dependency is not found, it throws immediately. For optional dependencies you need `@Inject Optional<T>`.

---

#### `<context:annotation-config>` vs `<context:component-scan>`

```xml
<!-- Registers BPPs for annotation processing on ALREADY-REGISTERED beans -->
<!-- Does NOT scan packages -->
<context:annotation-config/>

<!-- Scans packages AND registers the same BPPs as annotation-config -->
<!-- Use this instead of annotation-config when scanning -->
<context:component-scan base-package="com.example"/>
```

`<context:annotation-config>` registers:
- `AutowiredAnnotationBeanPostProcessor` — processes `@Autowired`, `@Value`, `@Inject`
- `CommonAnnotationBeanPostProcessor` — processes `@Resource`, `@PostConstruct`, `@PreDestroy`
- `PersistenceAnnotationBeanPostProcessor` — processes `@PersistenceContext`, `@PersistenceUnit`
- `EventListenerMethodProcessor` — processes `@EventListener`
- `DefaultEventListenerFactory`

`<context:component-scan>` does everything `<context:annotation-config>` does, PLUS scans packages for stereotype annotations.

---

### 2.2.3 — Java-Based Configuration (`@Configuration`)

Java config is the modern standard. It provides type-safety, IDE support, and refactoring capability that XML cannot.

#### `@Configuration` — CGLIB Proxy Behavior

This is one of the most important internal mechanisms in Spring:

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // calls serviceB() method
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

**Without CGLIB:** `serviceA()` calls `serviceB()` as a plain Java method call — creates a new `ServiceB` every time. `serviceA` and `serviceB` would get different `ServiceB` instances. Singleton guarantee broken.

**With CGLIB (actual Spring behavior):** Spring subclasses `AppConfig` using CGLIB. The generated subclass overrides every `@Bean` method. When `serviceA()` calls `serviceB()` inside `AppConfig`, the CGLIB override intercepts the call, checks the singleton cache, and returns the existing `ServiceB` instance if it exists.

```
AppConfig (user class)
    └── AppConfig$$SpringCGLIB$$0 (generated CGLIB subclass)
            └── @Bean method overrides → check singleton cache before delegating
```

This means:
- `@Configuration` classes CANNOT be `final` (CGLIB cannot subclass final classes)
- `@Bean` methods in `@Configuration` CANNOT be `final` or `private` (CGLIB cannot override them)
- `@Configuration` classes CANNOT be declared as `static` inner classes (unless they are `static`)

---

#### `@Configuration(proxyBeanMethods = false)` — Lite Mode

```java
@Configuration(proxyBeanMethods = false)  // NO CGLIB subclassing
public class LiteConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // PLAIN Java call — new ServiceB() every time!
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

In lite mode:
- No CGLIB proxy generated — class can be `final`
- Inter-bean method calls are plain Java calls — do NOT check singleton cache
- Better performance (no CGLIB overhead at startup)
- Used by Spring Boot auto-configuration classes for speed
- **You must use dependency injection (constructor/setter) instead of inter-bean calls to maintain singleton guarantee**

---

#### `@Bean` — Full Reference

```java
@Configuration
public class AppConfig {

    // Basic
    @Bean
    public OrderService orderService() {
        return new OrderService();
    }

    // Custom name + aliases
    @Bean(name = {"dataSource", "primaryDS", "mainDB"})
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    // Lifecycle methods
    @Bean(initMethod = "init", destroyMethod = "cleanup")
    public ConnectionPool connectionPool() {
        return new ConnectionPool();
    }

    // Scope
    @Bean
    @Scope("prototype")
    public ReportGenerator reportGenerator() {
        return new ReportGenerator();
    }

    // Conditional
    @Bean
    @Conditional(ProductionCondition.class)
    public AuditService auditService() {
        return new AuditService();
    }

    // Static @Bean — for BFPPs and BPPs that must be created early
    @Bean
    public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    // @Bean with injected parameters
    @Bean
    public OrderService orderService(PaymentService paymentService,
                                     @Value("${order.timeout}") int timeout) {
        return new OrderService(paymentService, timeout);
    }
}
```

**`@Bean` method parameter injection:** When a `@Bean` method has parameters, Spring injects them from the container — as if they were `@Autowired`. This is how `@Bean` methods get their dependencies without directly calling other `@Bean` methods.

**Static `@Bean` methods:** `BeanFactoryPostProcessor` and `BeanPostProcessor` beans MUST be declared as `static @Bean` methods. This is because BFPPs need to be instantiated very early in the container lifecycle — before the `@Configuration` class itself is instantiated (because CGLIB proxying happens after BFPPs run). Non-static BFPPs in `@Configuration` classes trigger a Spring warning and may not work correctly.

---

#### `@Import` — Composing Configurations

```java
@Configuration
@Import({DataConfig.class, ServiceConfig.class, SecurityConfig.class})
public class AppConfig {
    // Main configuration class
}
```

`@Import` can import:
1. **`@Configuration` classes** — their `@Bean` methods are included
2. **`ImportSelector` implementations** — programmatically select which configs to import
3. **`ImportBeanDefinitionRegistrar` implementations** — programmatically register BDs

```java
// ImportSelector example
public class DataModuleSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // Can read annotations from the importing class
        boolean useH2 = importingClassMetadata
            .hasAnnotation("com.example.UseH2");
        return useH2
            ? new String[]{"com.example.H2DataConfig"}
            : new String[]{"com.example.MySQLDataConfig"};
    }
}

@Configuration
@Import(DataModuleSelector.class)
public class AppConfig { }
```

```java
// ImportBeanDefinitionRegistrar example
public class ServiceRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {

        GenericBeanDefinition bd = new GenericBeanDefinition();
        bd.setBeanClass(AuditService.class);
        registry.registerBeanDefinition("auditService", bd);
    }
}
```

`@Import` is processed by `ConfigurationClassPostProcessor` (a `BeanDefinitionRegistryPostProcessor`), which runs in step 5 of `refresh()`. This is also how Spring Boot auto-configuration works — `@EnableAutoConfiguration` triggers an `ImportSelector` that reads `spring.factories` and imports selected auto-configuration classes.

---

#### `@ImportResource` — Mixing XML Into Java Config

```java
@Configuration
@ImportResource({
    "classpath:legacy/services.xml",
    "classpath:legacy/repositories.xml"
})
public class AppConfig {

    @Bean
    public NewModernService newService() {
        return new NewModernService();
    }
}
```

This imports XML bean definitions into a Java-config-based context. The XML beans and `@Bean` beans coexist in the same registry.

---

### 2.2.4 — Mixed Configuration

Spring fully supports mixing all three styles. Common patterns:

**Pattern 1: Java config as root, XML for legacy**
```java
@Configuration
@ImportResource("classpath:legacy-beans.xml")
@ComponentScan("com.example.newcode")
public class AppConfig {
    @Bean
    public NewService newService() { ... }
}
```

**Pattern 2: XML as root, with annotation scan**
```xml
<beans>
    <context:component-scan base-package="com.example"/>
    <bean id="legacyService" class="com.example.LegacyService"/>
    <import resource="datasource.xml"/>
</beans>
```

**Pattern 3: Annotation scan with Java config imported**
```java
@Configuration
@ComponentScan("com.example")
@Import(InfrastructureConfig.class)
@PropertySource("classpath:application.properties")
public class AppConfig { }
```

---

### 2.2.5 — Configuration Metadata Processing Order

Understanding the ORDER in which configuration is processed is critical:

```
ApplicationContext initialization (refresh())
        │
        ▼
Step 1: Register configuration metadata
        ├── XML files parsed by XmlBeanDefinitionReader
        ├── @Configuration classes registered by AnnotatedBeanDefinitionReader
        └── Package scan by ClassPathBeanDefinitionScanner

        ▼
Step 2: invokeBeanFactoryPostProcessors()
        └── ConfigurationClassPostProcessor runs (BeanDefinitionRegistryPostProcessor)
                ├── Processes @Configuration classes
                ├── Processes @ComponentScan → triggers scanning
                ├── Processes @Import → imports other configs
                ├── Processes @ImportResource → imports XML
                ├── Processes @Bean methods → registers BDs
                └── Processes @PropertySource → registers PropertySources

        ▼
Step 3: Other BeanFactoryPostProcessors run
        └── PropertySourcesPlaceholderConfigurer resolves ${} placeholders

        ▼
Step 4: registerBeanPostProcessors()

        ▼
Step 5: finishBeanFactoryInitialization()
        └── All beans instantiated
```

**`ConfigurationClassPostProcessor` is the most important `BeanFactoryPostProcessor` in the Spring framework.** It is what makes Java-based configuration work. It is registered automatically by `AnnotationConfigApplicationContext` and by `<context:annotation-config>`.

---

### Internal Mechanics — How `@Configuration` Classes Become CGLIB Proxies

The transformation happens inside `ConfigurationClassPostProcessor.postProcessBeanFactory()`:

```
1. ConfigurationClassPostProcessor detects all @Configuration classes
2. Calls ConfigurationClassEnhancer.enhance(configClass)
3. CGLIB creates a subclass: AppConfig$$SpringCGLIB$$0
4. The subclass is registered as the bean class (replaces AppConfig in BeanDefinition)
5. When AppConfig bean is instantiated, the CGLIB subclass is instantiated instead
6. CGLIB intercepts @Bean method calls:
       ─ Is there already an instance in the singleton cache? → return it
       ─ Is this a recursive call from within createBean()? → proceed normally
       ─ Otherwise: call beanFactory.getBean() → returns cached or creates new
```

The CGLIB callback is implemented as `BeanMethodInterceptor`. This is why calling a `@Bean` method on a `@Configuration` instance in a debugger or direct Java code (without Spring) will behave differently from when Spring calls it — Spring's CGLIB proxy intercepts and manages the singleton guarantee.

---

### Common Misconceptions

**Misconception 1:** "`@Component` classes are processed the same way as `@Configuration` classes."
Wrong. `@Configuration` classes get CGLIB-proxied. `@Component` classes with `@Bean` methods do NOT get CGLIB-proxied (they use "lite" mode automatically). Inter-bean method calls in `@Component` classes are plain Java calls — no singleton guarantee.

**Misconception 2:** "`<context:annotation-config>` enables component scanning."
Wrong. It only registers annotation processors. You need `<context:component-scan>` for scanning.

**Misconception 3:** "`@Autowired` always injects by type only."
Wrong. When multiple type matches exist, Spring falls back to name matching (using the field/parameter name as the bean name). This name fallback happens BEFORE throwing `NoUniqueBeanDefinitionException`.

**Misconception 4:** "`@Resource` is the same as `@Autowired`."
Wrong. `@Resource` injects BY NAME first (using the `name` attribute or field name), then falls back to type. `@Autowired` injects BY TYPE first, then falls back to name. This order reversal is the critical difference.

**Misconception 5:** "A `@Bean` method in a `@Configuration` class is called directly."
Wrong. CGLIB intercepts the call. The actual instantiation goes through `beanFactory.getBean()`, ensuring singleton caching, BPP processing, and lifecycle management.

**Misconception 6:** "You can use `@Autowired` and `@Resource` together on the same field."
Technically possible but undefined behavior — two different BPPs process them. In practice, `@Autowired` wins because `AutowiredAnnotationBeanPostProcessor` runs first. Never do this.

---

## ② CODE EXAMPLES

### Example 1 — XML vs Annotation vs Java Config — Same Bean, Three Ways

**The class:**
```java
public class OrderService {
    private PaymentService paymentService;
    private int timeout;

    public OrderService() { }

    public OrderService(PaymentService ps, int timeout) {
        this.paymentService = ps;
        this.timeout = timeout;
    }

    public void setPaymentService(PaymentService ps) { this.paymentService = ps; }
    public void setTimeout(int t) { this.timeout = t; }

    public void init() { System.out.println("Initializing OrderService"); }
    public void cleanup() { System.out.println("Cleaning up OrderService"); }
}
```

**XML:**
```xml
<bean id="paymentService" class="com.example.PaymentService"/>

<bean id="orderService"
      class="com.example.OrderService"
      init-method="init"
      destroy-method="cleanup">
    <property name="paymentService" ref="paymentService"/>
    <property name="timeout" value="5000"/>
</bean>
```

**Annotation:**
```java
@Service
public class PaymentService { }

@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;

    @Value("${order.timeout:5000}")
    private int timeout;

    @PostConstruct
    public void init() { System.out.println("Initializing OrderService"); }

    @PreDestroy
    public void cleanup() { System.out.println("Cleaning up OrderService"); }
}
```

**Java Config:**
```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }

    @Bean(initMethod = "init", destroyMethod = "cleanup")
    public OrderService orderService() {
        OrderService os = new OrderService();
        os.setPaymentService(paymentService()); // CGLIB ensures singleton
        os.setTimeout(5000);
        return os;
    }
}
```

---

### Example 2 — CGLIB Proof — Singleton Guarantee in `@Configuration`

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceB serviceB() {
        System.out.println("Creating ServiceB, hashCode: " +
            System.identityHashCode(new ServiceB()));
        return new ServiceB();
    }

    @Bean
    public ServiceA serviceA() {
        ServiceB b = serviceB(); // inter-bean call
        return new ServiceA(b);
    }

    @Bean
    public ServiceC serviceC() {
        ServiceB b = serviceB(); // another inter-bean call
        return new ServiceC(b);
    }
}

ApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

ServiceA a = ctx.getBean(ServiceA.class);
ServiceC c = ctx.getBean(ServiceC.class);
ServiceB bDirect = ctx.getBean(ServiceB.class);

// All three point to the SAME ServiceB instance:
System.out.println(a.getServiceB() == c.getServiceB()); // true
System.out.println(a.getServiceB() == bDirect);         // true
// "Creating ServiceB" printed only ONCE
```

```java
// Now with @Configuration(proxyBeanMethods = false):
@Configuration(proxyBeanMethods = false)
public class LiteConfig {
    @Bean
    public ServiceB serviceB() { return new ServiceB(); }

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // plain call — new ServiceB!
    }
}

// Now a.getServiceB() != ctx.getBean(ServiceB.class) — DIFFERENT instances!
```

---

### Example 3 — `@Autowired` Name Fallback (Critical Trap)

```java
@Configuration
public class DataConfig {

    @Bean
    public DataSource primaryDataSource() {
        return new HikariDataSource();
    }

    @Bean
    public DataSource secondaryDataSource() {
        return new BasicDataSource();
    }
}
```

```java
@Service
public class OrderService {

    @Autowired
    private DataSource primaryDataSource; // name matches bean "primaryDataSource"
    // No @Qualifier needed — name fallback resolves it!

    @Autowired
    private DataSource secondaryDataSource; // name matches "secondaryDataSource"
}
```

Both inject correctly via name fallback despite two `DataSource` beans existing. This is a common source of confusion — people think they need `@Qualifier` but name fallback handles it.

---

### Example 4 — `@Resource` vs `@Autowired` — Behavioral Difference

```java
@Configuration
public class Config {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }      // bean name: "dataSource"

    @Bean
    public DataSource specialDataSource() { return new BasicDataSource(); } // bean name: "specialDataSource"
}
```

```java
@Service
public class ServiceA {
    @Autowired
    private DataSource specialDataSource;
    // BY TYPE first → 2 DataSource beans → name fallback → "specialDataSource" → injects BasicDataSource ✓
}

@Service
public class ServiceB {
    @Resource
    private DataSource specialDataSource;
    // BY NAME first → looks for bean "specialDataSource" → injects BasicDataSource ✓
    // Same result here, but for different reasons
}

@Service
public class ServiceC {
    @Resource(name = "dataSource")
    private DataSource specialDataSource;
    // BY NAME → "dataSource" → injects HikariDataSource
    // Field name irrelevant when @Resource name is explicit
}
```

---

### Example 5 — Static `@Bean` for BeanFactoryPostProcessor

```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig {

    // MUST be static — BFPPs are created before @Configuration class is proxied
    @Bean
    public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    @Value("${server.port}") // Works because PSPC (BFPP) processes ${} before this
    private int serverPort;

    @Bean
    public ServerConfig serverConfig() {
        return new ServerConfig(serverPort);
    }
}
```

If `placeholderConfigurer()` is NOT static, Spring warns:
> "Cannot enhance @Configuration bean definition... not a static method"
> `@Value` injection may fail for properties in the same `@Configuration` class

---

### Example 6 — `@Import` with `ImportSelector`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(DatabaseSelector.class)
public @interface EnableDatabase {
    String type() default "mysql";
}

public class DatabaseSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        Map<String, Object> attrs =
            metadata.getAnnotationAttributes(EnableDatabase.class.getName());
        String type = (String) attrs.get("type");

        return switch (type) {
            case "mysql"    -> new String[]{"com.example.MySQLConfig"};
            case "postgres" -> new String[]{"com.example.PostgresConfig"};
            case "h2"       -> new String[]{"com.example.H2Config"};
            default         -> throw new IllegalArgumentException("Unknown DB: " + type);
        };
    }
}

@Configuration
@EnableDatabase(type = "postgres")
public class AppConfig { }
// Imports PostgresConfig automatically
```

---

### Example 7 — Mixed Configuration (Enterprise Pattern)

```java
@Configuration
@ComponentScan(basePackages = "com.example.services")
@ImportResource("classpath:legacy/repositories.xml")
@Import({SecurityConfig.class, CacheConfig.class})
@PropertySource({
    "classpath:application.properties",
    "classpath:application-${spring.profiles.active}.properties"
})
public class AppConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer configurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    @Bean
    @Primary
    public DataSource primaryDataSource(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        return new HikariDataSource(config);
    }
}
```

---

### Example 8 — `@Autowired` on Collections

```java
// Multiple beans of same type:
@Bean public Validator emailValidator() { return new EmailValidator(); }
@Bean public Validator phoneValidator() { return new PhoneValidator(); }
@Bean public Validator zipValidator()   { return new ZipValidator(); }

// Inject all of them:
@Service
public class CompositeValidatorService {

    @Autowired
    private List<Validator> validators;
    // Injects ALL Validator beans in registration order

    @Autowired
    private Map<String, Validator> validatorMap;
    // Map: key=beanName, value=bean instance
    // {"emailValidator": EmailValidator, "phoneValidator": PhoneValidator, ...}

    @Autowired
    private Set<Validator> validatorSet;
    // All Validator beans as a Set

    @Autowired
    @Qualifier("emailValidator")
    private Validator specificValidator;
    // Only the emailValidator bean
}
```

When injecting `List<T>` or `Set<T>`, beans that implement `Ordered` or are annotated with `@Order` are sorted by their order value. This is how Spring MVC's `HandlerInterceptor` chains are ordered.

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What is the primary reason `@Configuration` classes are CGLIB-proxied?

A) To enable circular dependency resolution
B) To ensure `@Bean` inter-method calls return the same singleton instance
C) To enable lazy initialization of `@Bean` methods
D) To enable AOP on configuration classes

**Answer: B**

---

**Q2 — MCQ:**
A `@Bean` method is declared as `private` in a `@Configuration` class. What happens?

A) Spring ignores the method silently
B) The bean is registered but not available for injection
C) CGLIB cannot override private methods — the bean may be registered but singleton guarantee is broken
D) `BeanDefinitionParsingException` is thrown at startup

**Answer: C**
CGLIB cannot override `private` methods. The `@Bean` method is still processed by `ConfigurationClassPostProcessor` and the bean IS registered. However, the singleton guarantee via inter-bean method calls is broken because CGLIB cannot intercept the private method call. Spring logs a warning.

---

**Q3 — Select all that apply:**
Which of the following are processed by `CommonAnnotationBeanPostProcessor`? (Select ALL that apply)

A) `@Autowired`
B) `@Resource`
C) `@PostConstruct`
D) `@PreDestroy`
E) `@Value`
F) `@Inject`

**Answer: B, C, D**
`@Autowired`, `@Value`, `@Inject` are processed by `AutowiredAnnotationBeanPostProcessor`.

---

**Q4 — Code Output Prediction:**

```java
@Configuration
public class Config {
    private int count = 0;

    @Bean
    public ServiceB serviceB() {
        count++;
        System.out.println("serviceB called, count=" + count);
        return new ServiceB();
    }

    @Bean
    public ServiceA serviceA() {
        serviceB();
        serviceB();
        return new ServiceA();
    }
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class);
```

What is printed?

A) `serviceB called, count=1` (once — CGLIB intercepts repeated calls)
B) `serviceB called, count=1` then `serviceB called, count=2` then `serviceB called, count=3`
C) `serviceB called, count=1` (only the first creation)
D) Nothing — `serviceB()` calls within `serviceA()` are always suppressed

**Answer: C**
CGLIB intercepts all calls to `serviceB()` after the first real creation. The first call creates the bean (`count=1`, prints). All subsequent calls (from `serviceA()`, and the container's own `serviceB` bean creation) return the cached singleton — the `count++` and `println` inside `serviceB()` do NOT execute again. Output: `serviceB called, count=1` exactly once.

---

**Q5 — Trick Question:**

```java
@Component  // NOT @Configuration
public class AppConfig {

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // inter-bean call
    }
}
```

`serviceA` and the `ServiceB` singleton bean — do they share the same `ServiceB` instance?

A) Yes — CGLIB ensures singleton even for `@Component`
B) No — `@Component` classes are NOT CGLIB-proxied; `serviceB()` is a plain Java call
C) Yes — Spring detects the pattern and applies singleton guarantee regardless
D) Depends on whether `@ComponentScan` includes the package

**Answer: B**
`@Component` classes with `@Bean` methods operate in "lite mode." No CGLIB proxy is generated. The `serviceB()` call inside `serviceA()` is a plain Java method call — it creates a NEW `ServiceB` instance that is NOT the singleton managed by the container. `serviceA.getServiceB() != ctx.getBean(ServiceB.class)`.

---

**Q6 — Select all that apply:**
Which statements about `@Autowired` dependency resolution are correct? (Select ALL that apply)

A) Type resolution happens before name resolution
B) `@Qualifier` takes effect after type narrowing
C) If one bean is `@Primary`, it always wins regardless of `@Qualifier`
D) Field name is used as a fallback name when multiple type matches exist
E) `@Autowired(required=false)` leaves the field `null` if no bean found
F) `@Autowired` on a `List<T>` injects all beans of type T

**Answer: A, B, D, E, F**
C is wrong: `@Qualifier` overrides `@Primary`. A field annotated with both `@Autowired @Qualifier("specific")` will inject the qualified bean, ignoring `@Primary`.

---

**Q7 — Compilation/Runtime Error:**

```java
@Configuration
public final class AppConfig {  // final class

    @Bean
    public OrderService orderService() {
        return new OrderService();
    }
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

What happens?

A) Compiles and runs perfectly — `final` has no effect on `@Configuration`
B) `BeanDefinitionParsingException` at startup
C) `CannotLoadBeanClassException` because CGLIB cannot subclass `final` class
D) Works but singleton guarantee for inter-bean calls is lost

**Answer: C**
CGLIB cannot subclass `final` classes. When Spring attempts to create the CGLIB proxy for `AppConfig`, it throws an exception. In Spring 5.2+ with `proxyBeanMethods=false`, `final` is allowed — but for standard `@Configuration`, `final` is forbidden.

---

**Q8 — Scenario:**
A developer has two `DataSource` beans and uses `@Resource` (no `name` attribute) on a field named `dataSource`. There is a bean named `"dataSource"` and another named `"secondaryDataSource"`. What is injected?

A) Throws `NoUniqueBeanDefinitionException` — two `DataSource` beans
B) Injects the `"dataSource"` bean (by name — field name matches bean name)
C) Injects the `@Primary` bean
D) Injects nothing — `@Resource` without `name` is ambiguous

**Answer: B**
`@Resource` without explicit `name` uses the field name as the lookup name. Field is named `dataSource` → looks for bean named `"dataSource"` → finds it → injects it. No ambiguity because name resolution happens first, before type resolution.

---

**Q9 — Drag-and-Drop: Order the `@Autowired` resolution algorithm:**

- Throw `NoUniqueBeanDefinitionException`
- Find all beans of type T
- Check `@Qualifier` annotation
- Check field/param name as fallback
- Check for `@Primary` bean
- Return single match (or throw `NoSuchBeanDefinitionException` if 0)

**Answer:**
1. Find all beans of type T
2. Return single match (or throw `NoSuchBeanDefinitionException` if 0)
3. Check for `@Primary` bean
4. Check `@Qualifier` annotation
5. Check field/param name as fallback
6. Throw `NoUniqueBeanDefinitionException`

---

**Q10 — MCQ:**
Which annotation would you use on a `@Bean` method returning a `BeanFactoryPostProcessor` inside a `@Configuration` class to ensure it works correctly?

A) `@Lazy`
B) `@Primary`
C) `static`
D) `@DependsOn`

**Answer: C**
`BeanFactoryPostProcessor` beans declared in `@Configuration` classes MUST be `static @Bean` methods. Non-static BFPPs in `@Configuration` classes are created too late in the lifecycle — after CGLIB proxying has already occurred — causing Spring to warn and the BFPP to potentially not function correctly.

---

## ④ TRICK ANALYSIS

**Trap 1: `@Component` vs `@Configuration` — lite mode vs full mode**
This is the single most tested distinction in this topic. `@Configuration` → CGLIB proxy → singleton guarantee for inter-bean calls. `@Component` (even with `@Bean` methods) → NO CGLIB → plain Java calls → new instance each time. Any question showing a `@Component` class with inter-bean calls and asking if the singleton contract holds → answer is NO.

**Trap 2: `final` `@Configuration` class**
A `final` class with `@Configuration` causes CGLIB to fail. This is a runtime exception, not a compile error. `final` is a compile-time keyword but the CGLIB failure happens at runtime during `refresh()`.

**Trap 3: `@Resource` name resolution vs `@Autowired` name fallback**
Both end up doing name-based resolution, but in different situations:
- `@Resource` → name is ALWAYS first (either explicit or field name)
- `@Autowired` → name fallback ONLY when type resolution produces multiple candidates
When there's ONE bean of the type, `@Autowired` injects it without ever checking the name. `@Resource` still tries the name first.

**Trap 4: `<context:annotation-config>` does NOT scan**
Common trap: question shows `<context:annotation-config>` and asks if `@Component` classes will be picked up. Answer: NO. Only BPPs for annotation processing on already-registered beans are enabled. No scanning.

**Trap 5: `@Bean` method parameters are container-injected**
A `@Bean` method with parameters does NOT use the constructor of the return type's class. The parameters are injected FROM THE CONTAINER. `@Bean public OrderService orderService(PaymentService ps)` → Spring calls `ctx.getBean(PaymentService.class)` to get `ps`, then calls your method.

**Trap 6: `@Qualifier` overrides `@Primary`**
When both `@Primary` and `@Qualifier` are in play, `@Qualifier` wins. `@Primary` is only the tiebreaker when no `@Qualifier` is present. Any question asking "bean A is `@Primary`, field has `@Qualifier("B")` — what is injected?" → bean B.

**Trap 7: Static `@Bean` for BFPPs**
Non-static BFPP `@Bean` methods in `@Configuration` classes cause a Spring warning and may not apply to all beans. The fix is always `static`. Any exam question showing a non-static BFPP `@Bean` method → problematic, produces a warning, may not work correctly.

**Trap 8: `@Inject` has no `required` attribute**
`@Inject` always behaves as `@Autowired(required=true)`. There is no `@Inject(required=false)`. For optional injection with JSR-330, use `@Inject Optional<T>` or `@Inject Provider<T>`.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `@Configuration` | CGLIB-proxied → singleton guarantee for inter-bean calls |
| `@Component` with `@Bean` | Lite mode → NO CGLIB → plain Java calls → new instance per call |
| `@Configuration(proxyBeanMethods=false)` | Lite mode explicitly → no CGLIB → faster startup |
| `final @Configuration` | Runtime exception — CGLIB cannot subclass `final` |
| `private @Bean` method | CGLIB cannot override → singleton guarantee broken (warning logged) |
| Static `@Bean` for BFPP/BPP | MUST be `static` — created before `@Configuration` is proxied |
| `@Autowired` resolution | Type → Primary → Qualifier → Name fallback → Exception |
| `@Resource` resolution | Name (field name or explicit) → Type fallback |
| `@Qualifier` vs `@Primary` | `@Qualifier` wins over `@Primary` always |
| `<context:annotation-config>` | Registers BPPs only — NO scanning |
| `<context:component-scan>` | Scans + registers BPPs (superset of annotation-config) |
| `@Inject` required | Always required — no `required` attribute |

---

### Configuration Style Comparison

| Aspect | XML | Annotation | Java Config |
|---|---|---|---|
| Type safety | ❌ | Partial | ✅ |
| IDE refactoring | ❌ | Partial | ✅ |
| External config | ✅ | ❌ | ❌ |
| Verbosity | High | Low | Medium |
| Compile-time checks | ❌ | Partial | ✅ |
| CGLIB proxying | N/A | N/A | `@Configuration` only |
| Conditional logic | Limited | Limited | ✅ full Java |
| Mixin/composition | `<import>` | `@ComponentScan` | `@Import` |

---

### `@Autowired` Resolution Algorithm (Memorize)

```
1. Find all beans of type T
       ↓
2. Zero found? → required=true → NoSuchBeanDefinitionException
               → required=false → null/skip
       ↓
3. One found? → inject it ✓
       ↓
4. Multiple found:
       a. @Primary present? → inject @Primary bean ✓
       b. @Qualifier present? → inject qualified bean ✓ (overrides @Primary)
       c. Name fallback: field/param name matches a bean name? → inject it ✓
       d. None of above → NoUniqueBeanDefinitionException ✗
```

---

### BPP Responsibility Map

| BPP | Processes |
|---|---|
| `AutowiredAnnotationBeanPostProcessor` | `@Autowired`, `@Value`, `@Inject` |
| `CommonAnnotationBeanPostProcessor` | `@Resource`, `@PostConstruct`, `@PreDestroy` |
| `PersistenceAnnotationBeanPostProcessor` | `@PersistenceContext`, `@PersistenceUnit` |
| `EventListenerMethodProcessor` | `@EventListener` |

---

### `@Import` Variants

```
@Import(SomeConfig.class)          → imports @Configuration class
@Import(SomeSelector.class)        → ImportSelector → programmatic config selection
@Import(SomeRegistrar.class)       → ImportBeanDefinitionRegistrar → programmatic BD registration
```

---

### Interview One-Liners

- "`@Configuration` classes are CGLIB-proxied to intercept `@Bean` method calls and enforce singleton semantics."
- "`@Component` classes with `@Bean` methods run in lite mode — no CGLIB, no singleton guarantee for inter-bean calls."
- "BeanFactoryPostProcessor `@Bean` methods MUST be `static` — they need to be created before `@Configuration` CGLIB proxying occurs."
- "`@Qualifier` overrides `@Primary` — qualifier is explicit targeting, primary is just a default preference."
- "`@Resource` resolves by name first; `@Autowired` resolves by type first — they are not equivalent."
- "`<context:component-scan>` is a superset of `<context:annotation-config>` — never use both."
- "`ConfigurationClassPostProcessor` is the BFPP that processes `@Configuration`, `@Import`, `@ComponentScan`, and `@Bean` — the foundation of Java-based configuration."
- "In the `@Autowired` resolution algorithm, `@Qualifier` always wins over `@Primary`, and name fallback is the last resort before `NoUniqueBeanDefinitionException`."

---
