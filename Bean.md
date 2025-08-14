**Core Capabilities**

* Automatic **construction** and **dependency injection**.
* **Lifecycle management** — creation → initialization → usage → destruction.
* **Scope control** — singleton, prototype, request, session, application, websocket.
* **Proxy wrapping** for AOP and transactional behavior.
* **Event awareness** — beans can listen to container and application events.
* **Customizable creation process** via post-processors.

---

**How Spring Creates a Bean (Implementation Flow)**

Internally, the **BeanFactory** / **ApplicationContext** uses this sequence:

1. **Bean Definition Loading**

   * `BeanDefinitionReader` reads configuration (`@Configuration`, XML, etc.).
   * Creates `BeanDefinition` objects that store:

     * Class name
     * Scope
     * Constructor args
     * Property values
     * Init/destroy methods

2. **Instantiation**

   * `InstantiationStrategy` decides *how* to create the object:

     * Direct constructor call via reflection.
     * CGLIB subclass (for proxy needs).
   * If `@Configuration` → enhanced with **CGLIB** to support method call interception.

3. **Dependency Injection**

   * Uses **AutowireCapableBeanFactory**:

     * Constructor injection.
     * Setter injection.
     * Field injection via reflection.

4. **Aware Interfaces**

   * If bean implements:

     * `BeanNameAware` → gets its name.
     * `BeanFactoryAware` → gets reference to factory.
     * `ApplicationContextAware` → gets context reference.

5. **BeanPostProcessors — Pre-initialization**

   * Any registered `BeanPostProcessor.postProcessBeforeInitialization()` is called.
   * This is where **@Autowired processing**, **@Value resolution**, etc. happen.

6. **Initialization**

   * Call custom init methods:

     * `@PostConstruct`
     * `InitializingBean.afterPropertiesSet()`
     * XML/Java-config defined `init-method`.

7. **BeanPostProcessors — Post-initialization**

   * `postProcessAfterInitialization()` is called.
   * This is where AOP proxies are created (e.g., for `@Transactional`).

8. **Bean Ready for Use**

   * Stored in **singleton cache** if scope is singleton.

---
