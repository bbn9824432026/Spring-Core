# Spring BeanFactoryPostProcessor — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

You're building a framework layer for your company's internal platform. The ticket:

```
TICKET-334: Make our platform configurable without touching application code
- Every team using our platform has different database pools (dev: 5, prod: 50)
- Some teams run in "strict mode" — all their @Service beans must be prototype
- Some teams need custom beans injected that THEY don't define (we provide them)
- Our platform needs to scan THEIR packages and register beans they didn't write
- ALL of this must happen before the app starts — not at runtime
- We cannot ask teams to change their @Configuration classes
```

You look at what tools you already know and try each one:

```java
// Attempt 1: Use @Bean to configure pool sizes
@Bean
public DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setMaximumPoolSize(50);    // hardcoded — teams can't override without changing code
    return ds;
}
// Problem: teams have to modify YOUR code to change the pool size.

// Attempt 2: Use @Value to inject pool size
@Bean
public DataSource dataSource(@Value("${pool.size:10}") int poolSize) {
    HikariDataSource ds = new HikariDataSource();
    ds.setMaximumPoolSize(poolSize);
    return ds;
}
// Problem: this only works for YOUR beans. You can't inject @Value into
// beans OTHER teams defined. And you can't change the SCOPE of their beans.

// Attempt 3: Use BeanPostProcessor to modify beans after creation
@Component
public class PoolSizeBPP implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof HikariDataSource ds) {
            ds.setMaximumPoolSize(50);    // WRONG: pool is ALREADY RUNNING by this point
        }                                  // Changing pool size on a live DataSource = undefined behavior
        return bean;
    }
}
// Problem: BPP runs AFTER beans are instantiated. The DataSource already
// started a connection pool with the old size. Changing it mid-life is dangerous.
// Also: you can't change scope on an already-created bean. Scope is metadata.
```

You hit a wall. Everything you know runs TOO LATE:

```
@Bean method              → creates ONE specific bean you wrote
@Value injection          → works only for beans you wrote
BeanPostProcessor         → runs AFTER beans are created — too late to change metadata
@Profile                  → swaps implementations — but you'd need to know all their bean names

What you NEED:
→ A hook that runs BEFORE any beans are created
→ That can see ALL bean definitions that have been registered
→ And modify/add/remove them before instantiation begins
```

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Run initialization code in a static block**

```java
public class PlatformConfig {
    static {
        // modify some global state before Spring starts?
        System.setProperty("pool.size", "50");
    }
}
```

**Why this fails:** Static blocks run when the class is loaded — you have no access to Spring's container at all. You can't see what beans are registered, you can't change their scopes, you can't add new bean definitions. You're operating completely outside Spring.

---

**Naive Approach 2: Use `ApplicationListener<ContextRefreshedEvent>` to post-configure**

```java
@Component
public class PostStartupConfig implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        ApplicationContext ctx = event.getApplicationContext();
        HikariDataSource ds = ctx.getBean(HikariDataSource.class);
        ds.setMaximumPoolSize(50);  // change after everything started
    }
}
```

**Why this fails:** By the time this event fires, ALL beans are already created, ALL connection pools are running, ALL scopes are locked in. You can change some mutable properties on live beans, but you cannot change a bean's scope (that's metadata — it only affects the NEXT time the bean is requested). You can't register new beans. You can't replace existing beans. You're rearranging deck chairs after the ship has sailed.

---

**Naive Approach 3: Extend `AnnotationConfigApplicationContext` and override `refresh()`**

```java
public class PlatformApplicationContext extends AnnotationConfigApplicationContext {
    @Override
    public void refresh() {
        // do some setup before calling super.refresh()?
        super.refresh();
    }
}
```

**Why this fails:** You're now asking every team to use YOUR custom `ApplicationContext` subclass instead of the standard one. Teams using Spring Boot can't use this at all — Boot manages the context. You've coupled your platform to the application's bootstrap code. And you still haven't accessed the bean definition registry inside `refresh()` in any structured, safe way.

---

**The correct solution emerges:**

Spring already designed exactly this extension point. The lifecycle of `refresh()` has a dedicated phase — after ALL bean definitions are loaded and registered, but BEFORE ANY bean is instantiated — where you can plug in code. That code receives the full bean factory and can do anything: modify existing bean definitions, add new ones, remove ones, read the whole catalog. Spring calls components that run in this phase `BeanFactoryPostProcessors`.

```
ALL your @Component, @Bean, @Configuration classes are parsed → BeanDefinitions registered
    ↓
[ YOUR CODE RUNS HERE ] ← BeanFactoryPostProcessor phase
  → sees all BeanDefinitions
  → can modify scope, lazy-init, property values, class
  → can ADD new BeanDefinitions (via BeanDefinitionRegistryPostProcessor)
  → can REMOVE BeanDefinitions
    ↓
Beans are actually created (instantiated) using the now-modified BeanDefinitions
```

---

## 🧠 3. Concept Introduction — Plain English First

### The Recipe Book Analogy

Imagine a restaurant kitchen. Before dinner service starts, there's a **prep phase** where:

1. All recipes are written into the recipe book (BeanDefinitions registered)
2. A **kitchen supervisor** reviews ALL recipes and can:
   - Change "serves 2" to "serves 10" (change scope from singleton to prototype)
   - Add new recipes to the book (register new BeanDefinitions)
   - Remove a recipe (remove a BeanDefinition)
   - Substitute an ingredient across all recipes (resolve `${placeholder}` values)
3. Actual cooking begins (beans instantiated from the modified recipes)

The kitchen supervisor IS the `BeanFactoryPostProcessor`. They operate on the **recipe book** (metadata), not on finished dishes (beans). By the time the cooking starts, the recipe book is final.

```
Phase 1 — WRITE RECIPES:
  @Component, @Configuration, @Bean → BeanDefinitions registered in registry
  (These are descriptions of HOW to make beans — not actual beans yet)

Phase 2 — SUPERVISOR REVIEWS (BeanFactoryPostProcessor runs):
  Sees: { name="dataSource", class=HikariDataSource, poolSize=10, scope=singleton }
  Modifies: scope → prototype, poolSize → 50
  Adds new recipe: { name="auditLogger", class=AuditLogger }
  
Phase 3 — COOK:
  BeanFactory reads final recipes → creates actual bean instances
  dataSource created as prototype with poolSize=50
  auditLogger created as singleton
```

### Two Levels of This Pattern

```
Level 1 — BeanFactoryPostProcessor (BFPP):
  Can READ and MODIFY existing BeanDefinitions
  Cannot ADD new BeanDefinitions (no access to the registry)
  "The supervisor can edit recipes but can't add pages to the book"

Level 2 — BeanDefinitionRegistryPostProcessor (BDRPP):
  Extends Level 1
  Can ALSO add new BeanDefinitions
  Runs BEFORE Level 1
  "The supervisor can edit AND add recipes, AND they review the book FIRST"
```

### Where in `refresh()` This All Happens

```
AbstractApplicationContext.refresh():

Step 3:  prepareBeanFactory()        ← internal Spring setup
Step 5:  invokeBeanFactoryPostProcessors()  ← ALL BFPPs run HERE
           │
           ├── Phase A: BDRPPs (can add beans)
           │     First: ConfigurationClassPostProcessor runs (scans @Component, processes @Bean)
           │     Then:  Your custom BDRPPs
           │
           ├── Phase B: postProcessBeanFactory() on all BDRPPs
           │     ConfigurationClassPostProcessor CGLIB-enhances @Configuration classes
           │
           └── Phase C: Plain BFPPs (modify existing beans)
                 PropertySourcesPlaceholderConfigurer resolves ${...}
                 Your custom BFPPs

Step 6:  registerBeanPostProcessors()   ← BPPs registered (NOT yet run)
Step 11: finishBeanFactoryInitialization() ← beans CREATED here (using modified definitions)
Step 11: BPPs run for each bean as it's created
```

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TODO 1: Write the simplest BFPP — inspect all BeanDefinitions (understand the API)
TODO 2: Write a BFPP that modifies bean scope/properties based on environment
TODO 3: Write a BDRPP that registers NEW BeanDefinitions programmatically
TODO 4: Understand ConfigurationClassPostProcessor — the BFPP you already use without knowing
TODO 5: Understand PropertySourcesPlaceholderConfigurer — why @Value works, why it must be static
TODO 6: Understand the ordering rules — PriorityOrdered vs Ordered vs plain
TODO 7: Wire a custom ImportBeanDefinitionRegistrar triggered by an annotation
TODO 8: Test infrastructure — replacing beans before the container finalizes
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: The simplest BFPP — learn the API

```java
// BeanFactoryPostProcessor: the interface with one method.
// postProcessBeanFactory() is called with the fully populated bean factory —
// all BeanDefinitions are registered, but NOT A SINGLE ordinary bean has been created.
// You get to see and modify the entire plan before execution.
@Component
public class BeanDefinitionInspector implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {

        // ConfigurableListableBeanFactory: the bean factory interface that gives you:
        // - getBeanDefinitionNames(): all registered bean names
        // - getBeanDefinition(name): the metadata for each bean
        // - getBeanNamesForType(): find beans by type
        // You CANNOT call getBean() safely here (see silent failure section below)

        System.out.println("=== BFPP: Bean Registry Snapshot (pre-instantiation) ===");
        System.out.println("Total beans registered: " + beanFactory.getBeanDefinitionCount());

        for (String beanName : beanFactory.getBeanDefinitionNames()) {

            // BeanDefinition: the metadata record that describes HOW to create a bean.
            // Think of it as a filled-in form:
            //   "Create a bean named X, of class Y, with scope Z, lazy=false, primary=false"
            // This is NOT the bean itself — it's the recipe.
            BeanDefinition bd = beanFactory.getBeanDefinition(beanName);

            System.out.printf("  Bean: %-40s | scope: %-10s | lazy: %-5s | class: %s%n",
                beanName,
                bd.getScope().isEmpty() ? "singleton" : bd.getScope(),
                bd.isLazyInit(),
                bd.getBeanClassName()
            );
        }
    }
}
```

**What you'll see when this runs:**

```
=== BFPP: Bean Registry Snapshot (pre-instantiation) ===
Total beans registered: 47
  Bean: orderService                        | scope: singleton   | lazy: false  | class: com.example.OrderService
  Bean: dataSource                          | scope: singleton   | lazy: false  | class: com.zaxxer.hikari.HikariDataSource
  Bean: transactionManager                  | scope: singleton   | lazy: false  | class: org.springframework.orm.jpa.JpaTransactionManager
  ...
```

Every bean your `@ComponentScan` found — plus Spring's internal beans — visible BEFORE any of them are created.

---

### TODO 2: BFPP that modifies bean definitions

```java
// This BFPP reads an environment variable and changes the scope of
// ALL @Service beans to prototype in "strict" mode.
// Teams using the platform just set STRICT_MODE=true in their environment.
// They don't change their @Service classes at all.
@Component
public class StrictModeBFPP implements BeanFactoryPostProcessor, Ordered {

    // Ordered: this interface lets you control when THIS bfpp runs relative to others.
    // getOrder() returns a number — LOWER number = runs EARLIER.
    // Ordered.LOWEST_PRECEDENCE = Integer.MAX_VALUE = run last among Ordered BFPPs.
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE; // run after platform's own BFPPs
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        boolean strictMode = Boolean.parseBoolean(
            System.getProperty("platform.strict-mode", "false"));

        if (!strictMode) return;

        System.out.println("STRICT MODE: Converting @Service beans to prototype scope");

        for (String beanName : beanFactory.getBeanDefinitionNames()) {
            BeanDefinition bd = beanFactory.getBeanDefinition(beanName);

            // getBeanClassName() returns null for inner beans and factory-produced beans.
            // Always guard against null before loading the class.
            String className = bd.getBeanClassName();
            if (className == null) continue;

            try {
                Class<?> beanClass = Class.forName(className);

                if (beanClass.isAnnotationPresent(Service.class)) {
                    // setScope(): change the scope on the BeanDefinition.
                    // This affects the NEXT time this bean is requested after context starts.
                    // In prototype scope: every getBean() creates a fresh instance.
                    // This ONLY works here — you cannot change scope on an already-created bean.
                    bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
                    System.out.println("  → " + beanName + " changed to prototype");
                }

            } catch (ClassNotFoundException e) {
                // Inner beans, synthetic beans — class might not be loadable directly
                // Safe to skip
            }
        }
    }
}
```

**More BeanDefinition modifications — the full toolkit:**

```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    BeanDefinition bd = beanFactory.getBeanDefinition("dataSource");

    // All of these modify the recipe BEFORE the bean is baked:

    // Change scope:
    bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);

    // Make it lazy — don't create until first requested:
    bd.setLazyInit(true);

    // Inject a property value that wasn't in the original bean definition:
    // (Works for setter injection, not constructor injection)
    bd.getPropertyValues().add("maximumPoolSize", 50);

    // Change the actual implementing class:
    bd.setBeanClassName("com.example.EnhancedDataSource");

    // Mark as primary (wins in autowiring conflicts):
    ((AbstractBeanDefinition) bd).setPrimary(true);

    // THE DANGEROUS THING — DON'T DO THIS IN A BFPP:
    // DataSource ds = beanFactory.getBean(DataSource.class);
    // → Forces premature instantiation NOW, before BPPs are registered
    // → The dataSource bean is created WITHOUT going through BPP processing
    // → AOP proxies, @Transactional, @Async — ALL silently skipped for this bean
    // → Spring logs a warning but doesn't stop you
}
```

---

### TODO 3: BeanDefinitionRegistryPostProcessor — adding new beans

A plain BFPP can only modify existing definitions. To ADD new ones, you need `BeanDefinitionRegistryPostProcessor` (BDRPP).

```java
// BeanDefinitionRegistryPostProcessor: extends BeanFactoryPostProcessor with one more method.
// postProcessBeanDefinitionRegistry() is called first — with BeanDefinitionRegistry access.
// postProcessBeanFactory() is called second — same as plain BFPP.
//
// BeanDefinitionRegistry: the object that OWNS the registry.
// It has: registerBeanDefinition(), removeBeanDefinition(), containsBeanDefinition()
// Plain BFPP does NOT have access to BeanDefinitionRegistry — it can't add beans.
@Component
public class PlatformBeansRegistrar implements BeanDefinitionRegistryPostProcessor, Ordered {

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }

    // PHASE 1: called first — can add/remove/modify BeanDefinitions
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {

        // Scan for team-defined classes annotated with our custom @PlatformService
        // Teams just put @PlatformService on their interfaces — we register the beans.
        ClassPathScanningCandidateComponentProvider scanner =
            new ClassPathScanningCandidateComponentProvider(false);
        // addIncludeFilter: "include classes that have this annotation"
        // AnnotationTypeFilter: matches by annotation presence
        scanner.addIncludeFilter(new AnnotationTypeFilter(PlatformService.class));

        // findCandidateComponents: scans the package, returns BeanDefinition-like objects
        // for each class that matches the filters
        Set<BeanDefinition> candidates = scanner.findCandidateComponents("com.example");

        for (BeanDefinition candidate : candidates) {
            String className = candidate.getBeanClassName();

            // RootBeanDefinition: a concrete BeanDefinition implementation.
            // You build the "recipe card" from scratch:
            GenericBeanDefinition bd = new GenericBeanDefinition();
            bd.setBeanClassName(className);
            bd.setScope(BeanDefinition.SCOPE_SINGLETON);
            bd.setLazyInit(false);

            // Add constructor arguments or property values if needed:
            // bd.getConstructorArgumentValues().addGenericArgumentValue("arg1");
            // bd.getPropertyValues().add("someProperty", "someValue");

            // Derive bean name from class name (e.g., MyService → myService):
            String simpleName = className.substring(className.lastIndexOf('.') + 1);
            String beanName = Character.toLowerCase(simpleName.charAt(0))
                            + simpleName.substring(1);

            // Guard: don't re-register if already present
            if (!registry.containsBeanDefinition(beanName)) {
                registry.registerBeanDefinition(beanName, bd);
                System.out.println("Platform registered: " + beanName);
            }
        }

        // You can also register beans completely from scratch:
        GenericBeanDefinition auditBd = new GenericBeanDefinition();
        auditBd.setBeanClass(AuditLogger.class);
        auditBd.getPropertyValues().add("applicationName", "MyPlatformApp");
        registry.registerBeanDefinition("platformAuditLogger", auditBd);
    }

    // PHASE 2: called second — same capability as plain BFPP
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        System.out.println("BDRPP phase 2: all definitions now registered, count = "
            + beanFactory.getBeanDefinitionCount());
        // Can still modify existing definitions here
    }
}
```

**The BDRPP discovery loop — the part that surprises everyone:**

```java
// BDRPP-A registers a new BeanDefinition for BDRPP-B in its postProcessBeanDefinitionRegistry():
registry.registerBeanDefinition("bdrppB", new RootBeanDefinition(BdRppB.class));

// Spring detects: "wait, a new BDRPP was just registered that I haven't run yet"
// → Spring calls BdRppB.postProcessBeanDefinitionRegistry() automatically
// → This repeats until no new BDRPPs are found

// Output:
// BDRPP-A: postProcessBeanDefinitionRegistry() → registers BDRPP-B
// BDRPP-B: postProcessBeanDefinitionRegistry() → runs automatically (discovered by loop)
// BDRPP-A: postProcessBeanFactory()
// BDRPP-B: postProcessBeanFactory()
```

---

### TODO 4: ConfigurationClassPostProcessor — the BFPP you already use

This is the most important BFPP in all of Spring. You use it every time you use `@Component`, `@Bean`, `@Import`, or `@Configuration` — you just don't see it.

```
ConfigurationClassPostProcessor is a BeanDefinitionRegistryPostProcessor
that implements PriorityOrdered (runs FIRST among all BDRPPs).

What it does in postProcessBeanDefinitionRegistry():

1. Finds all @Configuration, @Component, @ComponentScan classes in the registry
2. Parses @ComponentScan → scans packages → registers found @Component beans
3. Parses @Import:
   → @Import(SomeClass.class)              → registers SomeClass as a bean
   → @Import(ImportSelector.class)         → calls selectImports() → registers results
   → @Import(ImportBeanDefinitionRegistrar) → calls registerBeanDefinitions()
4. Parses @Bean methods → registers a BeanDefinition for each @Bean method
5. Parses @PropertySource → adds property files to Environment's PropertySource stack

What it does in postProcessBeanFactory():

6. CGLIB-enhances all @Configuration classes:
   Replaces MyConfig.class with MyConfig$$SpringCGLIB$$0.class in each BeanDefinition
   This is what makes inter-@Bean method calls return the SAME singleton instance.
```

**Why CGLIB enhancement happens HERE (in postProcessBeanFactory, not later):**

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // calls serviceB() directly
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

// Without CGLIB: serviceB() would be called as a regular Java method → new ServiceB()
// every time → two different ServiceB instances → singleton guarantee broken.

// ConfigurationClassPostProcessor.postProcessBeanFactory() replaces AppConfig.class
// with AppConfig$$SpringCGLIB$$0.class, which overrides serviceB() to check:
// "does the container already have this bean? yes → return existing. no → create new."

// This happens at BeanDefinition level — the BeanDefinition's beanClass
// is updated from AppConfig → AppConfig$$SpringCGLIB$$0
// When finishBeanFactoryInitialization() runs, it creates the CGLIB subclass instance.
```

---

### TODO 5: PropertySourcesPlaceholderConfigurer — why @Value works, and the static trap

```java
// PropertySourcesPlaceholderConfigurer (PSPC): a plain BFPP that runs after
// ConfigurationClassPostProcessor.
// What it does:
//   Walks through EVERY BeanDefinition's property values
//   Finds strings containing ${...} placeholders
//   Resolves them against the Environment's PropertySource stack
//   Replaces the placeholder string with the resolved value IN THE BEANDEFINITION
//
// After PSPC runs:
//   BeanDefinition for "dataSource" had: maximumPoolSize = "${db.pool.size}"
//   After PSPC:                          maximumPoolSize = "50"  (the actual value)
//
// When finishBeanFactoryInitialization() creates the DataSource,
// it reads "50" from the BeanDefinition — not "${db.pool.size}".

@Configuration
public class DataConfig {

    // THIS MUST BE STATIC. Here is exactly why:
    //
    // Spring's refresh() step 5 (invokeBeanFactoryPostProcessors) needs to:
    //   a) Instantiate the PSPC bean (by calling this @Bean method)
    //   b) Run PSPC.postProcessBeanFactory() to resolve ${...} placeholders
    //   c) THEN instantiate DataConfig (the @Configuration class)
    //   d) DataConfig needs @Value("${db.url}") resolved — which needs PSPC to have run
    //
    // If pspc() is NON-static:
    //   To call pspc(), Spring must first create the DataConfig bean (to call the method on)
    //   To create DataConfig, Spring must have CGLIB-enhanced it (postProcessBeanFactory)
    //   But postProcessBeanFactory hasn't run yet (we're trying to run PSPC first)
    //   DEADLOCK at infrastructure level
    //
    // What actually happens (no deadlock, but wrong result):
    //   Spring creates DataConfig WITHOUT CGLIB proxy (early, raw instantiation)
    //   Calls non-static pspc() → gets the PSPC bean
    //   Runs PSPC.postProcessBeanFactory()
    //   Now tries to inject @Value("${db.url}") into the already-created DataConfig
    //   But DataConfig was created before PSPC ran → @Value uses the default or literal string
    //
    // Silent failure: @Value("${db.url:localhost}") → always "localhost" even in production
    @Bean
    public static PropertySourcesPlaceholderConfigurer pspc() {
        PropertySourcesPlaceholderConfigurer p = new PropertySourcesPlaceholderConfigurer();
        p.setLocation(new ClassPathResource("app.properties"));
        p.setIgnoreResourceNotFound(true);     // don't fail if file missing
        p.setIgnoreUnresolvablePlaceholders(false); // DO fail if ${key} not found (safer)
        return p;
    }

    // With static pspc() above — this is resolved correctly:
    @Value("${db.url}")
    private String dbUrl;

    @Value("${db.pool.size:10}")
    private int poolSize;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(dbUrl);         // "jdbc:mysql://prod/mydb" — correctly resolved
        ds.setMaximumPoolSize(poolSize); // 50 — correctly resolved
        return ds;
    }
}
```

**Why Spring Boot projects don't need explicit PSPC:**

```
Spring Boot auto-configuration registers PropertySourcesPlaceholderConfigurer
via PropertyPlaceholderAutoConfiguration.

This means: in a @SpringBootApplication project, @Value just works.
In pure Spring (no Boot), you need the static @Bean PSPC, or @Value won't resolve.
```

---

### TODO 6: Ordering — the rule that trips everyone up

```java
// Three implementations of the same pattern — demonstrating ordering:

@Component
public class AlphaBFPP implements BeanFactoryPostProcessor, PriorityOrdered {
    @Override
    public int getOrder() { return 1; }  // lower = earlier

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        System.out.println("AlphaBFPP (PriorityOrdered, order=1)");
    }
}

@Component
public class BetaBFPP implements BeanFactoryPostProcessor, PriorityOrdered {
    @Override
    public int getOrder() { return -1; }  // runs before AlphaBFPP within PriorityOrdered

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        System.out.println("BetaBFPP (PriorityOrdered, order=-1)");
    }
}

@Component
public class GammaBFPP implements BeanFactoryPostProcessor, Ordered {
    @Override
    public int getOrder() { return Integer.MIN_VALUE; }  // LOWEST possible Ordered value

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        System.out.println("GammaBFPP (Ordered, order=MIN_VALUE)");
    }
}

// Output:
// BetaBFPP (PriorityOrdered, order=-1)     ← PriorityOrdered batch, sorted by order
// AlphaBFPP (PriorityOrdered, order=1)     ← PriorityOrdered batch, sorted by order
// GammaBFPP (Ordered, order=MIN_VALUE)     ← Ordered batch — AFTER PriorityOrdered ENTIRE BATCH
//
// GammaBFPP has order=MIN_VALUE (-2147483648) which is numerically LOWER than BetaBFPP's -1.
// But GammaBFPP is Ordered, not PriorityOrdered.
// PriorityOrdered = a COMPLETELY SEPARATE, EARLIER batch.
// No matter how low your Ordered.getOrder() value is, you run AFTER ALL PriorityOrdered BFPPs.
```

**The full ordering picture:**

```
PHASE 1 — BDRPPs:
  Batch A: PriorityOrdered BDRPPs (sorted by getOrder())
    → ConfigurationClassPostProcessor (PriorityOrdered, order=Integer.MIN_VALUE) ← ALWAYS FIRST
  Batch B: Ordered BDRPPs (sorted by getOrder())
  Batch C: Non-ordered BDRPPs (undefined order)
  [Loop repeats until no new BDRPPs discovered]

PHASE 2 — postProcessBeanFactory() on all BDRPPs (same order as phase 1)

PHASE 3 — Plain BFPPs:
  Batch D: PriorityOrdered BFPPs (sorted by getOrder())
    → PropertySourcesPlaceholderConfigurer (PriorityOrdered) ← resolves ${...}
  Batch E: Ordered BFPPs (sorted by getOrder())
  Batch F: Non-ordered BFPPs (undefined order)
```

---

### TODO 7: ImportBeanDefinitionRegistrar — annotation-triggered bean registration

This is NOT a BFPP itself — it's called BY `ConfigurationClassPostProcessor`. But it serves the same goal: register beans programmatically. Use it when you want to trigger registration via a custom annotation.

```java
// Step 1: Define your custom enable annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
// @Import: tells ConfigurationClassPostProcessor to call PlatformRegistrar
// when it sees this annotation on a @Configuration class
@Import(PlatformRegistrar.class)
public @interface EnablePlatform {
    String[] basePackages() default {};
    boolean strictMode() default false;
}

// Step 2: The registrar — called by ConfigurationClassPostProcessor
public class PlatformRegistrar implements ImportBeanDefinitionRegistrar {

    // registerBeanDefinitions() is called with:
    //   importingClassMetadata: annotation data from the class that has @EnablePlatform
    //   registry: the BeanDefinitionRegistry — full add/remove capability
    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {

        // Read annotation attributes from the @EnablePlatform annotation:
        Map<String, Object> attrs = importingClassMetadata
            .getAnnotationAttributes(EnablePlatform.class.getName());

        String[] basePackages = (String[]) attrs.get("basePackages");
        boolean strictMode = (boolean) attrs.get("strictMode");

        // Scan the packages specified in the annotation:
        ClassPathScanningCandidateComponentProvider scanner =
            new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(PlatformService.class));

        for (String pkg : basePackages) {
            for (BeanDefinition candidate : scanner.findCandidateComponents(pkg)) {
                String className = candidate.getBeanClassName();
                String simpleName = className.substring(className.lastIndexOf('.') + 1);
                String beanName = Character.toLowerCase(simpleName.charAt(0))
                                + simpleName.substring(1);

                GenericBeanDefinition bd = new GenericBeanDefinition();
                bd.setBeanClassName(className);
                if (strictMode) {
                    bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
                }

                if (!registry.containsBeanDefinition(beanName)) {
                    registry.registerBeanDefinition(beanName, bd);
                }
            }
        }

        // Always register the platform's core audit bean:
        GenericBeanDefinition auditBd = new GenericBeanDefinition();
        auditBd.setBeanClass(PlatformAuditLogger.class);
        auditBd.getPropertyValues().add("strictMode", strictMode);
        registry.registerBeanDefinition("platformAuditLogger", auditBd);
    }
}

// Step 3: Team's @Configuration — one annotation does everything
@Configuration
@EnablePlatform(
    basePackages = {"com.myteam.services"},
    strictMode = true
)
public class TeamAppConfig {
    // Nothing else needed — @EnablePlatform triggers all registration
}
```

**`ImportBeanDefinitionRegistrar` vs `BeanDefinitionRegistryPostProcessor`:**

```
ImportBeanDefinitionRegistrar:
  ✓ Triggered by @Import on a @Configuration class (annotation-driven)
  ✓ Gets annotation metadata from the importing class (@EnablePlatform attributes)
  ✓ Clean API for "enable this feature" annotations
  ✗ Must be triggered via @Import — can't run standalone
  ✗ Not a Spring bean itself — can't @Autowired dependencies into it

BeanDefinitionRegistryPostProcessor:
  ✓ Runs standalone — registered as a @Component or @Bean
  ✓ Can @Autowired dependencies (with limitations)
  ✓ Can be ordered
  ✓ Can chain (register new BDRPPs from within a BDRPP)
  ✗ No access to annotation metadata from importing classes
```

---

### TODO 8: Test infrastructure — replacing beans with mocks before context finalizes

```java
// A reusable test utility: inject this BFPP before refresh() to replace
// any registered bean with a mock — no @MockBean, no Mockito extensions needed.
public class MockBeanBFPP implements BeanDefinitionRegistryPostProcessor {

    private final Map<String, Object> mocks = new LinkedHashMap<>();

    public MockBeanBFPP mock(String beanName, Object mockObject) {
        mocks.put(beanName, mockObject);
        return this;
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        for (String beanName : mocks.keySet()) {
            if (registry.containsBeanDefinition(beanName)) {
                registry.removeBeanDefinition(beanName);
            }
            // Don't register a BeanDefinition — we'll register the mock singleton directly
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        for (Map.Entry<String, Object> entry : mocks.entrySet()) {
            // registerSingleton: bypasses the full bean lifecycle for this object.
            // Spring treats the mock object as a pre-created singleton.
            // No constructor called, no @PostConstruct, no BPP processing.
            // The mock is used AS-IS.
            beanFactory.registerSingleton(entry.getKey(), entry.getValue());
        }
    }
}

// Usage in tests — MUST add BFPP before refresh():
@Test
void orderServiceUsesEmailServiceCorrectly() {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();

    // Create mocks:
    EmailService mockEmail = mock(EmailService.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);
    when(mockPayment.charge(any(), any())).thenReturn(PaymentResult.success("txn-123"));

    // Add BFPP BEFORE refresh() — this is the key:
    // AFTER refresh(), it's too late — beans are created, context is sealed.
    ctx.addBeanFactoryPostProcessor(
        new MockBeanBFPP()
            .mock("emailService", mockEmail)
            .mock("paymentGateway", mockPayment)
    );

    ctx.register(AppConfig.class);
    ctx.refresh();  // BFPP runs here — replaces real beans with mocks

    // Get the real OrderService — it autowired the mock EmailService:
    OrderService orderService = ctx.getBean(OrderService.class);
    orderService.placeOrder(new Order("ORD-001", new BigDecimal("99.99")));

    verify(mockEmail).send(eq("customer@example.com"), contains("ORD-001"), any());
    verify(mockPayment).charge(eq("customer-id"), eq(new BigDecimal("99.99")));

    ctx.close();
}
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: BFPP vs BDRPP — which one do you need?**

```
Use plain BFPP (BeanFactoryPostProcessor) when:
  ✓ Modifying existing BeanDefinitions (scope, lazy, properties)
  ✓ Reading the registry to make decisions
  ✓ You don't need to add new beans

Use BDRPP (BeanDefinitionRegistryPostProcessor) when:
  ✓ Adding new BeanDefinitions
  ✓ Removing existing BeanDefinitions
  ✓ Chaining — registering other BDRPPs dynamically
  ✓ Implementing @Enable* annotations with ImportBeanDefinitionRegistrar

Use ImportBeanDefinitionRegistrar (not a BFPP) when:
  ✓ The trigger is a custom annotation (@EnablePlatform, @EnableFeature)
  ✓ You need annotation attribute data from the importing class
  ✓ You want annotation-driven, declarative registration
```

**Decision 2: `@Component` BFPP vs `ctx.addBeanFactoryPostProcessor()`**

```
@Component registration:
  ✓ Conventional — part of the normal component scan
  ✓ Can use @Autowired (with limitations — see lifecycle section)
  ✗ Discovered by ConfigurationClassPostProcessor — so CCP must have already run
  ✗ Can't be used in tests without a full Spring context

ctx.addBeanFactoryPostProcessor() (programmatic):
  ✓ Runs BEFORE bean-defined BFPPs (higher priority)
  ✓ Perfect for tests — add before refresh(), no @Component needed
  ✓ Can be added when Spring Boot is starting too (SpringApplication.addInitializers)
  ✗ More boilerplate — you manage the instance yourself
```

**Decision 3: AOP and @Transactional on BFPPs — why it never works**

```
BFPPs are instantiated during Step 5 of refresh().
AOP proxy creation (AbstractAutoProxyCreator) is a BPP.
BPPs are registered in Step 6 — AFTER BFPPs run.

When the BFPP bean is created:
  → Constructor ✓
  → @Autowired DI ✓ (AutowiredAnnotationBPP is PriorityOrdered — registered early)
  → @PostConstruct ✓ (CommonAnnotationBPP is also PriorityOrdered)
  → AOP proxy creation ✗ (AbstractAutoProxyCreator not registered yet)
  → @Transactional ✗ (TransactionInterceptor never wrapped around this bean)
  → @Async ✗ (AsyncAnnotationAdvisor never applied)
  → @Cacheable ✗ (CacheInterceptor never applied)

Result: these annotations compile fine, run fine, silently do nothing on BFPP classes.
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
class BFPPEdgeCaseTest {

    // TRAP 1: calling getBean() inside postProcessBeanFactory() — the silent killer
    @Component
    static class DangerousBFPP implements BeanFactoryPostProcessor {
        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
            // This COMPILES. It RUNS. Spring doesn't throw.
            // But DataSource is now created before BPPs are registered (Step 6 hasn't run yet).
            // AOP proxies, @Transactional, @Async — all silently skipped for dataSource.
            // Spring logs: "Bean 'dataSource' is not eligible for getting processed
            //               by all BeanPostProcessors (for example: not eligible for auto-proxying)"
            DataSource ds = bf.getBean(DataSource.class);  // ← DON'T DO THIS
        }
    }

    @Test
    void getBean_inBFPP_missesAOP() {
        // If your BFPP calls getBean(DataSource.class):
        // And DataSource has @Transactional methods:
        // Those methods will NOT have transaction management.
        // No exception. SQL runs without a transaction. Data corruption possible.
        // Only Spring's log warning tells you — easily missed.
    }

    // TRAP 2: @Transactional on BFPP — silently ignored
    @Component
    static class TransactionalBFPP implements BeanFactoryPostProcessor {
        @Transactional  // ← looks reasonable, does nothing
        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
            // This method runs WITHOUT any transaction.
            // If it does DB work, it runs in auto-commit mode.
            // No exception, no warning for @Transactional specifically.
        }
    }

    // TRAP 3: Non-static PSPC — @Value gets the default, not the property file value
    @Configuration
    static class BadConfig {
        @Bean
        // Missing 'static' keyword:
        public PropertySourcesPlaceholderConfigurer pspc() {
            PropertySourcesPlaceholderConfigurer p = new PropertySourcesPlaceholderConfigurer();
            p.setLocation(new ClassPathResource("test.properties")); // has timeout=9999
            return p;
        }

        @Value("${timeout:30}")
        private int timeout; // → 30 (default), NOT 9999 from the file

        @Bean
        public TimerService timerService() { return new TimerService(timeout); }
    }

    @Test
    void nonStaticPspc_silentlyUsesDefault() {
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext(BadConfig.class);
        TimerService ts = ctx.getBean(TimerService.class);
        // No exception. But the value is wrong:
        assertThat(ts.getTimeout()).isEqualTo(30); // 30, not 9999
        // In production: your app runs with dev defaults in prod. Silently.
        ctx.close();
    }

    // TRAP 4: setActiveProfiles() after refresh() — applies to BFPP instantiation too
    @Test
    void cannotModifyBFPP_afterRefresh() {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        ctx.register(AppConfig.class);
        ctx.refresh();

        // BFPPs have already run. Adding one now does nothing:
        ctx.addBeanFactoryPostProcessor(new StrictModeBFPP()); // silently does nothing
        // The BFPP never runs. No exception.
        ctx.close();
    }

    // TRAP 5: PriorityOrdered vs Ordered — the getOrder() value is irrelevant across tiers
    @Test
    void priorityOrderedBeatsOrderedRegardlessOfGetOrderValue() {
        // GammaBFPP implements Ordered, getOrder() = Integer.MIN_VALUE (-2147483648)
        // AlphaBFPP implements PriorityOrdered, getOrder() = Integer.MAX_VALUE (2147483647)
        //
        // You expect: GammaBFPP runs first (lower order number)
        // Reality:    AlphaBFPP runs first (PriorityOrdered = separate earlier batch)
        //
        // This is one of the most counterintuitive Spring ordering rules.
        // PriorityOrdered is a TIER above Ordered, not just a lower order number.
    }

    // TRAP 6: BDRPP registered INSIDE another BDRPP IS discovered
    @Component
    static class BdrppA implements BeanDefinitionRegistryPostProcessor {
        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
            // Register a new BDRPP — Spring's loop WILL discover and run this:
            registry.registerBeanDefinition("bdrppB",
                new RootBeanDefinition(BdrppB.class));
            System.out.println("A: registered BdrppB");
        }
        @Override public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {}
    }
    // Output: "A: registered BdrppB" then "B: postProcessBeanDefinitionRegistry runs"
    // NOT a common pattern, but good to understand it works.
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version problem:** Our `StrictModeBFPP` and `PlatformBeansRegistrar` are registered as `@Component` — they're mixed into the application's component scan. Teams using our platform see our internal infrastructure beans in their context. And since both use `System.getProperty()` directly, they're hard to test.

**Senior dev refactoring:** move to `ImportBeanDefinitionRegistrar` triggered by `@EnablePlatform`, register platform BFPPs programmatically with controlled ordering, and wrap everything in a `SpringApplicationRunListener` for Spring Boot integration:

```java
// The entire platform is activated by ONE annotation.
// Teams add this to their @SpringBootApplication class.
// Everything else is invisible to them.
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(PlatformAutoRegistrar.class)
public @interface EnablePlatform {
    String[] scanPackages();
    boolean strictMode() default false;
    int maxPoolSize() default 10;
}

public class PlatformAutoRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata metadata, BeanDefinitionRegistry registry) {

        Map<String, Object> attrs =
            metadata.getAnnotationAttributes(EnablePlatform.class.getName());

        String[] packages   = (String[]) attrs.get("scanPackages");
        boolean strictMode  = (boolean)  attrs.get("strictMode");
        int maxPoolSize     = (int)      attrs.get("maxPoolSize");

        // Register the platform's BFPP as a bean — it will participate in
        // invokeBeanFactoryPostProcessors() automatically:
        GenericBeanDefinition bfppDef = new GenericBeanDefinition();
        bfppDef.setBeanClass(PlatformConfigBFPP.class);
        bfppDef.getPropertyValues().add("strictMode", strictMode);
        bfppDef.getPropertyValues().add("maxPoolSize", maxPoolSize);
        bfppDef.getPropertyValues().add("scanPackages", packages);
        registry.registerBeanDefinition("platformConfigBFPP", bfppDef);

        // Register platform audit logger unconditionally:
        if (!registry.containsBeanDefinition("platformAuditLogger")) {
            GenericBeanDefinition auditDef = new GenericBeanDefinition();
            auditDef.setBeanClass(PlatformAuditLogger.class);
            auditDef.getPropertyValues().add("strictMode", strictMode);
            registry.registerBeanDefinition("platformAuditLogger", auditDef);
        }
    }
}

// The BFPP is now a proper bean with injected configuration — testable:
public class PlatformConfigBFPP implements BeanDefinitionRegistryPostProcessor, Ordered {

    // These are set via BeanDefinition.getPropertyValues() by the registrar above:
    private boolean strictMode;
    private int maxPoolSize;
    private String[] scanPackages;

    // Setters for property injection:
    public void setStrictMode(boolean strictMode) { this.strictMode = strictMode; }
    public void setMaxPoolSize(int maxPoolSize) { this.maxPoolSize = maxPoolSize; }
    public void setScanPackages(String[] scanPackages) { this.scanPackages = scanPackages; }

    @Override
    public int getOrder() { return Ordered.LOWEST_PRECEDENCE - 10; }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // Scan team's packages and register @PlatformService beans:
        ClassPathScanningCandidateComponentProvider scanner =
            new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(PlatformService.class));

        for (String pkg : scanPackages) {
            for (BeanDefinition candidate : scanner.findCandidateComponents(pkg)) {
                String name = deriveBeanName(candidate.getBeanClassName());
                if (!registry.containsBeanDefinition(name)) {
                    if (strictMode) candidate.setScope(BeanDefinition.SCOPE_PROTOTYPE);
                    registry.registerBeanDefinition(name, candidate);
                }
            }
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Adjust pool sizes on any DataSource bean:
        for (String name : beanFactory.getBeanNamesForType(DataSource.class)) {
            BeanDefinition bd = beanFactory.getBeanDefinition(name);
            bd.getPropertyValues().add("maximumPoolSize", maxPoolSize);
        }
    }

    private String deriveBeanName(String className) {
        String simple = className.substring(className.lastIndexOf('.') + 1);
        return Character.toLowerCase(simple.charAt(0)) + simple.substring(1);
    }
}

// Team usage — one line:
@SpringBootApplication
@EnablePlatform(
    scanPackages = {"com.myteam.services"},
    strictMode = true,
    maxPoolSize = 25
)
public class TeamApplication {
    public static void main(String[] args) { SpringApplication.run(TeamApplication.class, args); }
}
```

**What improved:**

- Platform internals (`PlatformConfigBFPP`, `PlatformAuditLogger`) are invisible to teams — no `@Component` scan leakage
- Configuration flows from the annotation's attributes — no `System.getProperty()` — fully testable
- `PlatformConfigBFPP` gets its values via property injection — can be tested by setting properties directly
- Adding the platform to a new team's app = add one annotation. Remove it = remove one annotation.

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **BeanDefinition** | A metadata record (a recipe card) describing HOW to create a bean. Has fields: className, scope, lazyInit, propertyValues, constructorArgs. Not the bean itself — just the description. |
| **BeanFactoryPostProcessor (BFPP)** | A component that runs AFTER all BeanDefinitions are registered but BEFORE any beans are created. It reads and modifies the recipes. Runs once per `refresh()`. |
| **BeanDefinitionRegistryPostProcessor (BDRPP)** | Extends BFPP. Has an extra method `postProcessBeanDefinitionRegistry()` that fires first and gives access to the full registry — enabling adding/removing BeanDefinitions. |
| **ConfigurableListableBeanFactory** | The object passed to `postProcessBeanFactory()`. Lets you iterate all BeanDefinitions, get/modify them. Does NOT let you add new ones (no registry access). |
| **BeanDefinitionRegistry** | The object passed to `postProcessBeanDefinitionRegistry()`. Lets you add, remove, check BeanDefinitions. The "recipe book with editing rights." |
| **ConfigurationClassPostProcessor** | The most important BFPP in Spring. A BDRPP with PriorityOrdered that processes all `@Configuration`, `@ComponentScan`, `@Bean`, `@Import`, `@PropertySource`. Runs absolutely first. |
| **PropertySourcesPlaceholderConfigurer** | A plain BFPP that resolves `${...}` placeholders in BeanDefinition property values. Must be a `static @Bean` in `@Configuration` classes. Auto-configured by Spring Boot. |
| **ImportBeanDefinitionRegistrar** | NOT a BFPP. An interface called BY `ConfigurationClassPostProcessor` when processing `@Import`. Gets annotation metadata from the importing class. Used to build `@Enable*` annotations. |
| **PriorityOrdered** | A marker interface. BFPPs implementing this run in a completely separate, earlier batch than `Ordered` BFPPs. Regardless of `getOrder()` value — the tier matters, not just the number. |
| **Ordered** | Interface with `getOrder()`. BFPPs implementing this run in the second batch, after all `PriorityOrdered` BFPPs. Within the batch: lower number = runs earlier. |
| **BDRPP discovery loop** | Spring's loop that re-discovers BDRPPs after each iteration — because a BDRPP can register new BDRPPs. Repeats until no new BDRPPs are found. |
| **Static @Bean for BFPP** | Required when declaring a BFPP as a `@Bean` in a `@Configuration` class. Static allows Spring to call the method without instantiating the `@Configuration` class first — avoiding a chicken-and-egg problem. |
| **`invokeBeanFactoryPostProcessors()`** | Step 5 of `refresh()` — the phase where ALL BFPPs run. After this step, BeanDefinitions are final. Before next step, beans are created. |

---

### The decision flowchart:

```
Do you need to hook into Spring's startup?
  ├── After beans are created (monitoring, validation)?
  │     → BeanPostProcessor (postProcessAfterInitialization)
  │
  └── Before beans are created (modify metadata, add beans)?
        ↓
        Do you only need to MODIFY existing BeanDefinitions?
          → BeanFactoryPostProcessor (postProcessBeanFactory)
             Examples: change scope, lazy-init, add property values

        Do you need to ADD or REMOVE BeanDefinitions?
          → BeanDefinitionRegistryPostProcessor (postProcessBeanDefinitionRegistry)
             Examples: scan packages, register programmatic beans

        Is the trigger a custom annotation (@EnableX)?
          → ImportBeanDefinitionRegistrar (via @Import)
             Examples: @EnableCaching, @EnableScheduling, @EnablePlatform

What order does my BFPP need?
  → Must run before ALL Ordered BFPPs (infrastructure-level)?
      → implements PriorityOrdered
  → Must run before some but after others?
      → implements Ordered, choose getOrder() value
  → Don't care, just run after everything else?
      → no ordering interface (runs in arbitrary order in the non-ordered batch)

Is @Transactional / @Async / AOP safe on my BFPP?
  → NO. NEVER. AOP is not applied to BFPP beans.
  → Silently does nothing. No error.

Can I call getBean() inside postProcessBeanFactory()?
  → AVOID. Causes premature instantiation. That bean misses BPP processing.
  → Spring warns in the log. The bean loses AOP, @Transactional, @Async.
```

### The timeline that makes everything click:

```
ctx.refresh() called:

Step 5: invokeBeanFactoryPostProcessors()
  ├── All BeanDefinitions exist (from @ComponentScan, @Bean, @Import)
  ├── ZERO ordinary beans exist
  │
  ├── ConfigurationClassPostProcessor runs (PriorityOrdered BDRPP — always FIRST):
  │     → Scans packages → registers @Component BeanDefinitions
  │     → Processes @Bean → registers method BeanDefinitions
  │     → Processes @Import → calls ImportBeanDefinitionRegistrars
  │     Then: postProcessBeanFactory() → CGLIB-enhances @Configuration classes
  │
  ├── PropertySourcesPlaceholderConfigurer runs (PriorityOrdered BFPP):
  │     → Resolves ${db.url} → "jdbc:mysql://prod/db" in every BeanDefinition
  │
  └── Your custom BFPPs/BDRPPs run (Ordered or non-ordered):
        → Modify scopes, add beans, inject properties

Step 6: registerBeanPostProcessors()
  → BPPs registered (but don't run yet)

Step 11: finishBeanFactoryInitialization()
  → Every singleton BeanDefinition → create the actual bean instance
  → Each bean goes through BPP processing (AOP, @Transactional, @Autowired)
  → This is the FIRST TIME any ordinary bean exists

The key insight:
  BFPP saw recipes (BeanDefinitions). It never saw beans.
  BPP sees beans (instances). It never saw the recipes.
  They operate on completely different things at completely different times.
```
