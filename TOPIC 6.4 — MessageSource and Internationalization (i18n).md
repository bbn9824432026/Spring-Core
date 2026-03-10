# TOPIC 6.4 — MessageSource and Internationalization (i18n)

---

## ① CONCEPTUAL EXPLANATION

### What is MessageSource?

`MessageSource` is Spring's abstraction for resolving messages — parameterized, locale-sensitive text strings — from external sources. It is the foundation of Spring's internationalization (i18n) support, enabling applications to present content in multiple languages without changing application code.

```
Without MessageSource:
    return "Order " + orderId + " placed successfully"; // English hardcoded
    return "Error: " + count + " items out of stock";   // English hardcoded

With MessageSource:
    return messageSource.getMessage("order.placed", new Object[]{orderId}, locale);
    // → "Order 12345 placed successfully" (en-US)
    // → "Bestellung 12345 erfolgreich aufgegeben" (de-DE)
    // → "注文 12345 が正常に完了しました" (ja-JP)
```

---

### 6.4.1 — MessageSource Interface

```java
public interface MessageSource {

    // Resolve message — returns null if not found and no default
    String getMessage(String code, Object[] args, String defaultMessage, Locale locale);

    // Resolve message — throws NoSuchMessageException if not found
    String getMessage(String code, Object[] args, Locale locale)
        throws NoSuchMessageException;

    // Resolve from MessageSourceResolvable (validation errors, etc.)
    String getMessage(MessageSourceResolvable resolvable, Locale locale)
        throws NoSuchMessageException;
}
```

**Three overloads:**

```java
// 1. With default message — NEVER throws
String msg1 = messageSource.getMessage(
    "welcome.message",           // code
    new Object[]{"Alice"},       // args (substituted into {0})
    "Welcome, Alice!",           // defaultMessage — returned if code not found
    Locale.ENGLISH               // locale
);

// 2. Without default — throws NoSuchMessageException if not found
String msg2 = messageSource.getMessage(
    "welcome.message",
    new Object[]{"Alice"},
    Locale.ENGLISH
);

// 3. From MessageSourceResolvable (used by Spring MVC validation)
String msg3 = messageSource.getMessage(
    new DefaultMessageSourceResolvable(
        new String[]{"email.invalid", "field.invalid"}, // try codes in order
        new Object[]{"user@example.com"},
        "Invalid field"                                  // default
    ),
    Locale.ENGLISH
);
```

---

### 6.4.2 — MessageSource Implementations

#### ResourceBundleMessageSource

The most commonly used implementation — backed by Java `ResourceBundle`:

```java
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();

    // Base name(s) of properties files
    ms.setBasename("messages");
    // OR multiple basenames:
    ms.setBasenames("messages", "validation-messages", "error-messages");

    // Default encoding for .properties files
    ms.setDefaultEncoding("UTF-8");

    // Cache duration for property files (-1 = cache forever, default)
    ms.setCacheSeconds(3600); // reload every hour (for hot-reload in dev)

    // Use code as default message (if code not found, return code itself)
    ms.setUseCodeAsDefaultMessage(true);

    // Fall back to system locale if message not found
    ms.setFallbackToSystemLocale(true);

    return ms;
}
```

**File naming convention:**

```
messages.properties              → default (fallback)
messages_en.properties           → English
messages_en_US.properties        → English, United States
messages_en_GB.properties        → English, Great Britain
messages_de.properties           → German
messages_de_DE.properties        → German, Germany
messages_de_AT.properties        → German, Austria
messages_ja.properties           → Japanese
messages_zh_CN.properties        → Chinese, Simplified
messages_zh_TW.properties        → Chinese, Traditional

Search order for Locale("en", "US"):
1. messages_en_US.properties  (most specific)
2. messages_en.properties     (language only)
3. messages.properties        (default fallback)
```

#### ReloadableResourceBundleMessageSource

Extends `ResourceBundleMessageSource` with hot-reload capability:

```java
@Bean
public MessageSource messageSource() {
    ReloadableResourceBundleMessageSource ms =
        new ReloadableResourceBundleMessageSource();

    ms.setBasename("classpath:i18n/messages");
    // Note: "classpath:" prefix — required for ReloadableResourceBundleMessageSource

    ms.setDefaultEncoding("UTF-8");
    ms.setCacheSeconds(60); // reload every 60 seconds

    // Can load from filesystem (not just classpath):
    // ms.setBasename("file:/app/messages/messages");

    return ms;
}
```

**Key difference from `ResourceBundleMessageSource`:**

```
ResourceBundleMessageSource:
    → Uses java.util.ResourceBundle internally
    → Cached at JVM level — cannot be reloaded without restart
    → "classpath:" prefix NOT needed (actually NOT supported)
    → Files must be on classpath root or specified relative path

ReloadableResourceBundleMessageSource:
    → Uses Spring's Resource abstraction
    → Can be reloaded at runtime (setCacheSeconds > 0)
    → Requires "classpath:", "file:" prefix
    → Can load from any Spring Resource location
    → Has caching with expiry
```

#### StaticMessageSource (Testing)

```java
// Programmatic message source — useful in tests
StaticMessageSource ms = new StaticMessageSource();
ms.addMessage("welcome", Locale.ENGLISH, "Welcome, {0}!");
ms.addMessage("welcome", Locale.GERMAN, "Willkommen, {0}!");
ms.addMessage("order.placed", Locale.ENGLISH, "Order {0} placed");
```

#### DelegatingMessageSource

```java
// Used when no MessageSource bean is defined — delegates to parent context
// ApplicationContext creates this automatically as fallback
// Rarely used directly by application code
```

---

### 6.4.3 — Message Format — Parameterized Messages

Spring `MessageSource` uses `java.text.MessageFormat` for parameterized messages:

```properties
# messages.properties
welcome.message=Welcome, {0}! You have {1} new messages.
order.placed=Order {0} has been placed on {1,date,long}.
order.total=Your total is {0,number,currency}.
items.count={0,choice,0#no items|1#one item|1<{0} items}
error.minValue=Value must be at least {0}.
```

```java
// Resolve with arguments:
messageSource.getMessage("welcome.message",
    new Object[]{"Alice", 5}, Locale.ENGLISH);
// → "Welcome, Alice! You have 5 new messages."

messageSource.getMessage("order.placed",
    new Object[]{"ORD-001", new Date()}, Locale.ENGLISH);
// → "Order ORD-001 has been placed on January 15, 2024."

messageSource.getMessage("order.total",
    new Object[]{new BigDecimal("1234.56")}, Locale.US);
// → "Your total is $1,234.56."

messageSource.getMessage("items.count",
    new Object[]{0}, Locale.ENGLISH);
// → "no items"

messageSource.getMessage("items.count",
    new Object[]{1}, Locale.ENGLISH);
// → "one item"

messageSource.getMessage("items.count",
    new Object[]{5}, Locale.ENGLISH);
// → "5 items"
```

**MessageFormat pattern types:**

```
{index}                  → simple substitution
{index,date}             → date format (short, medium, long, full, or pattern)
{index,time}             → time format
{index,number}           → number format (integer, currency, percent, or pattern)
{index,choice,pattern}   → conditional selection
```

---

### 6.4.4 — ApplicationContext as MessageSource

`ApplicationContext` extends `MessageSourceAware` and implements `MessageSource`:

```java
// ApplicationContext itself forwards to the registered MessageSource bean
ctx.getMessage("welcome.message", new Object[]{"Bob"}, Locale.ENGLISH);

// ApplicationContext delegates to the bean named "messageSource":
// If no bean named "messageSource" exists:
//   → Uses DelegatingMessageSource (which delegates to parent or returns default)
```

**CRITICAL:** The `MessageSource` bean MUST be named exactly `"messageSource"`:

```java
// CORRECT — Spring auto-detects this bean:
@Bean
public MessageSource messageSource() {  // method name "messageSource" → bean name "messageSource"
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
    ms.setBasename("messages");
    return ms;
}

// WRONG — NOT auto-detected as the application MessageSource:
@Bean("myMessageSource")
public MessageSource myMessageSource() {
    // Spring will NOT use this as the context's MessageSource
    // ctx.getMessage() will NOT use it
}
```

---

### 6.4.5 — HierarchicalMessageSource — Parent-Child Delegation

`HierarchicalMessageSource` adds parent delegation:

```java
public interface HierarchicalMessageSource extends MessageSource {
    void setParentMessageSource(MessageSource parent);
    MessageSource getParentMessageSource();
}
```

**Parent delegation chain:**

```
Child context MessageSource
    → not found? → Parent context MessageSource
        → not found? → throws NoSuchMessageException (or returns default)

Use case: Module-specific messages + global fallback
```

```java
// Setup parent-child MessageSource hierarchy:
ResourceBundleMessageSource parent = new ResourceBundleMessageSource();
parent.setBasename("global-messages");

ResourceBundleMessageSource child = new ResourceBundleMessageSource();
child.setBasename("module-messages");
child.setParentMessageSource(parent);
// child.getMessage("global.key") → not in module-messages → tries global-messages
```

---

### 6.4.6 — MessageSourceAware

Beans can receive the `MessageSource` via `MessageSourceAware`:

```java
@Component
public class OrderNotificationService implements MessageSourceAware {

    private MessageSource messageSource;

    @Override
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
        // Called during Aware processing (before @PostConstruct)
    }

    public String buildConfirmationMessage(Order order, Locale locale) {
        return messageSource.getMessage(
            "order.confirmation",
            new Object[]{order.getId(), order.getTotal()},
            locale
        );
    }
}

// OR: simply inject via @Autowired
@Component
public class OrderNotificationService {

    @Autowired
    private MessageSource messageSource;

    // Same result — Spring injects the "messageSource" bean
}
```

---

### 6.4.7 — Locale Resolution in Spring MVC

In web applications, the locale is typically determined per-request:

```java
// LocaleResolver — Spring MVC interface:
public interface LocaleResolver {
    Locale resolveLocale(HttpServletRequest request);
    void setLocale(HttpServletRequest request,
                   HttpServletResponse response,
                   Locale locale);
}
```

**Built-in LocaleResolver implementations:**

```java
// 1. AcceptHeaderLocaleResolver (default in Spring MVC)
// Reads Accept-Language HTTP header
// Browser sends: Accept-Language: de-DE,de;q=0.9,en;q=0.8
// → Locale: de_DE
@Bean
public LocaleResolver localeResolver() {
    AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
    resolver.setDefaultLocale(Locale.ENGLISH);
    resolver.setSupportedLocales(List.of(
        Locale.ENGLISH, Locale.GERMAN, Locale.JAPANESE));
    return resolver;
}

// 2. SessionLocaleResolver
// Stores locale in HTTP session — persists across requests
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.ENGLISH);
    return resolver;
}

// 3. CookieLocaleResolver
// Stores locale in browser cookie — persists across sessions
@Bean
public LocaleResolver localeResolver() {
    CookieLocaleResolver resolver = new CookieLocaleResolver();
    resolver.setCookieName("APP_LOCALE");
    resolver.setCookieMaxAge(3600 * 24 * 365); // 1 year
    resolver.setDefaultLocale(Locale.ENGLISH);
    return resolver;
}

// 4. FixedLocaleResolver
// Always returns the same locale — useful for testing
@Bean
public LocaleResolver localeResolver() {
    return new FixedLocaleResolver(Locale.ENGLISH);
}
```

**LocaleChangeInterceptor — change locale via request parameter:**

```java
// Register interceptor to change locale via "?lang=de" parameter
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang"); // ?lang=de switches to German
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
// Usage: GET /products?lang=de → all subsequent messages in German
// (works with SessionLocaleResolver or CookieLocaleResolver)
```

---

### 6.4.8 — MessageSource in Spring MVC — Thymeleaf and JSP

**Thymeleaf:**

```html
<!-- th:text with #{} notation for message resolution -->
<h1 th:text="#{welcome.title}">Welcome</h1>
<p th:text="#{welcome.message(${user.name})}">Welcome, User!</p>

<!-- Conditional with locale -->
<span th:text="#{order.status.${order.status}}">Status</span>
<!-- Resolves: order.status.PENDING, order.status.SHIPPED, etc. -->

<!-- Date formatting with locale -->
<span th:text="${#dates.format(order.date, #{date.format})}">Date</span>
```

**JSP with Spring tags:**

```jsp
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>

<!-- Simple message -->
<spring:message code="welcome.title"/>

<!-- With arguments -->
<spring:message code="welcome.message" arguments="${user.name}"/>

<!-- With default -->
<spring:message code="optional.key" text="Default Text"/>

<!-- Store in variable -->
<spring:message code="page.title" var="pageTitle"/>
<title>${pageTitle}</title>
```

---

### 6.4.9 — Validation Error Messages via MessageSource

Spring's validation framework uses `MessageSource` for error messages:

```java
// When errors.rejectValue("email", "email.invalid") is called,
// Spring generates error codes (DefaultMessageCodesResolver):
// 1. "email.invalid.userForm.email"
// 2. "email.invalid.email"
// 3. "email.invalid.java.lang.String"
// 4. "email.invalid"

// These codes are looked up in MessageSource:
// messages.properties:
// email.invalid={0} is not a valid email address
// email.invalid.userForm.email=Please enter a valid email in the registration form
// field.required=The field {0} is required
// Size.userForm.username=Username must be between {2} and {1} characters

// Spring MVC resolves messages for BindingResult errors:
// MessageCodesResolver → error codes → MessageSource lookup → resolved message
```

**Customizing error message codes:**

```java
// Custom MessageCodesResolver:
@Bean
public MessageCodesResolver messageCodesResolver() {
    DefaultMessageCodesResolver resolver = new DefaultMessageCodesResolver();
    resolver.setMessageCodeFormatter(MessageCodeFormatter.POSTFIX_ERROR_CODE);
    // Changes order: "fieldName.errorCode" instead of "errorCode.fieldName"
    return resolver;
}
```

---

### 6.4.10 — MessageSourceResolvable

`MessageSourceResolvable` is an interface for objects that know how to resolve themselves via a `MessageSource`:

```java
public interface MessageSourceResolvable {
    String[] getCodes();           // try these codes in order
    Object[] getArguments();       // substitution arguments
    String getDefaultMessage();    // fallback if no code found
}
```

`ObjectError` and `FieldError` (from `BindingResult`) implement `MessageSourceResolvable`:

```java
// Spring MVC internally:
BindingResult result = ...; // from controller validation

for (FieldError error : result.getFieldErrors()) {
    // FieldError implements MessageSourceResolvable
    String message = messageSource.getMessage(error, locale);
    // → tries error.getCodes() in order until found
    System.out.println(error.getField() + ": " + message);
}
```

---

### 6.4.11 — Properties File Encoding

Historically `.properties` files used ISO-8859-1 encoding (Java default). Modern Spring supports UTF-8:

```java
// ResourceBundleMessageSource — set encoding
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
    ms.setBasename("messages");
    ms.setDefaultEncoding("UTF-8");  // enables UTF-8 in .properties files
    return ms;
}
```

```properties
# messages_ja.properties (UTF-8 encoded)
welcome.message=ようこそ、{0}さん！
order.placed=注文 {0} を受け付けました。

# messages_ar.properties (UTF-8 encoded — right-to-left language)
welcome.message=مرحباً، {0}!
```

**Alternative: `.properties` with Unicode escapes (legacy, works without UTF-8):**

```properties
# messages_ja.properties (ISO-8859-1 with unicode escapes)
welcome.message=\u3088\u3046\u3053\u305D\u3001{0}\u3055\u3093\uff01
```

---

### Common Misconceptions

**Misconception 1:** "The `MessageSource` bean can have any name."
Wrong. The `ApplicationContext`'s special `MessageSource` detection uses the bean name `"messageSource"` exactly. A bean with any other name is NOT automatically used by `ctx.getMessage()`. It must be explicitly injected where needed.

**Misconception 2:** "`ResourceBundleMessageSource` supports `classpath:` prefix."
Wrong. `ResourceBundleMessageSource` uses Java's `ResourceBundle` internally and does NOT support Spring Resource prefixes like `classpath:`. Use `ReloadableResourceBundleMessageSource` for prefix-based resource loading.

**Misconception 3:** "Missing message code always throws an exception."
Wrong. `getMessage(code, args, defaultMessage, locale)` with a non-null `defaultMessage` NEVER throws — it returns the default. Only `getMessage(code, args, locale)` throws `NoSuchMessageException`. With `useCodeAsDefaultMessage=true`, even the throwing version returns the code as a fallback.

**Misconception 4:** "`AcceptHeaderLocaleResolver` can change locale via a request parameter."
Wrong. `AcceptHeaderLocaleResolver` is READ-ONLY — it reads the HTTP `Accept-Language` header but does NOT support locale change via request parameter. Use `SessionLocaleResolver` or `CookieLocaleResolver` with `LocaleChangeInterceptor` for user-controlled locale switching.

**Misconception 5:** "Locale resolution uses `Locale.getDefault()` as fallback in Spring MVC."
Partially wrong. `AcceptHeaderLocaleResolver` uses `Locale.getDefault()` only if no `Accept-Language` header is present AND no `defaultLocale` is configured. If `setDefaultLocale()` is called, that locale is used as fallback — NOT `Locale.getDefault()`.

**Misconception 6:** "All `MessageSource` implementations support hot-reload."
Wrong. `ResourceBundleMessageSource` uses Java's `ResourceBundle` which caches indefinitely at JVM level — it CANNOT be reloaded at runtime. `ReloadableResourceBundleMessageSource` provides hot-reload via `setCacheSeconds()`.

**Misconception 7:** "MessageFormat arguments use 1-based indexing."
Wrong. `MessageFormat` uses 0-based indexing: `{0}` is the first argument, `{1}` is the second.

---

## ② CODE EXAMPLES

### Example 1 — Complete MessageSource Configuration

```java
// Configuration
@Configuration
public class I18nConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource ms =
            new ReloadableResourceBundleMessageSource();

        // Multiple basenames — checked in order, first found wins
        ms.setBasenames(
            "classpath:i18n/messages",           // main messages
            "classpath:i18n/validation",          // validation messages
            "classpath:i18n/errors"               // error messages
        );

        ms.setDefaultEncoding("UTF-8");
        ms.setCacheSeconds(300);                  // reload every 5 minutes
        ms.setUseCodeAsDefaultMessage(false);     // don't use code as fallback
        ms.setFallbackToSystemLocale(false);      // don't fall back to system locale

        return ms;
    }

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }
}
```

```properties
# i18n/messages.properties (default/English)
app.name=My Application
welcome.title=Welcome to {0}
welcome.user=Hello, {0}! You have {1,choice,0#no messages|1#one message|1<{1} messages}.
order.placed=Order {0} placed successfully.
order.total=Order total: {0,number,currency}
date.format=MM/dd/yyyy

# i18n/messages_de.properties (German)
app.name=Meine Anwendung
welcome.title=Willkommen bei {0}
welcome.user=Hallo, {0}! Sie haben {1,choice,0#keine Nachrichten|1#eine Nachricht|1<{1} Nachrichten}.
order.placed=Bestellung {0} erfolgreich aufgegeben.
order.total=Bestellsumme: {0,number,currency}
date.format=dd.MM.yyyy

# i18n/messages_ja.properties (Japanese, UTF-8)
app.name=マイアプリ
welcome.title={0}へようこそ
welcome.user={0}さん、{1,choice,0#メッセージはありません|1#1件のメッセージ|1<{1}件のメッセージ}があります。
order.placed=注文{0}を受け付けました。
order.total=注文合計: {0,number,currency}
date.format=yyyy/MM/dd
```

---

### Example 2 — MessageSource Usage in Service Layer

```java
@Service
public class OrderConfirmationService {

    @Autowired
    private MessageSource messageSource;

    public OrderConfirmation buildConfirmation(Order order, Locale locale) {
        // Build locale-specific confirmation
        String subject = messageSource.getMessage(
            "email.order.subject",
            new Object[]{order.getId()},
            locale
        );

        String greeting = messageSource.getMessage(
            "email.greeting",
            new Object[]{order.getCustomer().getName()},
            locale
        );

        String orderSummary = messageSource.getMessage(
            "email.order.summary",
            new Object[]{
                order.getId(),
                order.getItems().size(),
                order.getTotal()
            },
            locale
        );

        // Format with locale-specific date
        String formattedDate = messageSource.getMessage(
            "date.format",
            null,
            "MM/dd/yyyy",
            locale
        );

        return OrderConfirmation.builder()
            .subject(subject)
            .greeting(greeting)
            .summary(orderSummary)
            .formattedDate(formattedDate)
            .build();
    }

    public String getStatusMessage(OrderStatus status, Locale locale) {
        // Dynamic code construction
        String code = "order.status." + status.name().toLowerCase();
        return messageSource.getMessage(code, null,
            status.name(), // default = enum name if no message found
            locale);
    }
}
```

---

### Example 3 — Locale Switching in Spring MVC

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver resolver = new CookieLocaleResolver();
        resolver.setCookieName("LOCALE");
        resolver.setCookieMaxAge(60 * 60 * 24 * 365); // 1 year
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setSupportedLocales(List.of(
            Locale.ENGLISH,
            Locale.GERMAN,
            Locale.JAPANESE,
            Locale.forLanguageTag("zh-CN")
        ));
        return resolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("locale");
        // Usage: GET /home?locale=de → switch to German
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}

// Controller — locale resolved automatically per request
@RestController
public class HomeController {

    @Autowired
    private MessageSource messageSource;

    @GetMapping("/welcome")
    public WelcomeResponse welcome(
            @RequestParam String username,
            Locale locale) {          // Spring injects current request locale
        return WelcomeResponse.builder()
            .title(messageSource.getMessage("welcome.title",
                new Object[]{"MyApp"}, locale))
            .message(messageSource.getMessage("welcome.user",
                new Object[]{username, 3}, locale))
            .build();
    }
}
```

---

### Example 4 — Validation Messages via MessageSource

```java
// messages.properties
// --- Bean Validation constraint messages ---
// Override default Hibernate Validator messages:
javax.validation.constraints.NotNull.message=This field is required
javax.validation.constraints.NotBlank.message=This field cannot be blank
javax.validation.constraints.Size.message=Must be between {min} and {max} characters
javax.validation.constraints.Email.message=Must be a valid email address
javax.validation.constraints.Min.message=Must be at least {value}
javax.validation.constraints.Max.message=Must be at most {value}

// --- Spring Validator error codes ---
field.required={0} is required
email.invalid={0} is not a valid email address
password.weak=Password does not meet security requirements
order.amount.negative=Order amount cannot be negative
dates.shipBeforeOrder=Ship date must be after order date

// --- FieldError with MessageSource resolution ---
@RestControllerAdvice
public class ValidationExceptionHandler {

    @Autowired
    private MessageSource messageSource;

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex,
            Locale locale) {

        Map<String, String> errors = new LinkedHashMap<>();

        for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
            // FieldError implements MessageSourceResolvable
            String message = messageSource.getMessage(fieldError, locale);
            errors.put(fieldError.getField(), message);
        }

        return ResponseEntity.badRequest().body(errors);
    }
}
```

---

### Example 5 — MessageSource with HierarchicalMessageSource

```java
// Parent: global application messages
@Bean("globalMessageSource")
public MessageSource globalMessageSource() {
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
    ms.setBasename("messages-global");
    ms.setDefaultEncoding("UTF-8");
    return ms;
}

// Child: module-specific messages, falls back to parent
@Bean
public MessageSource messageSource(
        @Qualifier("globalMessageSource") MessageSource parent) {
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
    ms.setBasename("messages-module");
    ms.setDefaultEncoding("UTF-8");
    ms.setParentMessageSource(parent); // delegate unfound codes to parent
    return ms;
}

// messages-module.properties:
// module.title=User Management Module
// user.created=User {0} created successfully

// messages-global.properties:
// app.name=Enterprise Application
// error.generic=An unexpected error occurred

// Lookup chain:
// messageSource.getMessage("app.name", ...) →
//   not in messages-module → try parent (messages-global) → found: "Enterprise Application"
// messageSource.getMessage("module.title", ...) →
//   found in messages-module → "User Management Module"
```

---

### Example 6 — StaticMessageSource for Testing

```java
@TestConfiguration
public class TestI18nConfig {

    @Bean
    @Primary
    public MessageSource messageSource() {
        StaticMessageSource ms = new StaticMessageSource();

        // Register messages programmatically
        ms.addMessage("welcome.message", Locale.ENGLISH,
            "Welcome, {0}!");
        ms.addMessage("order.placed", Locale.ENGLISH,
            "Order {0} placed");
        ms.addMessage("error.notFound", Locale.ENGLISH,
            "{0} not found");

        // German translations
        ms.addMessage("welcome.message", Locale.GERMAN,
            "Willkommen, {0}!");
        ms.addMessage("order.placed", Locale.GERMAN,
            "Bestellung {0} aufgegeben");

        return ms;
    }
}

@SpringBootTest
@Import(TestI18nConfig.class)
public class OrderServiceI18nTest {

    @Autowired
    private MessageSource messageSource;

    @Test
    public void testEnglishMessages() {
        String msg = messageSource.getMessage(
            "welcome.message", new Object[]{"Alice"}, Locale.ENGLISH);
        assertThat(msg).isEqualTo("Welcome, Alice!");
    }

    @Test
    public void testGermanMessages() {
        String msg = messageSource.getMessage(
            "welcome.message", new Object[]{"Alice"}, Locale.GERMAN);
        assertThat(msg).isEqualTo("Willkommen, Alice!");
    }

    @Test
    public void testMissingMessage() {
        assertThatThrownBy(() ->
            messageSource.getMessage("nonexistent", null, Locale.ENGLISH))
            .isInstanceOf(NoSuchMessageException.class);
    }
}
```

---

### Example 7 — MessageSource with LocaleContextHolder

```java
// In non-web code (services, batch jobs), locale must be set explicitly
@Service
public class BatchNotificationService {

    @Autowired
    private MessageSource messageSource;

    public void processBatch(List<Customer> customers) {
        for (Customer customer : customers) {
            Locale customerLocale = customer.getPreferredLocale();

            // Option 1: Pass locale explicitly (preferred — thread-safe)
            String message = messageSource.getMessage(
                "batch.processed",
                new Object[]{customer.getId()},
                customerLocale
            );

            // Option 2: Set on LocaleContextHolder (for current thread)
            LocaleContextHolder.setLocale(customerLocale);
            try {
                String msg2 = messageSource.getMessage(
                    "batch.processed",
                    new Object[]{customer.getId()},
                    LocaleContextHolder.getLocale() // gets from ThreadLocal
                );
                sendNotification(customer, msg2);
            } finally {
                LocaleContextHolder.resetLocaleContext(); // cleanup ThreadLocal
            }
        }
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What happens when `ApplicationContext.getMessage()` is called but NO bean named `"messageSource"` is registered?

A) `NoSuchBeanDefinitionException` — messageSource bean is required
B) `ApplicationContext` uses a `DelegatingMessageSource` that returns codes as default messages
C) `NullPointerException` — the internal reference is null
D) Falls back to `Locale.getDefault()` for all messages

**Answer: B**
When no `"messageSource"` bean is found, `AbstractApplicationContext` creates a `DelegatingMessageSource` as a fallback. This implementation delegates to the parent context if present, or returns the message code as the default message (or throws `NoSuchMessageException` if no default is provided).

---

**Q2 — Select all that apply:**
Which of the following are TRUE about `ResourceBundleMessageSource`? (Select ALL that apply)

A) Supports "classpath:" prefix in `setBasename()`
B) Uses Java's `java.util.ResourceBundle` internally
C) Cannot reload message files at runtime without restart
D) Requires UTF-8 explicitly set via `setDefaultEncoding()`
E) Searches parent MessageSource if code not found
F) The bean MUST be named "messageSource" to be auto-detected

**Answer: B, C, D, F**
A wrong — `ResourceBundleMessageSource` does NOT support `classpath:` prefix. E is partially wrong — it searches parent only if `setParentMessageSource()` is explicitly configured. B correct — uses `ResourceBundle`. C correct — Java ResourceBundle cache cannot be cleared at runtime. D correct — default encoding is ISO-8859-1 in older versions, must set UTF-8 explicitly. F correct — bean name "messageSource" required for auto-detection.

---

**Q3 — Code Output Prediction:**

```properties
# messages.properties
greeting=Hello, {0}! Count: {1,choice,0#none|1#one|1<many}.
```

```java
String msg = messageSource.getMessage("greeting",
    new Object[]{"Bob", 0}, Locale.ENGLISH);
System.out.println(msg);

String msg2 = messageSource.getMessage("greeting",
    new Object[]{"Alice", 1}, Locale.ENGLISH);
System.out.println(msg2);

String msg3 = messageSource.getMessage("greeting",
    new Object[]{"Charlie", 5}, Locale.ENGLISH);
System.out.println(msg3);
```

A)
```
Hello, Bob! Count: none.
Hello, Alice! Count: one.
Hello, Charlie! Count: many.
```
B)
```
Hello, Bob! Count: 0.
Hello, Alice! Count: 1.
Hello, Charlie! Count: 5.
```
C)
```
Hello, Bob! Count: none.
Hello, Alice! Count: 1.
Hello, Charlie! Count: many.
```
D)
```
Hello, {0}! Count: none.
Hello, {0}! Count: one.
Hello, {0}! Count: many.
```

**Answer: A**
`{0}` substitutes the name. `{1,choice,0#none|1#one|1<many}` uses choice format: `0#none` (exactly 0 → "none"), `1#one` (exactly 1 → "one"), `1<many` (greater than 1 → "many"). All three substitutions work correctly.

---

**Q4 — MCQ:**
What is the difference between `ReloadableResourceBundleMessageSource` and `ResourceBundleMessageSource`?

A) They are identical — both support hot-reload
B) `ReloadableResourceBundleMessageSource` supports `classpath:` prefix and hot-reload; `ResourceBundleMessageSource` uses Java `ResourceBundle` and cannot be reloaded
C) `ResourceBundleMessageSource` supports more locales
D) `ReloadableResourceBundleMessageSource` only works in Spring Boot

**Answer: B**
`ReloadableResourceBundleMessageSource` uses Spring's Resource abstraction (supporting `classpath:`, `file:` prefixes) and supports cache expiry via `setCacheSeconds()`. `ResourceBundleMessageSource` uses Java's `ResourceBundle` which does NOT support Spring resource prefixes and caches indefinitely.

---

**Q5 — Trick Question:**

```java
@Bean("appMessages")  // NOT named "messageSource"
public MessageSource applicationMessageSource() {
    ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
    ms.setBasename("messages");
    return ms;
}

// In a controller:
@Autowired
private ApplicationContext ctx;

public String getMessage() {
    return ctx.getMessage("welcome.title", null, Locale.ENGLISH);
}
```

What happens when `getMessage()` is called?

A) Returns the message from `messages.properties` — Spring finds the bean by type
B) Returns null — no message source configured
C) Uses `DelegatingMessageSource` — "welcome.title" code returned as message or `NoSuchMessageException`
D) `NoSuchBeanDefinitionException` — messageSource bean not found

**Answer: C**
The bean is named `"appMessages"` — NOT `"messageSource"`. `ApplicationContext` specifically looks for a bean named `"messageSource"`. Since it's not found, a `DelegatingMessageSource` is used. Calling `ctx.getMessage("welcome.title", null, Locale.ENGLISH)` either returns `"welcome.title"` (if `useCodeAsDefaultMessage=true` on the delegating source) or throws `NoSuchMessageException`.

---

**Q6 — Select all that apply:**
Which `LocaleResolver` implementations support locale switching via request parameter (with `LocaleChangeInterceptor`)? (Select ALL that apply)

A) `AcceptHeaderLocaleResolver`
B) `SessionLocaleResolver`
C) `CookieLocaleResolver`
D) `FixedLocaleResolver`
E) `AbstractLocaleContextResolver`

**Answer: B, C**
`LocaleChangeInterceptor` calls `localeResolver.setLocale()` to update the locale. `AcceptHeaderLocaleResolver` is read-only — `setLocale()` throws `UnsupportedOperationException`. `FixedLocaleResolver` is always fixed — `setLocale()` also throws. Only `SessionLocaleResolver` and `CookieLocaleResolver` support locale mutation.

---

**Q7 — MCQ:**
In `messages.properties`, what is the correct MessageFormat pattern for: "You have 1 new message" (singular) vs "You have N new messages" (plural)?

A) `msg=You have {0} new message{0,choice,1#|1<s}.`
B) `msg=You have {0,choice,0#no messages|1#one new message|1<{0} new messages}.`
C) `msg=You have {0} new {0,plural,one{message}other{messages}}.`
D) `msg=You have {0} new message(s).`

**Answer: B**
`java.text.MessageFormat` uses `choice` format for conditional text: `{0,choice,0#no messages|1#one new message|1<{0} new messages}`. The format is `value#text` with `|` separating cases. `value<text` means "greater than value". Spring's MessageSource uses standard `MessageFormat`.

---

**Q8 — Select all that apply:**
Which of the following correctly describe `MessageSourceResolvable`? (Select ALL that apply)

A) `FieldError` and `ObjectError` from `BindingResult` implement it
B) It provides a list of message codes to try in priority order
C) It can only be used with `ResourceBundleMessageSource`
D) Spring MVC uses it to resolve validation error messages via MessageSource
E) `DefaultMessageSourceResolvable` is a concrete implementation

**Answer: A, B, D, E**
C is wrong — `MessageSourceResolvable` works with any `MessageSource` implementation. A correct — both `FieldError` and `ObjectError` implement it. B correct — `getCodes()` returns multiple codes tried in order. D correct — Spring MVC's validation error message resolution uses `messageSource.getMessage(fieldError, locale)`. E correct.

---

**Q9 — Scenario-based:**

```properties
# messages.properties
order.status.pending=Pending
order.status.shipped=Shipped

# messages_de.properties
order.status.pending=Ausstehend
```

```java
// Locale.GERMAN, code = "order.status.shipped"
String msg = messageSource.getMessage(
    "order.status.shipped", null, null, Locale.GERMAN);
System.out.println(msg);
```

What is the output?

A) `"Shipped"` — falls back to default messages.properties
B) `null` — key not found in German messages, null default provided
C) `"order.status.shipped"` — code returned as default
D) `NoSuchMessageException` — key not found

**Answer: B**
The code `"order.status.shipped"` is NOT in `messages_de.properties`. Spring falls back to `messages.properties` ONLY if `fallbackToSystemLocale` is enabled (default: `true` for `ResourceBundleMessageSource`). Wait — with `fallbackToSystemLocale=true` (default), it searches `messages_de.properties`, then `messages.properties`. `messages.properties` has `order.status.shipped=Shipped`. So result: `"Shipped"` (A). BUT the `defaultMessage` parameter is `null`, and the overload with `defaultMessage` never throws — returns null if not found. With fallback enabled: **A** ("Shipped").

---

**Q10 — Drag-and-Drop: Match property file to search order for Locale("de", "DE"):**

Files available:
- `messages.properties`
- `messages_de.properties`
- `messages_de_DE.properties`
- `messages_en.properties`
- `messages_en_US.properties`

Order searched (first to last):

**Answer (priority order, most specific first):**
1. `messages_de_DE.properties` (language + country — most specific)
2. `messages_de.properties` (language only)
3. `messages.properties` (default fallback)

`messages_en.properties` and `messages_en_US.properties` are NOT searched for a German locale.

---

## ④ TRICK ANALYSIS

**Trap 1: MessageSource bean can have any name**
Wrong. `ApplicationContext` specifically looks for a bean named `"messageSource"` (exactly). A bean named anything else will NOT be auto-detected as the application's `MessageSource`. It can still be injected by name or type where needed, but `ctx.getMessage()` will use the fallback `DelegatingMessageSource`.

**Trap 2: ResourceBundleMessageSource supports classpath: prefix**
Wrong. `ResourceBundleMessageSource` uses Java's `ResourceBundle.getBundle()` which does not support Spring's resource prefixes. Use `ReloadableResourceBundleMessageSource` for `classpath:` or `file:` prefixes.

**Trap 3: Missing message always throws NoSuchMessageException**
Wrong. The three-argument `getMessage(code, args, defaultMessage, locale)` NEVER throws — returns `defaultMessage` when code not found. Only the two-argument `getMessage(code, args, locale)` throws. With `useCodeAsDefaultMessage=true`, even the throwing version returns the code itself.

**Trap 4: AcceptHeaderLocaleResolver supports setLocale()**
Wrong. `AcceptHeaderLocaleResolver.setLocale()` throws `UnsupportedOperationException`. It is read-only — reads `Accept-Language` header. `LocaleChangeInterceptor` requires a mutable `LocaleResolver` (`SessionLocaleResolver` or `CookieLocaleResolver`).

**Trap 5: MessageFormat uses 1-based argument indexing**
Wrong. `java.text.MessageFormat` uses 0-based indexing: `{0}` is first arg, `{1}` is second. This trips up developers from other frameworks.

**Trap 6: ReloadableResourceBundleMessageSource reloads automatically**
Partially wrong. Reload happens LAZILY — not on a background timer. The message source checks if files need reloading when a message IS requested, after `cacheSeconds` have elapsed since the last load. There is no background thread reloading files proactively.

**Trap 7: Fallback locale search goes from default to specific**
Wrong. Search goes from MOST SPECIFIC to LEAST SPECIFIC:
1. `messages_de_DE.properties` (most specific)
2. `messages_de.properties` (language only)
3. `messages.properties` (default)

The fallback chain is NEVER reversed.

**Trap 8: Spring's MessageSource and Java's ResourceBundle are equivalent**
Wrong. Java's `ResourceBundle` does NOT support parameterized messages natively (no `{0}` substitution). Spring's `MessageSource` wraps `ResourceBundle` AND applies `MessageFormat` for argument substitution. This is why `MessageFormat` patterns work in `.properties` files used with Spring.

---

## ⑤ SUMMARY SHEET

### MessageSource Implementation Comparison

```
┌──────────────────────────────────┬──────────────────────────┬──────────────────────────┐
│ Feature                          │ ResourceBundleMessageSrc  │ ReloadableResourceBundle │
├──────────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Underlying mechanism             │ java.util.ResourceBundle  │ Spring Resource           │
│ classpath: prefix support        │ NO                        │ YES                       │
│ file: prefix support             │ NO                        │ YES                       │
│ Hot-reload support               │ NO                        │ YES (setCacheSeconds)     │
│ Default encoding                 │ ISO-8859-1 (set UTF-8)    │ ISO-8859-1 (set UTF-8)   │
│ JVM-level caching                │ YES (permanent)           │ NO (controlled by Spring) │
│ Production recommendation        │ YES (performance)         │ DEV (convenience)        │
└──────────────────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

### Locale Resolution Search Order

```
For Locale("de", "DE"):
1. messages_de_DE.properties   ← most specific (language + country)
2. messages_de.properties      ← language only
3. messages.properties         ← default fallback

For Locale("en", "US"):
1. messages_en_US.properties
2. messages_en.properties
3. messages.properties
```

---

### LocaleResolver Comparison

```
┌────────────────────────────────┬──────────────┬──────────────┬───────────────┐
│ Feature                        │ AcceptHeader │ Session      │ Cookie        │
├────────────────────────────────┼──────────────┼──────────────┼───────────────┤
│ Storage                        │ HTTP Header  │ HttpSession  │ Browser cookie│
│ Supports setLocale()           │ NO           │ YES          │ YES           │
│ Persists across requests       │ NO (header)  │ Session life │ Cookie expiry │
│ LocaleChangeInterceptor works  │ NO           │ YES          │ YES           │
│ Default in Spring MVC          │ YES          │ NO           │ NO            │
└────────────────────────────────┴──────────────┴──────────────┴───────────────┘
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| MessageSource bean name | MUST be `"messageSource"` for auto-detection |
| `classpath:` in basename | NOT supported in `ResourceBundleMessageSource` |
| `classpath:` support | Only in `ReloadableResourceBundleMessageSource` |
| Fallback search order | Most specific → language only → default |
| MessageFormat indexing | 0-based: `{0}`, `{1}`, `{2}` |
| Hot-reload | Only `ReloadableResourceBundleMessageSource` supports it |
| `getMessage` with defaultMessage | NEVER throws — returns default if not found |
| `getMessage` without defaultMessage | Throws `NoSuchMessageException` if not found |
| `useCodeAsDefaultMessage=true` | Returns code itself — NEVER throws |
| `AcceptHeaderLocaleResolver` | Read-only — cannot switch via parameter |
| `setLocale()` support | Session and Cookie resolvers ONLY |
| `LocaleContextHolder` | ThreadLocal locale storage — manual management |

---

### MessageFormat Quick Reference

```
{0}                    → simple substitution (first arg)
{1}                    → second argument
{0,date}               → date format with default locale format
{0,date,short}         → short date: "1/15/24"
{0,date,long}          → long date: "January 15, 2024"
{0,date,yyyy-MM-dd}    → custom pattern: "2024-01-15"
{0,number}             → default number format
{0,number,currency}    → currency: "$1,234.56"
{0,number,percent}     → percent: "75%"
{0,number,#,##0.00}    → custom: "1,234.56"
{0,choice,0#none|1#one|1<many} → conditional/plural
```

---

### Interview One-Liners

- "The `MessageSource` bean MUST be named exactly `'messageSource'` for `ApplicationContext.getMessage()` to auto-detect it — any other name requires explicit injection."
- "`ResourceBundleMessageSource` does NOT support `classpath:` prefix — it uses Java's `ResourceBundle`. Use `ReloadableResourceBundleMessageSource` for Spring Resource prefix support and hot-reload."
- "Locale resolution search order goes from MOST SPECIFIC to LEAST SPECIFIC: `messages_de_DE.properties` → `messages_de.properties` → `messages.properties`."
- "`AcceptHeaderLocaleResolver` is READ-ONLY — it cannot be updated via `LocaleChangeInterceptor`. Use `SessionLocaleResolver` or `CookieLocaleResolver` for user-controlled locale switching."
- "`getMessage(code, args, defaultMessage, locale)` NEVER throws — returns `defaultMessage` if code not found. The two-argument version throws `NoSuchMessageException`."
- "`FieldError` and `ObjectError` implement `MessageSourceResolvable` — Spring MVC resolves validation error messages by calling `messageSource.getMessage(fieldError, locale)`."
- "`LocaleContextHolder` stores locale in a `ThreadLocal` — useful in non-web code (services, batch) for locale-aware message resolution without passing locale explicitly."
- "Java's `MessageFormat` uses 0-based argument indexing: `{0}` is the first argument. Plural/conditional forms use `choice` format: `{0,choice,0#none|1#one|1<many}`."

---
