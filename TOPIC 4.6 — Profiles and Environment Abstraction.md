# TOPIC 4.6 — Profiles and Environment Abstraction

---

## ① CONCEPTUAL EXPLANATION

### What is the Environment Abstraction?

The `Environment` interface is Spring's unified abstraction over two fundamental concepts:

```
Environment
├── Profiles   — named logical groupings of bean definitions
│               "Which beans should be active in this deployment?"
└── Properties — key-value configuration from multiple sources
                "What are the configuration values for this deployment?"
```

```java
public interface Environment extends PropertyResolver {

    // PROFILES:
    String[] getActiveProfiles();
    String[] getDefaultProfiles();
    boolean matchesProfiles(String... profileExpressions); // Spring 5.1+
    boolean acceptsProfiles(Profiles profiles);

    // PROPERTIES (inherited from PropertyResolver):
    boolean containsProperty(String key);
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);
    String getRequiredProperty(String key) throws IllegalStateException;
    String resolvePlaceholders(String text);       // resolves ${...}
    String resolveRequiredPlaceholders(String text); // throws if unresolved
}
```

The `Environment` is a first-class object in the Spring container — available everywhere via `@Autowired`, `EnvironmentAware`, and `ApplicationContext.getEnvironment()`.

---

### 4.6.1 — Profiles — Deep Architecture

#### What is a Profile?

A profile is a named logical group of bean definitions. Beans associated with a profile are only registered and instantiated when that profile is active.

```
Without profiles:
    ALL beans registered → ALL beans instantiated

With profiles:
    @Profile("dev") beans    → only when "dev" is active
    @Profile("prod") beans   → only when "prod" is active
    Unprofile beans          → ALWAYS registered (regardless of profiles)
```

#### How Profiles Work Internally

Profile activation happens in TWO phases:

**Phase 1 — BeanDefinition registration (ConfigurationClassPostProcessor):**

When `ConfigurationClassPostProcessor` processes `@Component` and `@Configuration` classes, it checks each bean's profile condition:

```java
// Inside ConfigurationClassParser:
private boolean isProfileActive(ConfigurableEnvironment environment, String[] profiles) {
    return environment.acceptsProfiles(Profiles.of(profiles));
}

// If profile NOT active → BeanDefinition is NOT registered
// If profile IS active → BeanDefinition IS registered
```

**Phase 2 — Environment profile state:**

```java
// In AbstractEnvironment:
private final Set<String> activeProfiles = new LinkedHashSet<>();
private final Set<String> defaultProfiles =
    new LinkedHashSet<>(getReservedDefaultProfiles());

// getReservedDefaultProfiles() returns: ["default"]
```

The `Environment` stores which profiles are active. `@Profile` conditions are evaluated against this set.

#### Profile Evaluation Logic

```
@Profile("A")          → active if "A" is in activeProfiles
@Profile("!A")         → active if "A" is NOT in activeProfiles
@Profile({"A", "B"})   → active if "A" OR "B" is in activeProfiles (OR logic)
@Profile("A & B")      → Spring 5.1+: active if "A" AND "B" are BOTH active (AND logic)
@Profile("A | B")      → Spring 5.1+: active if "A" OR "B" is active
@Profile("!(A & B)")   → Spring 5.1+: active if NOT (A AND B)
```

**The AND trap:** `@Profile({"A", "B"})` is NOT AND — it is OR. For AND logic, Spring 5.1+ provides the expression syntax.

#### Default Profile

The `"default"` profile is a special reserved profile:

```
Rule: If NO active profiles are set, beans with @Profile("default") ARE activated
Rule: If ANY profile is active, @Profile("default") beans are NOT activated
      (unless "default" is explicitly in the active profiles set)

This allows "fallback" beans:
    @Profile("default") → used when nothing else is specified
    @Profile("prod")    → used only when "prod" is explicitly activated
```

```java
@Configuration
public class DataSourceConfig {

    // Used in production
    @Bean
    @Profile("prod")
    public DataSource productionDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server/mydb");
        return ds;
    }

    // Used in dev
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    // Used when NO profile is active (test isolation, simple apps)
    @Bean
    @Profile("default")
    public DataSource defaultDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

---

### 4.6.2 — Profile Activation Mechanisms

#### Method 1 — Programmatic (before refresh())

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext();

// MUST be called BEFORE refresh()
ctx.getEnvironment().setActiveProfiles("prod", "cloud");
// OR: ctx.setConfigLocations(...);

ctx.register(AppConfig.class);
ctx.refresh(); // profiles evaluated here

// After refresh() → IllegalStateException if setActiveProfiles called
```

#### Method 2 — JVM System Property

```bash
# Command line
java -Dspring.profiles.active=prod,cloud -jar myapp.jar

# Spring reads: System.getProperty("spring.profiles.active")
```

#### Method 3 — Environment Variable

```bash
export SPRING_PROFILES_ACTIVE=prod,cloud
java -jar myapp.jar

# Spring reads: System.getenv("SPRING_PROFILES_ACTIVE")
```

#### Method 4 — Spring Boot application.properties

```properties
# application.properties
spring.profiles.active=prod,cloud
```

#### Method 5 — @ActiveProfiles in Tests

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AppConfig.class)
@ActiveProfiles({"test", "integration"})
public class MyIntegrationTest {
    // "test" and "integration" profiles activated for this test
}
```

#### Method 6 — Web Application (web.xml or Servlet context)

```xml
<!-- web.xml -->
<context-param>
    <param-name>spring.profiles.active</param-name>
    <param-value>prod</param-value>
</context-param>
```

#### Priority of Profile Activation Sources

When multiple sources specify profiles, the priority (highest to lowest):

```
1. Programmatic: ctx.getEnvironment().setActiveProfiles()
2. JVM system property: -Dspring.profiles.active=...
3. Environment variable: SPRING_PROFILES_ACTIVE=...
4. web.xml context-param
5. @ActiveProfiles in test (only for test context)
6. Spring Boot application.properties spring.profiles.active
```

---

### 4.6.3 — @Profile Annotation — All Use Cases

#### On @Component Classes

```java
@Component
@Profile("dev")
public class DevMailService implements MailService {
    // Only registered when "dev" is active
}

@Component
@Profile("prod")
public class SmtpMailService implements MailService {
    // Only registered when "prod" is active
}
```

#### On @Configuration Classes

```java
@Configuration
@Profile("dev")
public class DevConfig {
    // ALL @Bean methods in this class are only registered in "dev"
    @Bean public DataSource devDataSource() { ... }
    @Bean public MockPaymentGateway mockPaymentGateway() { ... }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean public DataSource prodDataSource() { ... }
    @Bean public RealPaymentGateway realPaymentGateway() { ... }
}
```

#### On @Bean Methods

```java
@Configuration
public class MixedConfig {

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() { ... }

    @Bean
    @Profile({"dev", "test"})  // OR logic: active in dev OR test
    public DataSource h2DataSource() { ... }

    @Bean
    // No @Profile — always registered
    public ConnectionPool connectionPool(DataSource ds) { ... }
}
```

#### NOT Logic

```java
@Component
@Profile("!prod")  // Active in ANY environment EXCEPT prod
public class DebugFilter implements Filter {
    // Safe in dev/test but excluded from prod
}
```

#### Spring 5.1+ Expression Syntax

```java
// AND: both profiles must be active
@Component
@Profile("cloud & aws")
public class AwsCloudService { ... }

// Complex expressions
@Component
@Profile("prod & !(eu-west | eu-central)")
public class NorthAmericaOnlyService { ... }
// Active when: prod is active AND neither eu-west nor eu-central is active

// Nested
@Component
@Profile("(dev | test) & !legacy")
public class ModernDevTestService { ... }
```

---

### 4.6.4 — PropertySources — The Property System Architecture

The property system in Spring is built on a layered abstraction:

```
Environment
    └── MutablePropertySources (ordered list of PropertySource<?> objects)
            ├── PropertySource 1 (highest priority)
            ├── PropertySource 2
            ├── PropertySource 3
            └── PropertySource N (lowest priority)
```

`PropertyResolver` searches this ordered list for a key — the first PropertySource that contains the key wins.

#### PropertySource Hierarchy

```java
public abstract class PropertySource<T> {
    protected final String name;
    protected final T source;       // the actual source (Properties, Map, env vars, etc.)

    public abstract Object getProperty(String name);
    public boolean containsProperty(String name) { return getProperty(name) != null; }
}
```

**Built-in PropertySource implementations:**

```
PropertiesPropertySource     → java.util.Properties
MapPropertySource            → java.util.Map<String, Object>
SystemEnvironmentPropertySource → System.getenv()
    (normalizes: server.port → SERVER_PORT, etc.)
CommandLinePropertySource    → command line arguments (Spring Boot)
ServletConfigPropertySource  → javax.servlet.ServletConfig init-params
ServletContextPropertySource → javax.servlet.ServletContext init-params
```

#### Default PropertySource Order (Spring Boot)

Spring Boot establishes a comprehensive PropertySource order (17 levels, simplified):

```
Priority (high to low):
1.  Devtools global settings (~/.spring-boot-devtools.properties)
2.  @TestPropertySource
3.  Spring Boot test properties
4.  Command line arguments (--server.port=9090)
5.  SPRING_APPLICATION_JSON (inline JSON)
6.  ServletConfig init parameters
7.  ServletContext init parameters
8.  JNDI attributes (java:comp/env)
9.  System.getProperties() (JVM system properties)
10. System.getenv() (OS environment variables)
11. application-{profile}.properties
12. application.properties (inside JAR)
13. @PropertySource annotations
14. Default properties (SpringApplication.setDefaultProperties)
```

**Critical rule:** Higher in the list = HIGHER priority = overrides lower sources.

---

### 4.6.5 — @PropertySource Annotation

```java
@Configuration
@PropertySource("classpath:database.properties")
public class DatabaseConfig {

    @Autowired
    private Environment env;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(env.getProperty("db.url"));
        ds.setUsername(env.getProperty("db.username"));
        ds.setPassword(env.getProperty("db.password"));
        return ds;
    }
}
```

**Multiple @PropertySource:**

```java
@Configuration
@PropertySources({
    @PropertySource("classpath:app.properties"),
    @PropertySource("classpath:database.properties"),
    @PropertySource(value = "classpath:override.properties",
                    ignoreResourceNotFound = true)
})
public class Config { }

// OR: Java 8+ repeatable annotation
@Configuration
@PropertySource("classpath:app.properties")
@PropertySource("classpath:database.properties")
@PropertySource(value = "classpath:override.properties", ignoreResourceNotFound = true)
public class Config2 { }
```

**When two @PropertySource files have the same key:**

The LAST `@PropertySource` registered wins for duplicate keys — because it is added at the END of the PropertySources list (lowest priority). This is COUNTERINTUITIVE:

```java
@PropertySource("classpath:app.properties")      // db.url=jdbc:h2:mem:app
@PropertySource("classpath:database.properties") // db.url=jdbc:mysql://server/db
// Result: db.url = jdbc:mysql://server/db (database.properties registered LAST → higher index)
// Wait — LOWER index = higher priority in MutablePropertySources
// So: app.properties (added first, lower index) WINS
```

Actually: `@PropertySource` files are added to the BOTTOM of the PropertySource list. Since search goes top-to-bottom (index 0 = highest priority), the FIRST `@PropertySource` listed has HIGHER priority than later ones. The first `@PropertySource` wins for duplicate keys.

---

### 4.6.6 — ConfigurableEnvironment — The Mutable Environment

```java
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {

    void setActiveProfiles(String... profiles);
    void addActiveProfile(String profile);
    void setDefaultProfiles(String... profiles);

    MutablePropertySources getPropertySources();  // get the ordered PropertySource list
    Map<String, Object> getSystemProperties();    // System.getProperties()
    Map<String, Object> getSystemEnvironment();   // System.getenv()

    void merge(ConfigurableEnvironment parent);
}
```

#### Programmatic PropertySource Manipulation

```java
@Component
public class DatabasePropertyLoader implements ApplicationContextAware {

    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        // Access the mutable environment
        ConfigurableEnvironment env = (ConfigurableEnvironment) ctx.getEnvironment();
        MutablePropertySources propertySources = env.getPropertySources();

        // Load properties from database
        Properties dbProps = loadFromDatabase();
        MapPropertySource dbPropertySource =
            new MapPropertySource("database-config", propertiesToMap(dbProps));

        // Add with HIGHEST priority (index 0)
        propertySources.addFirst(dbPropertySource);

        // OR: add with lowest priority
        // propertySources.addLast(dbPropertySource);

        // OR: add before/after a specific PropertySource
        // propertySources.addBefore("systemProperties", dbPropertySource);
        // propertySources.addAfter("systemEnvironment", dbPropertySource);
    }
}
```

---

### 4.6.7 — EnvironmentPostProcessor (Spring Boot)

Spring Boot provides `EnvironmentPostProcessor` — a hook that runs BEFORE the `ApplicationContext` is created, allowing PropertySources to be added very early:

```java
// META-INF/spring.factories:
// org.springframework.boot.env.EnvironmentPostProcessor=com.example.VaultPropertyPostProcessor

public class VaultPropertyPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                        SpringApplication application) {
        // Load secrets from HashiCorp Vault
        Map<String, Object> secrets = VaultClient.loadSecrets();
        MapPropertySource vaultSource =
            new MapPropertySource("vault", secrets);

        // Add with highest priority — secrets override everything
        environment.getPropertySources().addFirst(vaultSource);
    }
}
```

This runs during `SpringApplication.run()` — BEFORE `refresh()`, BEFORE any beans are created, BEFORE `@PropertySource` annotations are processed.

---

### 4.6.8 — @Value and Property Resolution

`@Value` is the most common way to inject property values into beans:

```java
@Component
public class AppConfig {

    // Simple property resolution
    @Value("${app.name}")
    private String appName;

    // With default value
    @Value("${app.timeout:5000}")
    private int timeout;

    // Type conversion (Spring converts String → int, boolean, List, etc.)
    @Value("${app.enabled:true}")
    private boolean enabled;

    // List injection
    @Value("${app.servers:localhost}")
    private List<String> servers; // comma-separated → List

    // SpEL expression
    @Value("#{T(java.lang.Math).PI}")
    private double pi;

    // SpEL + property combination
    @Value("#{environment['app.name'].toUpperCase()}")
    private String upperAppName;

    // Inject the entire Properties object
    @Value("#{systemProperties}")
    private Properties systemProps;
}
```

**@Value resolution process:**

```
1. AutowiredAnnotationBPP.postProcessProperties() called
2. For each @Value field:
   a. StringValueResolver.resolveStringValue("${app.name}")
   b. PropertySourcesPropertyResolver searches PropertySources in order
   c. Finds value in matching PropertySource
   d. TypeConverter converts String to target type (int, boolean, List, etc.)
   e. Sets field via reflection
```

**@Value vs Environment.getProperty():**

```java
// @Value — injected at startup, static
@Value("${db.url}")
private String dbUrl; // Cannot change after initialization

// Environment — dynamic lookup, can be called anytime
@Autowired Environment env;

public String getDbUrl() {
    return env.getProperty("db.url"); // Fresh lookup each time
}

// Use @Value for: values needed at bean initialization
// Use Environment for: values that may need to be resolved dynamically
```

---

### 4.6.9 — Property Type Conversion

Spring's `Environment` uses `ConversionService` for type conversion:

```java
// All of these work:
@Value("${timeout}")        private int timeout;        // String → int
@Value("${enabled}")        private boolean enabled;    // String → boolean
@Value("${rate}")           private double rate;        // String → double
@Value("${servers}")        private String[] servers;   // comma-split → array
@Value("${servers}")        private List<String> list;  // comma-split → List
@Value("${date}")           private LocalDate date;     // if converter registered
@Value("${myEnum}")         private MyEnum myEnum;      // String → enum (by name)
```

**Array/Collection from comma-separated:**

```properties
app.servers=server1,server2,server3
app.ports=8080,8081,8082
```

```java
@Value("${app.servers}")
private String[] servers; // ["server1", "server2", "server3"]

@Value("${app.ports}")
private List<Integer> ports; // [8080, 8081, 8082]
```

---

### 4.6.10 — Profiles vs @Conditional

`@Profile` is a specialization of `@Conditional`:

```java
// @Profile is implemented using @Conditional:
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(ProfileCondition.class)  // ← @Profile uses @Conditional internally
public @interface Profile {
    String[] value();
}

// ProfileCondition implementation:
class ProfileCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());
        if (attrs != null) {
            for (Object value : attrs.get("value")) {
                if (context.getEnvironment().matchesProfiles((String[]) value)) {
                    return true;
                }
            }
            return false;
        }
        return true;
    }
}
```

**`@Conditional` — the general-purpose extension point:**

```java
// Custom condition
public class OnProductionCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        String dbUrl = env.getProperty("db.url", "");
        return dbUrl.contains("prod-server"); // condition based on property VALUE
    }
}

@Bean
@Conditional(OnProductionCondition.class)
public ProductionMonitor productionMonitor() { return new ProductionMonitor(); }

// @Profile cannot do this — it can only check profile NAMES
// @Conditional can check ANY condition: properties, class presence, OS, etc.
```

---

### 4.6.11 — PropertyResolver and Placeholder Resolution

`PropertyResolver` is the base interface for property lookup:

```java
public interface PropertyResolver {

    boolean containsProperty(String key);

    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);
    <T> T getProperty(String key, Class<T> targetType, T defaultValue);

    String getRequiredProperty(String key) throws IllegalStateException;
    <T> T getRequiredProperty(String key, Class<T> targetType);

    // Resolves ${...} placeholders:
    String resolvePlaceholders(String text);          // unresolved → literal
    String resolveRequiredPlaceholders(String text);  // unresolved → IllegalArgumentException
}
```

**`getRequiredProperty()` vs `getProperty()`:**

```java
env.getProperty("missing.key")          // returns null
env.getProperty("missing.key", "def")   // returns "def"
env.getRequiredProperty("missing.key")  // throws IllegalStateException
```

**Placeholder resolution in strings:**

```java
env.resolvePlaceholders("Server: ${server.host:localhost}:${server.port:8080}")
// → "Server: myhost.com:9090" (if properties set)
// → "Server: localhost:8080" (if properties missing, uses defaults)

env.resolveRequiredPlaceholders("Server: ${server.host}")
// → throws IllegalArgumentException if server.host not found
```

---

### 4.6.12 — SystemEnvironmentPropertySource — Relaxed Binding

`SystemEnvironmentPropertySource` implements "relaxed binding" for environment variables:

```java
// Looking up "spring.datasource.url":
// SystemEnvironmentPropertySource checks:
// 1. spring.datasource.url
// 2. spring_datasource_url  (dots → underscores)
// 3. SPRING_DATASOURCE_URL  (uppercase)
// 4. SPRING.DATASOURCE.URL

// This means:
// OS env: SPRING_DATASOURCE_URL=jdbc:mysql://...
// is found when Spring looks for: spring.datasource.url
```

This relaxed binding is why OS environment variables can override `application.properties` values using uppercase underscored names.

---

### Common Misconceptions

**Misconception 1:** "`@Profile({'dev', 'test'})` means both dev AND test must be active."
Wrong. Multiple values in `@Profile` array use OR logic. Bean is registered if ANY of the listed profiles is active. For AND logic, use Spring 5.1+ expression syntax: `@Profile("dev & test")`.

**Misconception 2:** "The default profile is always active."
Wrong. The "default" profile is ONLY active when NO other profiles are set. As soon as any profile is explicitly activated, the "default" profile is no longer active (unless explicitly included).

**Misconception 3:** "Properties from later `@PropertySource` annotations override earlier ones."
Wrong. Properties are added to the END of the PropertySources list. Search goes from index 0 (highest priority) downward. Earlier `@PropertySource` declarations (added first = lower index = higher priority) WIN over later ones for duplicate keys.

**Misconception 4:** "`@Profile` and `@Conditional` are different mechanisms."
Wrong. `@Profile` IS a `@Conditional`. `@Profile` is implemented as `@Conditional(ProfileCondition.class)`. Every `@Profile` annotation is processed by Spring's condition evaluation infrastructure.

**Misconception 5:** "Profile activation can happen after `refresh()`."
Wrong. Profile activation MUST happen before `refresh()`. After `refresh()`, the container is active and all BeanDefinitions have been registered (or excluded). Calling `setActiveProfiles()` after refresh throws `IllegalStateException`.

**Misconception 6:** "Environment variables are case-sensitive in Spring."
Partially wrong. `SystemEnvironmentPropertySource` normalizes keys: it checks uppercase, underscore variants. So `SPRING_PROFILES_ACTIVE` and `spring.profiles.active` refer to the same effective property when Spring uses `SystemEnvironmentPropertySource`.

**Misconception 7:** "`@Value` is resolved at the same time as `@Autowired`."
Technically correct — both are resolved in `postProcessProperties()` by `AutowiredAnnotationBPP`. But `@Value` uses `StringValueResolver` (which uses `PropertySources`) while `@Autowired` uses `BeanFactory.resolveDependency()`. They run in the same BPP method but use different resolution mechanisms.

---

## ② CODE EXAMPLES

### Example 1 — Complete Profile Setup

```java
// Interface
public interface NotificationService {
    void send(String message);
}

// Dev implementation — logs only
@Service
@Profile("dev")
public class LoggingNotificationService implements NotificationService {
    @Override
    public void send(String message) {
        System.out.println("[DEV LOG] " + message);
    }
}

// Test implementation — captures for assertions
@Service
@Profile("test")
public class CaptureNotificationService implements NotificationService {
    private final List<String> captured = new ArrayList<>();
    @Override
    public void send(String message) { captured.add(message); }
    public List<String> getCaptured() { return captured; }
}

// Production implementation — real SMTP
@Service
@Profile("prod")
public class SmtpNotificationService implements NotificationService {
    @Autowired private JavaMailSender mailSender;
    @Override
    public void send(String message) {
        SimpleMailMessage mail = new SimpleMailMessage();
        mail.setText(message);
        mailSender.send(mail);
    }
}

// Default — used when no profile is active
@Service
@Profile("default")
public class NoOpNotificationService implements NotificationService {
    @Override
    public void send(String message) {
        System.out.println("[NO-OP] Notification discarded: " + message);
    }
}

// Consumer — unchanged regardless of profile
@Service
public class OrderService {
    @Autowired
    private NotificationService notificationService; // correct impl injected per profile

    public void placeOrder(Order order) {
        // business logic
        notificationService.send("Order placed: " + order.getId());
    }
}
```

---

### Example 2 — Profile Activation — All Mechanisms

```java
// Mechanism 1: Programmatic
public class Main {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext();
        ctx.getEnvironment().setActiveProfiles("prod", "aws");
        ctx.register(AppConfig.class);
        ctx.refresh();
        ctx.registerShutdownHook();
    }
}

// Mechanism 2: @SpringBootApplication entry point
// java -Dspring.profiles.active=prod,aws -jar app.jar

// Mechanism 3: Test
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AppConfig.class)
@ActiveProfiles("test")
class ServiceTest {
    @Autowired NotificationService notificationService;
    // CaptureNotificationService injected

    @Test
    void testSend() {
        notificationService.send("Hello");
        assertThat(((CaptureNotificationService)notificationService)
            .getCaptured()).contains("Hello");
    }
}
```

---

### Example 3 — Profile Expressions (Spring 5.1+)

```java
// AND: both cloud AND aws must be active
@Component
@Profile("cloud & aws")
public class AwsS3StorageService implements StorageService {
    // Only when both cloud and aws profiles are active
}

// NOT: active in any env except prod
@Component
@Profile("!prod")
public class DebugEndpoint {
    // Active in dev, test, staging — but NOT prod
}

// Complex: (dev OR test) AND NOT legacy
@Component
@Profile("(dev | test) & !legacy")
public class ModernFeatureService {
    // Active in dev or test environments, but not if running legacy mode
}

// Nested NOT with AND
@Component
@Profile("prod & !(eu | apac)")
public class NorthAmericaOnlyService {
    // Active when prod is active AND neither eu nor apac is active
}
```

---

### Example 4 — PropertySource Priority Demonstration

```java
// app-base.properties: timeout=5000, host=basehost
// app-override.properties: timeout=9999

@Configuration
@PropertySource("classpath:app-base.properties")     // added first → lower index = higher priority
@PropertySource("classpath:app-override.properties") // added second → higher index = lower priority
public class PropConfig {

    @Autowired Environment env;

    @PostConstruct
    public void show() {
        // timeout from app-base.properties (higher priority — added FIRST)
        // BUT app-override is added LAST → it has LOWER priority
        // So: timeout = 5000 (from app-base)
        System.out.println("timeout: " + env.getProperty("timeout"));
        // prints: 5000

        // This SURPRISES many developers who expect last-wins
        // The first @PropertySource wins for conflicts
    }
}
```

---

### Example 5 — MutablePropertySources Manipulation

```java
@Component
public class DynamicConfigLoader
        implements ApplicationContextAware, InitializingBean {

    private ConfigurableEnvironment env;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.env = (ConfigurableEnvironment) ctx.getEnvironment();
    }

    @Override
    public void afterPropertiesSet() {
        // Load config from external system (consul, vault, etc.)
        Map<String, Object> externalConfig = loadFromConsul();

        MapPropertySource consulSource =
            new MapPropertySource("consul", externalConfig);

        // Add at FRONT — highest priority (overrides all other sources)
        env.getPropertySources().addFirst(consulSource);

        System.out.println("Consul config loaded with " +
            externalConfig.size() + " properties");
    }

    private Map<String, Object> loadFromConsul() {
        // Simulated
        Map<String, Object> config = new HashMap<>();
        config.put("app.feature.enabled", "true");
        config.put("db.pool.size", "20");
        return config;
    }
}
```

---

### Example 6 — @Value Edge Cases

```java
@Component
public class ValueEdgeCases {

    // Required property — fails at startup if missing
    @Value("${required.key}")
    private String required; // IllegalArgumentException at startup if missing

    // Optional with default
    @Value("${optional.key:default-value}")
    private String optional; // "default-value" if key missing

    // Empty string default (when key missing, use empty string)
    @Value("${opt.key:}")
    private String emptyDefault; // "" if key missing

    // Boolean
    @Value("${feature.enabled:false}")
    private boolean featureEnabled;

    // List from CSV
    @Value("${servers:localhost}")
    private List<String> servers; // ["localhost"] if missing

    // Array
    @Value("${ports:8080}")
    private int[] ports; // [8080] if missing

    // SpEL — can reference other beans
    @Value("#{configBean.getTimeout() * 2}")
    private int doubleTimeout; // SpEL evaluated against Spring beans

    // SpEL with null-safe operator
    @Value("#{configBean?.featureFlag ?: 'default'}")
    private String safeFlag;

    // System property
    @Value("#{systemProperties['java.version']}")
    private String javaVersion;

    // Environment variable
    @Value("#{environment['HOME']}")
    private String homeDir;

    // Map injection
    @Value("#{${services.map:{key1:'val1'}}}")
    private Map<String, String> servicesMap;
    // With: services.map={key1:'val1',key2:'val2'} in properties
}
```

---

### Example 7 — @Conditional vs @Profile

```java
// @Profile: condition based on profile NAME only
@Component
@Profile("prod")
public class ProdService { }

// @Conditional: condition based on ANYTHING
public class DatabaseVendorCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata meta) {
        // Check if Oracle driver is on classpath
        try {
            Class.forName("oracle.jdbc.OracleDriver");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }
}

@Component
@Conditional(DatabaseVendorCondition.class)
public class OracleOptimizedRepository { }

// Spring Boot's @ConditionalOn* annotations:
@Component
@ConditionalOnClass(name = "com.mysql.jdbc.Driver")
public class MySQLAutoConfig { }

@Component
@ConditionalOnProperty(name = "feature.cache.enabled", havingValue = "true")
public class CacheConfig { }

@Component
@ConditionalOnMissingBean(CacheManager.class)
public class DefaultCacheManager implements CacheManager { }
```

---

### Example 8 — EnvironmentPostProcessor (Spring Boot)

```java
// src/main/resources/META-INF/spring.factories:
// org.springframework.boot.env.EnvironmentPostProcessor=\
//   com.example.config.SecretsEnvironmentPostProcessor

public class SecretsEnvironmentPostProcessor
        implements EnvironmentPostProcessor, Ordered {

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 10; // run late — after other sources loaded
    }

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                        SpringApplication application) {
        String profile = environment.getActiveProfiles().length > 0
            ? environment.getActiveProfiles()[0]
            : "default";

        // Load secrets based on active profile
        String secretsFile = "/secrets/" + profile + "/app.properties";
        Resource resource = new FileSystemResource(secretsFile);

        if (resource.exists()) {
            try {
                Properties secrets = new Properties();
                secrets.load(resource.getInputStream());
                PropertiesPropertySource source =
                    new PropertiesPropertySource("secrets-" + profile, secrets);
                // Secrets override everything
                environment.getPropertySources().addFirst(source);
                System.out.println("Loaded secrets from " + secretsFile);
            } catch (IOException e) {
                throw new IllegalStateException("Cannot load secrets: " + secretsFile, e);
            }
        }
    }
}
```

---

### Example 9 — Profile-Based Configuration Switching

```java
// Application properties:
// application-dev.properties:    db.url=jdbc:h2:mem:devdb
// application-staging.properties: db.url=jdbc:mysql://staging-host/db
// application-prod.properties:    db.url=jdbc:mysql://prod-cluster/db

@Configuration
public class DatabaseConfig {

    // Using profile-specific properties (not @Profile beans)
    // Single @Bean — different values per environment via properties
    @Bean
    public DataSource dataSource(
            @Value("${db.url}") String url,
            @Value("${db.username:sa}") String username,
            @Value("${db.password:}") String password,
            @Value("${db.pool.size:10}") int poolSize) {

        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        ds.setMaximumPoolSize(poolSize);
        return ds;
    }
}

// Run with: -Dspring.profiles.active=prod
// Spring loads application.properties + application-prod.properties
// application-prod.properties values override application.properties values
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
A bean is annotated with `@Profile({"dev", "test"})`. Under which conditions is this bean registered?

A) Only when BOTH "dev" AND "test" profiles are active simultaneously
B) When "dev" is active, OR when "test" is active, OR when both are active
C) Never — you cannot list multiple profiles in a single @Profile
D) Only when NEITHER "dev" NOR "test" is active (NOT logic)

**Answer: B**
Multiple values in `@Profile` array use OR logic. Bean is registered when ANY of the listed profiles is active.

---

**Q2 — MCQ:**
No profiles are explicitly set. Which beans are registered?

```java
@Bean @Profile("prod")    public ProdService prodService() { ... }
@Bean @Profile("default") public DefaultService defaultService() { ... }
@Bean                     public CommonService commonService() { ... }
```

A) All three beans
B) Only `commonService`
C) `defaultService` and `commonService`
D) `prodService` and `defaultService` and `commonService`

**Answer: C**
When no profiles are active, the "default" profile is implicitly active. `@Profile("default")` beans ARE registered. Unprofile beans (no `@Profile`) are ALWAYS registered. `@Profile("prod")` is NOT registered because "prod" is not active.

---

**Q3 — Select all that apply:**
Which of the following are valid ways to activate a Spring profile? (Select ALL that apply)

A) `-Dspring.profiles.active=prod` (JVM system property)
B) `@ActiveProfiles("prod")` in test class
C) `ctx.getEnvironment().setActiveProfiles("prod")` before `ctx.refresh()`
D) `ctx.getEnvironment().setActiveProfiles("prod")` after `ctx.refresh()`
E) `SPRING_PROFILES_ACTIVE=prod` (OS environment variable)
F) `spring.profiles.active=prod` in `application.properties` (Spring Boot)

**Answer: A, B, C, E, F**
D is wrong — calling `setActiveProfiles()` AFTER `refresh()` throws `IllegalStateException`. The container is already active and BeanDefinitions have been registered.

---

**Q4 — Code Output Prediction:**

```java
@Configuration
@PropertySource("classpath:first.properties")   // first.key=FROM_FIRST
@PropertySource("classpath:second.properties")  // first.key=FROM_SECOND
public class Config {
    @Autowired Environment env;

    @PostConstruct
    public void print() {
        System.out.println(env.getProperty("first.key"));
    }
}
```

What is printed?

A) `FROM_FIRST` — first PropertySource has higher priority
B) `FROM_SECOND` — last PropertySource wins (last-write-wins)
C) Both values concatenated
D) `IllegalStateException` — duplicate keys not allowed

**Answer: A**
`@PropertySource` files are added to the END of the PropertySources list. The search goes from index 0 (front) to the end. The FIRST `@PropertySource` is added before the second → it has a LOWER index → HIGHER priority. `FROM_FIRST` is found first and returned.

---

**Q5 — Trick Question:**

```java
@Configuration
public class Config {

    @Bean  // NOT static
    public PropertySourcesPlaceholderConfigurer pspc() {
        PropertySourcesPlaceholderConfigurer p = new PropertySourcesPlaceholderConfigurer();
        p.setLocation(new ClassPathResource("app.properties"));
        return p;
    }

    @Value("${app.timeout:30}")
    private int timeout;

    @Bean
    public TimerService timerService() { return new TimerService(timeout); }
}
```

With `app.properties` containing `app.timeout=60`, what timeout does `TimerService` receive?

A) 60 — correctly loaded from properties
B) 30 — default value used because PSPC is non-static
C) 0 — integer default when string resolution fails
D) `BeanCreationException` — circular dependency

**Answer: B**
Non-static PSPC `@Bean` method requires the `Config` class to be instantiated first (to call the method). But `Config` needs `@Value` resolved, which requires PSPC to run first. The chicken-and-egg results in `@Value("${app.timeout:30}")` using the default value `30` because PSPC hasn't run yet when `Config` is initialized.

---

**Q6 — Select all that apply:**
Which of the following are TRUE about the `"default"` profile? (Select ALL that apply)

A) Beans with `@Profile("default")` are registered when no profiles are explicitly active
B) `"default"` is automatically in the active profiles set at all times
C) When `"prod"` is set as the active profile, `@Profile("default")` beans are NOT registered (unless "default" is also explicitly activated)
D) `env.getDefaultProfiles()` returns `["default"]` when no default is changed
E) You can change the default profiles with `env.setDefaultProfiles("fallback")`

**Answer: A, C, D, E**
B is wrong — "default" is NOT always in activeProfiles. It is in `defaultProfiles` and used only when `activeProfiles` is empty. A: correct. C: correct — any explicit profile activation suppresses the "default" profile unless explicitly included. D: correct. E: correct.

---

**Q7 — MCQ:**
What is the correct Spring 5.1+ `@Profile` expression for a bean that should be active ONLY when "cloud" is active AND "aws" is active, but NOT when "eu" is also active?

A) `@Profile({"cloud", "aws", "!eu"})`
B) `@Profile("cloud & aws & !eu")`
C) `@Profile("cloud,aws,!eu")`
D) `@Profile("cloud AND aws AND NOT eu")`

**Answer: B**
Spring 5.1+ supports expression syntax: `&` for AND, `|` for OR, `!` for NOT, `()` for grouping. `@Profile("cloud & aws & !eu")` correctly expresses the requirement. Option A uses array notation which means OR logic. Options C and D use invalid syntax.

---

**Q8 — Scenario-based:**

```java
// system.properties: timeout=100
// application.properties: timeout=200
// JVM: -Dtimeout=300
```

```java
@Value("${timeout}")
private int timeout;
```

What value does `timeout` have in a Spring Boot application?

A) 100 — system.properties first
B) 200 — application.properties loaded first
C) 300 — JVM system properties override application.properties
D) Depends on the order of @PropertySource declarations

**Answer: C**
In Spring Boot's PropertySource priority, JVM system properties (System.getProperties()) have HIGHER priority than application.properties. `-Dtimeout=300` sets a JVM system property → `timeout = 300`.

---

**Q9 — MCQ:**
A `@Profile("!prod")` annotation on a `@Configuration` class means:

A) The class is only active when "prod" is in the active profiles
B) The class is active when "prod" is NOT in the active profiles (including when no profiles are set)
C) The class is never active — "!prod" is not a valid profile name
D) The class is active only when the "default" profile is active

**Answer: B**
`!` negates the profile condition. `@Profile("!prod")` means: register this bean when "prod" is NOT active. This includes: no profiles set, "dev" active, "test" active, "staging" active — any scenario where "prod" is not in the active profiles.

---

**Q10 — Select all that apply:**
Which of the following correctly describe the `Environment` abstraction? (Select ALL that apply)

A) `Environment` extends `PropertyResolver`
B) `ConfigurableEnvironment` allows adding new `PropertySource` objects
C) `Environment` manages both profiles AND properties
D) `getRequiredProperty()` returns null if key not found
E) `MutablePropertySources` maintains an ORDERED list of `PropertySource` objects
F) The first `PropertySource` in the list has the LOWEST priority

**Answer: A, B, C, E**
D is wrong — `getRequiredProperty()` throws `IllegalStateException` if key not found. F is wrong — the FIRST PropertySource (index 0) has the HIGHEST priority. Search proceeds from index 0 downward; the first match wins.

---

## ④ TRICK ANALYSIS

**Trap 1: @Profile array = AND logic**
The most common `@Profile` trap. `@Profile({"dev", "test"})` is OR logic — active when dev OR test. For AND, you need Spring 5.1+ expression syntax: `@Profile("dev & test")`. Any exam answer saying array = AND is wrong.

**Trap 2: Default profile always active**
Wrong. The "default" profile is ONLY active when NO other profiles are explicitly set. The moment you set `spring.profiles.active=prod`, the "default" profile becomes inactive (unless you also include "default" in the active profiles list). This is a common production bug: a "default" fallback bean intended for local dev suddenly becomes active in production when someone forgets to set profiles.

**Trap 3: Last @PropertySource wins for duplicate keys**
Wrong. `@PropertySource` files are added in order to the END of the PropertySources list. The FIRST `@PropertySource` declared (added first = lower index = nearer the front) has HIGHER priority than later ones. The FIRST wins, not the LAST.

**Trap 4: setActiveProfiles() can be called anytime**
Wrong. Must be called BEFORE `refresh()`. After `refresh()`, calling `setActiveProfiles()` throws `IllegalStateException`. Profile state is frozen at refresh time.

**Trap 5: @Profile is a fundamentally different mechanism from @Conditional**
Wrong. `@Profile` IS a `@Conditional`. It is implemented as `@Conditional(ProfileCondition.class)`. All `@Profile` evaluation goes through Spring's condition infrastructure.

**Trap 6: System environment variables are case-sensitive**
Partially wrong. `SystemEnvironmentPropertySource` normalizes keys. `SPRING_PROFILES_ACTIVE` is found when Spring looks for `spring.profiles.active`. Dots are normalized to underscores, and keys are checked in uppercase. This relaxed binding prevents case-sensitivity issues with OS env vars.

**Trap 7: Non-static PSPC @Bean just generates a warning**
Wrong consequence. Non-static PSPC causes `@Value` injections in the same `@Configuration` class to receive literal strings or default values instead of resolved property values. This is a data correctness bug — `@Value("${app.timeout:30}")` silently returns `30` instead of the actual configured value.

**Trap 8: @Profile("default") is active when default profile is mentioned explicitly**
Nuanced. `@Profile("default")` is active ONLY when activeProfiles is empty (using defaultProfiles) OR when "default" is explicitly included in activeProfiles. If you set `-Dspring.profiles.active=prod,default`, then both "prod" profile beans AND "default" profile beans are active simultaneously.

**Trap 9: Environment.getProperty() always does fresh lookup**
Wrong for `@Value` fields — they are resolved ONCE at startup and stored in the field. `@Value` fields cannot be updated after initialization. For dynamic lookup, use `Environment.getProperty()` method directly. `@Value` and `Environment.getProperty()` both use the same PropertySources but at different call times.

**Trap 10: @Profile on @Bean method vs @Profile on @Configuration class**
Different scope of effect. `@Profile` on a `@Configuration` class applies to ALL `@Bean` methods in that class — none are registered if the profile is inactive. `@Profile` on individual `@Bean` methods only excludes THAT specific method. A `@Configuration` class without `@Profile` can have individual `@Bean` methods with different `@Profile` conditions.

---

## ⑤ SUMMARY SHEET

### Profile Activation Sources — Priority Order

```
Priority (high → low):
1.  Programmatic: ctx.getEnvironment().setActiveProfiles()
2.  JVM system property: -Dspring.profiles.active=prod
3.  OS environment variable: SPRING_PROFILES_ACTIVE=prod
4.  web.xml <context-param>: spring.profiles.active
5.  Spring Boot: spring.profiles.active in application.properties
6.  @ActiveProfiles in tests
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `@Profile({"A","B"})` | OR logic — A OR B (not AND) |
| `@Profile("A & B")` | AND logic (Spring 5.1+ expression syntax) |
| `@Profile("!prod")` | Active when "prod" is NOT active |
| Default profile | Active ONLY when NO other profiles are set |
| Profile activation timing | MUST be before `refresh()` |
| `@PropertySource` priority | FIRST declared = higher priority (lower index) |
| `@Profile` implementation | `@Conditional(ProfileCondition.class)` |
| `getRequiredProperty()` | Throws `IllegalStateException` if key missing |
| `getProperty()` | Returns null if key missing |
| Static PSPC @Bean | REQUIRED — non-static causes @Value to use defaults |
| `SystemEnvironmentPropertySource` | Normalizes: `SPRING_DATASOURCE_URL` → `spring.datasource.url` |
| PropertySource search order | Index 0 = first = HIGHEST priority |

---

### PropertySource Default Priority (Spring Boot)

```
HIGH
 │  Command line args (--key=value)
 │  JVM system properties (-Dkey=value)
 │  OS environment variables
 │  application-{profile}.properties
 │  application.properties
 │  @PropertySource annotations
 ▼
LOW
```

---

### Profile Expression Quick Reference (Spring 5.1+)

```
@Profile("A")         → active when A is active
@Profile("!A")        → active when A is NOT active
@Profile("A & B")     → active when A AND B are BOTH active
@Profile("A | B")     → active when A OR B (or both) active
@Profile("!(A & B)")  → active when NOT (A AND B)
@Profile("(A|B) & C") → active when (A or B) AND C
```

---

### Environment Interface Hierarchy

```
PropertyResolver
    └── Environment
            └── ConfigurableEnvironment
                    └── StandardEnvironment (non-web)
                    └── StandardServletEnvironment (web)
                            (has ServletContext + ServletConfig PropertySources)
```

---

### Interview One-Liners

- "The `Environment` abstraction unifies two concerns: profiles (which beans to activate) and properties (what values to use) — both are accessed through the same `Environment` interface."
- "`@Profile({'A','B'})` uses OR logic — it is active when A OR B is active. For AND, use Spring 5.1+ expression syntax: `@Profile('A & B')`."
- "The 'default' profile is active ONLY when no other profiles are explicitly set — it is the fallback, not a permanent fixture."
- "`@Profile` is implemented as `@Conditional(ProfileCondition.class)` — profile evaluation is part of Spring's general condition evaluation infrastructure."
- "Profile activation must happen BEFORE `ctx.refresh()` — calling `setActiveProfiles()` after refresh throws `IllegalStateException`."
- "The first `@PropertySource` declared has the highest priority for duplicate keys — PropertySources are added to the end of the list (lower priority), and search starts from the front."
- "`PropertySourcesPlaceholderConfigurer` must be a static `@Bean` — a non-static BFPP `@Bean` causes `@Value` injections in the same `@Configuration` class to silently use default values."
- "`SystemEnvironmentPropertySource` normalizes OS environment variable names: `SPRING_DATASOURCE_URL` resolves when Spring looks for `spring.datasource.url`."

---
