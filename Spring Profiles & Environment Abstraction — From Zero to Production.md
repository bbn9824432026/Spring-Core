# Spring Profiles & Environment Abstraction — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

Your app has been running in production for six months. New ticket:

```
TICKET-203: Fix the staging environment database wipe incident
- Last week a developer ran the app locally pointing at staging DB
- The local config had DB_URL=jdbc:mysql://staging-server/mydb hardcoded
- They ran a data migration script. Wiped 3 days of staging test data.
- Also: local dev sends real emails to real customers (happened twice)
- Also: production needs connection pooling with 50 connections; 
         dev laptop can't handle that — it crashes IntelliJ
- Fix: the app must KNOW where it's running and behave accordingly
```

You look at the current codebase:

```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        // HARDCODED — the same config everywhere:
        ds.setJdbcUrl("jdbc:mysql://prod-server/mydb");   // ← everyone points at prod
        ds.setUsername("admin");
        ds.setPassword("super-secret-prod-password");     // ← committed to git
        ds.setMaximumPoolSize(50);                        // ← crashes dev laptops
        return ds;
    }

    @Bean
    public EmailService emailService() {
        SmtpEmailService service = new SmtpEmailService();
        service.setHost("smtp.mailgun.org");              // ← real email server
        service.setApiKey("real-api-key-here");           // ← also in git
        return service;
    }

    @Bean
    public PaymentGateway paymentGateway() {
        return new StripeGateway("real-stripe-key");      // ← real charges in dev
    }
}
```

The team has tried various "solutions":

```java
// Attempt 1: Comment things out before running locally (team of 5, everyone forgets)
// ds.setJdbcUrl("jdbc:mysql://prod-server/mydb");     // ← commented out by Alice
// ds.setJdbcUrl("jdbc:h2:mem:localdb");               // ← added by Alice, committed by mistake

// Attempt 2: if/else based on a system property
@Bean
public DataSource dataSource() {
    String env = System.getProperty("app.environment", "dev");
    if ("prod".equals(env)) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server/mydb");
        ds.setMaximumPoolSize(50);
        return ds;
    } else {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2).build();
    }
}
// Problem: every bean has its own if/else. Email bean has it. Payment bean has it.
// The checks are inconsistent — different property names, different string comparisons.
// Test classes don't know about this convention at all.

// Attempt 3: Multiple config files, manually swap them
// AppConfig.java      ← the real one
// AppConfigDev.java   ← copy-pasted, different values, ALWAYS goes out of sync
// Nobody remembers to update AppConfigDev when AppConfig changes.
```

**The core problems:**

```
Problem 1: Infrastructure components (DB, email, payment) are the same object
           regardless of where the app runs. There's no mechanism to swap them.

Problem 2: Credentials are in source code
           Committed to git. Everyone on the team (and git history) can see prod password.

Problem 3: "Which environment am I in?" is answered inconsistently
           Each developer invents their own convention. No single source of truth.

Problem 4: Tests use the same beans as production
           Unit test → triggers real payment charge → customer complaint

Problem 5: Configuration values (pool size, timeouts) can't vary by environment
           Without recompiling and redeploying the entire app.
```

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Environment-specific JAR builds**

```bash
# Build a different JAR for each environment
mvn package -Pdev    # produces app-dev.jar
mvn package -Pprod   # produces app-prod.jar
```

Where each Maven profile swaps in different config files at build time.

**Why this fails:** What you tested (app-dev.jar) is NOT what you deployed (app-prod.jar). The artifact that passed QA is different from the artifact in production — a fundamental violation of deployment hygiene. You can't certify "this exact binary is safe" because you have a different binary per environment. Also: adding an environment means a Maven build change, not just a config file.

---

**Naive Approach 2: One giant properties file with prefixed keys**

```properties
dev.db.url=jdbc:h2:mem:devdb
dev.email.host=localhost
prod.db.url=jdbc:mysql://prod-server/mydb
prod.email.host=smtp.mailgun.org
```

```java
@Value("${" + env + ".db.url}")  // somehow inject env name first
private String dbUrl;
```

**Why this fails:** Your properties file contains BOTH prod and dev secrets — you've now committed prod credentials to git alongside dev credentials. The whole point was to separate them. Additionally, the key naming convention (`prod.`, `dev.`) leaks environment concerns into every config key. Every new property needs the prefix. Adding a new environment means touching every single key.

---

**Naive Approach 3: Deploy different application.properties files via CI/CD**

```bash
# In CI pipeline:
cp config/prod/application.properties src/main/resources/application.properties
mvn package
```

**Why this fails:** You're still building different artifacts per environment. Developers can't run this reliably locally without the CI pipeline. The config file that works in prod is not the one sitting in your repository. It lives somewhere on a server managed by someone and you don't know which version is deployed. And you still haven't solved the problem of beans behaving differently per environment — different properties don't help if your code creates a `SmtpEmailService` regardless.

---

**The correct solution emerges:**

We need two separate mechanisms that work together:

```
Mechanism 1 — PROFILES: tell Spring "use THESE bean implementations in THIS environment"
  → In dev: use H2DataSource, FakeEmailService, MockPaymentGateway
  → In prod: use HikariDataSource, SmtpEmailService, StripeGateway
  → Same JAR. Different beans activated based on one environment signal.

Mechanism 2 — PROPERTIES: tell Spring "use THESE configuration VALUES in THIS environment"
  → Pool size: 5 in dev, 50 in prod
  → DB URL: h2 in dev, mysql in prod
  → Values come from OUTSIDE the JAR (env vars, properties files, config servers)
  → No secrets in source code.
```

---

## 🧠 3. Concept Introduction — Plain English First

### Profiles: The Light Switch Panel Analogy

Imagine a theatrical stage with multiple lighting rigs — one for daytime scenes, one for nighttime, one for storm sequences. The same stage has all the equipment installed, but a **switch panel** controls which rig is actually powered on for a given performance.

```
The stage (your application JAR)  ← same physical artifact, doesn't change
The lighting rigs (bean implementations):
  Dev rig:  H2 database + console email + mock payment
  Prod rig: MySQL database + SMTP email + real payment

The switch panel (spring.profiles.active):
  Set to "dev"  → powers on Dev rig  → Dev beans activated
  Set to "prod" → powers on Prod rig → Prod beans activated

Unlit equipment (beans without @Profile) = ALWAYS on regardless of which rig is active
```

The crucial insight: **the same JAR file contains all the equipment**. You're not building different JARs. You're choosing which beans are registered at startup time based on an external signal.

### Properties: The Dial on Each Piece of Equipment

Once you know WHICH rig is active, each piece of equipment has dials (properties) that can be tuned without replacing the equipment:

```
MySQL DataSource (prod rig) → dials:
  db.url = jdbc:mysql://prod-server/mydb     ← from environment variable
  pool.size = 50                             ← from application-prod.properties
  timeout = 30000                            ← from command line argument

These values come from OUTSIDE the code.
The code just says: @Value("${db.url}") — wherever it comes from, give it to me.
```

### The PropertySource Stack

Spring searches for property values in a priority-ordered stack. The FIRST source that contains the key wins:

```
HIGHER PRIORITY (checked first)
  ┌─────────────────────────────────────────┐
  │ 1. Command line args (--db.url=...)      │  ← deployment-time override
  │ 2. JVM system properties (-Ddb.url=...)  │  ← ops team secrets injection
  │ 3. OS environment variables              │  ← Docker/Kubernetes config
  │ 4. application-prod.properties           │  ← environment-specific file
  │ 5. application.properties                │  ← base defaults
  └─────────────────────────────────────────┘
LOWER PRIORITY (checked last)
```

When Spring sees `@Value("${db.url}")` — it walks this stack top to bottom. First hit wins. This lets ops override dev defaults without touching code.

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TODO 1: Define the DataSource as multiple @Bean implementations, each tagged @Profile
        → dev gets H2, prod gets HikariCP with MySQL
TODO 2: Define EmailService implementations per profile
        → dev gets console logger, prod gets real SMTP
TODO 3: Define a "not prod" safety net for dangerous services
        → @Profile("!prod") on MockPaymentGateway
TODO 4: Extract hardcoded values to properties files
        → application.properties (base) + application-prod.properties (overrides)
TODO 5: Inject values with @Value — no more hardcoded strings in @Bean methods
TODO 6: Activate profiles properly — programmatic, JVM property, env variable
TODO 7: Use @Profile("default") for zero-config local startup
TODO 8: Wire up test profiles with @ActiveProfiles
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: Define DataSource beans per profile

```java
// The WRONG first instinct — one bean with if/else:
// @Bean
// public DataSource dataSource() {
//     if (System.getProperty("env").equals("prod")) { ... }
// }
// Problem: every bean has scattered if/else. No central control. No standard contract.

// THE RIGHT WAY: separate @Bean methods, each tagged with @Profile.
// @Profile: tells Spring "only register this bean definition when this profile is active."
// If the profile isn't active, the method is never called — the bean doesn't exist.

@Configuration
public class DataSourceConfig {

    // @Profile("dev"): this bean is registered ONLY when the "dev" profile is active.
    // When "prod" is active: this method is never called. H2 never starts.
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // EmbeddedDatabaseBuilder: creates an in-memory H2 database.
        // Destroyed when the app shuts down. Safe — can't accidentally wipe real data.
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")   // creates tables
            .addScript("classpath:dev-data.sql") // loads test data
            .build();
    }

    // @Profile("prod"): registered ONLY when "prod" is active.
    // @Value: extracts the value from the PropertySource stack at runtime.
    // The actual value comes from environment variables, properties files, or CLI args.
    // No credentials in source code.
    @Bean
    @Profile("prod")
    public DataSource prodDataSource(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username,
            @Value("${db.password}") String password,
            @Value("${db.pool.size:10}") int poolSize) {   // default 10 if not specified

        // The SILENT FAILURE with non-@Profile configs:
        // If you don't tag this with @Profile("prod"), Spring registers it always.
        // In dev: it tries to connect to jdbc:mysql://prod-server and fails at startup.
        // With @Profile: in dev, this bean simply doesn't exist.

        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        ds.setMaximumPoolSize(poolSize);
        ds.setConnectionTimeout(30000);
        return ds;
    }

    // @Profile("default"): ONLY active when NO profiles are explicitly set.
    // Rule: if activeProfiles is empty → defaultProfiles is used → "default" is active.
    // Rule: as soon as ANY profile is set (dev, prod, test) → "default" becomes inactive.
    //
    // Use case: someone clones the repo and runs the app with zero configuration.
    // They don't set any profile. This bean activates → app starts without crashing.
    @Bean
    @Profile("default")
    public DataSource defaultDataSource() {
        System.out.println("WARNING: No profile active — using in-memory H2 database");
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

**What Spring does internally when profiles are evaluated:**

```
At ctx.refresh() time:
  ConfigurationClassPostProcessor runs
  For each @Bean method:
    → checks: does this method have @Profile?
    → if yes: asks Environment: "is this profile in your activeProfiles set?"
    → if profile NOT active: BeanDefinition is NEVER REGISTERED
      (the bean doesn't exist in the container — not lazy, not deferred — just absent)
    → if profile IS active: BeanDefinition registered normally

Key: this happens at registration time, before any beans are instantiated.
```

---

### TODO 2: Multiple implementations via @Profile on @Component classes

For services (not just configuration), you often have different implementations of the same interface. `@Profile` goes on the class itself:

```java
// The interface — shared contract regardless of environment
public interface EmailService {
    void send(String to, String subject, String body);
}

// @Component + @Profile("dev"): registered as a Spring bean ONLY in dev.
// Both classes implement EmailService — Spring knows which one to inject
// based on which profile is active. There's no ambiguity because only ONE
// implementation is registered at a time.
@Component
@Profile("dev")
public class ConsoleEmailService implements EmailService {

    @Override
    public void send(String to, String subject, String body) {
        // Safe: prints to console instead of sending real email
        System.out.println("=== EMAIL (DEV MODE — not sent) ===");
        System.out.println("To: " + to);
        System.out.println("Subject: " + subject);
        System.out.println("Body: " + body);
        System.out.println("===================================");
    }
}

@Component
@Profile("prod")
public class SmtpEmailService implements EmailService {

    @Value("${email.smtp.host}")
    private String host;

    @Value("${email.api.key}")
    private String apiKey;

    @Override
    public void send(String to, String subject, String body) {
        // Real SMTP call — only happens in prod
        mailgunClient.send(host, apiKey, to, subject, body);
    }
}

// Consumer: doesn't know which EmailService it gets — just injects the interface.
// The correct implementation is injected based on active profile.
@Service
public class OrderNotificationService {

    @Autowired
    private EmailService emailService;  // ← H2 in dev, SMTP in prod. Zero if/else here.

    public void notifyOrderPlaced(Order order) {
        emailService.send(
            order.getCustomerEmail(),
            "Order Confirmed",
            "Your order " + order.getId() + " has been placed."
        );
    }
}
```

---

### TODO 3: The "not prod" safety net — @Profile("!prod")

Some beans should be active in ALL environments EXCEPT production. A debug endpoint, a data reset controller, a mock payment gateway:

```java
// @Profile("!prod"): the ! means NOT.
// This bean is registered when "prod" is NOT in the active profiles.
// Active in: dev, test, staging, default, any custom profile you invent.
// NOT active in: prod.
//
// Silent failure without this:
// MockPaymentGateway registered in prod → "payments" succeed but no money changes hands.
// Real charges aren't processed. Customers think they paid. Finance team panics.
@Component
@Profile("!prod")
public class MockPaymentGateway implements PaymentGateway {

    @Override
    public PaymentResult charge(String customerId, BigDecimal amount) {
        System.out.println("MOCK CHARGE: " + amount + " from " + customerId);
        // Always succeeds — safe for testing
        return PaymentResult.success("mock-transaction-" + UUID.randomUUID());
    }
}

@Component
@Profile("prod")
public class StripePaymentGateway implements PaymentGateway {

    @Value("${stripe.api.key}")
    private String apiKey;

    @Override
    public PaymentResult charge(String customerId, BigDecimal amount) {
        return StripeClient.charge(apiKey, customerId, amount);
    }
}

// Debug endpoint — should NEVER exist in production:
@RestController
@Profile("!prod")   // entire controller absent in prod — no 404, no security risk, just gone
public class DataResetController {

    @Autowired
    private JdbcTemplate jdbc;

    @PostMapping("/dev/reset-data")
    public String resetData() {
        jdbc.execute("DELETE FROM orders");
        jdbc.execute("DELETE FROM customers");
        return "Data reset complete";
    }
}
```

**The AND trap — the most common @Profile mistake:**

```java
// WHAT YOU THINK THIS MEANS: active when BOTH "cloud" AND "aws" are active
@Profile({"cloud", "aws"})

// WHAT IT ACTUALLY MEANS: active when "cloud" OR "aws" is active (either one)
// Array = OR logic

// The silent failure: you deploy to GCP (profile "cloud", not "aws")
// You thought the AwsS3StorageService would stay inactive.
// But "cloud" alone matches {"cloud", "aws"} → AwsS3StorageService activates on GCP.
// It tries to connect to S3. Fails. App crashes.

// FIX — Spring 5.1+ expression syntax:
@Profile("cloud & aws")   // String with & = true AND
// Now: BOTH profiles must be active. GCP with only "cloud" → bean stays inactive.

// More complex expressions:
@Profile("(cloud | on-prem) & !legacy")   // (cloud or on-prem) AND NOT legacy
@Profile("prod & !(eu | apac)")           // prod AND NOT (eu OR apac)
```

---

### TODO 4: Properties files — values outside the code

```properties
# src/main/resources/application.properties
# Base defaults — used in ALL environments as fallback

# These are safe defaults for development:
db.pool.size=5
db.connection.timeout=5000
email.from=noreply@localhost

# DO NOT PUT CREDENTIALS HERE
# db.password=   ← leave empty or absent — must come from outside
```

```properties
# src/main/resources/application-dev.properties
# Loaded ADDITIONALLY when "dev" profile is active.
# Values here OVERRIDE application.properties values.
# Spring Boot auto-discovers: application-{profilename}.properties

db.pool.size=3          # smaller pool for dev laptops
db.url=jdbc:h2:mem:devdb
email.smtp.host=localhost
email.from=dev-test@localhost
```

```properties
# src/main/resources/application-prod.properties
# Loaded when "prod" profile is active.
# Sensitive values (passwords, keys) should NOT be here either —
# they should come from environment variables or a secrets manager.
# But non-sensitive prod config can live here safely.

db.pool.size=50
db.connection.timeout=30000
email.smtp.host=smtp.mailgun.org
email.from=noreply@myapp.com
# db.url, db.password, email.api.key → set as OS environment variables in prod deployment
```

**How Spring Boot loads these files:**

```
Active profile = "prod"

Spring Boot loads:
  1. application.properties        ← base, always loaded
  2. application-prod.properties   ← profile-specific, loaded on top

When BOTH files define the same key:
  application.properties:      db.pool.size=5
  application-prod.properties: db.pool.size=50

  → application-prod.properties WINS (higher priority than base)

Keys ONLY in application.properties: used as-is (no override)
Keys ONLY in application-prod.properties: used (add new prod-specific values)
```

---

### TODO 5: Inject values with @Value — the complete picture

```java
@Configuration
public class AppProperties {

    // Simple injection — fails at startup if db.url is not set anywhere:
    @Value("${db.url}")
    private String dbUrl;

    // With default — never fails:
    @Value("${db.pool.size:10}")        // default = 10 if key missing
    private int poolSize;

    // Empty default — empty string if missing (not null):
    @Value("${feature.flag.name:}")     // default = ""
    private String featureFlagName;

    // List from comma-separated property:
    // app.allowed-origins=https://myapp.com,https://admin.myapp.com
    @Value("${app.allowed-origins:http://localhost:3000}")
    private List<String> allowedOrigins; // ["http://localhost:3000"] in dev

    // The silent failure trap — ONLY applies to @Configuration classes
    // with a non-static PropertySourcesPlaceholderConfigurer:
    // If PSPC is non-static, @Value in the SAME @Configuration class
    // silently gets the DEFAULT value instead of the actual property.
    // The property file IS loaded — but too late for this class.
    // FIX: if you define PSPC as a @Bean, make it static.
    @Bean
    public static PropertySourcesPlaceholderConfigurer pspc() {
        // static: Spring can call this before the @Configuration class is instantiated
        // non-static: requires the @Configuration instance first → chicken-and-egg
        //             → @Value fields in this class silently fall back to defaults
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

**`@Value` vs `Environment.getProperty()` — when each wins:**

```java
@Service
public class FeatureService {

    // @Value: resolved ONCE at bean creation. Stored in the field.
    // ✓ Simple. No boilerplate.
    // ✗ Can't be refreshed after startup (value is frozen in the field).
    @Value("${feature.new-checkout:false}")
    private boolean newCheckoutEnabled;

    @Autowired
    private Environment env;

    // Environment.getProperty(): fresh lookup every call.
    // ✓ Can react to PropertySource changes (if you add/swap sources at runtime).
    // ✓ Can resolve keys constructed at runtime.
    // ✗ Verbose. Null check needed.
    public boolean isFeatureEnabled(String featureName) {
        // Dynamic key construction — @Value can't do this:
        return env.getProperty("feature." + featureName, Boolean.class, false);
    }

    // getRequiredProperty(): throws IllegalStateException if key not found.
    // Use when the app CANNOT function without this value:
    public String getDatabaseRegion() {
        return env.getRequiredProperty("db.region");
        // → "us-east-1" if set
        // → IllegalStateException("No value for key 'db.region'") if not set
        // Better than NullPointerException later — fails fast, at startup if injected
    }
}
```

---

### TODO 6: How to activate profiles — all the mechanisms

```java
// MECHANISM 1: Programmatic — used when writing application entry points
// MUST happen before ctx.refresh(). After refresh → IllegalStateException.
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("prod", "aws");   // multiple = all active
ctx.register(AppConfig.class);
ctx.refresh();   // profiles evaluated here — beans registered/excluded
// After this line: ctx.getEnvironment().setActiveProfiles() → IllegalStateException

// MECHANISM 2: JVM system property (operations team / startup scripts)
// java -Dspring.profiles.active=prod,aws -jar myapp.jar
// Comma-separated for multiple profiles.

// MECHANISM 3: OS environment variable (Docker, Kubernetes, CI/CD)
// export SPRING_PROFILES_ACTIVE=prod,aws
// Note: SystemEnvironmentPropertySource normalizes this:
// SPRING_PROFILES_ACTIVE → spring.profiles.active (dots↔underscores, case-insensitive)

// MECHANISM 4: Spring Boot application.properties
// spring.profiles.active=dev
// (Lowest priority of all — easily overridden by env var or JVM property)

// MECHANISM 5: Tests — @ActiveProfiles annotation
// @ActiveProfiles("test") → activates "test" profile for the test context
// See TODO 8 for full test example.
```

**Priority when multiple sources try to set the profile:**

```
Programmatic setActiveProfiles() > -Dspring.profiles.active > SPRING_PROFILES_ACTIVE > application.properties

In practice:
  Dev laptop: spring.profiles.active=dev in application.properties (lowest priority — convenience)
  CI/CD: SPRING_PROFILES_ACTIVE=test (overrides the dev default)
  Production deployment: -Dspring.profiles.active=prod (overrides everything)
  Emergency override: programmatic in main() (strongest)
```

---

### TODO 7: MutablePropertySources — adding external config at runtime

Sometimes you need to load config from a source Spring doesn't know about (Consul, Vault, a database):

```java
// ConfigurableEnvironment: the mutable version of Environment.
// Exposes getPropertySources() — the ordered list of sources.
// You can add/remove/reorder sources programmatically.
@Component
public class ConsulConfigLoader implements ApplicationContextAware {

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        ConfigurableEnvironment env = (ConfigurableEnvironment) ctx.getEnvironment();

        // MutablePropertySources: the ordered list of all PropertySource objects.
        // Think of it as an ordered list where index 0 = highest priority.
        MutablePropertySources sources = env.getPropertySources();

        // Load from external source:
        Map<String, Object> consulConfig = loadFromConsul();

        // MapPropertySource: wraps a Map<String, Object> as a PropertySource.
        // "consul" = the name of this source (used to reference it in addBefore/addAfter).
        MapPropertySource consulSource = new MapPropertySource("consul", consulConfig);

        // addFirst: inserts at index 0 → HIGHEST priority.
        // Any key in consul overrides ALL other sources (JVM props, application.properties, etc.)
        sources.addFirst(consulSource);

        // addLast: lowest priority — only used if key not found anywhere else.
        // sources.addLast(consulSource);

        // addBefore/addAfter: precise positioning.
        // Insert consul AFTER JVM system properties but BEFORE application.properties:
        // sources.addAfter("systemProperties", consulSource);
    }

    private Map<String, Object> loadFromConsul() {
        // Real implementation would call Consul's HTTP API
        return Map.of(
            "db.pool.size", "25",
            "feature.new-checkout", "true"
        );
    }
}
```

---

### TODO 8: Test setup — @ActiveProfiles

```java
// @ActiveProfiles: activates specified profiles for this test's ApplicationContext.
// Does NOT affect other tests. Each test class can have its own profile.
@SpringBootTest
@ActiveProfiles("test")   // ← activates "test" profile for this test class
class OrderServiceIntegrationTest {

    // With "test" profile active:
    // - ConsoleEmailService is NOT registered (@Profile("dev"))
    // - SmtpEmailService is NOT registered (@Profile("prod"))
    // - We need a "test" profile bean:
    @Autowired
    private EmailService emailService;  // must have a @Profile("test") implementation

    @Autowired
    private DataSource dataSource;      // must have a @Profile("test") DataSource

    @Test
    void orderPlacedSendsEmail() {
        orderService.placeOrder(testOrder);
        // Verify against captured emails — not real SMTP
        List<String> sent = ((CaptureEmailService) emailService).getSentEmails();
        assertThat(sent).hasSize(1);
    }
}

// The "test" profile implementations:
@Component
@Profile("test")
public class CaptureEmailService implements EmailService {
    private final List<String> sentEmails = new ArrayList<>();

    @Override
    public void send(String to, String subject, String body) {
        sentEmails.add(to + "|" + subject);  // capture instead of send
    }

    public List<String> getSentEmails() { return Collections.unmodifiableList(sentEmails); }
}

// Multiple profiles in one test:
@ActiveProfiles({"test", "feature-x"})  // OR logic: both "test" AND "feature-x" are active
class FeatureXTest { }
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: @Profile on class vs @Profile on individual @Bean methods**

```
@Profile on @Configuration class:
  → ALL @Bean methods in the class are excluded if profile is inactive
  → Use when: a whole set of beans belongs to one environment
  → Example: @Configuration @Profile("prod") class ProdConfig — entire class gone in dev

@Profile on individual @Bean methods:
  → Fine-grained: some beans in the same class are active, others aren't
  → Use when: most config is shared but a few beans differ per environment
  → Example: one @Configuration with @Bean @Profile("dev") dataSource()
             and @Bean @Profile("prod") dataSource()
```

**Decision 2: @Profile vs @Conditional**

```
@Profile:
  ✓ Simple. Declarative. Everyone understands it.
  ✗ Can only check profile NAMES, nothing else
  Use for: environment-based switching (dev/test/prod)

@Conditional (the general-purpose version):
  ✓ Can check ANYTHING: classpath contents, property values, OS, bean presence
  ✗ Requires a Condition class — more code
  Use for: library auto-configuration, conditional on class presence, feature flags

@Profile IS @Conditional (internally):
  @Profile("prod") = @Conditional(ProfileCondition.class)
  They're not different systems — @Profile is built on top of @Conditional.

Spring Boot's @ConditionalOn* (for libraries):
  @ConditionalOnClass(name = "com.mysql.jdbc.Driver")   → if MySQL driver on classpath
  @ConditionalOnProperty("feature.cache.enabled")        → if property set to "true"
  @ConditionalOnMissingBean(CacheManager.class)          → if no CacheManager yet
```

**Decision 3: Properties in application-{profile}.properties vs environment variables**

```
application-prod.properties (config files):
  ✓ Versioned in git (if non-sensitive)
  ✓ Visible to developers — easy to understand what prod looks like
  ✗ Can't contain passwords, API keys — those are sensitive
  Use for: pool sizes, timeouts, hostnames (non-secret)

OS environment variables:
  ✓ Not in source code — no git history of secrets
  ✓ Managed by ops/Kubernetes/Vault separately from code
  ✗ Not visible to developers without access to the deployment environment
  Use for: DB_PASSWORD, API_KEY, SECRET_KEY (anything sensitive)

Convention: non-secret config in properties files, secrets in environment variables.
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
@SpringBootTest
class ProfileEdgeCaseTest {

    // TRAP 1: @Profile array = OR, not AND — silent wrong behavior
    @Test
    void profileArrayIsOrNotAnd() {
        // @Profile({"cloud", "aws"}) — you expect: only active when BOTH are set
        // Reality: active when EITHER is set
        // Test with only "cloud" active:
        // → @Profile({"cloud","aws"}) bean IS registered ← WRONG if you intended AND
        //
        // Fix: @Profile("cloud & aws")
    }

    // TRAP 2: "default" profile disappears when you set ANY explicit profile
    @Configuration
    static class TestConfig {
        @Bean
        @Profile("default")
        String defaultMessage() { return "no profile set"; }

        @Bean
        @Profile("dev")
        String devMessage() { return "dev is active"; }
    }

    @Test
    @ActiveProfiles("dev")  // "dev" is active
    void defaultProfileInactiveWhenDevActive() {
        // "default" profile is NOT active when "dev" is active.
        // If anything autowires the defaultMessage bean → NoSuchBeanDefinitionException.
        // Silent bug: you add @Profile("default") as a "fallback" expecting it always there.
        // It's gone the moment anyone sets any profile.
        assertThat(ctx.containsBean("defaultMessage")).isFalse();
        assertThat(ctx.containsBean("devMessage")).isTrue();
    }

    // TRAP 3: Non-static PSPC makes @Value silently use defaults
    @Configuration
    static class BadConfig {
        @Bean
        // NON-static: this is the bug
        public PropertySourcesPlaceholderConfigurer pspc() {
            PropertySourcesPlaceholderConfigurer p = new PropertySourcesPlaceholderConfigurer();
            p.setLocation(new ClassPathResource("test.properties"));
            return p;
        }

        @Value("${test.timeout:30}")
        private int timeout; // silently = 30, even if test.properties has timeout=60

        @Bean
        public TimerService timerService() { return new TimerService(timeout); }
    }

    @Test
    void nonStaticPspcSilentlyIgnoresProperties() {
        // TimerService gets timeout=30 (default) instead of 60 (from file).
        // No error. No warning. Just wrong values at runtime.
        // Fix: make the @Bean static.
        assertThat(timerService.getTimeout()).isEqualTo(30); // documents the bug
    }

    // TRAP 4: setActiveProfiles() after refresh() throws — but which method?
    @Test
    void cannotSetProfilesAfterRefresh() {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        ctx.register(AppConfig.class);
        ctx.refresh();  // ← container is now active

        assertThatThrownBy(() -> ctx.getEnvironment().setActiveProfiles("dev"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Cannot change active profiles after the environment");
        ctx.close();
    }

    // TRAP 5: @PropertySource first declaration wins (not last)
    @Configuration
    @PropertySource("classpath:first.properties")   // first.key=FROM_FIRST
    @PropertySource("classpath:second.properties")  // first.key=FROM_SECOND
    static class DualSourceConfig { }

    @Test
    void firstPropertySourceWinsNotLast() {
        String value = env.getProperty("first.key");
        // Most devs expect "last write wins" like a Map.
        // But PropertySources search from index 0 (front of list).
        // "first.properties" was added first → lower index → higher priority.
        assertThat(value).isEqualTo("FROM_FIRST");  // not FROM_SECOND
    }

    // TRAP 6: Two @Profile beans for same interface with no active profile
    // If "dev" bean and "prod" bean exist, but active profile is "staging":
    // → NEITHER is registered → NoSuchBeanDefinitionException when injecting EmailService
    // Fix: always have a @Profile("default") or @Profile("!prod") fallback.
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version problem:** `@Profile` and `@Value` are scattered across every `@Configuration` class. When you add a new environment (staging), you search the whole codebase for `@Profile("dev")` and `@Profile("prod")` annotations. You miss a few. Staging behaves like prod in some areas and dev in others.

**Senior dev refactoring:** centralize all environment-specific decisions in dedicated `@Configuration` classes, one per environment, and create a clear layered structure:

```java
// ONE config class per environment — each developer knows exactly where to look.
// Adding "staging" = add StagingConfig.java. Touch nothing else.

// The base — shared beans, NO @Profile. Always registered.
@Configuration
public class BaseConfig {

    // Beans that are IDENTICAL in all environments live here.
    // No profile switching needed.
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource ms =
            new ReloadableResourceBundleMessageSource();
        ms.setBasename("classpath:i18n/messages");
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }
}

// Dev environment — H2, console services, no external calls
@Configuration
@Profile("dev")
public class DevConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .addScript("classpath:dev-data.sql")
            .build();
    }

    @Bean
    public EmailService emailService() {
        return new ConsoleEmailService();
    }

    @Bean
    public PaymentGateway paymentGateway() {
        return new MockPaymentGateway();
    }
}

// Production — real connections, values from environment variables
@Configuration
@Profile("prod")
public class ProdConfig {

    // Environment: the full Environment object — lets us read ANY property
    // with type conversion, defaults, and required checks in one place.
    @Autowired
    private Environment env;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        // getRequiredProperty: throws at startup if not set.
        // Better to fail at startup than to fail in production at 3am.
        ds.setJdbcUrl(env.getRequiredProperty("db.url"));
        ds.setUsername(env.getRequiredProperty("db.username"));
        ds.setPassword(env.getRequiredProperty("db.password"));
        ds.setMaximumPoolSize(env.getProperty("db.pool.size", Integer.class, 50));
        ds.setConnectionTimeout(env.getProperty("db.timeout", Long.class, 30000L));
        return ds;
    }

    @Bean
    public EmailService emailService() {
        SmtpEmailService service = new SmtpEmailService();
        service.setHost(env.getRequiredProperty("email.smtp.host"));
        service.setApiKey(env.getRequiredProperty("email.api.key"));
        return service;
    }

    @Bean
    public PaymentGateway paymentGateway() {
        return new StripeGateway(env.getRequiredProperty("stripe.api.key"));
    }
}

// Test — captured/mock services for assertions
@Configuration
@Profile("test")
public class TestConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .build();  // no test data — each test loads its own
    }

    // Singleton CaptureEmailService: tests can @Autowired it and check captured emails
    @Bean
    public CaptureEmailService emailService() {
        return new CaptureEmailService();
    }

    @Bean
    public PaymentGateway paymentGateway() {
        return new MockPaymentGateway();
    }
}
```

**What improved:**

- Adding "staging" environment = create `StagingConfig.java`, set `spring.profiles.active=staging`. Zero changes to existing files.
- Each config class is readable in isolation — "what does prod look like?" → open `ProdConfig.java`
- `env.getRequiredProperty()` in `ProdConfig` means prod crashes at startup if a required env var is missing — not at 3am when the first user hits the missing-config code path
- Test infrastructure is explicit and auditable — you know exactly what `@ActiveProfiles("test")` gives you

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **Profile** | A named label that groups bean definitions. Beans with a label are only registered when that label is "activated." Unlabeled beans always register. |
| **@Profile("dev")** | "Only register this bean when 'dev' is in the active profiles set." |
| **@Profile("!prod")** | "Register this bean in every environment EXCEPT when 'prod' is active." |
| **@Profile({"A","B"})** | OR logic — register when A or B (or both) are active. NOT "both required." |
| **@Profile("A & B")** | AND logic (Spring 5.1+) — register only when BOTH A and B are active. |
| **Default profile** | Active ONLY when zero other profiles are set. The moment you set any profile, "default" deactivates. |
| **Environment** | The unified interface that answers two questions: "Which profiles are active?" and "What is the value of this property?" |
| **ConfigurableEnvironment** | The mutable version — lets you add profiles and PropertySources programmatically. |
| **PropertySource** | A named wrapper around one source of key-value pairs (a file, a Map, environment variables, JVM system properties). |
| **MutablePropertySources** | The ordered list of all PropertySources. Index 0 = highest priority. First match wins. |
| **@PropertySource** | Annotation that adds a properties file to the PropertySources list. First declared = lower index = higher priority. |
| **@Value("${key:default}")** | Field injection — resolved once at bean creation. The `:default` part is used if the key isn't found anywhere in the stack. |
| **@Value("${key}")** | Same but throws at startup if key not found. Fail fast. |
| **getRequiredProperty()** | Throws `IllegalStateException` if key missing. Use for values the app cannot function without. |
| **getProperty()** | Returns null if key missing. Use for optional values. |
| **@Conditional** | The general-purpose mechanism. `@Profile` is built on top of it. Lets you register beans based on ANY condition (classpath, property values, OS, etc.). |
| **@ActiveProfiles** | Test annotation. Tells the test ApplicationContext which profiles to activate. Per-test-class. |
| **SPRING_PROFILES_ACTIVE** | The environment variable name Spring recognizes. Automatically normalized from `spring.profiles.active`. |
| **application-{profile}.properties** | Spring Boot auto-discovery. When "prod" is active, `application-prod.properties` is loaded on top of `application.properties`. |

---

### The decision flowchart:

```
Need different BEAN IMPLEMENTATIONS per environment?
  → Use @Profile on the class or @Bean method
  → One interface, multiple implementations, one registered at a time

Need different CONFIGURATION VALUES per environment?
  → Use application-{profile}.properties for non-sensitive values
  → Use OS environment variables for secrets (passwords, API keys)
  → Inject with @Value("${key}") or env.getProperty("key")

@Profile expression to use?
  → Active in ONE environment:          @Profile("prod")
  → Active in MULTIPLE environments:    @Profile("dev | test") [OR]
  → Active when ALL conditions met:     @Profile("cloud & aws") [AND]
  → Active in everything BUT one:       @Profile("!prod")
  → Complex:                            @Profile("(dev|test) & !legacy")
  → Array notation {"A","B"}:           OR only — never AND

Where to activate profiles?
  → Local dev (convenience):            spring.profiles.active=dev in application.properties
  → CI/CD pipeline:                     SPRING_PROFILES_ACTIVE=test (env var)
  → Production server:                  -Dspring.profiles.active=prod (JVM property)
  → Tests:                              @ActiveProfiles("test")
  → NEVER: setActiveProfiles() after refresh() → IllegalStateException

Property value missing — what to do?
  → App CAN work without it (optional):    @Value("${key:defaultValue}")
  → App CANNOT work without it (required): @Value("${key}") or env.getRequiredProperty("key")
    (fails at startup with clear message instead of NullPointerException in production)
```

### The execution timeline — where everything fits:

```
JVM starts
    ↓
Environment initialized:
  → activeProfiles set from: env var / JVM property / programmatic / application.properties
  → PropertySources stacked: CLI args → JVM props → OS env → application-prod.properties → application.properties

ctx.refresh() called:
  ConfigurationClassPostProcessor runs:
    For each @Component / @Bean:
      → Has @Profile? Ask Environment: "is this profile active?"
      → NO → skip, never register BeanDefinition (bean simply does not exist)
      → YES (or no @Profile) → register BeanDefinition normally
    @PropertySource annotations → add files to PropertySources stack
    @Value annotations → resolved from PropertySources stack (must be before instantiation)

Beans instantiated:
  @Autowired EmailService → exactly ONE implementation exists (the right one for this env)
  No ambiguity. No if/else in business code.

App runs:
  Dev: H2 + ConsoleEmail + MockPayment (safe, fast, isolated)
  Prod: MySQL + SmtpEmail + StripePayment (real, configured from env vars, no secrets in code)
```
