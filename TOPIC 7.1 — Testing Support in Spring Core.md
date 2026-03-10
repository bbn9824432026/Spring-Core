# TOPIC 7.1 — Testing Support in Spring Core

---

## ① CONCEPTUAL EXPLANATION

### Why Spring Test Support is Critical

Testing in Spring is not just about writing JUnit tests. Spring's test framework solves a fundamental problem: **how do you test components that depend on the Spring container without paying the cost of full application startup for every test?**

The answer is the **TestContext Framework** — a sophisticated infrastructure that manages `ApplicationContext` lifecycle, caches contexts across tests, handles dependency injection into test instances, and provides transaction management, all with minimal configuration.

---

### 7.1.1 — The TestContext Framework Architecture

The TestContext Framework is built on four core abstractions:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TestContext Framework                         │
├─────────────────────────────────────────────────────────────────┤
│  TestContextManager                                             │
│    → Central coordinator — manages one TestContext per test     │
│    → Notifies all registered TestExecutionListeners             │
│    → Created once per test class                                │
├─────────────────────────────────────────────────────────────────┤
│  TestContext                                                     │
│    → Holds current test class, method, and ApplicationContext   │
│    → Delegates to ContextLoader for context creation            │
│    → Tracks context cache key                                   │
├─────────────────────────────────────────────────────────────────┤
│  TestExecutionListener (TEL)                                    │
│    → Hooks into test lifecycle events                           │
│    → Default TELs registered automatically                      │
│    → Custom TELs can be added                                   │
├─────────────────────────────────────────────────────────────────┤
│  ContextLoader / SmartContextLoader                             │
│    → Loads and creates the ApplicationContext                   │
│    → DelegatingSmartContextLoader is the standard               │
└─────────────────────────────────────────────────────────────────┘
```

**Default TestExecutionListeners (in order):**

```
1. ServletTestExecutionListener     → sets up MockHttpServletRequest
2. DirtiesContextBeforeModesTestExecutionListener
3. ApplicationEventsTestExecutionListener
4. DependencyInjectionTestExecutionListener → @Autowired in test class
5. DirtiesContextTestExecutionListener      → handles @DirtiesContext
6. TransactionalTestExecutionListener       → @Transactional rollback
7. SqlScriptsTestExecutionListener          → @Sql execution
8. EventPublishingTestExecutionListener
```

---

### 7.1.2 — SpringExtension and @ExtendWith

`SpringExtension` is the JUnit 5 integration point. It integrates Spring's `TestContextManager` with JUnit 5's extension model:

```java
// Registers SpringExtension with JUnit 5:
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AppConfig.class)
public class MyTest {
    @Autowired
    private OrderService orderService;

    @Test
    void testPlaceOrder() {
        // orderService is fully injected from Spring context
    }
}
```

**What SpringExtension does at each lifecycle point:**

```
JUnit 5 Lifecycle Hook        → SpringExtension Action
─────────────────────────────────────────────────────
beforeAll()                   → Prepare TestContextManager for class
postProcessTestInstance()     → Inject dependencies into test instance
                                (@Autowired, @Value, @MockBean)
beforeEach()                  → Prepare test method
                                (begin transaction, set up request context)
afterEach()                   → After test method
                                (rollback transaction if @Transactional)
afterAll()                    → After all tests in class
```

---

### 7.1.3 — @SpringJUnitConfig — The Shortcut

`@SpringJUnitConfig` is a composed annotation that combines `@ExtendWith(SpringExtension.class)` and `@ContextConfiguration`:

```java
// These are equivalent:

// Verbose:
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AppConfig.class)
public class MyTest { }

// Shortcut:
@SpringJUnitConfig(AppConfig.class)
public class MyTest { }

// With component scan location:
@SpringJUnitConfig(locations = "classpath:test-context.xml")
public class MyTest { }

// With no config — uses inner @Configuration class:
@SpringJUnitConfig
public class MyTest {
    @Configuration
    static class TestConfig {
        @Bean
        public OrderService orderService() {
            return new OrderService(new InMemoryOrderRepository());
        }
    }
}
```

**@SpringJUnitConfig inner class detection:**

When no configuration is specified, Spring looks for a `static` inner `@Configuration` class in the test class and its superclasses. If found, it uses it automatically.

---

### 7.1.4 — ApplicationContext Loading and Context Caching

**Context caching is the single most important performance optimization** in Spring test support. Creating an `ApplicationContext` is expensive — loading all beans, running BFPPs, creating proxies. Without caching, every test class would restart the context.

**How the cache works:**

```
┌─────────────────────────────────────────────────────────────┐
│              Context Cache (static, JVM-level)              │
├─────────────────────────────────────────────────────────────┤
│  Key: MergedContextConfiguration                            │
│    → Set of configuration classes/locations                 │
│    → Active profiles                                        │
│    → Context initializers                                   │
│    → Context loader class                                   │
│    → Parent context                                         │
│    → @TestPropertySource locations + properties             │
├─────────────────────────────────────────────────────────────┤
│  Value: ApplicationContext                                  │
├─────────────────────────────────────────────────────────────┤
│  Tests with IDENTICAL cache keys SHARE the same context     │
│  Tests with DIFFERENT cache keys get DIFFERENT contexts     │
└─────────────────────────────────────────────────────────────┘
```

```java
// These two test classes SHARE the same ApplicationContext:
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
public class OrderServiceTest { }

@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
public class ProductServiceTest { }  // same cache key → same context

// These use DIFFERENT contexts (different config):
@SpringJUnitConfig(AppConfig.class)
public class OrderTest { }          // cache key A

@SpringJUnitConfig(classes = {AppConfig.class, ExtraConfig.class})
public class OrderTest2 { }         // cache key B — different context
```

**What invalidates the cache:**

```
@DirtiesContext on test class  → context closed AFTER the class
@DirtiesContext on test method → context closed AFTER that method
@MockBean/@SpyBean             → each unique combination creates new context
                                 (mock definitions are part of the cache key)
```

---

### 7.1.5 — @ContextConfiguration — Loading the Context

`@ContextConfiguration` specifies how to load the `ApplicationContext`:

```java
// From @Configuration class(es):
@ContextConfiguration(classes = {AppConfig.class, DatabaseConfig.class})

// From XML:
@ContextConfiguration(locations = "classpath:test-applicationContext.xml")

// From both:
@ContextConfiguration(
    classes = AppConfig.class,
    locations = "classpath:test-overrides.xml"
)

// Inherit parent configuration + add more:
@SpringJUnitConfig(AppConfig.class)
public class BaseTest { }

@SpringJUnitConfig(classes = ExtraConfig.class,
                   inheritLocations = true)  // default: true
public class DerivedTest extends BaseTest { }
// DerivedTest context has: AppConfig + ExtraConfig

// With custom initializer:
@ContextConfiguration(
    classes = AppConfig.class,
    initializers = CustomContextInitializer.class
)
```

**Context initializers** — `ApplicationContextInitializer<ConfigurableApplicationContext>`:

```java
// Applied before context refresh — for programmatic setup
public class CustomContextInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        // Set test-specific properties programmatically
        TestPropertySourceUtils.addInlinedPropertiesToEnvironment(ctx,
            "spring.datasource.url=jdbc:h2:mem:testdb",
            "app.feature.enabled=true"
        );
    }
}
```

---

### 7.1.6 — @TestPropertySource

Overrides property values specifically for tests:

```java
// Load test-specific properties file (HIGHEST priority among PropertySources)
@SpringJUnitConfig(AppConfig.class)
@TestPropertySource("classpath:test.properties")
public class OrderServiceTest { }

// Inline properties (even higher priority than file)
@SpringJUnitConfig(AppConfig.class)
@TestPropertySource(properties = {
    "app.timeout=5000",
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "feature.flag.enabled=true"
})
public class OrderServiceTest { }

// Both file and inline (inline OVERRIDES file):
@TestPropertySource(
    locations = "classpath:test.properties",
    properties = "app.timeout=5000"  // overrides same key in file
)
```

**Priority order (highest to lowest):**

```
1. Inline @TestPropertySource properties
2. @TestPropertySource file
3. System properties
4. Environment variables
5. @PropertySource files
6. Default values
```

**Inheritance:**

```java
// inheritProperties = true (default) — child INHERITS parent's properties
@SpringJUnitConfig(AppConfig.class)
@TestPropertySource(properties = "base.timeout=100")
public class BaseTest { }

@SpringJUnitConfig(AppConfig.class)
@TestPropertySource(properties = "extra.flag=true")
// inheritProperties=true by default — has BOTH base.timeout AND extra.flag
public class ChildTest extends BaseTest { }
```

---

### 7.1.7 — @ActiveProfiles

Activates Spring profiles for the test context:

```java
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
public class OrderServiceTest { }

// Multiple profiles:
@ActiveProfiles({"test", "integration"})
public class IntegrationTest { }

// Profile inheritance — child adds profiles to parent's:
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
public class BaseTest { }

@ActiveProfiles("slow")  // with inheritProfiles=true (default)
public class SlowTest extends BaseTest { }
// SlowTest has profiles: "test" AND "slow"

// Programmatic profile resolution:
@ActiveProfiles(resolver = CustomActiveProfilesResolver.class)
public class DynamicProfileTest { }
```

**Custom `ActiveProfilesResolver`:**

```java
public class EnvironmentBasedResolver implements ActiveProfilesResolver {
    @Override
    public String[] resolve(Class<?> testClass) {
        String env = System.getenv("TEST_ENV");
        return env != null ? new String[]{env} : new String[]{"local"};
    }
}
```

---

### 7.1.8 — @TestConfiguration

`@TestConfiguration` is a special `@Configuration` that adds beans ONLY to the test context, without replacing the main configuration:

```java
// Main config:
@Configuration
public class AppConfig {
    @Bean
    public OrderService orderService(PaymentGateway gateway) {
        return new OrderService(gateway);
    }
}

// Test config — adds/overrides beans for testing:
@TestConfiguration
public class TestAppConfig {

    // Override PaymentGateway with a test double
    @Bean
    @Primary  // required to override existing bean
    public PaymentGateway paymentGateway() {
        return new MockPaymentGateway();
    }

    // Add test-only beans
    @Bean
    public TestDataFactory testDataFactory() {
        return new TestDataFactory();
    }
}

// Usage in test:
@SpringJUnitConfig(AppConfig.class)
@Import(TestAppConfig.class)
public class OrderServiceTest {
    @Autowired
    private OrderService orderService; // uses MockPaymentGateway
}
```

**Key difference from `@Configuration`:**

```
@Configuration:  Replaces/overrides configuration when used as @ContextConfiguration
@TestConfiguration: ADDS to the existing context — not a replacement

@Configuration in inner class of test → also picked up, but REPLACES all config
@TestConfiguration in inner class of test → ADDED to loaded config
```

---

### 7.1.9 — @MockBean and @SpyBean

`@MockBean` and `@SpyBean` are Spring Boot Test features (also usable in Spring Framework tests with the right dependencies):

```java
@SpringJUnitConfig(AppConfig.class)
public class OrderServiceTest {

    // Creates a Mockito mock and registers it as a bean in the context
    // Replaces any existing bean of the same type
    @MockBean
    private PaymentGateway paymentGateway;

    // Creates a Mockito spy wrapping the REAL bean from context
    @SpyBean
    private OrderRepository orderRepository;

    @Autowired
    private OrderService orderService;  // injected with mocked PaymentGateway

    @Test
    void testSuccessfulPayment() {
        Order order = new Order("CUST-001", new BigDecimal("99.99"));

        // Mock behavior
        when(paymentGateway.charge(any(), any()))
            .thenReturn(PaymentResult.success("TXN-123"));

        // Real repo called but can verify/stub
        doReturn(order).when(orderRepository).findById("ORD-001");

        orderService.processOrder("ORD-001");

        verify(paymentGateway).charge(eq("CUST-001"), eq(new BigDecimal("99.99")));
    }
}
```

**Critical: @MockBean invalidates context cache:**

```java
// These tests use DIFFERENT contexts — cannot share cache:
@SpringJUnitConfig(AppConfig.class)
public class TestA {
    @MockBean PaymentGateway gateway; // cache key includes MockBean
}

@SpringJUnitConfig(AppConfig.class)
public class TestB {
    @MockBean PaymentGateway gateway; // SAME mock type → SAME cache key → SHARED
    @MockBean EmailService email;     // DIFFERENT mocks → DIFFERENT cache key
}
```

**@MockBean vs @Mock:**

```
@Mock (Mockito):      Pure Mockito — NOT registered in Spring context
                      Bean in context is NOT replaced
                      Manual injection needed

@MockBean (Spring):   Creates Mockito mock AND registers in Spring context
                      Replaces existing bean of same type
                      @Autowired uses the mock automatically
                      Resets mock between tests (default)
```

---

### 7.1.10 — @Transactional in Tests

When `@Transactional` is applied to a test class or method, Spring wraps each test in a transaction that is **rolled back automatically** after the test completes:

```java
@SpringJUnitConfig(AppConfig.class)
@Transactional  // applied to all test methods in this class
public class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void testSaveOrder() {
        Order order = new Order("CUST-001");
        orderRepository.save(order);

        Order found = orderRepository.findById(order.getId());
        assertThat(found).isNotNull();
        // Transaction ROLLED BACK after this test
        // Database is clean for the next test
    }

    @Test
    @Commit  // override rollback — this test commits
    void testSaveAndCommit() {
        // This change PERSISTS to database
        orderRepository.save(new Order("CUST-002"));
    }

    @Test
    @Rollback(false)  // equivalent to @Commit
    void testAlsoCommits() { }

    @Test
    @Rollback  // explicit rollback (this is default anyway)
    void testExplicitRollback() { }
}
```

**Transaction propagation in test context:**

```
Test method begins → TransactionalTestExecutionListener opens transaction
  → Test method executes (all DB operations in same transaction)
  → @BeforeTransaction method runs BEFORE transaction opens
  → @AfterTransaction method runs AFTER transaction closes
Test method ends → transaction rolled back (or committed if @Commit)

// @BeforeTransaction and @AfterTransaction:
@BeforeTransaction
void setupBeforeTransaction() {
    // Runs BEFORE transaction opens — good for data setup that must commit
    // (like reference data that test transaction reads)
}

@AfterTransaction
void verifyAfterTransaction() {
    // Runs AFTER transaction is rolled back — verify DB state
}
```

**Common misconception — transaction on @BeforeEach:**

```java
@Transactional
public class OrderTest {

    @BeforeEach
    void setUp() {
        // This runs INSIDE the test transaction
        // The setup data is rolled back after each test
        orderRepository.save(testOrder);
    }
}
```

---

### 7.1.11 — @Sql and @SqlGroup

`@Sql` executes SQL scripts against the test database:

```java
@SpringJUnitConfig(AppConfig.class)
@Transactional
public class OrderRepositoryTest {

    // Execute before each test method (default phase = BEFORE_TEST_METHOD)
    @Test
    @Sql("classpath:test-data/orders.sql")
    void testFindByCustomer() {
        // orders.sql has been executed — test data available
        List<Order> orders = orderRepository.findByCustomerId("CUST-001");
        assertThat(orders).hasSize(3);
    }

    // Execute AFTER test (for verification or cleanup)
    @Test
    @Sql(scripts = "classpath:verify-state.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void testWithPostSql() { }

    // Multiple scripts in order:
    @Test
    @Sql(scripts = {
        "classpath:schema.sql",
        "classpath:test-data.sql"
    })
    void testWithMultipleScripts() { }

    // Inline SQL statements:
    @Test
    @Sql(statements = {
        "DELETE FROM orders WHERE status = 'CANCELLED'",
        "INSERT INTO orders (id, customer_id) VALUES (1, 'CUST-001')"
    })
    void testWithInlineSql() { }
}

// Class-level @Sql runs for ALL test methods:
@SpringJUnitConfig(AppConfig.class)
@Sql("classpath:test-data/base-data.sql")  // runs before each test
public class ProductTest {
    @Test
    @Sql("classpath:extra-data.sql")  // method-level ADDS to class-level
    void testWithExtraData() { }      // both scripts execute
}
```

**@SqlGroup — multiple @Sql annotations:**

```java
@Test
@SqlGroup({
    @Sql(scripts = "classpath:setup.sql",
         executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD),
    @Sql(scripts = "classpath:cleanup.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
})
void testWithSetupAndCleanup() { }
```

**@SqlConfig — script execution configuration:**

```java
@Sql(
    scripts = "classpath:test-data.sql",
    config = @SqlConfig(
        encoding = "UTF-8",
        separator = ";",           // statement separator (default ";")
        commentPrefix = "--",      // SQL comment prefix
        blockCommentStartDelimiter = "/*",
        blockCommentEndDelimiter = "*/",
        errorMode = SqlConfig.ErrorMode.FAIL_ON_ERROR,  // or IGNORE_ERRORS, CONTINUE_ON_ERROR
        transactionMode = SqlConfig.TransactionMode.INFERRED  // or ISOLATED
    )
)
```

---

### 7.1.12 — @DirtiesContext

`@DirtiesContext` marks that the test has dirtied (modified in non-rollback-able ways) the context, causing Spring to close and remove it from the cache:

```java
// Dirties context after the entire TEST CLASS:
@SpringJUnitConfig(AppConfig.class)
@DirtiesContext(classMode = ClassMode.AFTER_CLASS)  // default
public class StatefulServiceTest { }

// Dirties context BEFORE the test class runs:
@DirtiesContext(classMode = ClassMode.BEFORE_CLASS)
public class ResetBeforeTest { }

// Dirties context AFTER EACH method:
@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
public class EachMethodDirtiesTest { }

// Dirties context BEFORE EACH method:
@DirtiesContext(classMode = ClassMode.BEFORE_EACH_TEST_METHOD)
public class EachMethodFreshContextTest { }

// On specific method — closes context after THIS method:
public class MixedTest {
    @Test
    @DirtiesContext  // method-level — closes context after this method only
    void testThatChangesStaticState() { }

    @Test
    void normalTest() { }  // normal test — uses (possibly new) cached context
}
```

**When to use @DirtiesContext:**

```
✓ Test modifies static state (e.g., System.setProperty(), ThreadLocal)
✓ Test starts/stops external resources (embedded servers, message brokers)
✓ Test uses @MockBean and you need a fresh context for the next test
✓ Test modifies singleton bean state that cannot be reset
✗ Normal DB tests — use @Transactional rollback instead
✗ Property override tests — use @TestPropertySource instead
```

---

### 7.1.13 — Context Hierarchy — @ContextHierarchy

For testing components in parent-child context setups (like Spring MVC root context + DispatcherServlet context):

```java
@SpringJUnitConfig
@ContextHierarchy({
    @ContextConfiguration(name = "root",
                          classes = RootConfig.class),       // parent
    @ContextConfiguration(name = "dispatcher",
                          classes = WebConfig.class)         // child
})
public class WebLayerTest {

    @Autowired
    private OrderService orderService;   // from root context

    @Autowired
    private OrderController controller;  // from dispatcher context

    @Test
    void testController() { }
}

// Partial context override — only change specific level:
@ContextHierarchy({
    @ContextConfiguration(name = "root",
                          classes = RootConfig.class),
    @ContextConfiguration(name = "dispatcher",
                          classes = TestWebConfig.class)  // override dispatcher
})
public class CustomWebTest extends WebLayerTest { }
```

---

### 7.1.14 — Dependency Injection in Test Classes

Test instances themselves get full Spring DI:

```java
@SpringJUnitConfig(AppConfig.class)
public class InjectionTest {

    // Field injection
    @Autowired
    private OrderService orderService;

    // @Value injection
    @Value("${app.timeout}")
    private int timeout;

    // Constructor injection (JUnit 5 + SpringExtension support)
    private final OrderService orderService2;
    private final ProductService productService;

    @Autowired
    InjectionTest(OrderService orderService, ProductService productService) {
        this.orderService2 = orderService;
        this.productService = productService;
    }

    // @Qualifier
    @Autowired
    @Qualifier("premiumOrderService")
    private OrderService premiumService;

    // Inject ApplicationContext itself
    @Autowired
    private ApplicationContext ctx;

    // @Resource by name
    @Resource(name = "orderService")
    private OrderService namedService;
}
```

**Parameter injection in test methods (JUnit 5):**

```java
@SpringJUnitConfig(AppConfig.class)
public class ParameterTest {

    @Test
    void testWithInjectedParams(
            @Autowired OrderService orderService,     // injected from context
            @Value("${app.name}") String appName,    // injected from properties
            ApplicationContext ctx) {                 // injected automatically
        assertThat(orderService).isNotNull();
        assertThat(appName).isEqualTo("MyApp");
    }
}
```

---

### 7.1.15 — TestExecutionListener — Custom Extension

```java
// Custom TEL:
public class TimingTestExecutionListener
        extends AbstractTestExecutionListener {

    @Override
    public int getOrder() { return 5000; }  // run order

    @Override
    public void beforeTestMethod(TestContext testContext) {
        testContext.setAttribute("startTime", System.currentTimeMillis());
    }

    @Override
    public void afterTestMethod(TestContext testContext) {
        long start = (Long) testContext.getAttribute("startTime");
        long duration = System.currentTimeMillis() - start;
        System.out.println(testContext.getTestMethod().getName()
            + " took " + duration + "ms");
    }
}

// Register via annotation:
@SpringJUnitConfig(AppConfig.class)
@TestExecutionListeners(
    listeners = TimingTestExecutionListener.class,
    mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS
    // MERGE_WITH_DEFAULTS: adds to default listeners
    // REPLACE_DEFAULTS: replaces all default listeners
)
public class TimedTest { }

// OR register globally via META-INF/spring.factories:
// org.springframework.test.context.TestExecutionListener=\
//   com.example.TimingTestExecutionListener
```

---

### 7.1.16 — Common Pitfalls and Enterprise Patterns

**Pitfall 1 — Over-using @DirtiesContext:**

```java
// BAD — @DirtiesContext on every test defeats caching:
@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
public class SlowTest { }

// GOOD — use @Transactional for DB cleanup:
@Transactional
public class FastTest { }
```

**Pitfall 2 — @MockBean proliferation:**

```java
// BAD — unique @MockBean combos = unique contexts = slow suite:
public class TestA { @MockBean ServiceA a; }
public class TestB { @MockBean ServiceA a; @MockBean ServiceB b; }
public class TestC { @MockBean ServiceA a; @MockBean ServiceB b; @MockBean ServiceC c; }
// 3 different contexts loaded

// GOOD — consolidate mock combos in a base class:
@SpringJUnitConfig(AppConfig.class)
public abstract class BaseMockTest {
    @MockBean ServiceA a;
    @MockBean ServiceB b;
}
public class TestA extends BaseMockTest { }  // same context
public class TestB extends BaseMockTest { }  // same context — SHARED
```

**Pitfall 3 — @BeforeEach vs @BeforeTransaction:**

```java
@Transactional
public class DbTest {

    @BeforeEach  // runs INSIDE the test transaction — setup rolled back too
    void setup() { repo.save(testData); }

    @BeforeTransaction  // runs OUTSIDE transaction — data committed
    void setupReferenceData() { refDataRepo.save(referenceData); }
}
```

---

## ② CODE EXAMPLES

### Example 1 — Complete Test Class with All Features

```java
@SpringJUnitConfig(classes = {AppConfig.class, DatabaseConfig.class})
@ActiveProfiles("test")
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb;MODE=PostgreSQL",
    "app.payment.mock=true"
})
@Transactional
public class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @MockBean
    private PaymentGateway paymentGateway;

    @MockBean
    private EmailNotificationService emailService;

    @Value("${app.payment.mock}")
    private boolean paymentMockEnabled;

    @Test
    @Sql("classpath:test-data/customers.sql")
    void placeOrder_withValidCustomer_succeeds() {
        // Arrange
        when(paymentGateway.charge(any(), any()))
            .thenReturn(PaymentResult.success("TXN-001"));

        Order order = Order.builder()
            .customerId("CUST-001")
            .items(List.of(new OrderItem("PROD-A", 2, new BigDecimal("49.99"))))
            .build();

        // Act
        Order saved = orderService.placeOrder(order);

        // Assert
        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(paymentGateway).charge(eq("CUST-001"), any(BigDecimal.class));
        verify(emailService).sendConfirmation(eq("CUST-001"), any());

        // DB assertion — within same transaction
        Order fromDb = orderRepository.findById(saved.getId()).orElseThrow();
        assertThat(fromDb.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        // Transaction auto-rolled back after this method
    }

    @Test
    @Sql(scripts = "classpath:test-data/customers.sql")
    @DirtiesContext  // this test modifies singleton state
    void placeOrder_withPaymentFailure_rollsBack() {
        when(paymentGateway.charge(any(), any()))
            .thenThrow(new PaymentException("Card declined"));

        Order order = Order.builder()
            .customerId("CUST-001")
            .items(List.of(new OrderItem("PROD-A", 1, new BigDecimal("99.99"))))
            .build();

        assertThatThrownBy(() -> orderService.placeOrder(order))
            .isInstanceOf(PaymentException.class)
            .hasMessage("Card declined");

        // Verify no order saved:
        assertThat(orderRepository.count()).isEqualTo(0);
    }

    @BeforeTransaction
    void insertReferenceData() {
        // This runs outside the test transaction — committed to DB
        // and visible to all test transactions
    }

    @AfterTransaction
    void verifyCleanState() {
        // Verify DB is clean after rollback
    }
}
```

---

### Example 2 — Context Caching Demonstration

```java
// Base test class — defines context
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
@Transactional
public abstract class BaseIntegrationTest {
    @Autowired
    protected ApplicationContext ctx;

    @MockBean  // ALL subclasses use the same mock config
    protected PaymentGateway paymentGateway;
}

// These ALL share the same ApplicationContext:
public class OrderServiceTest extends BaseIntegrationTest {
    @Autowired OrderService orderService;
}

public class ProductServiceTest extends BaseIntegrationTest {
    @Autowired ProductService productService;
}

public class CustomerServiceTest extends BaseIntegrationTest {
    @Autowired CustomerService customerService;
}
// One context load for ALL three test classes

// This causes a NEW context (different profile):
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles({"test", "premium"})
public class PremiumOrderTest { }  // DIFFERENT cache key
```

---

### Example 3 — @Sql with Cleanup

```java
@SpringJUnitConfig(AppConfig.class)
public class ReportingRepositoryTest {
    // Note: NOT @Transactional — some report queries need committed data

    @Autowired
    private ReportingRepository reportingRepository;

    @Test
    @SqlGroup({
        @Sql(
            scripts = "classpath:sql/setup-report-data.sql",
            executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD,
            config = @SqlConfig(
                transactionMode = SqlConfig.TransactionMode.ISOLATED,
                errorMode = SqlConfig.ErrorMode.FAIL_ON_ERROR
            )
        ),
        @Sql(
            statements = {
                "DELETE FROM report_data",
                "DELETE FROM report_summary"
            },
            executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD,
            config = @SqlConfig(
                transactionMode = SqlConfig.TransactionMode.ISOLATED
            )
        )
    })
    void testMonthlyReport() {
        MonthlyReport report = reportingRepository
            .generateReport(YearMonth.of(2024, 1));

        assertThat(report.getTotalOrders()).isEqualTo(5);
        assertThat(report.getTotalRevenue())
            .isEqualByComparingTo(new BigDecimal("1250.00"));
    }
}
```

---

### Example 4 — @ContextHierarchy for Web Layer

```java
// Root context config (services, repositories)
@Configuration
@ComponentScan("com.example.service")
public class RootConfig { }

// Web context config (controllers, MVC)
@Configuration
@EnableWebMvc
@ComponentScan("com.example.web")
public class WebConfig { }

// Test with context hierarchy:
@SpringJUnitConfig
@ContextHierarchy({
    @ContextConfiguration(name = "root", classes = RootConfig.class),
    @ContextConfiguration(name = "web",  classes = WebConfig.class)
})
public class OrderControllerTest {

    @Autowired  // from web context (child)
    private OrderController orderController;

    @Autowired  // from root context (parent) — visible to child
    private OrderService orderService;

    @Test
    void testGetOrder() {
        ModelAndView mav = orderController.getOrder("ORD-001", new ModelMap());
        assertThat(mav.getViewName()).isEqualTo("order-detail");
        assertThat(mav.getModel().get("order")).isNotNull();
    }
}
```

---

### Example 5 — Custom TestExecutionListener

```java
@Component
public class AuditTestExecutionListener
        extends AbstractTestExecutionListener {

    @Override
    public int getOrder() { return 2000; }

    @Override
    public void beforeTestClass(TestContext ctx) {
        System.out.println("=== Test class: "
            + ctx.getTestClass().getSimpleName() + " ===");
    }

    @Override
    public void beforeTestMethod(TestContext ctx) {
        // Can access the ApplicationContext from test context
        ApplicationContext appCtx = ctx.getApplicationContext();
        AuditService auditService = appCtx.getBean(AuditService.class);
        auditService.startTestAudit(ctx.getTestMethod().getName());
    }

    @Override
    public void afterTestMethod(TestContext ctx) {
        Throwable ex = ctx.getTestException();
        ApplicationContext appCtx = ctx.getApplicationContext();
        AuditService auditService = appCtx.getBean(AuditService.class);
        auditService.endTestAudit(
            ctx.getTestMethod().getName(),
            ex == null ? "PASSED" : "FAILED"
        );
    }
}

// Using it:
@SpringJUnitConfig(AppConfig.class)
@TestExecutionListeners(
    listeners = AuditTestExecutionListener.class,
    mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS
)
public class AuditedTest { }
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which component is responsible for injecting `@Autowired` dependencies into a Spring test instance?

A) `SpringExtension.beforeEach()`
B) `DependencyInjectionTestExecutionListener`
C) `AutowiredAnnotationBeanPostProcessor`
D) `TestContextManager.prepareTestInstance()`

**Answer: B**
`DependencyInjectionTestExecutionListener.prepareTestInstance()` is responsible for performing dependency injection on the test instance. It uses the test's `ApplicationContext` to inject `@Autowired`, `@Value`, and `@Resource` into the test class fields. `SpringExtension` delegates to `TestContextManager`, which calls all registered `TestExecutionListener` implementations including this one.

---

**Q2 — Select all that apply:**
Which of the following contribute to the context cache key? (Select ALL that apply)

A) The set of `@ContextConfiguration` classes or locations
B) `@ActiveProfiles` values
C) `@TestPropertySource` locations and inline properties
D) The test class name
E) `@MockBean` definitions
F) `ContextLoader` class used
G) The number of test methods in the class

**Answer: A, B, C, E, F**
D is wrong — the test class name is NOT part of the cache key. Multiple different test classes with identical configuration share the same context. G is wrong — method count is irrelevant. The cache key is formed from: config classes/locations, active profiles, property sources, context loader, context initializers, parent context configuration, and MockBean definitions.

---

**Q3 — Code Scenario:**

```java
@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
@Transactional
public class TestA {
    @MockBean ServiceX x;
}

@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
@Transactional
public class TestB {
    @MockBean ServiceX x;
}

@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("test")
@Transactional
public class TestC {
    @MockBean ServiceX x;
    @MockBean ServiceY y;
}
```

How many `ApplicationContext` instances are created when running A, B, and C?

A) 1 — all three share the same context
B) 2 — TestA and TestB share, TestC is different
C) 3 — each creates its own context
D) It depends on test execution order

**Answer: B**
TestA and TestB have IDENTICAL cache keys: same config (`AppConfig.class`), same profile (`"test"`), same `@MockBean` set (`ServiceX`). They share one context. TestC has `@MockBean ServiceX + ServiceY` — a different mock set = different cache key = different context. Total: **2 contexts**.

---

**Q4 — Trick Question:**

```java
@SpringJUnitConfig(AppConfig.class)
@Transactional
public class OrderTest {

    @Autowired
    private OrderRepository repo;

    @BeforeEach
    void setup() {
        repo.deleteAll();
        repo.save(new Order("CUST-001"));
    }

    @Test
    void testFindOrder() {
        List<Order> orders = repo.findAll();
        assertThat(orders).hasSize(1);
    }

    @Test
    void testCountOrders() {
        assertThat(repo.count()).isEqualTo(1);
    }
}
```

Both tests pass in isolation. Will they reliably pass when run together?

A) Yes — `@BeforeEach` sets up fresh data for each test
B) No — `deleteAll()` in `@BeforeEach` is part of the test transaction and is rolled back; the next test starts with leftover data from the previous test's rolled-back transaction

**Answer: A — Yes they pass.**
`@BeforeEach` in a `@Transactional` test class runs INSIDE the test transaction. The full sequence for each test is: begin transaction → `@BeforeEach` (deleteAll + save) → test method → rollback. Since both `@BeforeEach` and the test run in the SAME transaction, and that transaction rolls back, the next test starts with a CLEAN state. The setup is isolated per test. Tests pass reliably.

---

**Q5 — MCQ:**
What is the default behavior of `@Transactional` on a Spring test method?

A) Commits after the test completes
B) Rolls back after the test completes
C) Creates a new transaction regardless of propagation
D) Rolls back only if the test throws an exception

**Answer: B**
`@Transactional` on Spring test methods **rolls back by default** after the test completes — regardless of whether the test passes or fails. This ensures test isolation. Use `@Commit` or `@Rollback(false)` to force a commit.

---

**Q6 — Select all that apply:**
When does `@DirtiesContext` cause the context to be closed and removed from the cache? (Select ALL that apply)

A) `ClassMode.AFTER_CLASS` — after all tests in the class complete
B) `ClassMode.BEFORE_CLASS` — before any test in the class runs
C) `ClassMode.AFTER_EACH_TEST_METHOD` — after each test method
D) Method-level `@DirtiesContext` — after that specific method
E) Whenever `@MockBean` is used
F) `ClassMode.BEFORE_EACH_TEST_METHOD` — before each test method

**Answer: A, B, C, D, F**
E is wrong — `@MockBean` doesn't trigger `@DirtiesContext` behavior. It adds to the cache key so a separate context is created, but `@DirtiesContext` itself is not involved. All ClassMode values (A, B, C, F) and method-level annotation (D) are valid.

---

**Q7 — MCQ:**
What is the difference between `@Mock` (Mockito) and `@MockBean` (Spring)?

A) They are identical — both create Mockito mocks
B) `@Mock` creates a Mockito mock but does NOT register it in the Spring context; `@MockBean` creates a mock AND registers it as a Spring bean replacing any existing bean of the same type
C) `@MockBean` can only be used in `@SpringBootTest`; `@Mock` works everywhere
D) `@Mock` resets between tests; `@MockBean` does not

**Answer: B**
`@Mock` is pure Mockito — it creates a mock object but does NOT interact with the Spring context. Beans in the context remain real. `@MockBean` creates a Mockito mock AND registers it in the Spring `ApplicationContext`, replacing any existing bean of the same type. Spring auto-injects this mock wherever that type is needed.

---

**Q8 — Select all that apply:**
Which of the following annotations cause `@BeforeEach` to run INSIDE the test transaction? (Select ALL that apply)

A) `@Transactional` on the test class
B) `@Transactional` on each test method individually
C) `@BeforeEach` always runs inside the transaction
D) `@BeforeTransaction` runs before the transaction opens
E) `@Sql` scripts run inside the test transaction by default

**Answer: A, B, D, E**
C is wrong — `@BeforeEach` only runs inside the transaction when the test IS transactional (A or B). D is correct — `@BeforeTransaction` is specifically designed to run BEFORE the transaction opens (outside). E is correct — `@Sql` scripts with `transactionMode = INFERRED` (default) run inside the existing transaction.

---

**Q9 — Code Output Prediction:**

```java
@SpringJUnitConfig(AppConfig.class)
public class TestA {
    @Autowired ApplicationContext ctxA;
}

@SpringJUnitConfig(AppConfig.class)
public class TestB {
    @Autowired ApplicationContext ctxB;
}

@SpringJUnitConfig(AppConfig.class)
@ActiveProfiles("prod")
public class TestC {
    @Autowired ApplicationContext ctxC;
}
```

Which of the following is TRUE?

A) `ctxA == ctxB` (same reference), `ctxA != ctxC`
B) `ctxA != ctxB != ctxC` (all different references)
C) `ctxA == ctxB == ctxC` (all same reference)
D) `ctxA != ctxB` (different references), `ctxA == ctxC`

**Answer: A**
TestA and TestB have identical cache keys (`AppConfig.class`, no profiles, no property sources). They share the SAME `ApplicationContext` — `ctxA == ctxB` (same object reference). TestC adds `@ActiveProfiles("prod")` — different cache key — different context. `ctxA != ctxC`.

---

**Q10 — Scenario-based:**
A test suite runs 50 test classes. All use `@SpringJUnitConfig(AppConfig.class)`. Developer adds `@DirtiesContext` to one test class. What is the WORST CASE number of context loads?

A) 1 — context still cached for all others
B) 2 — one context before the dirty class, one after
C) 50 — each class creates a new context
D) Depends on where the dirty class is in execution order

**Answer: D**
If the `@DirtiesContext` class runs FIRST, Spring loads 1 context, marks it dirty (closes after class), then the remaining 49 classes share a NEW context — total: **2 loads**. If it runs LAST, 49 classes share the cached context, then the last class loads a new one — total: **2 loads**. If it runs in the MIDDLE (position N), classes 1 to N-1 share one context, then class N dirties it, then classes N+1 to 50 share a new context — total: **2 loads** (assuming all have same config). Actually, worst case is consistently **2** — the dirty class causes exactly one additional load.

---

## ④ TRICK ANALYSIS

**Trap 1: @Transactional test commits by default**
The single most common misconception. `@Transactional` on a TEST method/class **rolls back by default**. In production code, `@Transactional` commits. In test code, it rolls back. Use `@Commit` or `@Rollback(false)` to override.

**Trap 2: @MockBean always creates a fresh context**
Wrong. `@MockBean` becomes part of the cache key. If two test classes declare the same `@MockBean` types (same set), they SHARE the same context. Only unique combinations of `@MockBean` types trigger new context creation.

**Trap 3: @BeforeEach runs outside the transaction**
Wrong when class is `@Transactional`. `@BeforeEach` runs INSIDE the test transaction and its changes are rolled back. Use `@BeforeTransaction` for setup that must commit and be visible across transactions.

**Trap 4: @TestConfiguration replaces AppConfig**
Wrong. `@TestConfiguration` ADDS beans to the existing context — it does NOT replace the main configuration. To override a bean, the `@TestConfiguration` bean needs `@Primary` or the override feature. `@Configuration` in a test's inner class can act as the full context config if used in `@ContextConfiguration`.

**Trap 5: Context cache is per test class**
Wrong. The context cache is a **static JVM-level cache** shared across ALL test classes in the same JVM run. Multiple test classes with the same configuration share one `ApplicationContext`. This is the key to test suite performance.

**Trap 6: @Sql runs outside transactions**
Wrong by default. `@Sql` with `transactionMode = INFERRED` (the default) runs inside the current test transaction. Changes made by `@Sql` are rolled back with the test transaction. Use `transactionMode = ISOLATED` for `@Sql` to run in its own separate transaction that commits independently.

**Trap 7: @DirtiesContext is needed for @Transactional tests**
Wrong. `@Transactional` tests don't need `@DirtiesContext` for database cleanup — the rollback handles isolation. `@DirtiesContext` is for non-rollback-able state changes (static fields, caches, embedded servers).

**Trap 8: @SpringJUnitConfig inner @Configuration replaces all config**
Nuanced. An inner `static @Configuration` class inside a `@SpringJUnitConfig` test (with no explicit config specified) acts as the ENTIRE context configuration — not supplemental. There is no "merging" with other configs unless you explicitly use `@Import`.

**Trap 9: ActiveProfiles on derived test class replaces parent's profiles**
Wrong with default `inheritProfiles = true`. With the default, derived class ADDS profiles to parent's profiles. `inheritProfiles = false` replaces parent's profiles. Many candidates assume replacement by default.

**Trap 10: @Value fields in test class are re-evaluated per test method**
Wrong. `@Value` fields are injected ONCE when the test instance is created (before the first test method). They are NOT re-injected between test methods. Test instance fields are populated once per instance. JUnit 5 creates a new test instance per method by default, so `@Value` IS re-evaluated per test method when using JUnit 5's default `PER_METHOD` lifecycle.

**Trap 11: @Sql class-level and method-level annotations replace each other**
Wrong. Class-level `@Sql` and method-level `@Sql` are BOTH executed. The class-level script runs first, then the method-level script. They are additive, not exclusive.

**Trap 12: SpringExtension is automatically applied with @SpringJUnitConfig**
TRUE — and this is the correct behaviour. `@SpringJUnitConfig` is a meta-annotation that INCLUDES `@ExtendWith(SpringExtension.class)`. You do NOT need both.

---

## ⑤ SUMMARY SHEET

### TestContext Framework Component Roles

```
TestContextManager       → Central coordinator per test class
                           Creates TestContext, notifies all TELs
TestContext              → Holds test class/method/instance + ApplicationContext
                           Caches context per MergedContextConfiguration key
TestExecutionListener    → Hooks for before/after test class, method, instance
ContextLoader            → Creates the ApplicationContext from config
SmartContextLoader       → Extended — processes both classes and locations
MergedContextConfiguration → The cache key object
```

---

### Default TestExecutionListener Execution Order

```
Order  | Listener                                    | Responsibility
-------|---------------------------------------------|---------------------------
1000   | ServletTestExecutionListener                | MockHttpServletRequest setup
1500   | DirtiesContextBeforeModesTestExecListener   | @DirtiesContext BEFORE modes
1900   | ApplicationEventsTestExecutionListener      | ApplicationEvents recording
2000   | DependencyInjectionTestExecutionListener    | @Autowired injection
3000   | DirtiesContextTestExecutionListener         | @DirtiesContext AFTER modes
4000   | TransactionalTestExecutionListener          | @Transactional rollback
5000   | SqlScriptsTestExecutionListener             | @Sql execution
```

---

### Context Cache Key Components

```
Cache Key = MergedContextConfiguration containing:
  ✓ Configuration classes (from @ContextConfiguration classes=)
  ✓ Configuration locations (from @ContextConfiguration locations=)
  ✓ Active profiles (@ActiveProfiles)
  ✓ Property source locations (@TestPropertySource locations=)
  ✓ Inline properties (@TestPropertySource properties=)
  ✓ Context initializers
  ✓ Context loader class
  ✓ Parent context MergedContextConfiguration
  ✓ MockBean definitions (@MockBean types)

  ✗ Test class name (NOT part of key)
  ✗ Test method names (NOT part of key)
  ✗ Number of test methods (NOT part of key)
```

---

### @Transactional Test Behavior

| Annotation | Default behavior | Override |
|---|---|---|
| `@Transactional` on class | All methods roll back | `@Commit` on method |
| `@Transactional` on method | That method rolls back | `@Commit` on same method |
| `@Commit` | Commits | — |
| `@Rollback(false)` | Commits | — |
| `@Rollback(true)` | Rolls back (explicit) | — |
| `@BeforeEach` | Inside transaction | Use `@BeforeTransaction` |
| `@AfterEach` | Inside transaction | Use `@AfterTransaction` |

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `@SpringJUnitConfig` includes | `@ExtendWith(SpringExtension.class)` — do NOT add separately |
| Context cache scope | Static JVM-level — shared across ALL test classes in same run |
| Cache invalidators | `@DirtiesContext`, unique `@MockBean` combos |
| `@Transactional` test default | **ROLLBACK** — opposite of production code default |
| `@BeforeEach` + `@Transactional` class | Runs INSIDE transaction — rolled back |
| `@BeforeTransaction` | Runs OUTSIDE transaction — committed |
| `@MockBean` cache key | Same type set = same key = shared context |
| `@TestConfiguration` | ADDS beans — does NOT replace main config |
| `@TestPropertySource` priority | HIGHEST among PropertySources |
| `@ActiveProfiles` inheritance | `inheritProfiles=true` (default) ADDS profiles |
| `@Sql` default phase | `BEFORE_TEST_METHOD` |
| `@Sql` class + method | BOTH execute — additive, not exclusive |
| `@Sql` transactionMode default | `INFERRED` — runs inside existing test transaction |
| `@DirtiesContext` for DB cleanup | NOT needed — use `@Transactional` rollback |
| Inner `@Configuration` in test | Acts as ENTIRE config (not supplemental) |
| Inner `@TestConfiguration` in test | SUPPLEMENTAL — adds to loaded config |

---

### Annotation Quick Reference

```
Core:
  @SpringJUnitConfig          = @ExtendWith(SpringExtension) + @ContextConfiguration
  @SpringJUnitWebConfig       = @ExtendWith(SpringExtension) + @ContextConfiguration
                                + @WebAppConfiguration

Context config:
  @ContextConfiguration       → specify config classes/XML locations
  @ContextHierarchy           → parent-child context setup
  @ActiveProfiles             → activate profiles (additive with inheritProfiles=true)
  @TestPropertySource         → highest-priority property overrides

Bean overrides:
  @TestConfiguration          → add beans to existing context
  @MockBean                   → Mockito mock registered as Spring bean
  @SpyBean                    → Mockito spy wrapping real bean

Transaction control:
  @Transactional (test)       → ROLLBACK by default (opposite of production)
  @Commit                     → force commit
  @Rollback(true/false)       → explicit control
  @BeforeTransaction          → runs BEFORE transaction opens
  @AfterTransaction           → runs AFTER transaction closes

Database scripts:
  @Sql                        → execute SQL scripts/statements
  @SqlGroup                   → multiple @Sql annotations
  @SqlConfig                  → configure script execution
  @SqlMergeMode               → how class+method @Sql combine

Context lifecycle:
  @DirtiesContext             → close context after test (don't use for DB)
  @TestExecutionListeners     → register custom/additional TELs
```

---

### Interview One-Liners

- "The Spring test context cache is a **static JVM-level** cache — multiple test classes with identical configuration share a single `ApplicationContext`, never reloading it."
- "`@Transactional` on a test method **rolls back by default** — the opposite of production behavior. Use `@Commit` to force persistence."
- "`@MockBean` becomes part of the context cache key — two test classes with the same `@MockBean` set share one context. Each unique mock combination forces a new context load."
- "`@BeforeEach` in a `@Transactional` test class runs **inside** the test transaction and is rolled back with it. Use `@BeforeTransaction` for setup that must outlive the test transaction."
- "`@SpringJUnitConfig` is a composed annotation that already includes `@ExtendWith(SpringExtension.class)` — adding it separately is redundant."
- "`@TestConfiguration` **adds** beans to the existing context; it does NOT replace it. Inner `@Configuration` classes act as the ENTIRE context configuration."
- "`@TestPropertySource` has the **highest priority** among all `PropertySources` — it overrides system properties, environment variables, and `@PropertySource` files."
- "`@DirtiesContext` is NOT needed for database test isolation — `@Transactional` rollback handles that. Use `@DirtiesContext` only for non-rollback-able state changes like static fields or embedded servers."
- "Class-level and method-level `@Sql` annotations are **additive** — both execute. The class-level script runs first, then the method-level script."
- "`@ActiveProfiles` with `inheritProfiles=true` (the default) **adds** to parent class profiles — it does NOT replace them. Use `inheritProfiles=false` to replace."
- "The context cache key does NOT include the test class name — multiple test classes with identical config are treated as identical consumers of the same context."
- "`@Sql` with default `transactionMode=INFERRED` runs **inside** the existing test transaction and is rolled back with it. Use `transactionMode=ISOLATED` for `@Sql` to commit independently."

---
