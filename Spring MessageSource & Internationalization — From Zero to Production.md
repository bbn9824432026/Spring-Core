# Spring MessageSource & Internationalization — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

Your e-commerce app is expanding. The business ticket arrives:

```
TICKET-891: Launch in Germany and Japan
- All UI text, emails, error messages must appear in the user's language
- German users see German. Japanese users see Japanese. Default: English.
- User can switch language via ?lang=de in the URL
- Validation errors must ALSO be translated (not just page text)
- Emails sent to customers must be in THEIR language, not the server's language
- Works in: controllers, service layer, email templates, Thymeleaf views
```

You sit down and write the obvious first attempt:

```java
@Service
public class OrderService {

    public String buildConfirmationEmail(Order order, String userLanguage) {

        if ("de".equals(userLanguage)) {
            return "Sehr geehrte(r) " + order.getCustomerName() + ",\n"
                 + "Ihre Bestellung " + order.getId() + " wurde aufgegeben.\n"
                 + "Gesamtbetrag: " + order.getTotal() + " EUR";
        } else if ("ja".equals(userLanguage)) {
            return order.getCustomerName() + "様\n"
                 + "ご注文 " + order.getId() + " を受け付けました。\n"
                 + "合計金額: " + order.getTotal() + " 円";
        } else {
            return "Dear " + order.getCustomerName() + ",\n"
                 + "Your order " + order.getId() + " has been placed.\n"
                 + "Total: $" + order.getTotal();
        }
    }
}
```

Two days later:

```java
@RestController
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<?> getProduct(@PathVariable String id, String lang) {

        if (product == null) {
            if ("de".equals(lang)) {
                return ResponseEntity.status(404)
                    .body("Produkt " + id + " nicht gefunden");
            } else if ("ja".equals(lang)) {
                return ResponseEntity.status(404)
                    .body("商品 " + id + " が見つかりません");
            } else {
                return ResponseEntity.status(404)
                    .body("Product " + id + " not found");
            }
        }
        // ...
    }
}
```

By week two you have this same `if("de")... else if("ja")... else...` pattern in:

- 8 controllers
- 4 service classes
- 2 email builders
- validation error messages

And now the business says: **add French**. You now touch 14 files. You miss three places. French users see English error messages. QA files bugs.

**The core problems:**

```
Problem 1: Language logic is INLINED everywhere
           Adding a new language = touching every file that ever builds a string

Problem 2: Translatable text is buried in Java code
           Translators can't edit .java files — they need text files they can open in Excel

Problem 3: User's language is passed manually everywhere
           Every method takes a String lang parameter — it leaks through every layer

Problem 4: Validation errors speak a different language than the rest of the app
           Your @NotBlank says "must not be blank" in English even when everything else is German

Problem 5: Currency and date formatting is locale-specific
           "$1,234.56" vs "1.234,56 €" — this is a country-level concern, not a dev concern
```

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Create a TranslationService with a big map**

```java
@Service
public class TranslationService {
    private static final Map<String, Map<String, String>> translations = new HashMap<>();

    static {
        translations.put("order.placed", Map.of(
            "en", "Order {0} placed",
            "de", "Bestellung {0} aufgegeben",
            "ja", "注文{0}を受け付けました"
        ));
        // ... 200 more entries
    }

    public String get(String key, String lang, Object... args) {
        return MessageFormat.format(
            translations.getOrDefault(key, Map.of()).getOrDefault(lang, key),
            args
        );
    }
}
```

**Why this fails:** Every translation change requires a recompile and redeploy. Your translators (non-developers) can't touch this. The class becomes 3000 lines. You need to restart the server to fix a typo in a translation. Adding French means a Java code change, not a text file addition. This is the same problem as before — just consolidated.

---

**Naive Approach 2: Load text from a database**

```java
@Service
public class TranslationService {
    @Autowired JdbcTemplate jdbc;

    public String get(String key, String locale) {
        return jdbc.queryForObject(
            "SELECT text FROM translations WHERE key=? AND locale=?",
            String.class, key, locale);
    }
}
```

**Why this fails:** Every rendered string now hits the database. A page with 50 strings = 50 queries. You add caching, now you have cache invalidation problems. You still need to pass the locale to every method. You've built a poor version of something that already exists: `.properties` files with caching, which is exactly what Spring ships.

---

**Naive Approach 3: One `Locale` parameter passed everywhere**

```java
// At least pull text out of code into properties files...
public String buildEmail(Order order, Locale locale) {
    ResourceBundle bundle = ResourceBundle.getBundle("messages", locale);
    return bundle.getString("order.placed").replace("{0}", order.getId());
}
```

**Why this fails:** `ResourceBundle` does no argument substitution (`{0}` doesn't work — you do it yourself, inconsistently). Every method still takes a `Locale` parameter that has to be threaded from the HTTP request all the way down into the service layer. And you're reinventing Spring's `MessageSource` by hand — same file format, none of the features.

---

**The correct solution emerges:**

We need three things that work together:

```
1. Text lives in FILES (.properties), not in Java code
   → Translators edit files, not Java
   → Adding a language = adding a file, touching zero Java

2. A CENTRAL RESOLVER that knows: "for Locale X, look up file Y, apply MessageFormat"
   → One place in the whole app: MessageSource
   → Every component asks IT for text, not each other

3. The CURRENT LOCALE lives somewhere accessible to the whole request
   → HTTP request carries it (Accept-Language header, cookie, session)
   → Spring extracts it once per request, makes it available everywhere
   → Service layer gets the locale injected — doesn't need an HTTP dependency
```

---

## 🧠 3. Concept Introduction — Plain English First

### The Library Analogy

Imagine a library with books in every language. The library has:

- **Shelves organized by language** — one shelf for German, one for Japanese, one for English (the fallback shelf).
- **A librarian** (MessageSource) who knows the system: "You want 'order.placed' in German? Let me check the German shelf first. Not there? Check the German shelf for just language... not there? Try the English fallback shelf."
- **A membership card** that each user carries — it says what language they prefer (the `Locale`).
- **A front-desk receptionist** (LocaleResolver) who reads the membership card from the user's pocket when they walk in, so the librarian doesn't need to ask every time.

```
User arrives (HTTP request)
    ↓
Receptionist reads their preference card (LocaleResolver):
  "Accept-Language: de-DE" → Locale = de_DE

Library lookup (MessageSource):
  "order.placed" in de_DE →
    1. Check messages_de_DE.properties  ← most specific shelf
    2. Check messages_de.properties     ← language shelf
    3. Check messages.properties        ← English fallback shelf
  Found: "Bestellung {0} aufgegeben"

MessageFormat fills in the blanks:
  {0} → "ORD-001"
  Result: "Bestellung ORD-001 aufgegeben"
```

### The Properties File Structure

```
src/main/resources/
  i18n/
    messages.properties          ← English (default/fallback)
    messages_de.properties       ← German
    messages_de_DE.properties    ← German, Germany specifically
    messages_de_AT.properties    ← German, Austria specifically
    messages_ja.properties       ← Japanese

Search order for a German user from Germany:
  de_DE → de → (default)    ← always most-specific-first
```

### Where Does the Locale Come From?

In a web app, the locale is a per-request thing. Spring resolves it ONCE at the start of the request and stores it in a **ThreadLocal** — a variable tied to the current thread (which handles exactly one request at a time). Any code running on that thread can read the current locale without needing it passed in.

```
HTTP Request
    ↓
[LocaleResolver reads Accept-Language header or cookie]
    ↓
LocaleContextHolder.setLocale(de_DE)   ← ThreadLocal storage
    ↓
[Your controller runs]  → gets Locale via method parameter injection
    ↓
[Your service runs]     → reads from LocaleContextHolder or gets Locale passed in
    ↓
HTTP Response sent
    ↓
LocaleContextHolder.resetLocaleContext()  ← cleaned up
```

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TODO 1: Create properties files — pull all text out of Java code
TODO 2: Configure MessageSource bean — the central resolver, named exactly right
TODO 3: Configure LocaleResolver — how the app reads the user's language preference
TODO 4: Add LocaleChangeInterceptor — allow ?lang=de to switch language
TODO 5: Use MessageSource in service layer — pass Locale explicitly
TODO 6: Use MessageSource in controller — let Spring inject Locale
TODO 7: Wire validation error messages through MessageSource too
TODO 8: Handle the "no translation found" edge cases gracefully
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: Create properties files

```properties
# src/main/resources/i18n/messages.properties  (English — always the fallback)
welcome.title=Welcome to {0}
welcome.user=Hello, {0}! You have {1,choice,0#no messages|1#one message|1<{1} messages}.
order.placed=Order {0} has been placed successfully.
order.total=Order total: {0,number,currency}
order.status.pending=Pending
order.status.shipped=Shipped
order.status.delivered=Delivered
product.not.found=Product {0} was not found.
error.generic=An unexpected error occurred. Please try again.
date.format=MM/dd/yyyy
```

```properties
# src/main/resources/i18n/messages_de.properties  (German)
welcome.title=Willkommen bei {0}
welcome.user=Hallo, {0}! Sie haben {1,choice,0#keine Nachrichten|1#eine Nachricht|1<{1} Nachrichten}.
order.placed=Bestellung {0} wurde erfolgreich aufgegeben.
order.total=Bestellsumme: {0,number,currency}
order.status.pending=Ausstehend
order.status.shipped=Versendet
order.status.delivered=Zugestellt
product.not.found=Produkt {0} wurde nicht gefunden.
error.generic=Ein unerwarteter Fehler ist aufgetreten. Bitte versuchen Sie es erneut.
date.format=dd.MM.yyyy
```

```properties
# src/main/resources/i18n/messages_ja.properties  (Japanese — file must be saved as UTF-8)
welcome.title={0}へようこそ
welcome.user={0}さん、{1,choice,0#メッセージはありません|1#1件のメッセージ|1<{1}件のメッセージ}があります。
order.placed=ご注文{0}を受け付けました。
order.total=注文合計: {0,number,currency}
order.status.pending=保留中
order.status.shipped=発送済み
order.status.delivered=配達済み
product.not.found=商品{0}が見つかりません。
error.generic=予期しないエラーが発生しました。もう一度お試しください。
date.format=yyyy/MM/dd
```

**The `{0,choice,...}` pattern — how plurals work in MessageFormat:**

```
{1,choice,0#no messages|1#one message|1<{1} messages}
         ↑             ↑             ↑
     arg index      exactly 0    exactly 1   greater than 1
                                             (uses arg value in text)

0#no messages   → when arg[1] == 0  → "no messages"
1#one message   → when arg[1] == 1  → "one message"
1<{1} messages  → when arg[1] > 1   → "5 messages" (inserts the number)
```

**Important: MessageFormat uses 0-based indexing. `{0}` is the first argument, `{1}` is the second. This trips up almost everyone the first time.**

---

### TODO 2: Configure MessageSource — the ONE thing that cannot be named wrong

```java
@Configuration
public class I18nConfig {

    // MessageSource: the central component that resolves a (code, locale) pair
    // to a human-readable string, using properties files as the source of truth.
    //
    // THE NAME IS NOT NEGOTIABLE:
    // ApplicationContext specifically looks for a bean named EXACTLY "messageSource".
    // Any other name → the context creates a fallback DelegatingMessageSource
    // that just returns the code itself or throws NoSuchMessageException.
    // No error is shown. Your app silently shows "order.placed" instead of text.
    @Bean
    public MessageSource messageSource() {

        // ReloadableResourceBundleMessageSource: the modern implementation.
        // Uses Spring's Resource abstraction — supports "classpath:" prefix.
        // Can reload files at runtime when they change (for dev convenience).
        //
        // The other option is ResourceBundleMessageSource (uses Java's ResourceBundle).
        // CRITICAL DIFFERENCE:
        //   ResourceBundleMessageSource: NO "classpath:" prefix, NO hot-reload
        //   ReloadableResourceBundleMessageSource: YES "classpath:", YES hot-reload
        ReloadableResourceBundleMessageSource messageSource =
            new ReloadableResourceBundleMessageSource();

        // setBasename: the path prefix for your properties files.
        // "classpath:i18n/messages" means Spring looks for:
        //   classpath:i18n/messages.properties
        //   classpath:i18n/messages_de.properties
        //   etc.
        // The suffix ".properties" is added automatically — don't include it.
        messageSource.setBasename("classpath:i18n/messages");

        // setDefaultEncoding: MUST be set for any non-Latin characters.
        // Without this, Japanese/Chinese/Arabic characters become garbage "????".
        // Default encoding is ISO-8859-1 — fine for Western Europe, broken for everything else.
        messageSource.setDefaultEncoding("UTF-8");

        // setCacheSeconds: how long to cache the loaded properties before re-reading the file.
        // -1 = cache forever (production default — fast, no file IO per request)
        //  0 = never cache (reload every request — only for debugging)
        // 60 = reload every 60 seconds (good for dev — edit translation and see it without restart)
        messageSource.setCacheSeconds(3600);

        // setFallbackToSystemLocale: if no file exists for the requested locale,
        // should Spring fall back to the JVM's system locale before trying messages.properties?
        // false = skip JVM locale, go straight to the default messages.properties
        // This is SAFER — your fallback is predictable (always English) not server-dependent.
        messageSource.setFallbackToSystemLocale(false);

        return messageSource;
    }
}
```

**The silent failure trap — wrong name:**

```java
// WRONG — Spring ignores this for ctx.getMessage():
@Bean("translations")
public MessageSource translations() {
    ReloadableResourceBundleMessageSource ms = new ReloadableResourceBundleMessageSource();
    ms.setBasename("classpath:i18n/messages");
    return ms;
}

// Your app starts fine. No error. But:
ctx.getMessage("order.placed", new Object[]{"ORD-001"}, Locale.GERMAN);
// → NoSuchMessageException: No message found under code 'order.placed'
//   (because it's using the fallback DelegatingMessageSource, not your bean)

// Fix: the method name determines the bean name in @Bean methods.
// Name your method "messageSource":
@Bean
public MessageSource messageSource() { ... }  // bean name = "messageSource" ✓

// OR be explicit:
@Bean("messageSource")
public MessageSource translations() { ... }   // explicit name = "messageSource" ✓
```

**`ResourceBundleMessageSource` vs `ReloadableResourceBundleMessageSource` — the one decision that matters:**

```
ResourceBundleMessageSource:
  ✓ Slightly simpler to configure
  ✗ Basename: "i18n/messages" (NO "classpath:" prefix — it breaks)
  ✗ JVM-level cache: once loaded, never reloads until server restart
  → Use in production where you never need hot-reload

ReloadableResourceBundleMessageSource:
  ✓ Basename: "classpath:i18n/messages" (Spring Resource prefix — required here)
  ✓ setCacheSeconds() actually works — files reload after expiry
  ✓ Can load from "file:/some/external/path" (not just classpath)
  → Use everywhere — the safe default
```

---

### TODO 3: Configure LocaleResolver — how does Spring know the user's language?

```java
// LocaleResolver: the component Spring MVC calls at the START of every request
// to determine which Locale to use for that request.
// Spring MVC has NO built-in locale — it MUST ask a LocaleResolver.
// The resolved Locale is then available via: Locale locale in controller params.

// OPTION A: SessionLocaleResolver (good for traditional web apps)
// Stores the user's chosen locale in their HTTP session.
// Once set: persists across all requests in the same session.
// Survives page refreshes. Lost when session expires.
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    // defaultLocale: what to use before the user has ever set a preference.
    resolver.setDefaultLocale(Locale.ENGLISH);
    return resolver;
}

// OPTION B: CookieLocaleResolver (good for long-lived preference)
// Stores locale in a browser cookie.
// Survives session expiry, browser restart.
// @Bean
// public LocaleResolver localeResolver() {
//     CookieLocaleResolver resolver = new CookieLocaleResolver();
//     resolver.setCookieName("LOCALE");
//     resolver.setCookieMaxAge(60 * 60 * 24 * 365); // 1 year
//     resolver.setDefaultLocale(Locale.ENGLISH);
//     return resolver;
// }

// OPTION C: AcceptHeaderLocaleResolver (good for APIs / mobile clients)
// Reads the HTTP "Accept-Language: de-DE" header sent by the browser/client.
// READ-ONLY: cannot be changed via request parameter.
// @Bean
// public LocaleResolver localeResolver() {
//     AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
//     resolver.setDefaultLocale(Locale.ENGLISH);
//     resolver.setSupportedLocales(List.of(Locale.ENGLISH, Locale.GERMAN));
//     return resolver;
// }
```

**The AcceptHeaderLocaleResolver silent failure trap:**

```java
// You configure AcceptHeaderLocaleResolver AND LocaleChangeInterceptor (?lang=de)
// expecting them to work together.

// Result: clicking "Switch to German" does NOTHING.
// No error. The page just reloads in the same language.

// Why:
// LocaleChangeInterceptor works by calling localeResolver.setLocale(request, response, newLocale)
// AcceptHeaderLocaleResolver.setLocale() → throws UnsupportedOperationException
// → Spring catches it internally in some versions, or silently ignores it

// Fix: use SessionLocaleResolver or CookieLocaleResolver with LocaleChangeInterceptor.
// AcceptHeaderLocaleResolver is only for read-only locale detection (like REST APIs).
```

---

### TODO 4: Add LocaleChangeInterceptor — ?lang=de switches language

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // LocaleChangeInterceptor: an interceptor that runs before every controller method.
    // It checks the request for a specific parameter (?lang=de).
    // If found, it calls localeResolver.setLocale() to update the stored locale.
    // Requires a MUTABLE LocaleResolver (Session or Cookie — NOT AcceptHeader).
    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        // paramName: the URL parameter that triggers a locale change
        interceptor.setParamName("lang");
        // Usage: GET /products?lang=de  → switches to German for all future requests
        //         GET /products?lang=ja  → switches to Japanese
        //         GET /products?lang=en  → switches back to English
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // addInterceptor: registers the interceptor to run for ALL requests.
        // The interceptor fires before your controller method body.
        registry.addInterceptor(localeChangeInterceptor());
    }

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }
}
```

**What happens on `GET /products?lang=de`:**

```
Request arrives: GET /products?lang=de

1. LocaleChangeInterceptor fires (before controller):
   → sees parameter "lang" = "de"
   → calls sessionLocaleResolver.setLocale(request, response, Locale.GERMAN)
   → Locale.GERMAN saved in session

2. Controller method runs:
   → Spring injects Locale locale = Locale.GERMAN (from session)
   → messageSource.getMessage("order.placed", ..., Locale.GERMAN)
   → "Bestellung ORD-001 wurde erfolgreich aufgegeben."

3. Next request (no ?lang= parameter):
   → SessionLocaleResolver reads from session: still Locale.GERMAN
   → User stays in German until they explicitly switch back
```

---

### TODO 5: Use MessageSource in service layer — the right way

```java
@Service
public class OrderEmailService {

    // Inject the MessageSource by name — Spring gives you the "messageSource" bean.
    // OR implement MessageSourceAware and Spring calls setMessageSource() automatically.
    @Autowired
    private MessageSource messageSource;

    // The RIGHT pattern: accept Locale as a parameter.
    // The service doesn't know HOW the locale was determined — it just uses it.
    // This keeps the service testable (pass any Locale in tests)
    // and decoupled from HTTP (usable in batch jobs too).
    public String buildConfirmationEmail(Order order, Locale locale) {

        // getMessage(code, args, locale):
        //   code: the key in your .properties files
        //   args: substituted into {0}, {1}, etc. via MessageFormat
        //   locale: which language file to look in
        // Throws NoSuchMessageException if not found AND no default provided.
        String greeting = messageSource.getMessage(
            "email.greeting",
            new Object[]{order.getCustomerName()},
            locale
        );

        // getMessage(code, args, defaultMessage, locale):
        //   Same but NEVER throws — returns defaultMessage if code not found.
        // Use this when the text is OPTIONAL or you have a safe fallback.
        String footer = messageSource.getMessage(
            "email.footer",
            null,               // no args
            "Thank you for shopping with us.",  // fallback if key missing
            locale
        );

        // Dynamic code construction — powerful but use carefully:
        // If order.getStatus() = OrderStatus.SHIPPED,
        // this looks up "order.status.shipped" in the properties file.
        String statusText = messageSource.getMessage(
            "order.status." + order.getStatus().name().toLowerCase(),
            null,
            order.getStatus().name(),  // fallback = enum name in English
            locale
        );

        return greeting + "\n\n"
             + "Order " + order.getId() + " — Status: " + statusText + "\n\n"
             + footer;
    }

    // For batch processing: each customer gets their preferred locale
    public void sendBulkConfirmations(List<Order> orders) {
        for (Order order : orders) {
            // Each customer may have a different preferred locale
            Locale customerLocale = order.getCustomer().getPreferredLocale();
            // Pass it explicitly — no thread-local magic in batch jobs
            String email = buildConfirmationEmail(order, customerLocale);
            emailSender.send(order.getCustomer().getEmail(), email);
        }
    }
}
```

**The `LocaleContextHolder` alternative — when you CAN'T pass Locale as a parameter:**

```java
// Sometimes you're deep in a call stack and can't add a Locale parameter
// to every method. In a web request, Spring has already stored the locale
// in a ThreadLocal (LocaleContextHolder). You can read it from there.
//
// WARNING: Only works in the web request thread.
// In @Async methods, scheduled tasks, or batch jobs: the ThreadLocal is EMPTY.
// Always reset it in finally blocks if you set it manually.
@Service
public class LegacyOrderService {

    @Autowired
    private MessageSource messageSource;

    public String getStatusText(OrderStatus status) {
        // LocaleContextHolder: a ThreadLocal holder for the current locale.
        // In a web request, Spring MVC sets this automatically.
        // Read it here without needing a Locale parameter.
        Locale locale = LocaleContextHolder.getLocale();

        return messageSource.getMessage(
            "order.status." + status.name().toLowerCase(),
            null,
            status.name(),
            locale
        );
    }
}
```

---

### TODO 6: Use MessageSource in controller — Spring injects Locale automatically

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private MessageSource messageSource;

    @Autowired
    private OrderEmailService orderEmailService;

    // Spring MVC sees "Locale locale" in the method signature and automatically
    // injects the locale resolved by LocaleResolver for this request.
    // You don't annotate it — Spring recognizes Locale as a special injected type.
    @PostMapping
    public ResponseEntity<Map<String, String>> placeOrder(
            @Valid @RequestBody OrderRequest request,
            BindingResult result,
            Locale locale) {               // ← Spring injects this automatically

        if (result.hasErrors()) {
            // More on this in TODO 7
            return ResponseEntity.badRequest().build();
        }

        Order order = orderService.create(request);

        // Build the success message in the user's language:
        String message = messageSource.getMessage(
            "order.placed",
            new Object[]{order.getId()},
            locale
        );

        // German user → "Bestellung ORD-001 wurde erfolgreich aufgegeben."
        // Japanese user → "ご注文ORD-001を受け付けました。"
        return ResponseEntity.ok(Map.of("message", message));
    }

    @GetMapping("/{id}")
    public ResponseEntity<?> getOrder(@PathVariable String id, Locale locale) {
        Order order = orderService.findById(id);

        if (order == null) {
            String error = messageSource.getMessage(
                "product.not.found",
                new Object[]{id},
                locale
            );
            return ResponseEntity.status(404).body(Map.of("error", error));
        }

        return ResponseEntity.ok(order);
    }
}
```

---

### TODO 7: Validation errors — also translated through MessageSource

Validation errors need to speak the same language as the rest of your app. Spring's `FieldError` knows how to look itself up in `MessageSource`.

```properties
# Add to i18n/messages.properties
# Bean Validation constraint codes — override Hibernate Validator's built-in English messages.
# The key format matches what DefaultMessageCodesResolver generates.
NotBlank.orderRequest.customerId=Customer ID cannot be blank
NotNull.orderRequest.amount=Amount is required
FutureOrPresent.orderRequest.orderDate=Order date cannot be in the past
# More generic fallbacks (apply to any field):
NotBlank=This field is required
NotNull=This field is required
Email=Please enter a valid email address
Size=Must be between {2} and {1} characters

# Custom Spring Validator error codes:
order.amount.positive=Amount must be positive, was: {0}
dates.shipBeforeOrder=Ship date cannot be before the order date
```

```properties
# i18n/messages_de.properties
NotBlank.orderRequest.customerId=Kundennummer darf nicht leer sein
NotNull.orderRequest.amount=Betrag ist erforderlich
FutureOrPresent.orderRequest.orderDate=Bestelldatum darf nicht in der Vergangenheit liegen
NotBlank=Dieses Feld ist erforderlich
NotNull=Dieses Feld ist erforderlich
Email=Bitte eine gültige E-Mail-Adresse eingeben
Size=Muss zwischen {2} und {1} Zeichen lang sein

order.amount.positive=Betrag muss positiv sein, war: {0}
dates.shipBeforeOrder=Versanddatum kann nicht vor dem Bestelldatum liegen
```

```java
// FieldError (from BindingResult) implements MessageSourceResolvable.
// MessageSourceResolvable: an interface that says "I know what codes to try
// and in what order — just give me a MessageSource and a Locale."
// FieldError carries:
//   getCodes() → ["NotBlank.orderRequest.customerId", "NotBlank.customerId",
//                 "NotBlank.java.lang.String", "NotBlank"]
//   getArguments() → the constraint parameters (min, max, etc.)
//   getDefaultMessage() → Hibernate Validator's built-in English message

@RestControllerAdvice
public class ValidationExceptionHandler {

    @Autowired
    private MessageSource messageSource;

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, List<String>> handleValidation(
            MethodArgumentNotValidException ex,
            Locale locale) {           // ← current request locale injected here too

        Map<String, List<String>> errors = new LinkedHashMap<>();

        for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {

            // messageSource.getMessage(MessageSourceResolvable, Locale):
            // Spring tries fieldError.getCodes() in order:
            //   1. "NotBlank.orderRequest.customerId" → in messages_de? Yes → "Kundennummer..."
            //   2. (if not found): "NotBlank.customerId" → in messages_de?
            //   3. (if not found): "NotBlank.java.lang.String" → in messages_de?
            //   4. (if not found): "NotBlank" → in messages_de? Yes → "Dieses Feld..."
            //   5. (if nothing in messages_de): try messages.properties fallback
            //   6. (if still nothing): fieldError.getDefaultMessage() (Hibernate's English)
            String message = messageSource.getMessage(fieldError, locale);

            errors.computeIfAbsent(fieldError.getField(), k -> new ArrayList<>())
                  .add(message);
        }

        // Global errors (cross-field, from our Spring Validator)
        for (ObjectError globalError : ex.getBindingResult().getGlobalErrors()) {
            String message = messageSource.getMessage(globalError, locale);
            errors.computeIfAbsent("_global", k -> new ArrayList<>())
                  .add(message);
        }

        return errors;
    }
}
```

**How error code resolution works — the specificity ladder:**

```
errors.rejectValue("customerId", "NotBlank") on object "orderRequest":

Spring generates codes from MOST to LEAST specific:
  1. "NotBlank.orderRequest.customerId"     ← objectName + fieldName (most specific)
  2. "NotBlank.customerId"                  ← fieldName only
  3. "NotBlank.java.lang.String"            ← field type
  4. "NotBlank"                             ← code only (most general)

Spring checks messages_de.properties for each in order.
First one found wins. If none found, falls back to messages.properties.
If still not found, uses the defaultMessage passed to rejectValue().

This means you can be as specific or general as you want:
  - "NotBlank=Dieses Feld ist erforderlich"                ← catches ALL @NotBlank
  - "NotBlank.customerId=Kundennummer ist erforderlich"     ← only for customerId fields
  - "NotBlank.orderRequest.customerId=..."                  ← only for this specific form field
```

---

### TODO 8: Handle missing messages gracefully

```java
// The three getMessage() behaviors — know which one you're calling:

// 1. WITH default message — NEVER throws, NEVER returns null:
String safe = messageSource.getMessage(
    "might.not.exist",
    null,
    "Fallback text if key missing",   // ← returned when key not found
    locale
);
// → "Fallback text if key missing" if key absent

// 2. WITHOUT default message — throws NoSuchMessageException if key not found:
try {
    String strict = messageSource.getMessage(
        "must.exist",
        null,
        locale                          // ← no defaultMessage parameter
    );
} catch (NoSuchMessageException e) {
    // Key was missing from ALL properties files for this locale
    log.error("Missing translation key: {}", e.getMessage());
}

// 3. useCodeAsDefaultMessage = true — NEVER throws (returns the key itself):
// messageSource.setUseCodeAsDefaultMessage(true);
// messageSource.getMessage("missing.key", null, locale)
// → returns "missing.key" (the code itself — visible to users but won't crash)
// Use in development to spot missing keys. Don't use in production.
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: SessionLocaleResolver vs CookieLocaleResolver vs AcceptHeaderLocaleResolver**

```
AcceptHeaderLocaleResolver:
  ✓ Zero configuration — reads browser's Accept-Language header automatically
  ✓ Best for REST APIs and mobile apps (client sends their locale)
  ✗ Read-only: user can't switch language via ?lang= parameter
  ✗ Depends on browser/client correctly sending Accept-Language
  Use when: building APIs consumed by browsers/mobile apps that manage locale

SessionLocaleResolver:
  ✓ Supports locale switching (?lang=de works)
  ✓ Locale persists across all requests in one session
  ✗ Lost on session expiry (user has to switch again after logout)
  ✗ Requires session (stateful — doesn't work in stateless APIs)
  Use when: traditional web apps with user sessions

CookieLocaleResolver:
  ✓ Supports locale switching
  ✓ Persists even after session expiry (cookie lifetime)
  ✗ Requires cookie consent in GDPR regions
  ✗ User's choice is stored on their device (device-specific)
  Use when: public websites where you want locale preference to survive sessions
```

**Decision 2: ResourceBundleMessageSource vs ReloadableResourceBundleMessageSource**

```
Production: ResourceBundleMessageSource
  → JVM-level cache = fastest possible lookup
  → No file system access after initial load
  → Translations don't change at runtime — restart for new translations

Development: ReloadableResourceBundleMessageSource
  → setCacheSeconds(5) = translations reload every 5 seconds
  → Edit translations.properties, see changes without restart
  → MUST use "classpath:" prefix (unlike ResourceBundleMessageSource)
```

**Decision 3: Passing Locale vs using LocaleContextHolder**

```
Prefer: explicit Locale parameter
  public String buildEmail(Order order, Locale locale) { ... }
  ✓ Works in batch jobs, @Async methods, unit tests
  ✓ Intent is clear — no hidden dependency on ThreadLocal
  ✓ Easy to test: pass any Locale

Use LocaleContextHolder only when:
  → Locale can't be threaded through legacy code without major refactoring
  → You're certain the code only runs on web request threads
  → Always check: am I in a new Thread? (If yes: LocaleContextHolder is empty)
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
@SpringBootTest
class MessageSourceTest {

    @Autowired
    private MessageSource messageSource;

    // TRAP 1: Missing UTF-8 encoding — Japanese becomes garbage
    @Test
    void japaneseCharactersMustBeReadable() {
        String msg = messageSource.getMessage("welcome.title", null, Locale.JAPANESE);
        // Without setDefaultEncoding("UTF-8"):
        // → "{0}????????????????" (garbage characters)
        // With UTF-8 set:
        // → "{0}へようこそ" (correct)
        assertThat(msg).contains("ようこそ");
        assertThat(msg).doesNotContain("?"); // garbage characters indicator
    }

    // TRAP 2: setFallbackToSystemLocale(true) + server in unexpected locale
    @Test
    void fallbackBehaviorMustBePredictable() {
        // If the server JVM runs in French (Locale.getDefault() = fr_FR)
        // and a user requests German (de_DE) but messages_de.properties is missing:
        // With fallbackToSystemLocale=true:  falls back to fr_FR → French messages
        // With fallbackToSystemLocale=false: falls back to messages.properties → English
        //
        // A server in France showing French to German users is a bug.
        // Always set fallbackToSystemLocale(false) for predictable behavior.
        String msg = messageSource.getMessage("order.placed",
            new Object[]{"ORD-001"}, new Locale("zh")); // Chinese — no file exists
        // Should fall back to English, not whatever the server's locale is:
        assertThat(msg).isEqualTo("Order ORD-001 has been placed successfully.");
    }

    // TRAP 3: Dynamic code construction with invalid enum names
    @Test
    void dynamicCodesMustHandleMissingKeys() {
        // messageSource.getMessage("order.status." + status.name().toLowerCase(), ...)
        // If someone adds OrderStatus.IN_TRANSIT to the enum but forgets
        // to add "order.status.in_transit" to ALL properties files:
        // → NoSuchMessageException (if no default provided)
        // → silently shows "IN_TRANSIT" to users (if default = status.name())
        //
        // Always provide a defaultMessage when using dynamic codes:
        String status = messageSource.getMessage(
            "order.status.unknown_status",  // doesn't exist
            null,
            "Unknown",                      // safe fallback
            Locale.ENGLISH
        );
        assertThat(status).isEqualTo("Unknown"); // doesn't throw
    }

    // TRAP 4: MessageFormat special characters break substitution
    @Test
    void singleQuotesInMessagesNeedEscaping() {
        // MessageFormat treats single quotes specially: ' starts a literal string
        // "Don't worry" in messages.properties → MessageFormat sees "Dont worry"!
        // The apostrophe swallows the next character.
        //
        // Fix: escape single quotes by doubling them:
        // messages.properties: order.note=Don''t forget your order!
        // Result: "Don't forget your order!"
        //
        // Or: avoid single quotes in messages where possible.
        String msg = messageSource.getMessage("order.note", null, Locale.ENGLISH);
        assertThat(msg).isEqualTo("Don't forget your order!"); // NOT "Dont forget..."
    }

    // TRAP 5: LocaleContextHolder is empty in @Async methods
    @Test
    void asyncMethodsLoseLocaleContext() throws Exception {
        // In a web request: LocaleContextHolder.getLocale() = de_DE (set by Spring MVC)
        // In an @Async method (runs on a different thread):
        //   LocaleContextHolder.getLocale() = Locale.getDefault() (JVM default — WRONG)
        //
        // Fix: capture locale BEFORE the @Async call and pass it as a parameter.
        // NEVER rely on LocaleContextHolder inside @Async or new Thread().
    }

    // TRAP 6: Bean Validation messages use a DIFFERENT properties file by default
    @Test
    void beanValidationUsesValidationMessages() {
        // Hibernate Validator's @NotBlank, @Size etc. look for:
        //   ValidationMessages.properties (NOT messages.properties)
        //
        // To make them use YOUR messages.properties:
        // Register a Validator bean that uses your MessageSource:
        // @Bean
        // public LocalValidatorFactoryBean validator(MessageSource messageSource) {
        //     LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
        //     factory.setValidationMessageSource(messageSource);
        //     return factory;
        // }
        // Without this, @NotBlank shows Hibernate's English message even when
        // your app is in German and messages_de.properties has the German version.
    }
}
```

**The `LocalValidatorFactoryBean` fix for Trap 6 — critical for fully translated validation errors:**

```java
@Configuration
public class I18nConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource ms =
            new ReloadableResourceBundleMessageSource();
        ms.setBasename("classpath:i18n/messages");
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }

    // LocalValidatorFactoryBean: the Spring-managed Bean Validation validator.
    // By default it uses Hibernate Validator's built-in messages (English only).
    // By setting validationMessageSource to YOUR messageSource:
    // → @NotBlank, @Size etc. now look up their messages in YOUR messages_de.properties
    // → German user gets German validation errors from @NotBlank too, not just from
    //   your custom Validators
    @Bean
    public LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
        factory.setValidationMessageSource(messageSource());
        return factory;
    }
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version problem:** every controller and service method takes `Locale locale` as a parameter and calls `messageSource.getMessage(...)` directly. After 20+ endpoints, you have the same pattern repeated everywhere. And writing `messageSource.getMessage("order.placed", new Object[]{id}, locale)` is verbose.

**Senior dev refactoring:** wrap `MessageSource` in a thin, request-aware helper that the whole team can use cleanly:

```java
// MessageHelper: a thin wrapper around MessageSource that removes boilerplate.
// Not a replacement for MessageSource — just a convenience layer.
// Lives in the web layer since it uses LocaleContextHolder.
@Component
public class MessageHelper {

    @Autowired
    private MessageSource messageSource;

    // Simple lookup — uses current request's locale from ThreadLocal.
    // Only call this from web request threads (controllers, interceptors).
    public String get(String code, Object... args) {
        return messageSource.getMessage(
            code,
            args.length > 0 ? args : null,
            LocaleContextHolder.getLocale()
        );
    }

    // Lookup with explicit locale — safe for service layer and batch.
    public String get(String code, Locale locale, Object... args) {
        return messageSource.getMessage(
            code,
            args.length > 0 ? args : null,
            locale
        );
    }

    // Safe lookup with fallback — never throws, never returns null.
    public String getOrDefault(String code, String defaultMessage, Object... args) {
        return messageSource.getMessage(
            code,
            args.length > 0 ? args : null,
            defaultMessage,
            LocaleContextHolder.getLocale()
        );
    }

    // Resolve a validation error to a message.
    // Used in exception handlers to translate FieldError → human string.
    public String resolve(MessageSourceResolvable resolvable) {
        return messageSource.getMessage(resolvable, LocaleContextHolder.getLocale());
    }
}

// BEFORE refactoring — verbose and repetitive:
@RestController
public class OrderController {
    @Autowired private MessageSource messageSource;

    @PostMapping
    public ResponseEntity<?> place(@RequestBody OrderRequest req, Locale locale) {
        String message = messageSource.getMessage(
            "order.placed", new Object[]{req.getId()}, locale);
        return ResponseEntity.ok(Map.of("message", message));
    }
}

// AFTER refactoring — clean and concise:
@RestController
public class OrderController {
    @Autowired private MessageHelper msg;

    @PostMapping
    public ResponseEntity<?> place(@RequestBody OrderRequest req) {
        return ResponseEntity.ok(Map.of(
            "message", msg.get("order.placed", req.getId())
        ));
    }
}
```

**What improved:**

- No `Locale locale` parameter in every controller method (cleaner signatures)
- `msg.get("order.placed", orderId)` is readable like English
- The locale is still resolved correctly (from ThreadLocal set by Spring MVC)
- Service layer still uses explicit `Locale` parameter via `msg.get(code, locale, args)` — safe for batch

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **MessageSource** | The central resolver: given a key + a locale, finds the right translated text in the right file. Like a library's lookup system. |
| **ResourceBundleMessageSource** | MessageSource backed by Java's ResourceBundle. Fastest (JVM cache). No "classpath:" prefix. No hot-reload. Use in production. |
| **ReloadableResourceBundleMessageSource** | MessageSource backed by Spring's file loading. Supports "classpath:" prefix and hot-reload. Use everywhere. |
| **StaticMessageSource** | MessageSource backed by a HashMap you fill in code. Only for tests. |
| **DelegatingMessageSource** | The fallback Spring creates when you forget to name your bean "messageSource". Just returns the code itself or throws. |
| **Locale** | A language+country tag. `Locale.GERMAN` = "de". `new Locale("de", "DE")` = "de_DE" (Germany-specific). |
| **MessageFormat** | Java's template engine for parameterized strings. `{0}` = first arg. `{1,choice,...}` = plural forms. Used inside all .properties values. |
| **LocaleResolver** | The component Spring MVC calls once per request to determine which Locale this user wants. |
| **AcceptHeaderLocaleResolver** | Reads browser's `Accept-Language` header. Read-only — can't be changed by user clicking a language button. |
| **SessionLocaleResolver** | Stores user's chosen locale in their HTTP session. Supports locale switching. |
| **CookieLocaleResolver** | Stores user's chosen locale in a browser cookie. Survives session expiry. |
| **LocaleChangeInterceptor** | Middleware that watches for `?lang=de` in URLs and tells the LocaleResolver to switch. Only works with mutable resolvers (Session/Cookie). |
| **LocaleContextHolder** | A ThreadLocal that holds the current request's Locale. Set by Spring MVC at request start. Empty in @Async threads. |
| **MessageSourceResolvable** | An interface that `FieldError`/`ObjectError` implement. Says "try these codes in this order, with these args, with this default." |
| **HierarchicalMessageSource** | A MessageSource that delegates to a parent when a key isn't found locally. Module-specific → global fallback. |
| **DefaultMessageCodesResolver** | The component that generates multiple error codes from a single `rejectValue("field", "NotBlank")` call — from most specific to least specific. |

---

### The decision flowchart:

```
How does the user's locale reach my code?
  ├── REST API / mobile app sends Accept-Language header
  │     → AcceptHeaderLocaleResolver (read-only, no switching needed)
  ├── Web app, user can switch language with a button
  │     → SessionLocaleResolver + LocaleChangeInterceptor
  └── Web app, preference should survive logout
        → CookieLocaleResolver + LocaleChangeInterceptor

Which MessageSource implementation?
  ├── Production (performance matters, no hot-reload needed)
  │     → ResourceBundleMessageSource (NO "classpath:" prefix)
  └── Development/Default (want hot-reload, Spring Resource support)
        → ReloadableResourceBundleMessageSource (REQUIRES "classpath:" prefix)

How to get the message?
  ├── Key definitely exists (required translation)?
  │     → getMessage(code, args, locale) — throws if missing (catches bugs)
  ├── Key might not exist (optional feature text)?
  │     → getMessage(code, args, defaultMessage, locale) — never throws
  └── Development mode (want to see missing keys visually)?
        → setUseCodeAsDefaultMessage(true) — returns code itself if not found

Where to get the Locale from?
  ├── In a controller method → add Locale locale parameter (Spring injects)
  ├── In service layer → accept Locale as a method parameter (always pass explicitly)
  └── Deep in legacy code (can't add parameters) → LocaleContextHolder.getLocale()
        ⚠️ Only safe on web request threads — empty in @Async/batch
```

### The request lifecycle — where everything fits:

```
HTTP Request: GET /orders/123  (Accept-Language: de-DE)
              or GET /orders/123?lang=de

    LocaleChangeInterceptor fires first:
      → sees ?lang=de → calls sessionResolver.setLocale(request, response, de_DE)

    LocaleResolver reads locale:
      → SessionLocaleResolver reads from session: Locale = de_DE

    LocaleContextHolder stores it:
      → ThreadLocal["locale"] = de_DE

    Controller method runs:
      Locale locale (parameter)    → Spring injects de_DE here
      msg.get("order.placed", id)  → reads de_DE from LocaleContextHolder

    MessageSource resolves:
      code = "order.placed", locale = de_DE
      1. Check i18n/messages_de_DE.properties → not found
      2. Check i18n/messages_de.properties    → FOUND: "Bestellung {0}..."
      3. MessageFormat substitutes {0} → "Bestellung ORD-001..."

    Response: { "message": "Bestellung ORD-001 wurde erfolgreich aufgegeben." }

    Thread finishes → LocaleContextHolder.resetLocaleContext()
```
