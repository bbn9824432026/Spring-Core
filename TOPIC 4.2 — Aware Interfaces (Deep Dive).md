# TOPIC 4.2 — Aware Interfaces (Deep Dive)

---

## ① CONCEPTUAL EXPLANATION

### What Are Aware Interfaces?

Aware interfaces are Spring's mechanism for beans to receive references to Spring infrastructure objects. They follow a consistent naming convention: `XxxAware` where `Xxx` is the infrastructure component being injected.

The fundamental design question: **why not just use `@Autowired` for everything?**

```
@Autowired ApplicationContext ctx;  // works
vs
implements ApplicationContextAware  // also works

Why do Aware interfaces exist at all?
```

**Reasons Aware interfaces exist:**

1. **Pre-annotation era:** Aware interfaces predate Spring's annotation support. They were the only mechanism before Spring 2.5.

2. **Timing guarantee:** Aware callbacks (specifically BeanNameAware, BeanClassLoaderAware, BeanFactoryAware) are invoked BEFORE BPP processing — before `@Autowired` injection could theoretically be overridden or interfered with.

3. **No annotation processing required:** Aware interfaces work even in a plain `BeanFactory` without annotation processing BPPs registered.

4. **Framework internals:** Spring's own internal beans use Aware interfaces to avoid circular dependency issues with the container itself.

5. **Reliability in edge cases:** Works for BFPP beans (which don't get full BPP processing), infrastructure beans, and beans that need guaranteed early access to container components.

---

### 4.2.1 — Complete Aware Interface Taxonomy

Spring provides a rich hierarchy of Aware interfaces:

#### Group 1 — Core Container Aware (invoked directly, not via BPP)

```java
// 1. BeanNameAware — provides the bean's name as registered in the container
public interface BeanNameAware extends Aware {
    void setBeanName(String name);
}

// 2. BeanClassLoaderAware — provides the ClassLoader used to load the bean class
public interface BeanClassLoaderAware extends Aware {
    void setBeanClassLoader(ClassLoader classLoader);
}

// 3. BeanFactoryAware — provides the BeanFactory that owns this bean
public interface BeanFactoryAware extends Aware {
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}
```

These three are invoked by `AbstractAutowireCapableBeanFactory.invokeAwareMethods()`:

```java
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware beanNameAware) {
            beanNameAware.setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                beanClassLoaderAware.setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware beanFactoryAware) {
            beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

**Critical:** `setBeanFactory()` provides the `AbstractAutowireCapableBeanFactory` (the internal factory), NOT the `ApplicationContext`. The factory reference is the raw `BeanFactory`, which is a narrower view than `ApplicationContext`.

#### Group 2 — ApplicationContext Aware (invoked via BPP)

These are handled by `ApplicationContextAwareProcessor` — a `BeanPostProcessor` registered in `prepareBeanFactory()` during `refresh()`:

```java
// Invocation order within ApplicationContextAwareProcessor:

// 4. EnvironmentAware — access to the Spring Environment (profiles, properties)
public interface EnvironmentAware extends Aware {
    void setEnvironment(Environment environment);
}

// 5. EmbeddedValueResolverAware — access to ${...} placeholder resolution
public interface EmbeddedValueResolverAware extends Aware {
    void setEmbeddedValueResolver(StringValueResolver resolver);
}

// 6. ResourceLoaderAware — access to resource loading capabilities
public interface ResourceLoaderAware extends Aware {
    void setResourceLoader(ResourceLoader resourceLoader);
}

// 7. ApplicationEventPublisherAware — access to event publishing
public interface ApplicationEventPublisherAware extends Aware {
    void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
}

// 8. MessageSourceAware — access to internationalization (i18n) support
public interface MessageSourceAware extends Aware {
    void setMessageSource(MessageSource messageSource);
}

// 9. ApplicationContextAware — full access to the ApplicationContext
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

`ApplicationContextAwareProcessor` implementation:

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {

    private final ConfigurableApplicationContext applicationContext;
    private final StringValueResolver embeddedValueResolver;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof EnvironmentAware ||
            bean instanceof EmbeddedValueResolverAware ||
            bean instanceof ResourceLoaderAware ||
            bean instanceof ApplicationEventPublisherAware ||
            bean instanceof MessageSourceAware ||
            bean instanceof ApplicationContextAware) {
            invokeAwareInterfaces(bean);
        }
        return bean;
    }

    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof EnvironmentAware ea) {
            ea.setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware evra) {
            evra.setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware rla) {
            rla.setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware aepa) {
            aepa.setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware msa) {
            msa.setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware aca) {
            aca.setApplicationContext(this.applicationContext);
        }
    }
}
```

**Key insight:** `ApplicationContext` implements `ResourceLoader`, `ApplicationEventPublisher`, and `MessageSource`. So the SAME `ApplicationContext` object is passed for all of these — they are just different views of the same object.

---

### 4.2.2 — Web-Specific Aware Interfaces

```java
// 10. ServletContextAware — provides the javax.servlet.ServletContext
public interface ServletContextAware extends Aware {
    void setServletContext(ServletContext servletContext);
}

// 11. ServletConfigAware — provides the javax.servlet.ServletConfig
public interface ServletConfigAware extends Aware {
    void setServletConfig(ServletConfig servletConfig);
}
```

These are handled by `ServletContextAwareProcessor` in web ApplicationContexts. They are NOT called in non-web environments.

---

### 4.2.3 — Import-Aware and Other Framework Aware Interfaces

```java
// ImportAware — for @Import-ed @Configuration classes that need to know their import metadata
public interface ImportAware extends Aware {
    void setImportMetadata(AnnotationMetadata importMetadata);
}

// LoadTimeWeaverAware — for beans needing access to load-time weaving infrastructure
public interface LoadTimeWeaverAware extends Aware {
    void setLoadTimeWeaver(LoadTimeWeaver loadTimeWeaver);
}

// NotificationPublisherAware — JMX notification publishing
public interface NotificationPublisherAware extends Aware {
    void setNotificationPublisher(NotificationPublisher notificationPublisher);
}
```

`ImportAware` is particularly important — it allows `@Configuration` classes that were `@Import`-ed to access the annotation metadata of the class that imported them:

```java
@Configuration
public class MainConfig {
    @Import(ConditionalConfig.class)
    // Passes AnnotationMetadata of MainConfig to ConditionalConfig
}

@Configuration
public class ConditionalConfig implements ImportAware {

    private AnnotationMetadata importMetadata;

    @Override
    public void setImportMetadata(AnnotationMetadata metadata) {
        this.importMetadata = metadata;
        // Can now inspect annotations on the importing class
    }
}
```

This pattern is used extensively in Spring Boot's auto-configuration and `@Enable*` annotations.

---

### 4.2.4 — BeanNameAware — Deep Dive

```java
public interface BeanNameAware extends Aware {
    void setBeanName(String name);
}
```

**What name is provided?**
The name is the bean's primary name as registered in the `BeanDefinitionRegistry`. It is NOT an alias. If the bean has multiple names, only the primary name (first name) is provided.

```java
@Bean(name = {"primary", "secondary", "tertiary"})
public MyService myService() { return new MyService(); }
// setBeanName receives: "primary" (the first name)
```

**Use cases for BeanNameAware:**
1. Logging — include the bean name in log messages
2. JMX — register the bean with a name-based MBean
3. Cache key prefixing — use bean name as part of a cache key
4. Dynamic configuration — look up properties based on bean name

**Pitfall:** Using `setBeanName()` to store the name for later `getBean(name)` calls creates tight coupling. Prefer `@Autowired` or `ObjectProvider` for dependency access.

---

### 4.2.5 — BeanFactoryAware — Deep Dive

```java
public interface BeanFactoryAware extends Aware {
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}
```

**What `BeanFactory` reference is provided?**

The reference is the INTERNAL `DefaultListableBeanFactory` (or a wrapper that exposes it). It is the same factory that the `ApplicationContext` delegates to. Crucially, it is NOT the `ApplicationContext` itself.

```java
@Component
public class MyBean implements BeanFactoryAware {
    private ConfigurableListableBeanFactory factory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        // Cast to access more methods:
        this.factory = (ConfigurableListableBeanFactory) beanFactory;
        // Now can access: getBeanDefinition(), getBeanNamesForType(), etc.
    }
}
```

**Use cases for BeanFactoryAware:**
1. Programmatic bean lookup (service locator pattern)
2. Prototype bean retrieval from singleton scope
3. BeanDefinition inspection at runtime
4. Programmatic bean registration (via `DefaultListableBeanFactory`)

**Anti-pattern warning:** `BeanFactoryAware` for routine dependency injection is an anti-pattern — it makes the service locator pattern appear, hiding dependencies and making testing harder. Use `@Autowired` or `ObjectProvider` instead.

**Legitimate use:** `BeanFactoryAware` is legitimate when:
- The type of dependency is not known at compile time
- The bean is a framework component that needs to manage other beans
- You are implementing a custom scope

---

### 4.2.6 — ApplicationContextAware — Deep Dive

```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

`ApplicationContextAware` provides the richest access — the full `ApplicationContext` interface, which extends:
- `BeanFactory` — bean retrieval
- `ListableBeanFactory` — bulk bean queries
- `HierarchicalBeanFactory` — parent factory access
- `MessageSource` — i18n
- `ApplicationEventPublisher` — event publishing
- `ResourcePatternResolver` — resource loading

**When to use ApplicationContextAware vs @Autowired ApplicationContext:**

Both work and produce the same result. The differences are subtle:

| | `ApplicationContextAware` | `@Autowired ApplicationContext` |
|---|---|---|
| Timing | Guaranteed before `@PostConstruct` | Same timing (both before init callbacks... actually same BPP step) |
| Works in BeanFactory | Only `BeanFactoryAware` works in plain BF | `@Autowired` requires annotation processing BPP |
| Works for BFPPs | Yes (direct call) | No — BFPPs don't get full BPP processing |
| Explicit contract | Yes — class declares it needs the context | Less obvious |
| Testability | Must set mock via setter | Must inject via test infrastructure |
| Spring recommendation | Use `@Autowired` for most cases | Preferred for application code |

---

### 4.2.7 — EmbeddedValueResolverAware — Deep Dive

This is one of the least-known but most useful Aware interfaces:

```java
public interface EmbeddedValueResolverAware extends Aware {
    void setEmbeddedValueResolver(StringValueResolver resolver);
}
```

The `StringValueResolver` can resolve:
- `${property.key}` — property placeholders from `PropertySources`
- `#{SpEL expression}` — Spring Expression Language expressions

```java
@Component
public class ConfigurableProcessor implements EmbeddedValueResolverAware {

    private StringValueResolver resolver;

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.resolver = resolver;
    }

    public void processConfig(String configKey) {
        // Resolve a property at runtime (after the key is known)
        String value = resolver.resolveStringValue("${" + configKey + "}");
        System.out.println(configKey + " = " + value);
    }

    public String evaluateExpression(String spel) {
        // Evaluate SpEL at runtime
        return resolver.resolveStringValue("#{" + spel + "}");
    }
}
```

This is more powerful than `@Value` because the key to resolve can be determined at runtime (not compile-time).

---

### 4.2.8 — ResourceLoaderAware — Deep Dive

```java
public interface ResourceLoaderAware extends Aware {
    void setResourceLoader(ResourceLoader resourceLoader);
}
```

`ResourceLoader` provides access to Spring's `Resource` abstraction for loading files, classpaths, URLs, and other resources:

```java
@Component
public class TemplateLoader implements ResourceLoaderAware {

    private ResourceLoader resourceLoader;

    @Override
    public void setResourceLoader(ResourceLoader loader) {
        this.resourceLoader = loader;
    }

    public String loadTemplate(String location) throws IOException {
        Resource resource = resourceLoader.getResource(location);
        // location examples:
        //   "classpath:templates/email.html"
        //   "file:/etc/config/template.html"
        //   "https://config.example.com/template.html"

        try (InputStream is = resource.getInputStream()) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}
```

**Resource prefix conventions:**
- `classpath:` — load from classpath
- `classpath*:` — load from ALL classpath locations (including JARs)
- `file:` — load from filesystem
- `https:` — load from URL
- No prefix — depends on context type (`ClassPathXmlApplicationContext` → classpath, `FileSystemXmlApplicationContext` → filesystem)

---

### 4.2.9 — ImportAware — Framework Pattern

`ImportAware` enables the `@Enable*` annotation pattern used throughout Spring:

```java
// Step 1: Define the @Enable annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
    int order() default Ordered.LOWEST_PRECEDENCE;
}

// Step 2: The imported @Configuration implements ImportAware
@Configuration
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration implements ImportAware {

    @Override
    public void setImportMetadata(AnnotationMetadata importMetadata) {
        // Read @EnableAsync attributes from the class that used @EnableAsync
        Map<String, Object> enableAsync =
            importMetadata.getAnnotationAttributes(EnableAsync.class.getName());
        this.enableMethodAnnotation = (boolean) enableAsync.get("proxyTargetClass");
        this.mode = (AdviceMode) enableAsync.get("mode");
        this.order = (Integer) enableAsync.get("order");
    }
}
```

This pattern allows `@EnableAsync` to configure its behavior based on what attributes the user put on `@EnableAsync` — without hardcoding them in the `@Configuration` class.

---

### 4.2.10 — Aware Interface Invocation Order — Complete Picture

The complete ordered sequence of Aware callbacks during a bean's lifecycle:

```
Step 1: Bean instantiated (constructor)
Step 2: DI (populateBean)

Step 3: invokeAwareMethods() — DIRECT, before any BPP:
    a. BeanNameAware.setBeanName()
    b. BeanClassLoaderAware.setBeanClassLoader()
    c. BeanFactoryAware.setBeanFactory()

Step 4: BPP.postProcessBeforeInitialization() — VIA ApplicationContextAwareProcessor:
    d. EnvironmentAware.setEnvironment()
    e. EmbeddedValueResolverAware.setEmbeddedValueResolver()
    f. ResourceLoaderAware.setResourceLoader()
    g. ApplicationEventPublisherAware.setApplicationEventPublisher()
    h. MessageSourceAware.setMessageSource()
    i. ApplicationContextAware.setApplicationContext()

    THEN (in the same postProcessBeforeInitialization step, via CommonAnnotationBPP):
    j. @PostConstruct

Step 5: invokeInitMethods():
    k. InitializingBean.afterPropertiesSet()
    l. Custom init-method

Step 6: BPP.postProcessAfterInitialization():
    → AOP proxy created
```

---

### When Aware Interfaces Are NOT Needed (Modern Spring)

In modern Spring (4.x+), almost all Aware interface use cases have `@Autowired` equivalents:

```java
// OLD: Aware interface
@Component
public class OldStyle implements ApplicationContextAware {
    private ApplicationContext ctx;
    public void setApplicationContext(ApplicationContext ctx) { this.ctx = ctx; }
}

// MODERN: @Autowired
@Component
public class NewStyle {
    @Autowired
    private ApplicationContext ctx; // equivalent — simpler
}

// OLD:
public class OldBean implements BeanNameAware {
    private String beanName;
    public void setBeanName(String name) { this.beanName = name; }
}

// MODERN:
public class NewBean {
    // No equivalent for BeanNameAware — must use Aware interface
    // OR inject ApplicationContext and call getBeansOfType + identify self
    // BeanNameAware is the only clean way for this
}
```

**`BeanNameAware` has no `@Autowired` equivalent** — it is the one Aware interface that remains relevant in modern Spring with no clean alternative.

---

### Common Misconceptions

**Misconception 1:** "`BeanFactoryAware` provides an `ApplicationContext`."
Wrong. It provides the raw `BeanFactory` (specifically the `AbstractAutowireCapableBeanFactory`), NOT the `ApplicationContext`. To get the `ApplicationContext`, use `ApplicationContextAware`.

**Misconception 2:** "All Aware callbacks are invoked by `ApplicationContextAwareProcessor`."
Wrong. `BeanNameAware`, `BeanClassLoaderAware`, and `BeanFactoryAware` are invoked DIRECTLY by `invokeAwareMethods()` — not via any BPP. Only callbacks 4-9 are invoked by `ApplicationContextAwareProcessor`.

**Misconception 3:** "Aware interfaces are obsolete in Spring 5+."
Wrong. `BeanNameAware` has no clean annotation alternative. `BeanFactoryAware` and `ApplicationContextAware` are still valid for framework/library code. `ImportAware` is still the only mechanism for reading `@Enable*` annotation attributes.

**Misconception 4:** "`setBeanFactory()` is called AFTER `@PostConstruct`."
Wrong. `setBeanFactory()` is called in `invokeAwareMethods()` BEFORE `postProcessBeforeInitialization()` (which contains `@PostConstruct`). `BeanFactoryAware` is always available by the time `@PostConstruct` runs.

**Misconception 5:** "`setApplicationContext()` provides a different object than `@Autowired ApplicationContext`."
Wrong. Both provide the SAME `ApplicationContext` instance. `ApplicationContextAwareProcessor` stores the `ApplicationContext` at construction time and provides it via `setApplicationContext()`.

**Misconception 6:** "`ResourceLoaderAware` provides a separate object from `ApplicationContext`."
Wrong. `ApplicationContext` implements `ResourceLoader`. `ApplicationContextAwareProcessor` passes `this.applicationContext` as the `ResourceLoader`. It is the SAME object.

---

## ② CODE EXAMPLES

### Example 1 — All Core Aware Interfaces

```java
@Component
public class CoreAwareDemo
        implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware {

    private String beanName;
    private ClassLoader classLoader;
    private ConfigurableListableBeanFactory factory;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("Bean name: " + name);
        // Use case: dynamic logging, JMX naming, cache key prefix
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
        System.out.println("ClassLoader: " + classLoader.getClass().getSimpleName());
        // Use case: dynamic class loading, custom resource loading
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        // Cast to richer interface for BeanDefinition access
        this.factory = (ConfigurableListableBeanFactory) beanFactory;

        // Check BeanDefinition for this bean:
        BeanDefinition bd = factory.getBeanDefinition(this.beanName != null ? this.beanName : "");
        System.out.println("Scope: " + bd.getScope());
        System.out.println("Lazy: " + bd.isLazyInit());
    }

    @PostConstruct
    public void verifyOrder() {
        // All Aware callbacks already ran:
        System.out.println("In @PostConstruct — beanName: " + beanName); // not null ✓
        System.out.println("In @PostConstruct — factory: " + (factory != null)); // true ✓
    }
}
```

---

### Example 2 — ApplicationContextAware for Dynamic Bean Lookup

```java
@Service
public class ReportGeneratorFactory implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    // Service locator pattern — legitimate use when type is dynamic
    public ReportGenerator getGenerator(String format) {
        // All beans of type ReportGenerator keyed by bean name
        Map<String, ReportGenerator> generators =
            ctx.getBeansOfType(ReportGenerator.class);

        ReportGenerator gen = generators.get(format.toLowerCase() + "ReportGenerator");
        if (gen == null) {
            throw new UnsupportedOperationException("No generator for format: " + format);
        }
        return gen;
    }

    // Also: prototype bean retrieval — new instance each time
    public ShoppingCart createNewCart() {
        return ctx.getBean("shoppingCart", ShoppingCart.class); // prototype scope
    }
}
```

---

### Example 3 — BeanNameAware for Self-Identification

```java
@Component
public class MetricsCollector implements BeanNameAware {

    private String beanName;
    private MeterRegistry meterRegistry;

    @Autowired
    public MetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    @PostConstruct
    public void registerMetrics() {
        // Register gauge with bean-name-based metric name
        meterRegistry.gauge("spring.bean." + beanName + ".active",
            this, m -> m.isActive() ? 1 : 0);
    }

    // What BeanNameAware provides that @Autowired cannot:
    // The bean's own name as registered in the container
    // There is NO annotation equivalent for this
    public boolean isActive() { return true; }
}
```

---

### Example 4 — EmbeddedValueResolverAware for Dynamic Resolution

```java
@Component
public class DynamicPropertyResolver implements EmbeddedValueResolverAware {

    private StringValueResolver resolver;

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.resolver = resolver;
    }

    // Static resolution — known at compile time (use @Value instead)
    // @Value("${app.timeout}") int timeout — easier

    // Dynamic resolution — key is determined at runtime (EmbeddedValueResolverAware needed)
    public String resolveProperty(String key) {
        return resolver.resolveStringValue("${" + key + ":DEFAULT_VALUE}");
    }

    // SpEL evaluation at runtime
    public String evaluateSpEL(String expression) {
        return resolver.resolveStringValue("#{" + expression + "}");
    }

    // Real use case: plugin system where each plugin has its own property namespace
    public Map<String, String> loadPluginConfig(String pluginName) {
        Map<String, String> config = new HashMap<>();
        String[] keys = {"host", "port", "timeout", "retries"};
        for (String key : keys) {
            String value = resolveProperty("plugin." + pluginName + "." + key);
            if (value != null) config.put(key, value);
        }
        return config;
    }
}
```

---

### Example 5 — ResourceLoaderAware for File Loading

```java
@Component
public class TemplateEngine implements ResourceLoaderAware {

    private ResourceLoader resourceLoader;
    private final Map<String, String> templateCache = new ConcurrentHashMap<>();

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    public String loadTemplate(String templatePath) {
        return templateCache.computeIfAbsent(templatePath, path -> {
            try {
                Resource resource = resourceLoader.getResource(path);

                if (!resource.exists()) {
                    throw new TemplateNotFoundException("Template not found: " + path);
                }

                return new String(
                    resource.getInputStream().readAllBytes(),
                    StandardCharsets.UTF_8
                );
            } catch (IOException e) {
                throw new TemplateLoadException("Failed to load: " + path, e);
            }
        });
    }

    @PostConstruct
    public void preloadCriticalTemplates() {
        // Pre-load templates at startup
        loadTemplate("classpath:templates/email-welcome.html");
        loadTemplate("classpath:templates/email-reset.html");
        System.out.println("Critical templates preloaded");
    }
}
```

---

### Example 6 — ImportAware for Custom @Enable Annotation

```java
// Step 1: Define the enable annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AuditConfiguration.class)
public @interface EnableAudit {
    String auditTable() default "audit_log";
    boolean includeReadOperations() default false;
    AuditLevel level() default AuditLevel.WRITE_ONLY;

    enum AuditLevel { WRITE_ONLY, ALL }
}

// Step 2: ImportAware @Configuration reads the annotation attributes
@Configuration
public class AuditConfiguration implements ImportAware {

    private String auditTable;
    private boolean includeReadOps;
    private EnableAudit.AuditLevel level;

    @Override
    public void setImportMetadata(AnnotationMetadata importMetadata) {
        // Read the @EnableAudit attributes from the importing class
        Map<String, Object> attrs =
            importMetadata.getAnnotationAttributes(EnableAudit.class.getName());

        this.auditTable      = (String)               attrs.get("auditTable");
        this.includeReadOps  = (Boolean)              attrs.get("includeReadOperations");
        this.level           = (EnableAudit.AuditLevel) attrs.get("level");

        System.out.println("Audit configured: table=" + auditTable
            + ", readOps=" + includeReadOps + ", level=" + level);
    }

    @Bean
    public AuditService auditService(DataSource ds) {
        return new AuditService(ds, auditTable, includeReadOps, level);
    }

    @Bean
    public AuditInterceptor auditInterceptor(AuditService auditService) {
        return new AuditInterceptor(auditService);
    }
}

// Step 3: User enables the feature
@Configuration
@EnableAudit(
    auditTable = "security_audit",
    includeReadOperations = true,
    level = EnableAudit.AuditLevel.ALL
)
public class AppConfig {
    // AuditConfiguration.setImportMetadata() receives AnnotationMetadata of THIS class
    // AuditService created with security_audit table, read ops enabled, ALL level
}
```

---

### Example 7 — BeanFactoryAware for Programmatic Prototype Creation

```java
@Service
public class CommandFactory implements BeanFactoryAware {

    private ConfigurableListableBeanFactory factory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.factory = (ConfigurableListableBeanFactory) beanFactory;
    }

    // Create new prototype command instance — legitimate BeanFactoryAware use
    public <T extends Command> T createCommand(Class<T> commandType) {
        // getBean() on a prototype always returns new instance
        return factory.getBean(commandType);
    }

    // Inspect bean metadata programmatically
    public void inspectBean(String beanName) {
        if (factory.containsBeanDefinition(beanName)) {
            BeanDefinition bd = factory.getBeanDefinition(beanName);
            System.out.println(beanName + ":");
            System.out.println("  scope: " + bd.getScope());
            System.out.println("  class: " + bd.getBeanClassName());
            System.out.println("  lazy:  " + bd.isLazyInit());
            System.out.println("  primary: " + bd.isPrimary());
        }
    }
}
```

---

### Example 8 — Aware Interface Timing Verification

```java
@Component
public class TimingVerificationBean
        implements BeanNameAware, ApplicationContextAware {

    private String beanName;
    private ApplicationContext ctx;

    // This field is @Autowired — injected in populateBean (Step 4)
    @Autowired
    private SomeService someService;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        // At this point: someService is ALREADY injected (populateBean ran in Step 4)
        // Aware callbacks run AFTER DI
        System.out.println("setBeanName — someService is: " +
            (someService != null ? "INJECTED" : "NULL"));
        // prints: INJECTED ← DI happens before Aware callbacks
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
        System.out.println("setApplicationContext — beanName: " + beanName);
        // beanName is already set ← BeanNameAware runs before ApplicationContextAware
    }

    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct — ctx: " +
            (ctx != null ? "AVAILABLE" : "NULL"));
        // ctx is available ← ApplicationContextAware runs before @PostConstruct
    }
}
```

Output:
```
setBeanName — someService is: INJECTED
setApplicationContext — beanName: timingVerificationBean
@PostConstruct — ctx: AVAILABLE
```

---

### Example 9 — Aware Interfaces in BFPP Context

```java
// BFPPs get ONLY BeanNameAware, BeanClassLoaderAware, BeanFactoryAware
// They do NOT get ApplicationContextAware (ApplicationContextAwareProcessor not yet registered)
@Component
public class CustomBFPP implements BeanFactoryPostProcessor,
                                    BeanNameAware,
                                    ApplicationContextAware { // This WON'T be called!

    private String beanName;
    private ApplicationContext ctx; // Will remain NULL

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("BFPP setBeanName: " + name); // ← called ✓
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        // This is called if BFPP is managed as a regular bean
        // But the order is implementation-dependent and potentially unreliable
        this.ctx = ctx;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        System.out.println("BFPP beanName: " + beanName); // beanName set ✓
        // ctx may or may not be set — unreliable for BFPPs
        // Always use the provided beanFactory parameter instead
    }
}
```

---

### Example 10 — Custom Aware Interface (Framework Extension)

```java
// Define a custom Aware interface for your framework
public interface TenantContextAware extends Aware {
    void setTenantContext(TenantContext tenantContext);
}

// Register a BPP to process it
@Component
public class TenantContextAwareProcessor implements BeanPostProcessor {

    @Autowired
    private TenantContext tenantContext;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof TenantContextAware tca) {
            tca.setTenantContext(tenantContext);
        }
        return bean;
    }
}

// Any bean can implement TenantContextAware to receive the context
@Service
public class TenantAwareReportService implements TenantContextAware {

    private TenantContext tenantContext;

    @Override
    public void setTenantContext(TenantContext tenantContext) {
        this.tenantContext = tenantContext;
    }

    public List<Report> getReports() {
        return reportRepository.findByTenant(tenantContext.getCurrentTenantId());
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which Aware interface callback is invoked BEFORE `postProcessBeforeInitialization()` runs?

A) `ApplicationContextAware.setApplicationContext()`
B) `EnvironmentAware.setEnvironment()`
C) `BeanFactoryAware.setBeanFactory()`
D) `MessageSourceAware.setMessageSource()`

**Answer: C**
`BeanFactoryAware.setBeanFactory()` is invoked by `invokeAwareMethods()` — a direct call BEFORE any BPP processing. Options A, B, and D are all handled by `ApplicationContextAwareProcessor` which is a BPP and runs in `postProcessBeforeInitialization()`.

---

**Q2 — MCQ:**
A bean implements both `BeanFactoryAware` and `ApplicationContextAware`. In `setBeanFactory()`, the bean tries to call `getEnvironment()` on the provided `BeanFactory` reference. What happens?

A) Works — `BeanFactory` provides `getEnvironment()`
B) Compile error — `BeanFactory` interface doesn't have `getEnvironment()`
C) `ClassCastException` at runtime
D) Returns null — environment not yet configured

**Answer: B**
`BeanFactory` interface does NOT declare `getEnvironment()`. That method is on `ConfigurableApplicationContext` (and `ConfigurableEnvironment`). The code won't compile if the reference is typed as `BeanFactory`. Cast to `ConfigurableApplicationContext` (if it is one) to access `getEnvironment()`.

---

**Q3 — Select all that apply:**
Which of the following are handled by `ApplicationContextAwareProcessor`? (Select ALL that apply)

A) `BeanNameAware`
B) `EnvironmentAware`
C) `BeanFactoryAware`
D) `ResourceLoaderAware`
E) `ApplicationEventPublisherAware`
F) `BeanClassLoaderAware`

**Answer: B, D, E**
`BeanNameAware` (A), `BeanFactoryAware` (C), and `BeanClassLoaderAware` (F) are handled by `invokeAwareMethods()` directly. `EnvironmentAware`, `ResourceLoaderAware`, and `ApplicationEventPublisherAware` are handled by `ApplicationContextAwareProcessor`.

---

**Q4 — Code Output Prediction:**

```java
@Component
public class OrderCheck implements BeanNameAware, ApplicationContextAware {

    private String beanName;
    private ApplicationContext ctx;

    @Autowired
    private DataSource dataSource; // injected in populateBean

    @Override
    public void setBeanName(String name) {
        System.out.println("setBeanName: ds=" + (dataSource != null));
        this.beanName = name;
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("setApplicationContext: name=" + beanName);
        this.ctx = ctx;
    }

    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct: ctx=" + (ctx != null));
    }
}
```

What is the output?

A)
```
setBeanName: ds=false
setApplicationContext: name=null
@PostConstruct: ctx=true
```

B)
```
setBeanName: ds=true
setApplicationContext: name=orderCheck
@PostConstruct: ctx=true
```

C)
```
setBeanName: ds=true
setApplicationContext: name=null
@PostConstruct: ctx=true
```

D)
```
setBeanName: ds=false
setApplicationContext: name=orderCheck
@PostConstruct: ctx=true
```

**Answer: B**
`populateBean()` (DI) runs BEFORE `invokeAwareMethods()`. So `dataSource` IS injected when `setBeanName()` is called → `ds=true`. `setBeanName()` sets `this.beanName = name` BEFORE `setApplicationContext()` is called → `name=orderCheck`. `ctx` is set before `@PostConstruct` → `ctx=true`.

---

**Q5 — Trick Question:**
What does `ResourceLoaderAware.setResourceLoader()` receive in a standard Spring `ApplicationContext`?

A) A dedicated `DefaultResourceLoader` instance
B) A `ClassPathXmlApplicationContext`-specific resource loader
C) The `ApplicationContext` itself (which implements `ResourceLoader`)
D) A `PathMatchingResourcePatternResolver` instance

**Answer: C**
`ApplicationContextAwareProcessor` passes `this.applicationContext` as the `ResourceLoader`. Since `ApplicationContext` extends `ResourceLoader`, the same `ApplicationContext` object is provided. No separate `ResourceLoader` instance is created.

---

**Q6 — Select all that apply:**
Which Aware interfaces are still necessary in modern Spring (no clean `@Autowired` equivalent)? (Select ALL that apply)

A) `ApplicationContextAware` — `@Autowired ApplicationContext` is equivalent
B) `BeanNameAware` — no annotation equivalent for getting the bean's own registered name
C) `BeanFactoryAware` — `@Autowired BeanFactory` provides the same reference
D) `ImportAware` — only mechanism for `@Import`-ed `@Configuration` to read importer's metadata
E) `ResourceLoaderAware` — `@Autowired ResourceLoader` is equivalent
F) `EnvironmentAware` — `@Autowired Environment` is equivalent

**Answer: B, D**
A is not necessary (`@Autowired ApplicationContext` works). C — `@Autowired BeanFactory` provides the factory but through `AutowiredAnnotationBeanPostProcessor` which runs AFTER `invokeAwareMethods()`. For framework code needing early factory access, `BeanFactoryAware` is still relevant. E and F have annotation equivalents. `BeanNameAware` (B) and `ImportAware` (D) have no clean annotation alternatives.

---

**Q7 — Scenario-based:**
A `BeanFactoryPostProcessor` implements `ApplicationContextAware`. Will `setApplicationContext()` be called reliably?

A) Yes — Spring always calls `ApplicationContextAware` before `postProcessBeanFactory()`
B) No — BFPPs are instantiated before `ApplicationContextAwareProcessor` is registered
C) Yes — but only if `@DependsOn("applicationContextAwareProcessor")` is added
D) Yes — `AbstractApplicationContext` explicitly calls `setApplicationContext()` on all BFPPs

**Answer: B**
BFPPs are instantiated and executed in `invokeBeanFactoryPostProcessors()` — Step 5 of `refresh()`. `ApplicationContextAwareProcessor` is registered in `prepareBeanFactory()` — Step 3. But the BPP is REGISTERED in Step 3 and APPLIED in Step 6 (registerBeanPostProcessors + finishBeanFactoryInitialization). For BFPPs instantiated in Step 5, `ApplicationContextAwareProcessor` has been registered but may not reliably process them because BFPP instantiation happens outside the normal `preInstantiateSingletons()` flow. The behavior is implementation-dependent and not guaranteed.

---

**Q8 — MCQ:**
A `@Bean` method returns an object of type `Closeable`. The `@Bean` has no explicit `destroyMethod`. What happens at context close?

A) Nothing — no destroy-method configured
B) `close()` is called — inferred from `Closeable.close()`
C) `destroy()` is called — Spring always calls `destroy()` on `DisposableBean`
D) `finalize()` is called — JVM handles it

**Answer: B**
`Closeable` declares `close()`. With `destroyMethod = "(inferred)"` (the default for `@Bean`), Spring detects `close()` and calls it at context shutdown.

---

**Q9 — Code Output Prediction:**

```java
@Configuration
@EnableSomething(feature = "caching")
public class AppConfig { }

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SomethingConfig.class)
public @interface EnableSomething {
    String feature();
}

@Configuration
public class SomethingConfig implements ImportAware {

    @Override
    public void setImportMetadata(AnnotationMetadata metadata) {
        Map<String, Object> attrs =
            metadata.getAnnotationAttributes(EnableSomething.class.getName());
        System.out.println("Feature: " + attrs.get("feature"));
    }
}
```

What is printed?

A) `Feature: caching`
B) `Feature: null`
C) `NullPointerException` — `attrs` is null
D) Nothing — `ImportAware` is not called for `@Import`-ed configs

**Answer: A**
`ImportAware.setImportMetadata()` receives the `AnnotationMetadata` of `AppConfig` (the class that used `@EnableSomething`). The `@EnableSomething` attributes are read from that metadata. `feature = "caching"` is printed.

---

**Q10 — Drag-and-Drop: Order these Aware/init callbacks from first to last:**

- `ApplicationContextAware.setApplicationContext()`
- `@PostConstruct`
- `BeanClassLoaderAware.setBeanClassLoader()`
- `InitializingBean.afterPropertiesSet()`
- `EnvironmentAware.setEnvironment()`
- `BeanNameAware.setBeanName()`
- `BeanFactoryAware.setBeanFactory()`
- `BPP.postProcessAfterInitialization()` (AOP proxy)

**Answer:**
1. `BeanNameAware.setBeanName()`
2. `BeanClassLoaderAware.setBeanClassLoader()`
3. `BeanFactoryAware.setBeanFactory()`
4. `EnvironmentAware.setEnvironment()`
5. `ApplicationContextAware.setApplicationContext()`
6. `@PostConstruct`
7. `InitializingBean.afterPropertiesSet()`
8. `BPP.postProcessAfterInitialization()`

---

## ④ TRICK ANALYSIS

**Trap 1: All Aware callbacks via ApplicationContextAwareProcessor**
The most common trap. Only 6 callbacks go through `ApplicationContextAwareProcessor`: Environment, EmbeddedValueResolver, ResourceLoader, ApplicationEventPublisher, MessageSource, ApplicationContext. The core three (`BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`) are invoked DIRECTLY by `invokeAwareMethods()` — before any BPP runs. Any question stating "all Aware callbacks are handled by BPPs" is wrong.

**Trap 2: BeanFactoryAware provides ApplicationContext**
`BeanFactoryAware.setBeanFactory()` provides `BeanFactory` — NOT `ApplicationContext`. You cannot call `getEnvironment()`, `publishEvent()`, or `getMessage()` on the raw `BeanFactory` reference. For those, implement `ApplicationContextAware`.

**Trap 3: ResourceLoaderAware provides a separate object**
`setResourceLoader()` receives the `ApplicationContext` itself (since `ApplicationContext extends ResourceLoader`). There is no separate `ResourceLoader` object. The exam may present options suggesting a standalone resource loader — those are wrong.

**Trap 4: BeanNameAware provides primary bean name**
`setBeanName()` provides the PRIMARY registered name, not aliases. If `@Bean(name = {"a", "b", "c"})`, the name passed to `setBeanName()` is `"a"` — the first alias. Not `"b"`, not `"c"`.

**Trap 5: Aware callbacks before or after DI?**
Aware callbacks (`invokeAwareMethods()` and `ApplicationContextAwareProcessor`) both run AFTER `populateBean()` (DI). So ALL injected dependencies are available when any Aware callback is invoked. Questions suggesting Aware callbacks are called on uninitialized beans are wrong.

**Trap 6: ImportAware scope**
`ImportAware.setImportMetadata()` receives the metadata of the CLASS THAT IMPORTED via `@Import`, NOT the metadata of the `ImportAware` class itself. This is commonly confused — the metadata describes the importer, not the importee.

**Trap 7: Custom Aware interfaces vs @Autowired**
Spring does NOT automatically process custom Aware interfaces. You must register a `BeanPostProcessor` to handle custom Aware interfaces. The `Aware` interface itself is just a marker — no magic processing happens automatically for custom sub-interfaces.

**Trap 8: BeanFactoryAware in web context**
In a web application, `BeanFactoryAware.setBeanFactory()` provides the internal `DefaultListableBeanFactory`, NOT the `WebApplicationContext`. If you need web-specific features (session scopes, `getServletContext()`), implement `WebApplicationContextUtils` or `ServletContextAware` instead.

**Trap 9: EmbeddedValueResolverAware vs @Value**
`@Value("${key}")` is resolved at injection time by `AutowiredAnnotationBeanPostProcessor`. `EmbeddedValueResolverAware` provides access to the same resolver but allows RUNTIME key determination. They use the same underlying mechanism but serve different use cases. `EmbeddedValueResolverAware` is NOT a replacement for `@Value`.

**Trap 10: Aware order within Group 2**
Within `ApplicationContextAwareProcessor`, the order is FIXED: Environment → EmbeddedValueResolver → ResourceLoader → ApplicationEventPublisher → MessageSource → ApplicationContext. Exam questions may ask about relative ordering within this group. The answer follows the order in `ApplicationContextAwareProcessor.invokeAwareInterfaces()`.

---

## ⑤ SUMMARY SHEET

### Aware Interface Complete Reference

```
GROUP 1 — Invoked by invokeAwareMethods() DIRECTLY (before BPP step):
┌─────────────────────────────┬──────────────────────────────────────┐
│ BeanNameAware               │ Bean's registered primary name       │
│ BeanClassLoaderAware        │ ClassLoader for bean's class         │
│ BeanFactoryAware            │ Internal BeanFactory (NOT ctx)       │
└─────────────────────────────┴──────────────────────────────────────┘

GROUP 2 — Invoked by ApplicationContextAwareProcessor BPP (in BPP-before step):
┌─────────────────────────────┬──────────────────────────────────────┐
│ EnvironmentAware            │ Environment (profiles, properties)   │
│ EmbeddedValueResolverAware  │ ${...} and #{...} resolver           │
│ ResourceLoaderAware         │ ApplicationContext (implements RL)   │
│ ApplicationEventPublisherAware│ ApplicationContext (implements AEP)│
│ MessageSourceAware          │ ApplicationContext (implements MS)   │
│ ApplicationContextAware     │ Full ApplicationContext              │
└─────────────────────────────┴──────────────────────────────────────┘

SPECIAL PURPOSE:
┌─────────────────────────────┬──────────────────────────────────────┐
│ ImportAware                 │ Importer class's AnnotationMetadata  │
│ LoadTimeWeaverAware         │ LTW infrastructure                   │
│ ServletContextAware         │ Web: javax.servlet.ServletContext    │
└─────────────────────────────┴──────────────────────────────────────┘
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| Core 3 Aware callbacks | `invokeAwareMethods()` — DIRECT, before BPP processing |
| ApplicationContext 6 callbacks | `ApplicationContextAwareProcessor` BPP — in `postProcessBeforeInitialization()` |
| DI before Aware | `populateBean()` (DI) runs BEFORE all Aware callbacks |
| `BeanFactoryAware` provides | Internal `DefaultListableBeanFactory` — NOT `ApplicationContext` |
| `ResourceLoaderAware` provides | The `ApplicationContext` itself (implements `ResourceLoader`) |
| `BeanNameAware` provides | Primary bean name — NOT aliases |
| `ImportAware.setImportMetadata()` | Receives metadata of the IMPORTER class |
| Custom Aware interface | Must register a BPP to process it — not automatic |
| `BeanNameAware` modern alternative | NONE — still required for self-identification |
| `ImportAware` modern alternative | NONE — still required for `@Enable*` pattern |

---

### Aware Interface vs @Autowired Comparison

```
@Autowired ApplicationContext  → identical result to ApplicationContextAware
@Autowired BeanFactory         → similar but via BPP (later than BeanFactoryAware)
@Autowired Environment         → identical result to EnvironmentAware
@Autowired ResourceLoader      → identical result to ResourceLoaderAware

NO @Autowired equivalent for:
    BeanNameAware              → only way to get bean's own registered name
    ImportAware                → only way to read @Enable* annotation attributes
    BeanClassLoaderAware       → @Autowired ClassLoader works too
```

---

### Execution Order Diagram

```
Step 4: populateBean() ─────────────────────── DI complete
Step 5: invokeAwareMethods():
    ├── BeanNameAware
    ├── BeanClassLoaderAware
    └── BeanFactoryAware
Step 6: postProcessBeforeInitialization():
    ├── ApplicationContextAwareProcessor:
    │       EnvironmentAware
    │       EmbeddedValueResolverAware
    │       ResourceLoaderAware
    │       ApplicationEventPublisherAware
    │       MessageSourceAware
    │       ApplicationContextAware
    └── CommonAnnotationBPP:
            @PostConstruct
Step 7: invokeInitMethods():
    ├── InitializingBean.afterPropertiesSet()
    └── Custom init-method
Step 8: postProcessAfterInitialization():
    └── AOP proxy creation
```

---

### Interview One-Liners

- "`BeanNameAware`, `BeanClassLoaderAware`, and `BeanFactoryAware` are invoked directly by `invokeAwareMethods()` before any BPP runs — they are not processed by `ApplicationContextAwareProcessor`."
- "`ApplicationContextAware.setApplicationContext()` is handled by a BPP (`ApplicationContextAwareProcessor`), not by a direct call — unlike `BeanFactoryAware`."
- "`BeanFactoryAware.setBeanFactory()` provides the internal `DefaultListableBeanFactory`, NOT the `ApplicationContext`. Use `ApplicationContextAware` to access the full context."
- "All Aware callbacks (both groups) run AFTER dependency injection — `@Autowired` fields are always set when any Aware callback fires."
- "`ResourceLoaderAware.setResourceLoader()` receives the `ApplicationContext` itself — `ApplicationContext` implements `ResourceLoader`, so no separate object is created."
- "`ImportAware.setImportMetadata()` receives the `AnnotationMetadata` of the CLASS THAT IMPORTED the configuration — allowing `@Enable*` annotations to pass attributes to their supporting `@Configuration` classes."
- "`BeanNameAware` has no `@Autowired` equivalent — it is the only way for a bean to discover its own registered primary name."
- "Custom `Aware` interfaces require a custom `BeanPostProcessor` to process them — Spring does not automatically handle custom sub-interfaces of `Aware`."

---
