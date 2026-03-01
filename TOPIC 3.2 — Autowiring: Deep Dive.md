# TOPIC 3.2 — Autowiring: Deep Dive

---

## ① CONCEPTUAL EXPLANATION

### What is Autowiring?

Autowiring is Spring's mechanism for automatically resolving and injecting dependencies without explicit wiring configuration. Instead of specifying exactly which bean goes where, you declare the *intent* (type, name, qualifier) and Spring resolves the dependency from its registry.

Autowiring exists at two levels in Spring:

```
Level 1: XML autowire attribute (legacy, coarse-grained)
         → applies to ALL properties of a bean at once
         → modes: no | byName | byType | constructor | autodetect

Level 2: Annotation-based autowiring (modern, fine-grained)
         → @Autowired, @Qualifier, @Primary, @Resource, @Inject
         → applies to specific fields/methods/constructors
         → processed by BeanPostProcessors
```

In modern Spring, "autowiring" almost always means annotation-based autowiring. Understanding the internal resolution algorithm and every edge case is critical.

---

### 3.2.1 — XML Autowiring Modes

XML-level autowiring is configured via the `autowire` attribute on `<bean>`:

#### `autowire="no"` (default)
No automatic wiring. All dependencies must be explicitly configured via `<property>` or `<constructor-arg>`.

#### `autowire="byName"`
Spring looks for a bean whose **name** matches the **property name**:

```xml
<bean id="orderService" class="com.example.OrderService" autowire="byName"/>
```

```java
public class OrderService {
    private PaymentService paymentService; // Spring looks for bean named "paymentService"
    private DataSource dataSource;         // Spring looks for bean named "dataSource"

    public void setPaymentService(PaymentService ps) { this.paymentService = ps; }
    public void setDataSource(DataSource ds) { this.dataSource = ds; }
}
```

Requirements: setter methods must exist. The bean name must exactly match the property name. No type checking — if the types don't match, a `BeanNotOfRequiredTypeException` is thrown at injection time.

#### `autowire="byType"`
Spring looks for a bean whose **type** matches the **property type**. Throws `NoUniqueBeanDefinitionException` if multiple beans of the same type exist (unless `@Primary` or `autowire-candidate=false` disambiguates):

```xml
<bean id="orderService" class="com.example.OrderService" autowire="byType"/>
```

#### `autowire="constructor"`
Same as `byType` but applies to constructor parameters instead of setter properties:

```xml
<bean id="orderService" class="com.example.OrderService" autowire="constructor"/>
```

#### `autowire="autodetect"` (deprecated in Spring 3.0, removed in Spring 6)
Spring first tries `constructor`, then falls back to `byType`. Never use this.

**Default autowiring mode for the entire XML file:**
```xml
<beans default-autowire="byType">  <!-- applies to all beans in this file -->
    <bean id="orderService" class="com.example.OrderService"/>
    <bean id="orderService2" class="com.example.OrderService2" autowire="no"/> <!-- override -->
</beans>
```

---

### 3.2.2 — `@Autowired` — Complete Internal Architecture

`@Autowired` is processed by `AutowiredAnnotationBeanPostProcessor` (AABPP). This BPP is registered automatically by any ApplicationContext (via `<context:annotation-config>` or `<context:component-scan>` in XML, or automatically in `AnnotationConfigApplicationContext`).

#### When AABPP Runs

During `AbstractAutowireCapableBeanFactory.populateBean()`, which is called per-bean after instantiation:

```
createBean(beanName, mbd, args)
    └── doCreateBean()
            ├── createBeanInstance()        ← instantiation (constructor)
            ├── applyMergedBeanDefinitionPostProcessors()
            │       └── AABPP.postProcessMergedBeanDefinition()
            │               → scans class for @Autowired metadata
            │               → builds InjectionMetadata (cached per class)
            ├── addSingletonFactory()       ← early exposure for circular deps
            └── populateBean()
                    └── AABPP.postProcessProperties()
                            → InjectionMetadata.inject()
                            → for each injection point:
                                    └── resolveDependency()
```

#### `InjectionMetadata` — The Metadata Cache

When AABPP first processes a class, it scans the class hierarchy (including superclasses) for `@Autowired` and `@Value` injection points. The result is cached in an `InjectionMetadata` object:

```java
class InjectionMetadata {
    private final Class<?> targetClass;
    private final Collection<InjectedElement> injectedElements;
    // Each InjectedElement is either:
    // - AutowiredFieldElement  (for @Autowired fields)
    // - AutowiredMethodElement (for @Autowired methods)
}
```

This cache prevents repeated class scanning on every bean instantiation (important for prototype beans). The cache is stored in a `ConcurrentHashMap<String, InjectionMetadata>` inside AABPP.

#### `resolveDependency()` — The Resolution Engine

The actual dependency resolution happens in `DefaultListableBeanFactory.resolveDependency()`. This is the core resolution algorithm:

```java
public Object resolveDependency(DependencyDescriptor descriptor,
                                 String requestingBeanName,
                                 Set<String> autowiredBeanNames,
                                 TypeConverter typeConverter) {

    // Step 1: Special type handling
    if (descriptor.getDependencyType() == Optional.class) {
        return createOptionalDependency(descriptor, requestingBeanName);
    }
    if (ObjectFactory.class.isAssignableFrom(descriptor.getDependencyType())) {
        return new DependencyObjectFactory(descriptor, requestingBeanName);
    }
    if (javaxInjectProviderClass != null &&
        descriptor.getDependencyType() == javaxInjectProviderClass) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    }

    // Step 2: Lazy resolution proxy
    Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
        descriptor, requestingBeanName);
    if (result == null) {

        // Step 3: Actual resolution
        result = doResolveDependency(descriptor, requestingBeanName,
                                     autowiredBeanNames, typeConverter);
    }
    return result;
}
```

The core is in `doResolveDependency()`:

```
doResolveDependency()
│
├── Step 1: Check for @Value shortcut
│       → if @Value annotation present → resolve via EmbeddedValueResolver
│
├── Step 2: Check for Collection/Map/Array injection
│       → if field type is List<T>, Set<T>, Map<String,T>, T[]
│       → findAutowireCandidates() for type T
│       → collect ALL matching beans
│       → sort by @Order / Ordered interface
│
├── Step 3: findAutowireCandidates(requiredType)
│       → getBeanNamesForType(requiredType, true, eager)
│       → check each candidate:
│               ├── isAutowireCandidate()     ← checks autowire-candidate=false
│               ├── isQualifierMatch()         ← @Qualifier matching
│               └── resolvableDependencies?    ← ApplicationContext, BeanFactory, etc.
│
├── Step 4: Determine unique candidate
│       ├── 0 candidates → required? → NoSuchBeanDefinitionException
│       │                  optional? → null
│       ├── 1 candidate  → use it ✓
│       └── Multiple candidates:
│               ├── determinePrimaryCandidate()    → @Primary bean?
│               ├── determineHighestPriorityCandidate() → @Priority ordering?
│               └── matchesBeanName()              → field/param name matches?
│                       ├── Match found → use it ✓
│                       └── No match → NoUniqueBeanDefinitionException
│
└── Step 5: Resolve the selected bean name
        → beanFactory.getBean(selectedBeanName)
        → record dependency: registerDependentBean()
```

---

### 3.2.3 — `@Qualifier` — Complete Mechanics

`@Qualifier` is NOT just a simple name filter. It is a metadata-matching mechanism that compares qualifier metadata between injection points and bean definitions.

#### How `@Qualifier` Matching Works Internally

`AutowireCandidateQualifier` objects are stored in `BeanDefinition`. When a `@Qualifier("name")` is present at the injection point, Spring calls `QualifierAnnotationAutowireCandidateResolver.isQualifierMatch()` which:

1. Reads the `@Qualifier` annotation at the injection point
2. For each candidate bean:
   a. Check if the bean definition has a matching `AutowireCandidateQualifier`
   b. If not, check if the **bean name** matches the qualifier value
   c. If not, check the bean class for `@Qualifier` annotation
   d. If not, check factory method for `@Qualifier` annotation

```java
// Qualifier on @Bean method
@Bean
@Qualifier("fast")
public DataSource fastDataSource() { return new HikariDataSource(); }

@Bean
@Qualifier("slow")
public DataSource slowDataSource() { return new BasicDataSource(); }
```

```java
// Qualifier at injection point
@Autowired
@Qualifier("fast")
private DataSource dataSource; // matches fastDataSource
```

#### Custom Qualifier Annotations

You can create type-safe qualifier annotations:

```java
@Target({ElementType.FIELD, ElementType.METHOD,
         ElementType.PARAMETER, ElementType.TYPE,
         ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface DatabaseType {
    String value() default "primary";
    Environment env() default Environment.PRODUCTION;

    enum Environment { PRODUCTION, TESTING, DEVELOPMENT }
}
```

```java
@Bean
@DatabaseType(value = "analytics", env = DatabaseType.Environment.PRODUCTION)
public DataSource analyticsDataSource() { ... }
```

```java
@Autowired
@DatabaseType(value = "analytics", env = DatabaseType.Environment.PRODUCTION)
private DataSource dataSource; // matched by ALL qualifier attributes
```

**Critical:** Custom qualifier matching compares ALL annotation attributes. If any attribute differs, it's not a match. This enables extremely fine-grained disambiguation.

#### `@Qualifier` on `@Component` Classes

```java
@Component
@Qualifier("premium")
public class PremiumPaymentService implements PaymentService { }

@Component
@Qualifier("standard")
public class StandardPaymentService implements PaymentService { }
```

```java
@Autowired
@Qualifier("premium")
private PaymentService paymentService; // injects PremiumPaymentService
```

---

### 3.2.4 — `@Primary` — Complete Mechanics

`@Primary` sets a flag in the `BeanDefinition` (`primary=true`). During dependency resolution, after `findAutowireCandidates()` returns multiple results, `determinePrimaryCandidate()` scans the candidates and returns the one with `primary=true`.

**Key rules:**
- Only ONE bean of a type should be `@Primary` — if multiple are `@Primary`, Spring throws `NoUniqueBeanDefinitionException` ("more than one 'primary' bean found")
- `@Primary` only activates when there are multiple candidates of the same type
- `@Qualifier` at the injection point OVERRIDES `@Primary`
- `@Primary` affects XML autowiring (`byType`) AND annotation-based autowiring

```java
@Bean @Primary
public DataSource primaryDS() { return new HikariDataSource(); } // @Primary

@Bean
public DataSource secondaryDS() { return new BasicDataSource(); }

@Bean
public DataSource archiveDS() { return new ArchiveDataSource(); }
```

```java
@Autowired DataSource ds;           // gets primaryDS (@Primary wins)
@Autowired @Qualifier("secondaryDS") DataSource ds2; // gets secondaryDS (@Qualifier overrides @Primary)
```

---

### 3.2.5 — `@Autowired` on Collections and Maps

One of the most powerful and underused features of `@Autowired`:

```java
// All beans of type Validator, collected into a List
@Autowired
private List<Validator> validators;

// Same, as a Set (no ordering)
@Autowired
private Set<Validator> validatorSet;

// Map: key=beanName, value=bean instance
@Autowired
private Map<String, Validator> validatorMap;

// Array form
@Autowired
private Validator[] validatorArray;
```

**Ordering in List/Array injection:**
Beans implementing `org.springframework.core.Ordered` or annotated with `@Order` are sorted by their order value (lower = earlier in list). `@Priority` (JSR-250) also works. Beans without ordering are appended in registration order.

```java
@Component @Order(1)
public class EmailValidator implements Validator { } // first in list

@Component @Order(2)
public class PhoneValidator implements Validator { } // second in list

@Component @Order(Ordered.LOWEST_PRECEDENCE)
public class FallbackValidator implements Validator { } // last in list
```

**`Map<String, T>` injection:**
The map key is the bean name (String), the value is the bean instance. This is extremely useful for strategy patterns:

```java
@Component("PDF")
public class PdfReportGenerator implements ReportGenerator { }

@Component("EXCEL")
public class ExcelReportGenerator implements ReportGenerator { }

@Service
public class ReportService {
    @Autowired
    private Map<String, ReportGenerator> generators;
    // {"PDF": PdfReportGenerator, "EXCEL": ExcelReportGenerator}

    public void generate(String format, Data data) {
        ReportGenerator gen = generators.get(format); // strategy pattern
        if (gen == null) throw new UnsupportedFormatException(format);
        gen.generate(data);
    }
}
```

**Critical:** When injecting `Map<String, T>`, the type parameter `T` determines which beans are collected. ALL beans of type `T` (including subtypes) are included, keyed by their bean name.

---

### 3.2.6 — Optional Dependencies — All Mechanisms

Spring provides multiple mechanisms for expressing that a dependency is optional:

#### Mechanism 1: `required=false`
```java
@Autowired(required = false)
private AuditService auditService; // null if not found
```

#### Mechanism 2: `Optional<T>`
```java
@Autowired
private Optional<AuditService> auditService; // empty Optional if not found
```
`Optional<T>` is unwrapped by `resolveDependency()` — Spring recognizes `Optional` as a special type and returns `Optional.empty()` if the bean doesn't exist, rather than throwing.

#### Mechanism 3: `@Nullable`
```java
@Autowired
@Nullable
private AuditService auditService; // null if not found, no exception
```
Spring's `@Nullable` (from `org.springframework.lang`) and JSR-305's `@Nullable` are both recognized. Also recognizes Lombok's `@Nullable` and other common variants via annotation name matching (`"Nullable"`).

#### Mechanism 4: `ObjectProvider<T>`
```java
@Autowired
private ObjectProvider<AuditService> auditProvider;

// Usage:
auditProvider.ifAvailable(audit -> audit.log(event));  // run if bean exists
auditProvider.getIfAvailable(() -> new DefaultAudit()); // with default supplier
AuditService audit = auditProvider.getIfUnique();       // null if not unique
auditProvider.stream().forEach(a -> a.initialize());    // stream all matching
```

`ObjectProvider<T>` is the most powerful optional dependency mechanism. It provides:
- Lazy resolution (bean not fetched until `getObject()` called)
- Safe optional access (no exception on missing bean)
- Collection-like streaming of all matching beans
- Default supplier on `getIfAvailable(Supplier<T>)`

#### Mechanism 5: `@Inject Optional<T>` (JSR-330)
```java
@Inject
private Optional<AuditService> auditService;
```

#### Comparison Table

| Mechanism | Lazy | Exception on miss | Multiple beans |
|---|---|---|---|
| `required=false` | No | No (null) | Picks one |
| `Optional<T>` | No | No (empty) | Picks one |
| `@Nullable` | No | No (null) | Picks one |
| `ObjectProvider<T>` | Yes | No | ✅ stream() |
| `Provider<T>` (JSR-330) | Yes | Yes | No |

---

### 3.2.7 — Autowiring Limitations and Exclusions

#### Limitation 1: Primitive types (XML byType)
XML `autowire="byType"` cannot wire simple types (`int`, `String`, `boolean`). You must use `<property value="...">` for primitives and strings. Annotation `@Value` handles this for annotation-based config.

#### Limitation 2: Multiple beans of same type (byType)
XML `autowire="byType"` throws `NoUniqueBeanDefinitionException` when multiple beans of the required type exist (unless `@Primary` or `autowire-candidate=false` resolves it). Annotation `@Autowired` has the additional fallback to name matching.

#### Limitation 3: Null values
Autowiring cannot inject `null` values. If a dependency resolves to `null`, Spring treats it as "not found."

#### Excluding Beans from Autowiring

```xml
<!-- XML: exclude specific bean from type-based autowiring -->
<bean id="legacyService" class="com.example.LegacyService"
      autowire-candidate="false"/>
```

```java
// Java config: exclude bean from autowiring
@Bean(autowireCandidate = false)
public DataSource legacyDataSource() { ... }
```

```java
// Exclude entire type from autowiring registration
// (done programmatically in BeanFactoryPostProcessor)
factory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
```

**Default autowire candidates:** All beans are autowire candidates (`autowireCandidate=true`) by default. Setting to `false` removes the bean from type-based resolution but keeps it accessible by name.

---

### 3.2.8 — `@Resource` vs `@Autowired` — Resolution Deep Dive

Both inject dependencies, but through different resolution strategies:

#### `@Autowired` Resolution Chain:
```
1. Determine required type T from field/param declaration
2. Find all beans of type T (getBeanNamesForType)
3. Filter by @Qualifier (if present)
4. If 1 result → inject
5. If multiple: check @Primary → check name fallback → throw
6. If 0: check required flag → throw or skip
```

#### `@Resource` Resolution Chain (CommonAnnotationBeanPostProcessor):
```
1. Determine name:
   a. @Resource(name="explicit") → use "explicit"
   b. No name → use field name / method property name
2. Find bean by name (getBean(name))
3. If found → inject (type check performed after)
4. If not found by name:
   a. Fall back to type resolution (like @Autowired byType)
5. If neither → NoSuchBeanDefinitionException
```

#### Critical Behavioral Difference:

```java
@Configuration
public class Config {
    @Bean public DataSource dataSource() { return new HikariDataSource(); }
    @Bean public DataSource backupDataSource() { return new BasicDataSource(); }
}

@Service
public class ServiceA {
    @Autowired
    private DataSource dataSource;
    // Type resolution → 2 DataSource beans → name fallback: "dataSource" → HikariDataSource ✓
}

@Service
public class ServiceB {
    @Resource
    private DataSource dataSource;
    // Name resolution: field name = "dataSource" → getBean("dataSource") → HikariDataSource ✓
    // Same result, different path
}

@Service
public class ServiceC {
    @Autowired
    private DataSource backupDataSource;
    // Type → 2 beans → name fallback: "backupDataSource" → BasicDataSource ✓
}

@Service
public class ServiceD {
    @Resource
    private DataSource backupDataSource;
    // Name: field name = "backupDataSource" → getBean("backupDataSource") → BasicDataSource ✓
}

// Where they DIVERGE:
@Service
public class ServiceE {
    @Autowired
    private DataSource mySpecialSource; // field name doesn't match any bean name
    // Type → 2 beans → name fallback: "mySpecialSource" → NO MATCH → NoUniqueBeanDefinitionException!
}

@Service
public class ServiceF {
    @Resource
    private DataSource mySpecialSource; // field name doesn't match any bean name
    // Name: "mySpecialSource" → not found → fallback to type → 2 beans → NoUniqueBeanDefinitionException
    // SAME exception but different path
}
```

The divergence appears when `@Autowired` has `@Qualifier` capability but `@Resource` does not (it has its own `name` attribute instead):

```java
@Autowired @Qualifier("backupDataSource")
private DataSource myDS;  // works — @Qualifier specifies exact bean

@Resource(name = "backupDataSource")
private DataSource myDS;  // equivalent — explicit name on @Resource
```

---

### 3.2.9 — JSR-330 `@Inject` Behavior

`@Inject` is the JSR-330 standard annotation, equivalent to `@Autowired` with these differences:

| | `@Autowired` | `@Inject` |
|---|---|---|
| `required` attribute | Yes (`required=false` for optional) | No — always required |
| Optional injection | Via `required=false`, `Optional<T>`, `@Nullable` | Via `Optional<T>` or `Provider<T>` |
| Target | Fields, methods, constructors | Fields, methods, constructors |
| Qualifier companion | `@Qualifier` (Spring) | `@Named` (JSR-330) |
| Standard | Spring | JSR-330 |
| Processed by | `AutowiredAnnotationBeanPostProcessor` | Same AABPP |

```java
import javax.inject.Inject;
import javax.inject.Named;
import javax.inject.Provider;

@Named("orderService")          // = @Component("orderService")
public class OrderService {

    @Inject                     // = @Autowired (required=true always)
    private PaymentService paymentService;

    @Inject
    @Named("primary")           // = @Qualifier("primary")
    private DataSource dataSource;

    @Inject
    private Provider<ReportGen> reportGenProvider; // lazy + prototype support
    // provider.get() → new prototype instance
}
```

`Provider<T>` (JSR-330) is equivalent to `ObjectFactory<T>` (Spring). Both provide lazy access to beans, enabling prototype injection from singletons.

---

### `@Autowired` Constructor Injection — Spring 4.3+ Implicit Detection

From Spring 4.3, `@Autowired` is **implicit** for single-constructor classes:

```java
// Spring 4.3+ — @Autowired not needed
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Implicitly @Autowired — Spring infers this is the injection constructor
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

This works because `AutowiredAnnotationBeanPostProcessor` calls `SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()`, which:
1. Checks for explicit `@Autowired` constructors
2. If none found, checks if there's exactly ONE constructor
3. If exactly one non-default constructor → uses it as the injection constructor

This implicit behavior ONLY applies when there is exactly ONE constructor. Two or more constructors requires explicit `@Autowired`.

---

### `@Autowired` and Generics — Type-safe Collection Injection

Spring 4+ supports generic type information in autowiring:

```java
// Generic repository interface
public interface Repository<T> { ... }

@Component
public class UserRepository implements Repository<User> { }

@Component
public class OrderRepository implements Repository<Order> { }
```

```java
@Service
public class UserService {

    @Autowired
    private Repository<User> userRepository;
    // Spring resolves generic type parameter
    // Only UserRepository matches Repository<User>
    // OrderRepository does NOT match — different generic param
}
```

This works through `ResolvableType` — Spring's abstraction over Java's `Type` system that can inspect generic type parameters at runtime (via `ParameterizedType`). Without this, both repositories would appear as the same raw type `Repository`.

---

### `@Autowired` Self-Reference — Circular via Proxy

```java
@Service
public class TransactionalService {

    @Autowired
    private TransactionalService self; // inject itself!

    @Transactional
    public void doWork() { ... }

    public void callTransactional() {
        self.doWork(); // goes through AOP proxy → @Transactional works
        // vs this.doWork() → bypasses proxy → @Transactional ignored
    }
}
```

This is a legitimate pattern for self-invocation to preserve AOP proxy behavior. Spring 4.3 added explicit support for self-references via `@Autowired`. The self-injected reference is the AOP proxy, not `this`.

---

### Common Misconceptions

**Misconception 1:** "`@Autowired` always resolves by type only."
Wrong. `@Autowired` resolves by type first, but has name fallback. When multiple type candidates exist, Spring uses the field/parameter name as a tiebreaker before throwing `NoUniqueBeanDefinitionException`.

**Misconception 2:** "`@Primary` wins over `@Qualifier`."
Wrong. `@Qualifier` ALWAYS wins over `@Primary`. Qualifier is explicit targeting; Primary is just a default preference.

**Misconception 3:** "`@Resource` does not support type-based resolution."
Wrong. `@Resource` falls back to type resolution if name-based resolution fails.

**Misconception 4:** "Multiple `@Primary` beans cause silent undefined behavior."
Wrong. Multiple `@Primary` beans for the same type cause `NoUniqueBeanDefinitionException` at startup — it's not silent.

**Misconception 5:** "`@Inject` is a Spring annotation."
Wrong. `@Inject` is JSR-330 (`javax.inject`). Spring supports it for compatibility.

**Misconception 6:** "`ObjectProvider` always returns a single bean."
Wrong. `ObjectProvider` supports `stream()` to iterate ALL matching beans.

**Misconception 7:** "XML `autowire="byType"` and `@Autowired` have the same fallback behavior."
Wrong. XML `byType` does NOT have name fallback. If multiple beans of the same type exist, it throws immediately. `@Autowired` has name fallback as an additional resolution step.

---

## ② CODE EXAMPLES

### Example 1 — Complete Resolution Algorithm Demo

```java
@Configuration
public class ResolverDemo {

    @Bean
    public PaymentService stripePaymentService() {
        return new StripePaymentService();
    }

    @Bean @Primary
    public PaymentService paypalPaymentService() {
        return new PaypalPaymentService();
    }

    @Bean
    @Qualifier("express")
    public PaymentService expressPaymentService() {
        return new ExpressPaymentService();
    }
}

@Service
public class OrderService {

    // Case 1: Multiple beans, @Primary wins
    @Autowired
    private PaymentService paymentService;
    // → paypalPaymentService (@Primary)

    // Case 2: @Qualifier overrides @Primary
    @Autowired @Qualifier("express")
    private PaymentService expressService;
    // → expressPaymentService

    // Case 3: Name fallback
    @Autowired
    private PaymentService stripePaymentService; // field name matches bean name
    // → stripePaymentService (name fallback, even though @Primary exists)

    // Case 4: All beans in a list, ordered by @Order
    @Autowired
    private List<PaymentService> allServices;
    // → [stripePaymentService, paypalPaymentService, expressPaymentService]
    // (in registration order if no @Order)

    // Case 5: Map — bean name to instance
    @Autowired
    private Map<String, PaymentService> serviceMap;
    // → {"stripePaymentService": Stripe..., "paypalPaymentService": Paypal...,
    //     "expressPaymentService": Express...}
}
```

---

### Example 2 — Custom Qualifier with Multiple Attributes

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER,
         ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface DataSourceQualifier {
    String region();
    boolean readOnly() default false;
}
```

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @DataSourceQualifier(region = "US", readOnly = false)
    public DataSource usWriteDataSource() { return buildDS("us-primary"); }

    @Bean
    @DataSourceQualifier(region = "US", readOnly = true)
    public DataSource usReadDataSource() { return buildDS("us-replica"); }

    @Bean
    @DataSourceQualifier(region = "EU", readOnly = false)
    public DataSource euWriteDataSource() { return buildDS("eu-primary"); }
}

@Service
public class RegionalOrderService {

    @Autowired
    @DataSourceQualifier(region = "US", readOnly = false)
    private DataSource writeDs; // matches usWriteDataSource EXACTLY

    @Autowired
    @DataSourceQualifier(region = "EU", readOnly = false)
    private DataSource euDs;    // matches euWriteDataSource EXACTLY
}
```

---

### Example 3 — `@Autowired` with Generics

```java
public interface EventProcessor<T extends DomainEvent> {
    void process(T event);
}

@Component
public class OrderEventProcessor implements EventProcessor<OrderEvent> {
    public void process(OrderEvent event) { ... }
}

@Component
public class PaymentEventProcessor implements EventProcessor<PaymentEvent> {
    public void process(PaymentEvent event) { ... }
}

@Service
public class EventRouter {

    @Autowired
    private EventProcessor<OrderEvent> orderProcessor;
    // Only OrderEventProcessor matches — generic type resolved

    @Autowired
    private EventProcessor<PaymentEvent> paymentProcessor;
    // Only PaymentEventProcessor matches

    @Autowired
    private List<EventProcessor<?>> allProcessors;
    // Both processors collected
}
```

---

### Example 4 — ObjectProvider — Full Feature Demo

```java
@Service
public class NotificationService {

    // ObjectProvider — lazy, safe, streamable
    @Autowired
    private ObjectProvider<EmailSender> emailSenderProvider;

    @Autowired
    private ObjectProvider<SmsSender> smsSenderProvider;

    @Autowired
    private ObjectProvider<PushNotificationSender> pushProvider;

    public void sendNotification(Notification notification) {

        // Send via email if available
        emailSenderProvider.ifAvailable(sender ->
            sender.send(notification.getTo(), notification.getMessage()));

        // Send via SMS with fallback default
        SmsSender sms = smsSenderProvider.getIfAvailable(
            () -> new NoOpSmsSender());
        sms.send(notification.getPhone(), notification.getMessage());

        // Use unique bean or null (no exception if multiple or zero)
        PushNotificationSender push = pushProvider.getIfUnique();
        if (push != null) push.send(notification.getDeviceToken(), notification.getMessage());

        // Stream all senders for bulk operation
        pushProvider.stream()
            .filter(PushNotificationSender::isAvailable)
            .forEach(s -> s.send(notification.getDeviceToken(), notification.getMessage()));
    }
}
```

---

### Example 5 — `@Resource` Edge Cases

```java
@Configuration
public class Config {
    @Bean public DataSource primaryDataSource() { return new HikariDataSource(); }
    @Bean public DataSource secondaryDataSource() { return new C3P0DataSource(); }
}
```

```java
@Service
public class ResourceDemo {

    // Case 1: Explicit name — injects primaryDataSource regardless of field name
    @Resource(name = "primaryDataSource")
    private DataSource anyNameHere; // field name irrelevant

    // Case 2: No name → uses field name "secondaryDataSource"
    @Resource
    private DataSource secondaryDataSource; // looks for bean "secondaryDataSource"

    // Case 3: No name, field name doesn't match any bean → falls back to type
    @Resource
    private DataSource myDataSource;
    // Name "myDataSource" → not found
    // Falls back to type → 2 DataSource beans → NoUniqueBeanDefinitionException!

    // Case 4: No name on setter → uses property name from method name
    @Resource
    public void setDataSource(DataSource ds) { }
    // Property name = "dataSource" (from "setDataSource")
    // Looks for bean "dataSource" → not found (beans are "primaryDataSource", "secondaryDataSource")
    // Falls back to type → 2 beans → NoUniqueBeanDefinitionException!
}
```

---

### Example 6 — Self-Reference Injection

```java
@Service
public class CachingService {

    @Autowired
    private CachingService self; // inject AOP proxy of self

    @Cacheable("products")
    public List<Product> getProducts() {
        return repository.findAll();
    }

    public List<Product> getFilteredProducts(String category) {
        // If we called this.getProducts(), AOP proxy is bypassed — no caching
        // self.getProducts() goes through proxy — caching works
        return self.getProducts().stream()
            .filter(p -> p.getCategory().equals(category))
            .collect(toList());
    }
}
```

---

### Example 7 — XML Autowiring Mode Comparison

```java
public class OrderService {
    private PaymentService paymentService;
    private DataSource dataSource;
    private String serviceName;

    // Setters for all fields
    public void setPaymentService(PaymentService ps) { this.paymentService = ps; }
    public void setDataSource(DataSource ds) { this.dataSource = ds; }
    public void setServiceName(String name) { this.serviceName = name; }
}
```

```xml
<!-- byName — matches property name to bean name -->
<bean id="orderService" class="com.example.OrderService" autowire="byName">
    <!-- paymentService property → looks for bean "paymentService" ✓ -->
    <!-- dataSource property → looks for bean "dataSource" ✓ -->
    <!-- serviceName → looks for bean "serviceName" → not found → left null -->
    <property name="serviceName" value="OrderProcessing"/>  <!-- explicit override for string -->
</bean>

<!-- byType — matches property type to bean type -->
<bean id="orderService2" class="com.example.OrderService" autowire="byType">
    <!-- PaymentService type → 1 bean found → injected ✓ -->
    <!-- DataSource type → 1 bean found → injected ✓ -->
    <!-- String type → MANY beans of type String → NoUniqueBeanDefinitionException! -->
</bean>
```

---

### Example 8 — `@Autowired` Fallback to Name — Proof

```java
@Configuration
public class Config {
    @Bean public DataSource primaryDataSource() { return new HikariDataSource(); }
    @Bean public DataSource secondaryDataSource() { return new BasicDataSource(); }
    // No @Primary on either
}

@Service
public class ProofService {

    // Field name = "primaryDataSource" → type resolution → 2 found → name fallback
    @Autowired
    private DataSource primaryDataSource; // ✓ HikariDataSource

    // Field name = "secondaryDataSource" → type resolution → 2 found → name fallback
    @Autowired
    private DataSource secondaryDataSource; // ✓ BasicDataSource

    // Field name = "unknownDataSource" → type resolution → 2 found → name fallback FAILS
    @Autowired
    private DataSource unknownDataSource; // ✗ NoUniqueBeanDefinitionException
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What happens when `@Autowired` finds multiple beans of the required type and none is `@Primary`, no `@Qualifier` is present, and the field name does NOT match any bean name?

A) Spring picks the first registered bean
B) Spring picks the bean with the smallest memory footprint
C) `NoUniqueBeanDefinitionException` is thrown
D) The field is left null

**Answer: C**
After exhausting all resolution strategies (type → primary → qualifier → name fallback), if no unique candidate is found, Spring throws `NoUniqueBeanDefinitionException`.

---

**Q2 — MCQ:**
A `@Bean` method is annotated with both `@Primary` and `@Qualifier("special")`. An injection point has `@Autowired @Qualifier("other")`. What is injected?

A) The `@Primary` bean — it takes precedence
B) The bean qualified with `"other"` — `@Qualifier` overrides `@Primary`
C) `NoUniqueBeanDefinitionException` — conflicting metadata
D) The `@Primary` bean named `"special"` — both annotations combine

**Answer: B**
`@Qualifier` at the injection point ALWAYS overrides `@Primary`. The `@Primary` on the bean definition only activates when no explicit `@Qualifier` is present at the injection point.

---

**Q3 — Select all that apply:**
Which mechanisms allow injecting an optional dependency that leaves the field `null` (or equivalent) when the bean doesn't exist, WITHOUT throwing an exception? (Select ALL that apply)

A) `@Autowired(required = false)`
B) `@Autowired` with `Optional<T>`
C) `@Autowired` with `@Nullable`
D) `@Inject` with no modifier
E) `@Autowired` with `ObjectProvider<T>`
F) `@Resource` with no modifiers

**Answer: A, B, C, E**
`@Inject` (D) has no `required` attribute and always throws if the bean is missing. `@Resource` (F) without explicit name falls back to type, but still throws if no match found.

---

**Q4 — Code Output Prediction:**

```java
@Configuration
public class Config {
    @Bean @Order(3) public Interceptor loggingInterceptor()   { return new LoggingInterceptor(); }
    @Bean @Order(1) public Interceptor securityInterceptor()  { return new SecurityInterceptor(); }
    @Bean @Order(2) public Interceptor metricsInterceptor()   { return new MetricsInterceptor(); }
}

@Service
public class WebService {
    @Autowired
    private List<Interceptor> interceptors;

    public void printOrder() {
        interceptors.forEach(i -> System.out.println(i.getClass().getSimpleName()));
    }
}

ctx.getBean(WebService.class).printOrder();
```

What is the output?

A)
```
LoggingInterceptor
SecurityInterceptor
MetricsInterceptor
```
B)
```
SecurityInterceptor
MetricsInterceptor
LoggingInterceptor
```
C)
```
MetricsInterceptor
SecurityInterceptor
LoggingInterceptor
```
D) Registration order (undefined)

**Answer: B**
When injecting `List<T>`, Spring sorts by `@Order` value. Lower value = higher priority = earlier in list. Order 1 (security) → Order 2 (metrics) → Order 3 (logging).

---

**Q5 — Trick Question:**

```java
@Configuration
public class Config {
    @Bean @Primary public DataSource ds1() { return new HikariDataSource(); }
    @Bean @Primary public DataSource ds2() { return new BasicDataSource(); }
}

@Service
public class Service {
    @Autowired
    private DataSource dataSource;
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class);
```

What happens?

A) `ds1` injected — registered first, so first `@Primary` wins
B) `ds2` injected — last registered wins
C) `NoUniqueBeanDefinitionException` — multiple `@Primary` beans of same type
D) `UnsatisfiedDependencyException` wrapping `NoUniqueBeanDefinitionException`

**Answer: D**
Multiple `@Primary` beans of the same type cause Spring to fail. Technically a `NoUniqueBeanDefinitionException` is thrown internally, but it's wrapped in `UnsatisfiedDependencyException` when reported. The root cause is the multiple `@Primary` beans.

---

**Q6 — Select all that apply:**
Which of the following correctly describe `@Resource` behavior? (Select ALL that apply)

A) Resolves by name first (field name if no explicit name given)
B) Supports `@Qualifier` annotation for disambiguation
C) Falls back to type-based resolution if name-based fails
D) Processed by `CommonAnnotationBeanPostProcessor`
E) Can inject `null` for optional dependencies with no additional annotation
F) Works on fields and setter methods only (not constructors)

**Answer: A, C, D, F**
B is wrong — `@Resource` uses its own `name` attribute, not `@Qualifier`.
E is wrong — `@Resource` throws if neither name nor type resolution finds a match.
F is correct — `@Resource` is defined for fields and methods only (JSR-250 spec), not constructors.

---

**Q7 — Scenario-based:**
A class has `@Autowired private Map<String, DataSource> dataSources`. The context has three `DataSource` beans named `"primary"`, `"secondary"`, and `"archive"`. What does `dataSources` contain?

A) Only the `@Primary` DataSource, keyed by "primary"
B) A map with keys `"primary"`, `"secondary"`, `"archive"` and their respective instances
C) Throws `NoUniqueBeanDefinitionException`
D) An empty map — `Map` injection requires explicit `@Qualifier`

**Answer: B**
`Map<String, T>` injection collects ALL beans of type T. The key is the bean name, the value is the instance. No `@Primary` preference applies to collection injection.

---

**Q8 — Code Output Prediction:**

```java
@Service
public class ServiceA {
    @Autowired(required = false)
    private UnknownService unknownService;

    @PostConstruct
    public void init() {
        System.out.println("unknownService is: " + unknownService);
    }
}
// UnknownService has NO bean definition in the context
```

What is printed?

A) `unknownService is: null`
B) `NoSuchBeanDefinitionException` during context initialization
C) `UnsatisfiedDependencyException`
D) `unknownService is: [empty]`

**Answer: A**
`@Autowired(required = false)` suppresses the exception. The field remains `null`. `@PostConstruct` runs after injection and prints `null`.

---

**Q9 — Compilation/Runtime Error:**

```java
@Service
public class OrderService {
    @Autowired @Resource(name = "primary")
    private DataSource dataSource;
}
```

What happens?

A) Both annotations are processed — `@Resource` wins (processed later)
B) Both annotations are processed — `@Autowired` wins (processed first)
C) Compile error — cannot use both on same field
D) Both annotations processed — undefined/unpredictable behavior

**Answer: B**
Both `@Autowired` and `@Resource` CAN be placed on the same field (no compile error — they're just annotations). `AutowiredAnnotationBeanPostProcessor` runs first (higher priority) and injects via `@Autowired` type-based resolution. `CommonAnnotationBeanPostProcessor` then processes `@Resource` and overwrites with the `"primary"` bean. **Effective result: `@Resource` wins** because it runs second and overwrites the `@Autowired`-injected value. This is undefined behavior and you should never mix them.

---

**Q10 — Drag-and-Drop: Complete the `@Autowired` resolution algorithm in order:**

Steps (unordered):
- Check name fallback (field/param name matches a bean name?)
- Find all beans of type T
- Check if `@Qualifier` annotation narrows the candidates
- Return the unique candidate or throw
- Check if exactly one `@Primary` bean among candidates
- If 0 candidates: check `required` flag

**Answer:**
1. Find all beans of type T
2. If 0 candidates: check `required` flag
3. Check if `@Qualifier` annotation narrows the candidates
4. Check if exactly one `@Primary` bean among candidates
5. Check name fallback (field/param name matches a bean name?)
6. Return the unique candidate or throw

---

## ④ TRICK ANALYSIS

**Trap 1: `@Qualifier` vs `@Primary` precedence**
The single most common trap. `@Qualifier` at the injection point ALWAYS wins over `@Primary` on the bean definition. `@Primary` only activates when no `@Qualifier` is present. Exam questions that suggest `@Primary` "wins" are wrong.

**Trap 2: Multiple `@Primary` beans — not silent**
Two `@Primary` beans of the same type → `NoUniqueBeanDefinitionException` at startup. Not silent. Not a warning. A hard failure. Questions that say "the first registered one wins" or "undefined behavior" are wrong.

**Trap 3: `@Autowired` on `List<T>` vs single `T`**
`@Autowired List<Validator>` collects ALL `Validator` beans. `@Autowired Validator` (single) throws `NoUniqueBeanDefinitionException` if multiple exist (unless `@Primary`). The List/Set/Map variants are fundamentally different behavior, not just a convenience.

**Trap 4: XML `byType` has NO name fallback**
XML `autowire="byType"` throws on multiple matches. Unlike annotation `@Autowired`, there is no name fallback for XML autowiring. Adding `@Primary` to one of the beans (if `@Primary` is supported in that context) or using `autowire-candidate=false` is the only resolution.

**Trap 5: `@Resource` on constructor**
`@Resource` is NOT supported on constructors — only fields and setter methods. Placing `@Resource` on a constructor parameter has no effect (no BPP processes it there). Only `@Autowired`, `@Inject`, and `@Value` work on constructor parameters.

**Trap 6: `Map<String, T>` vs `List<T>` ordering**
`List<T>` respects `@Order`. `Map<String, T>` does NOT have a guaranteed ordering (it's a `LinkedHashMap` in practice, so insertion/registration order, but this is an implementation detail and not guaranteed by the contract). Don't assume ordering in maps.

**Trap 7: `ObjectProvider.getIfUnique()` behavior**
`getIfUnique()` returns null if ZERO OR MORE THAN ONE matching bean exists. It only returns a value if exactly ONE bean matches. This is different from `getIfAvailable()` which throws on multiple matches. Questions testing "what does getIfUnique return when 2 beans exist" → null (no exception).

**Trap 8: `@Autowired` processes superclass fields**
`@Autowired` scanning traverses the class hierarchy. If a superclass has `@Autowired` fields, they ARE processed for subclass instances. This is a source of confusion when injection "works by magic" in inherited fields.

**Trap 9: `@Autowired` on `Optional<T>` — Optional itself is never null**
```java
@Autowired Optional<RareService> rareService;
```
`rareService` is NEVER null. If `RareService` bean doesn't exist, `rareService` is `Optional.empty()`. If it exists, `rareService` is `Optional.of(theBean)`. Checking `rareService == null` is always false.

**Trap 10: JSR-330 `@Inject` has no `required=false`**
`@Inject` is always equivalent to `@Autowired(required=true)`. You cannot write `@Inject(required=false)` — `@Inject` has no attributes at all. For optional JSR-330 injection, use `@Inject Optional<T>`.

---

## ⑤ SUMMARY SHEET

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `@Autowired` resolution order | Type → @Qualifier → @Primary → Name fallback → Exception |
| `@Qualifier` vs `@Primary` | `@Qualifier` ALWAYS wins |
| Multiple `@Primary` of same type | `NoUniqueBeanDefinitionException` at startup |
| XML `byType` name fallback | NONE — throws immediately on multiple matches |
| `@Resource` primary strategy | BY NAME first (field name or explicit name) |
| `@Inject` required attribute | NONE — always required |
| `List<T>` injection | Collects ALL beans of T, sorted by `@Order` |
| `Map<String, T>` injection | All beans of T, keyed by bean name |
| `@Resource` on constructors | NOT supported — fields and methods only |
| `ObjectProvider.getIfUnique()` | Returns null if 0 or 2+ beans — no exception |
| `Optional<T>` injection | Never null — either `Optional.of(bean)` or `Optional.empty()` |
| `@Autowired` on superclass | Processes superclass `@Autowired` fields for subclass beans |

---

### `@Autowired` Resolution Algorithm

```
Type Resolution
    ↓
0 found? → required=true  → NoSuchBeanDefinitionException
         → required=false → null / Optional.empty()
    ↓
1 found? → inject ✓
    ↓
Multiple found:
    → @Qualifier present? → match by qualifier → 1 found → inject ✓
    → @Primary present? (exactly 1) → inject @Primary ✓
    → multiple @Primary → NoUniqueBeanDefinitionException
    → name fallback: field/param name = bean name? → inject ✓
    → NoUniqueBeanDefinitionException ✗
```

---

### Injection Annotation Comparison Table

| Feature | `@Autowired` | `@Resource` | `@Inject` |
|---|---|---|---|
| Standard | Spring | JSR-250 | JSR-330 |
| Primary strategy | By type | By name | By type |
| Optional support | `required=false` | None | `Optional<T>` |
| Qualifier | `@Qualifier` | `name` attribute | `@Named` |
| Constructor support | ✅ | ❌ | ✅ |
| Processed by | AABPP | CABPP | AABPP |
| Spring 4.3 implicit | ✅ single ctor | ❌ | ❌ |

---

### `ObjectProvider<T>` API Quick Reference

```java
getObject()            → inject, throw if missing/multiple
getIfAvailable()       → null if 0; throw if multiple
getIfAvailable(Supplier) → default if 0; throw if multiple
getIfUnique()          → null if 0 or multiple; bean if exactly 1
ifAvailable(Consumer)  → run consumer if exactly 1 bean available
stream()               → stream of ALL matching beans
forEach(Consumer)      → iterate ALL matching beans
```

---

### Interview One-Liners

- "`@Qualifier` at the injection point always overrides `@Primary` on the bean definition — qualifier is explicit, primary is just a preference."
- "Multiple `@Primary` beans of the same type cause `NoUniqueBeanDefinitionException` at startup — not silent."
- "`@Autowired` resolves by type first, then uses name fallback when multiple candidates exist. XML `byType` has no name fallback."
- "`@Resource` resolves by name first (field name as default), then falls back to type. It is NOT supported on constructors."
- "`@Inject` has no `required` attribute — it is always required. Use `@Inject Optional<T>` for optional JSR-330 injection."
- "`@Autowired List<T>` collects all beans of type T sorted by `@Order`. `@Autowired Map<String, T>` keys beans by name."
- "`ObjectProvider<T>` is the most powerful optional dependency mechanism — lazy, safe, streamable, with default supplier support."
- "`AutowiredAnnotationBeanPostProcessor` caches injection metadata per class in `InjectionMetadata` to avoid repeated class scanning for prototype beans."

---
