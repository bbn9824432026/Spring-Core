# TOPIC 6.3 — Type Conversion and Validation

---

## ① CONCEPTUAL EXPLANATION

### What is Type Conversion in Spring?

Type conversion is the mechanism by which Spring transforms values from one type to another during data binding, `@Value` injection, SpEL evaluation, and web request parameter binding. Spring has TWO parallel conversion systems:

```
System 1 (Legacy): PropertyEditor (java.beans.PropertyEditor)
    → Java Beans specification
    → Stateful — NOT thread-safe
    → String ↔ Object only
    → Still used internally in some places

System 2 (Modern): ConversionService (Spring 3.0+)
    → Spring's own abstraction
    → Stateless — thread-safe
    → Any type ↔ Any type (not just String)
    → Composable, extensible
    → Preferred for all new code
```

---

### 6.3.1 — PropertyEditor — The Legacy System

`PropertyEditor` is from the `java.beans` package. Spring uses it extensively internally but it has serious limitations:

```java
public interface PropertyEditor {
    void setValue(Object value);
    Object getValue();
    void setAsText(String text) throws IllegalArgumentException;
    String getAsText();
    // ... other methods
}
```

**Spring's built-in `PropertyEditor` implementations:**

```
ClassEditor          → String → Class<?>
FileEditor           → String → java.io.File
InputStreamEditor    → String → InputStream
LocaleEditor         → String → java.util.Locale
PathEditor           → String → java.nio.file.Path
PatternEditor        → String → java.util.regex.Pattern
PropertiesEditor     → String → java.util.Properties
ResourceEditor       → String → Resource
TimeZoneEditor       → String → java.util.TimeZone
URIEditor            → String → java.net.URI
URLEditor            → String → java.net.URL
UUIDEditor           → String → java.util.UUID
CharsetEditor        → String → java.nio.charset.Charset
CurrencyEditor       → String → java.util.Currency
```

**Custom `PropertyEditor`:**

```java
// Convention: for type Foo, create FooEditor
public class MoneyEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        // "USD 100.50" → Money object
        String[] parts = text.split(" ");
        if (parts.length != 2) throw new IllegalArgumentException("Invalid money: " + text);
        Currency currency = Currency.getInstance(parts[0]);
        BigDecimal amount = new BigDecimal(parts[1]);
        setValue(new Money(currency, amount));
    }

    @Override
    public String getAsText() {
        Money money = (Money) getValue();
        return money.getCurrency().getCurrencyCode() + " " + money.getAmount();
    }
}
```

**Registering custom `PropertyEditor` — three mechanisms:**

```java
// Mechanism 1: CustomEditorConfigurer (BFPP — most common for global registration)
@Bean
public static CustomEditorConfigurer customEditorConfigurer() {
    CustomEditorConfigurer cec = new CustomEditorConfigurer();
    Map<Class<?>, Class<? extends PropertyEditor>> editors = new HashMap<>();
    editors.put(Money.class, MoneyEditor.class);
    cec.setCustomEditors(editors);
    return cec; // MUST be static — it's a BFPP
}

// Mechanism 2: @InitBinder (Controller scope — Spring MVC)
@Controller
public class PaymentController {
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(Money.class, new MoneyEditor());
    }
}

// Mechanism 3: WebBindingInitializer (global Spring MVC)
public class GlobalBindingInitializer implements WebBindingInitializer {
    @Override
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(Money.class, new MoneyEditor());
    }
}
```

**Why `PropertyEditor` is limited:**

```
1. NOT thread-safe — holds state (current value) as instance variable
   → Spring creates a NEW instance per conversion (expensive)
   → Cannot be registered as a singleton — must be a fresh instance each time

2. String ↔ Object ONLY — cannot convert Object → Object directly
   → e.g., cannot convert LocalDate → String in a type-safe way

3. No generic support — requires casting
4. No coercion/widening — no automatic type widening
```

---

### 6.3.2 — ConversionService — The Modern System

`ConversionService` is Spring's own type conversion framework:

```java
public interface ConversionService {

    // Check if conversion is possible
    boolean canConvert(Class<?> sourceType, Class<?> targetType);
    boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

    // Perform the conversion
    <T> T convert(Object source, Class<T> targetType);
    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

**`ConversionService` implementations:**

```
DefaultConversionService
    → All built-in converters registered
    → Handles primitives, wrappers, enums, String conversions

FormattingConversionService
    → Extends DefaultConversionService
    → Adds @NumberFormat, @DateTimeFormat support
    → Printing + Parsing support (Formatter API)

DefaultFormattingConversionService
    → Standard implementation used by Spring MVC and Boot
    → All built-in + formatting converters

ApplicationConversionService
    → Spring Boot's version
    → Additional Spring Boot-specific converters
```

---

### 6.3.3 — Converter — The Core Extension Point

```java
// Converter<S, T>: converts from Source type S to Target type T
@FunctionalInterface
public interface Converter<S, T> {
    T convert(S source);
}
```

**Thread-safe and stateless** — unlike `PropertyEditor`, `Converter` implementations hold no state.

```java
// Example: String → Money
public class StringToMoneyConverter implements Converter<String, Money> {

    @Override
    public Money convert(String source) {
        // "USD 100.50"
        String[] parts = source.split(" ");
        if (parts.length != 2) {
            throw new IllegalArgumentException("Invalid money format: " + source);
        }
        return new Money(
            Currency.getInstance(parts[0]),
            new BigDecimal(parts[1])
        );
    }
}

// Example: Money → String
public class MoneyToStringConverter implements Converter<Money, String> {
    @Override
    public String convert(Money source) {
        return source.getCurrency().getCurrencyCode() + " " + source.getAmount();
    }
}

// Registration
@Configuration
public class ConversionConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultConversionService service = new DefaultConversionService();
        service.addConverter(new StringToMoneyConverter());
        service.addConverter(new MoneyToStringConverter());
        return service;
    }
}
```

---

### 6.3.4 — ConverterFactory — Hierarchical Conversion

`ConverterFactory<S, R>` creates converters for a family of related target types:

```java
public interface ConverterFactory<S, R> {
    <T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

**Classic use case: String → Enum (any enum type)**

```java
// Converts String to ANY Enum subtype
public class StringToEnumConverterFactory
        implements ConverterFactory<String, Enum> {

    @Override
    @SuppressWarnings("unchecked")
    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
        return source -> (T) Enum.valueOf(targetType, source.trim().toUpperCase());
    }
}

// Usage:
// String "ACTIVE" → UserStatus.ACTIVE (works for ANY enum)
// String "PENDING" → OrderStatus.PENDING (same factory, different target)
```

**Another use case: Number → Number widening/narrowing**

```java
public class NumberToNumberConverterFactory
        implements ConverterFactory<Number, Number> {

    @Override
    public <T extends Number> Converter<Number, T> getConverter(Class<T> targetType) {
        return source -> NumberUtils.convertNumberToTargetClass(source, targetType);
    }
}
// Integer → Long, Double → Integer, etc.
```

---

### 6.3.5 — GenericConverter — Most Flexible

`GenericConverter` works with complex type pairs and has access to full type descriptor information:

```java
public interface GenericConverter {

    // Which source-target type pairs this converter handles
    Set<ConvertiblePair> getConvertibleTypes();

    // Perform conversion — has access to full TypeDescriptor
    // (including generics info: List<String> vs List<Integer>)
    Object convert(Object source,
                   TypeDescriptor sourceType,
                   TypeDescriptor targetType);
}
```

**Use case: Collection to Collection conversion**

```java
// Converts List<S> → Set<T> using an underlying S→T converter
public class ListToSetConverter implements GenericConverter {

    private final ConversionService conversionService;

    @Override
    public Set<ConvertiblePair> getConvertibleTypes() {
        return Set.of(new ConvertiblePair(List.class, Set.class));
    }

    @Override
    public Object convert(Object source,
                           TypeDescriptor sourceType,
                           TypeDescriptor targetType) {
        if (source == null) return null;
        List<?> list = (List<?>) source;

        TypeDescriptor elementSourceType = sourceType.getElementTypeDescriptor();
        TypeDescriptor elementTargetType = targetType.getElementTypeDescriptor();

        Set<Object> result = new LinkedHashSet<>(list.size());
        for (Object element : list) {
            Object converted = conversionService.convert(element,
                elementSourceType, elementTargetType);
            result.add(converted);
        }
        return result;
        }
}
```

**`ConditionalConverter` — add matching condition:**

```java
// ConditionalConverter: converter only applies when condition matches
public class SmartStringToDateConverter
        implements Converter<String, LocalDate>, ConditionalConverter {

    @Override
    public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
        // Only convert if target has @DateFormat annotation
        return targetType.hasAnnotation(DateFormat.class);
    }

    @Override
    public LocalDate convert(String source) {
        DateFormat annotation = /* get from TypeDescriptor */;
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern(annotation.pattern());
        return LocalDate.parse(source, fmt);
    }
}
```

---

### 6.3.6 — Formatter — Locale-Aware Conversion

`Formatter<T>` is an extension for locale-aware printing and parsing:

```java
public interface Formatter<T> extends Printer<T>, Parser<T> {
    // from Printer:
    String print(T object, Locale locale);
    // from Parser:
    T parse(String text, Locale locale) throws ParseException;
}
```

**Purpose:** Locale-sensitive String ↔ Type conversion. Used for displaying and parsing dates, numbers, currencies in a locale-appropriate way.

```java
// Custom Formatter for Money
public class MoneyFormatter implements Formatter<Money> {

    @Override
    public String print(Money money, Locale locale) {
        // Format for the given locale
        NumberFormat nf = NumberFormat.getCurrencyInstance(locale);
        nf.setCurrency(money.getCurrency());
        return nf.format(money.getAmount());
    }

    @Override
    public Money parse(String text, Locale locale) throws ParseException {
        // Parse "$100.50" or "€100,50" depending on locale
        NumberFormat nf = NumberFormat.getCurrencyInstance(locale);
        Number amount = nf.parse(text);
        Currency currency = nf.getCurrency();
        return new Money(currency, new BigDecimal(amount.toString()));
    }
}

// Registration:
@Configuration
public class FormatConfig {

    @Bean
    public FormattingConversionService conversionService() {
        DefaultFormattingConversionService service =
            new DefaultFormattingConversionService();
        service.addFormatter(new MoneyFormatter());
        return service;
    }
}
```

---

### 6.3.7 — @NumberFormat and @DateTimeFormat

Spring provides annotations for declarative formatting:

```java
public class OrderForm {

    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
    // iso = "yyyy-MM-dd" — parses "2024-01-15"
    private LocalDate orderDate;

    @DateTimeFormat(pattern = "dd/MM/yyyy HH:mm")
    // Custom pattern
    private LocalDateTime deliveryTime;

    @NumberFormat(style = NumberFormat.Style.CURRENCY)
    // "$1,234.56" in en-US locale
    private BigDecimal totalAmount;

    @NumberFormat(pattern = "#,##0.00")
    // "1,234.56" regardless of locale
    private Double discount;

    @NumberFormat(style = NumberFormat.Style.PERCENT)
    // "75%" → 0.75
    private Double taxRate;
}
```

**How `@DateTimeFormat` works internally:**

`FormattingConversionService` registers `DateTimeFormatterRegistrar` which processes `@DateTimeFormat` annotations. When converting, `AnnotationFormatterFactory` implementations detect the annotation and create locale-aware formatters.

```java
public interface AnnotationFormatterFactory<A extends Annotation> {
    Set<Class<?>> getFieldTypes();  // which field types this handles
    Printer<?> getPrinter(A annotation, Class<?> fieldType);
    Parser<?> getParser(A annotation, Class<?> fieldType);
}
```

---

### 6.3.8 — ConversionService Registration

```java
// Option 1: Bean named "conversionService" — Spring auto-detects
@Bean
public ConversionService conversionService() {
    DefaultFormattingConversionService service =
        new DefaultFormattingConversionService();
    service.addConverter(new StringToMoneyConverter());
    service.addFormatter(new MoneyFormatter());
    return service;
}

// Option 2: Spring MVC WebMvcConfigurer
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToMoneyConverter());
        registry.addFormatter(new MoneyFormatter());
        registry.addConverterFactory(new StringToEnumConverterFactory());
    }
}

// Option 3: Spring Boot — automatically configures DefaultFormattingConversionService
// Just register converters/formatters as @Bean and Spring Boot picks them up:
@Bean
public Converter<String, Money> stringToMoneyConverter() {
    return new StringToMoneyConverter();
}
```

---

### 6.3.9 — Spring Validation — The Validator Interface

Spring has its own `Validator` interface (separate from Bean Validation / JSR-380):

```java
public interface Validator {

    // Does this validator support validating objects of the given class?
    boolean supports(Class<?> clazz);

    // Validate the given object and record errors in the Errors object
    void validate(Object target, Errors errors);
}
```

**Errors interface:**

```java
public interface Errors {
    // Record field-level error
    void rejectValue(String field, String errorCode);
    void rejectValue(String field, String errorCode, String defaultMessage);
    void rejectValue(String field, String errorCode, Object[] errorArgs, String defaultMessage);

    // Record object-level (global) error
    void reject(String errorCode);
    void reject(String errorCode, String defaultMessage);

    // Check errors
    boolean hasErrors();
    boolean hasFieldErrors(String field);
    List<FieldError> getFieldErrors();
    List<ObjectError> getGlobalErrors();
    int getErrorCount();
}
```

**Custom Validator implementation:**

```java
@Component
public class OrderValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return Order.class.isAssignableFrom(clazz);
        // true for Order and all its subclasses
    }

    @Override
    public void validate(Object target, Errors errors) {
        Order order = (Order) target;

        // Field validation
        if (order.getCustomerId() == null || order.getCustomerId().isBlank()) {
            errors.rejectValue("customerId", "field.required",
                "Customer ID is required");
        }

        if (order.getItems() == null || order.getItems().isEmpty()) {
            errors.rejectValue("items", "items.empty",
                "Order must have at least one item");
        }

        if (order.getTotalAmount() != null &&
            order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            errors.rejectValue("totalAmount", "amount.positive",
                new Object[]{order.getTotalAmount()},
                "Total amount must be positive, was: {0}");
        }

        // Cross-field validation (global error)
        if (order.getShipDate() != null && order.getOrderDate() != null &&
            order.getShipDate().isBefore(order.getOrderDate())) {
            errors.reject("dates.invalid",
                "Ship date cannot be before order date");
        }

        // Nested object validation
        if (order.getShippingAddress() != null) {
            errors.pushNestedPath("shippingAddress");
            ValidationUtils.invokeValidator(
                new AddressValidator(), order.getShippingAddress(), errors);
            errors.popNestedPath();
        }
    }
}
```

**`ValidationUtils` — helper methods:**

```java
// Rejects if field value is null or empty (blank string)
ValidationUtils.rejectIfEmpty(errors, "name", "name.required");
ValidationUtils.rejectIfEmptyOrWhitespace(errors, "name", "name.required");

// Invoke another validator (for nested objects)
ValidationUtils.invokeValidator(addressValidator, address, errors);
```

---

### 6.3.10 — Bean Validation (JSR-380 / Jakarta Validation)

Spring integrates with Bean Validation (Hibernate Validator is the reference implementation):

```java
// Standard Jakarta Validation annotations:
public class UserRegistrationForm {

    @NotNull
    @Size(min = 3, max = 50)
    @Pattern(regexp = "[a-zA-Z0-9]+")
    private String username;

    @NotNull
    @Email
    private String email;

    @NotNull
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @NotNull
    @Past  // must be in the past
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
    private LocalDate birthDate;

    @Min(0) @Max(150)
    private int age;

    @NotEmpty  // not null AND not empty collection
    private List<String> roles;

    @Valid  // cascade validation to nested object
    @NotNull
    private Address address;
}

public class Address {
    @NotBlank
    private String street;

    @NotBlank
    @Size(min = 2, max = 10)
    private String postalCode;

    @NotBlank
    private String country;
}
```

**Triggering Bean Validation:**

```java
// Option 1: @Valid or @Validated in controller
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<User> createUser(
            @Valid @RequestBody UserRegistrationForm form,
            BindingResult result) {  // BindingResult catches validation errors
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().build();
        }
        return ResponseEntity.ok(userService.create(form));
    }
}

// Option 2: @Validated on service method (Spring AOP)
@Service
@Validated  // enables method-level validation via AOP
public class OrderService {

    public Order placeOrder(
            @Valid @NotNull Order order,        // validate parameter
            @NotNull @Size(min=1) String userId) {
        // ...
    }
}

// Option 3: Programmatic validation
@Component
public class ManualValidator {

    @Autowired
    private javax.validation.Validator beanValidator;

    public void validate(Object object) {
        Set<ConstraintViolation<Object>> violations = beanValidator.validate(object);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
}
```

---

### 6.3.11 — Custom Bean Validation Constraint

```java
// Step 1: Define annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = CurrencyCodeValidator.class)
@Documented
public @interface ValidCurrencyCode {
    String message() default "Invalid currency code: ${validatedValue}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Step 2: Implement ConstraintValidator
public class CurrencyCodeValidator
        implements ConstraintValidator<ValidCurrencyCode, String> {

    private static final Set<String> VALID_CURRENCIES =
        Currency.getAvailableCurrencies()
            .stream()
            .map(Currency::getCurrencyCode)
            .collect(Collectors.toSet());

    @Override
    public void initialize(ValidCurrencyCode annotation) {
        // Called once — access annotation attributes here
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;  // @NotNull handles null check
        boolean valid = VALID_CURRENCIES.contains(value.toUpperCase());
        if (!valid) {
            // Custom violation message with value
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "'" + value + "' is not a valid ISO 4217 currency code")
                .addConstraintViolation();
        }
        return valid;
    }
}

// Step 3: Usage
public class PaymentRequest {
    @ValidCurrencyCode
    private String currency;  // "USD" ✓, "XYZ" ✗
}
```

---

### 6.3.12 — DataBinder — The Bridge Between Conversion and Validation

`DataBinder` is the central component that binds request parameters to objects AND performs validation:

```java
// Programmatic DataBinder usage:
Order order = new Order();
DataBinder binder = new DataBinder(order, "order");

// Register validators
binder.addValidators(new OrderValidator());
// OR
binder.setValidator(new OrderValidator());

// Bind properties from a Map
MutablePropertyValues pvs = new MutablePropertyValues();
pvs.add("customerId", "CUST-001");
pvs.add("totalAmount", "100.50");
pvs.add("status", "PENDING");
binder.bind(pvs);

// Validate
binder.validate();

// Check results
BindingResult result = binder.getBindingResult();
if (result.hasErrors()) {
    for (FieldError error : result.getFieldErrors()) {
        System.out.println(error.getField() + ": " + error.getDefaultMessage());
    }
}
```

**`BindingResult` — extended `Errors`:**

```java
// BindingResult extends Errors
public interface BindingResult extends Errors {
    Object getTarget();           // the bound object
    Map<String, Object> getModel(); // model map for Spring MVC
    Object getRawFieldValue(String field); // before type conversion
    PropertyEditor findEditor(String field, Class<?> valueType);
    PropertyEditorRegistry getPropertyEditorRegistry();
    String[] resolveMessageCodes(String errorCode, String field);
    void addError(ObjectError error);
}
```

---

### 6.3.13 — Validation Groups

Bean Validation supports validation groups — subsets of constraints applied in specific scenarios:

```java
// Define group marker interfaces (empty — just markers)
public interface OnCreate {}
public interface OnUpdate {}

public class UserForm {

    @NotNull(groups = OnCreate.class)  // required only when creating
    @Null(groups = OnUpdate.class)     // must be null when updating
    private String id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})  // always required
    private String username;

    @Email(groups = {OnCreate.class, OnUpdate.class})
    private String email;

    @NotBlank(groups = OnCreate.class)  // required only on create
    private String initialPassword;
}

// Use groups in controller:
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<User> create(
            @Validated(OnCreate.class) @RequestBody UserForm form,
            BindingResult result) {
        // Only OnCreate group constraints validated
        if (result.hasErrors()) return ResponseEntity.badRequest().build();
        return ResponseEntity.ok(userService.create(form));
    }

    @PutMapping("/users/{id}")
    public ResponseEntity<User> update(
            @PathVariable String id,
            @Validated(OnUpdate.class) @RequestBody UserForm form,
            BindingResult result) {
        // Only OnUpdate group constraints validated
        if (result.hasErrors()) return ResponseEntity.badRequest().build();
        return ResponseEntity.ok(userService.update(id, form));
    }
}
```

---

### 6.3.14 — @Validated vs @Valid

```
@Valid (javax.validation / jakarta.validation):
    → Standard Bean Validation annotation
    → Triggers Bean Validation on the annotated parameter/field
    → Supports cascaded validation (@Valid on nested fields)
    → Does NOT support validation groups in method parameters
    → Can be used anywhere

@Validated (org.springframework.validation.annotation):
    → Spring's own annotation
    → EXTENDS @Valid with GROUP support
    → @Validated(OnCreate.class) → runs only OnCreate group
    → When on CLASS: enables method-level validation via AOP
    → When on METHOD PARAMETER: same as @Valid + groups
```

---

### 6.3.15 — MessageSource and Error Message Internationalization

Validation errors use message codes that are resolved against a `MessageSource`:

```java
// Error code resolution strategy (DefaultMessageCodesResolver):
// For rejectValue("email", "email.invalid"):
//   1. "email.invalid.userForm.email"  (code.objectName.fieldName)
//   2. "email.invalid.email"           (code.fieldName)
//   3. "email.invalid.java.lang.String" (code.fieldType)
//   4. "email.invalid"                 (code only — fallback)

// messages.properties:
// email.invalid.userForm.email=The email {0} in registration form is invalid
// email.invalid={0} is not a valid email address
// field.required=The field {0} is required
// items.empty=Order must contain at least one item
```

---

### Common Misconceptions

**Misconception 1:** "`PropertyEditor` and `Converter` are interchangeable."
Wrong. `PropertyEditor` is stateful, String-only, NOT thread-safe. `Converter` is stateless, any-type-to-any-type, thread-safe. For new code always use `Converter`. `PropertyEditor` is legacy.

**Misconception 2:** "`ConversionService` replaces `PropertyEditor` completely."
Wrong in practice. `PropertyEditor` is still used in some internal Spring places (XML bean definitions, some binding scenarios). Both coexist. `ConversionService` is preferred but `PropertyEditor` is still active in the container.

**Misconception 3:** "`@Valid` and `@Validated` are equivalent."
Wrong. `@Valid` is standard Bean Validation — no group support in method parameters. `@Validated` is Spring's extension — adds group support. `@Validated` on a class enables method-level validation via AOP.

**Misconception 4:** "Spring's `Validator` interface and Bean Validation are the same."
Wrong. Spring's `Validator` (the `org.springframework.validation` package) is Spring's own validation contract. Bean Validation is the Jakarta/JSR-380 standard. They are separate systems. Spring integrates both via `SmartValidator` and `SpringValidatorAdapter` which bridges between them.

**Misconception 5:** "`BindingResult` must be declared right after the validated parameter."
True — this is a strict rule: `BindingResult` (or `Errors`) MUST immediately follow the `@Valid`/`@Validated` parameter it captures. If not adjacent, Spring throws an exception because it cannot determine which binding result corresponds to which parameter.

**Misconception 6:** "`ConversionService` converts between types automatically everywhere in Spring."
Wrong. `ConversionService` is used explicitly when registered. Specific parts of Spring use it: `DataBinder`, SpEL evaluation, `@Value` injection (via `TypeConverter`). Direct field access or other mechanisms may not use it.

**Misconception 7:** "`Formatter` is the same as `Converter`."
Wrong. `Formatter<T>` converts String ↔ T with LOCALE awareness. `Converter<S, T>` converts any S → T without locale. `Formatter` is specifically for locale-sensitive presentation (dates, numbers, currencies). Both can be added to `FormattingConversionService`.

---

## ② CODE EXAMPLES

### Example 1 — Complete ConversionService Setup

```java
// Custom types
public record Money(Currency currency, BigDecimal amount) {}
public record Distance(double value, String unit) {}

// Converters
public class StringToMoneyConverter implements Converter<String, Money> {
    @Override
    public Money convert(String source) {
        String[] parts = source.trim().split("\\s+");
        if (parts.length != 2) throw new IllegalArgumentException("Format: 'USD 100.50'");
        return new Money(Currency.getInstance(parts[0]), new BigDecimal(parts[1]));
    }
}

public class MoneyToStringConverter implements Converter<Money, String> {
    @Override
    public String convert(Money source) {
        return source.currency().getCurrencyCode() + " " + source.amount().toPlainString();
    }
}

public class StringToDistanceConverter implements Converter<String, Distance> {
    private static final Pattern PATTERN = Pattern.compile("(\\d+(?:\\.\\d+)?)(km|m|mi)");

    @Override
    public Distance convert(String source) {
        Matcher m = PATTERN.matcher(source.trim());
        if (!m.matches()) throw new IllegalArgumentException("Format: '100km', '5.5mi'");
        return new Distance(Double.parseDouble(m.group(1)), m.group(2));
    }
}

// Registration
@Configuration
public class ConversionConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService service =
            new DefaultFormattingConversionService();

        // Add custom converters
        service.addConverter(new StringToMoneyConverter());
        service.addConverter(new MoneyToStringConverter());
        service.addConverter(new StringToDistanceConverter());

        return service;
    }
}

// Usage in service
@Service
public class PricingService {

    @Autowired
    private ConversionService conversionService;

    public void process() {
        // Type-safe conversions
        Money price = conversionService.convert("USD 99.99", Money.class);
        String formatted = conversionService.convert(price, String.class);
        Distance dist = conversionService.convert("150km", Distance.class);

        System.out.println("Price: " + price);       // Money[currency=USD, amount=99.99]
        System.out.println("Formatted: " + formatted); // USD 99.99
        System.out.println("Distance: " + dist);       // Distance[value=150.0, unit=km]
    }
}
```

---

### Example 2 — ConverterFactory for Enum Family

```java
// A generic StringToEnum converter factory
@Component
public class StringToEnumConverterFactory implements ConverterFactory<String, Enum<?>> {

    private final Map<Class<?>, Converter<String, ?>> cache = new ConcurrentHashMap<>();

    @Override
    @SuppressWarnings("unchecked")
    public <T extends Enum<?>> Converter<String, T> getConverter(Class<T> targetType) {
        return (Converter<String, T>) cache.computeIfAbsent(targetType,
            clazz -> source -> {
                String normalized = source.trim().toUpperCase().replace("-", "_");
                try {
                    return (T) Enum.valueOf((Class<Enum>) clazz, normalized);
                } catch (IllegalArgumentException e) {
                    // Try case-insensitive match
                    for (T constant : targetType.getEnumConstants()) {
                        if (constant.name().equalsIgnoreCase(normalized)) {
                            return constant;
                        }
                    }
                    throw new IllegalArgumentException(
                        "No enum constant " + clazz.getSimpleName() + "." + source);
                }
            });
    }
}

// Works for ANY enum:
enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
enum PaymentStatus { PENDING, PROCESSED, FAILED, REFUNDED }

// "pending" → OrderStatus.PENDING
// "SHIPPED"  → OrderStatus.SHIPPED
// "FAILED"   → PaymentStatus.FAILED
```

---

### Example 3 — Complete Spring Validator

```java
// Domain object
public class Product {
    private String sku;
    private String name;
    private BigDecimal price;
    private int stockCount;
    private String category;
    private List<String> tags;
    private ProductDimensions dimensions;
    // getters/setters
}

// Nested object
public class ProductDimensions {
    private double length;
    private double width;
    private double height;
    private String unit; // "cm", "in"
    // getters/setters
}

// Validators
@Component
public class ProductDimensionsValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return ProductDimensions.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        ProductDimensions dim = (ProductDimensions) target;

        if (dim.getLength() <= 0) {
            errors.rejectValue("length", "positive.required",
                new Object[]{"length"}, "Length must be positive");
        }
        if (dim.getWidth() <= 0) {
            errors.rejectValue("width", "positive.required",
                new Object[]{"width"}, "Width must be positive");
        }
        if (dim.getHeight() <= 0) {
            errors.rejectValue("height", "positive.required",
                new Object[]{"height"}, "Height must be positive");
        }
        if (!Set.of("cm", "in", "mm").contains(dim.getUnit())) {
            errors.rejectValue("unit", "unit.invalid",
                new Object[]{dim.getUnit()},
                "Unit must be one of: cm, in, mm");
        }
    }
}

@Component
public class ProductValidator implements Validator {

    @Autowired
    private ProductDimensionsValidator dimensionsValidator;

    @Autowired
    private SkuFormatService skuFormatService;

    @Override
    public boolean supports(Class<?> clazz) {
        return Product.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Product product = (Product) target;

        // Required fields
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "sku", "field.required",
            new Object[]{"SKU"}, "SKU is required");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "name", "field.required",
            new Object[]{"name"}, "Name is required");

        // SKU format validation
        if (product.getSku() != null &&
            !skuFormatService.isValidFormat(product.getSku())) {
            errors.rejectValue("sku", "sku.invalid",
                new Object[]{product.getSku()},
                "SKU format invalid: must be ABC-12345");
        }

        // Price validation
        if (product.getPrice() != null) {
            if (product.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
                errors.rejectValue("price", "price.positive",
                    "Price must be positive");
            }
            if (product.getPrice().scale() > 2) {
                errors.rejectValue("price", "price.scale",
                    "Price must have at most 2 decimal places");
            }
        } else {
            errors.rejectValue("price", "field.required",
                new Object[]{"price"}, "Price is required");
        }

        // Stock count
        if (product.getStockCount() < 0) {
            errors.rejectValue("stockCount", "stock.negative",
                "Stock count cannot be negative");
        }

        // Tags validation
        if (product.getTags() != null) {
            for (int i = 0; i < product.getTags().size(); i++) {
                String tag = product.getTags().get(i);
                if (tag == null || tag.isBlank()) {
                    errors.rejectValue("tags[" + i + "]", "tag.empty",
                        "Tag at index " + i + " cannot be empty");
                }
                if (tag != null && tag.length() > 30) {
                    errors.rejectValue("tags[" + i + "]", "tag.tooLong",
                        new Object[]{30}, "Tag must be at most {0} characters");
                }
            }
        }

        // Nested object validation
        if (product.getDimensions() != null) {
            errors.pushNestedPath("dimensions");
            ValidationUtils.invokeValidator(
                dimensionsValidator, product.getDimensions(), errors);
            errors.popNestedPath();
        }
    }
}
```

---

### Example 4 — Bean Validation with Custom Constraint

```java
// Cross-field validation constraint
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
@Documented
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String password();
    String confirmPassword();
}

@Component
public class PasswordMatchValidator
        implements ConstraintValidator<PasswordMatch, Object> {

    private String passwordField;
    private String confirmPasswordField;

    @Override
    public void initialize(PasswordMatch annotation) {
        this.passwordField = annotation.password();
        this.confirmPasswordField = annotation.confirmPassword();
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        try {
            Object password = BeanUtils.getPropertyDescriptor(
                value.getClass(), passwordField)
                .getReadMethod().invoke(value);
            Object confirm = BeanUtils.getPropertyDescriptor(
                value.getClass(), confirmPasswordField)
                .getReadMethod().invoke(value);

            boolean valid = Objects.equals(password, confirm);
            if (!valid) {
                context.disableDefaultConstraintViolation();
                context
                    .buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                    .addPropertyNode(confirmPasswordField)
                    .addConstraintViolation();
            }
            return valid;
        } catch (Exception e) {
            return false;
        }
    }
}

// Usage on class:
@PasswordMatch(password = "password", confirmPassword = "confirmPassword")
public class RegistrationForm {
    @NotBlank
    @Size(min = 8)
    private String password;

    @NotBlank
    private String confirmPassword;

    @NotBlank
    @Email
    private String email;
}
```

---

### Example 5 — Formatter with Locale Support

```java
// Locale-aware Money formatter
@Component
public class MoneyFormatter implements Formatter<Money> {

    @Override
    public Money parse(String text, Locale locale) throws ParseException {
        // Parse locale-specific format: "$1,234.56" (en-US) or "1.234,56 €" (de-DE)
        NumberFormat format = NumberFormat.getCurrencyInstance(locale);
        ParsePosition pos = new ParsePosition(0);
        Number number = format.parse(text, pos);
        if (pos.getIndex() == 0 || pos.getIndex() < text.length()) {
            throw new ParseException("Cannot parse '" + text + "' as Money for locale " + locale, 0);
        }
        Currency currency = ((DecimalFormat) format).getCurrency();
        return new Money(currency, new BigDecimal(number.toString()));
    }

    @Override
    public String print(Money money, Locale locale) {
        NumberFormat format = NumberFormat.getCurrencyInstance(locale);
        format.setCurrency(money.currency());
        return format.format(money.amount());
        // en-US → "$1,234.56"
        // de-DE → "1.234,56 €"
        // ja-JP → "¥1,235"
    }
}

// Registration and usage:
@Configuration
public class FormatterConfig {

    @Bean
    public FormattingConversionService conversionService() {
        DefaultFormattingConversionService service =
            new DefaultFormattingConversionService();
        service.addFormatter(new MoneyFormatter());
        return service;
    }
}

@Controller
public class PriceController {

    @GetMapping("/price")
    @ResponseBody
    public String showPrice(Locale locale) {
        Money price = new Money(Currency.getInstance("USD"), new BigDecimal("1234.56"));
        // Uses MoneyFormatter.print() with current request locale
        return conversionService.convert(price, String.class);
    }
}
```

---

### Example 6 — DataBinder Programmatic Validation

```java
@Service
public class OrderProcessingService {

    @Autowired
    private OrderValidator orderValidator;

    @Autowired
    private ConversionService conversionService;

    public ProcessResult processOrder(Map<String, String> formData) {
        // Create target object
        Order order = new Order();

        // Create DataBinder
        DataBinder binder = new DataBinder(order, "order");
        binder.setConversionService(conversionService);
        binder.addValidators(orderValidator);

        // Prevent binding sensitive fields
        binder.setAllowedFields("customerId", "items", "shippingAddress",
            "totalAmount", "orderDate");
        // OR: binder.setDisallowedFields("id", "version", "createdAt");

        // Bind form data
        MutablePropertyValues pvs = new MutablePropertyValues(formData);
        binder.bind(pvs);

        // Validate
        binder.validate();

        BindingResult result = binder.getBindingResult();
        if (result.hasErrors()) {
            List<String> errorMessages = result.getAllErrors()
                .stream()
                .map(e -> e instanceof FieldError fe
                    ? fe.getField() + ": " + fe.getDefaultMessage()
                    : e.getDefaultMessage())
                .collect(Collectors.toList());
            return ProcessResult.failure(errorMessages);
        }

        return ProcessResult.success(saveOrder(order));
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which of the following is the MAIN limitation of `PropertyEditor` compared to `Converter`?

A) `PropertyEditor` only works with String input
B) `PropertyEditor` is not thread-safe because it is stateful
C) `PropertyEditor` cannot convert primitives
D) Both A and B

**Answer: D**
`PropertyEditor` has TWO critical limitations: (1) it ONLY converts from String (not Object to Object), and (2) it is STATEFUL (holds current value as instance state) making it NOT thread-safe. Spring must create new instances per use. `Converter` is stateless (thread-safe) and works with any source/target types.

---

**Q2 — Select all that apply:**
Which statements about `ConversionService` are TRUE? (Select ALL that apply)

A) A bean named `"conversionService"` is automatically detected by Spring
B) `DefaultConversionService` supports locale-aware formatting
C) `FormattingConversionService` extends `ConversionService` with `Formatter` support
D) `Converter<S,T>` is thread-safe and can be registered as a singleton
E) `ConversionService` replaces `PropertyEditor` completely in Spring internals
F) `canConvert()` returns true only if the conversion will succeed

**Answer: A, C, D**
B wrong — `DefaultConversionService` does NOT handle locale-aware formatting; `FormattingConversionService` does. E wrong — `PropertyEditor` still used in some internal Spring places. F wrong — `canConvert()` checks if a converter EXISTS for the type pair, not whether it will succeed at runtime (source data may still be invalid).

---

**Q3 — Code Output Prediction:**

```java
DefaultConversionService service = new DefaultConversionService();

System.out.println(service.convert("42", Integer.class));
System.out.println(service.convert(42, String.class));
System.out.println(service.convert("true", Boolean.class));
System.out.println(service.convert(3.14, Integer.class));
System.out.println(service.canConvert(String.class, LocalDate.class));
```

A)
```
42
42
true
3
false
```
B)
```
42
42
true
null
true
```
C)
```
42
42
true
3
true
```
D)
```
42
"42"
true
3
false
```

**Answer: A**
`String→Integer`: `42`. `Integer→String`: `"42"` (prints as `42`). `String→Boolean`: `true`. `Double→Integer`: `3` (narrowing via `NumberUtils`). `String→LocalDate`: `false` — `DefaultConversionService` does NOT include Java 8 date converters by default (they require `FormattingConversionService` with `DateTimeFormatterRegistrar`).

---

**Q4 — Trick Question:**

```java
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<?> create(
            @Valid @RequestBody UserForm form,
            String userId,               // <-- notice: no @Valid
            BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().body(result.getAllErrors());
        }
        return ResponseEntity.ok().build();
    }
}
```

What happens at startup or runtime?

A) Works normally — BindingResult captures errors for `form`
B) `IllegalStateException` at startup — BindingResult not adjacent to `@Valid` parameter
C) Works — BindingResult captures errors for both `form` and `userId`
D) `NullPointerException` — BindingResult cannot find its target

**Answer: B**
`BindingResult` MUST immediately follow its corresponding `@Valid`/`@Validated` parameter. In this code, `String userId` is between `@Valid @RequestBody UserForm form` and `BindingResult result`. Spring detects this and throws `IllegalStateException`: "An Errors/BindingResult argument is expected to be declared immediately after the model attribute, the @RequestBody or the @RequestPart arguments".

---

**Q5 — Select all that apply:**
Which `TransactionPhase` behaviors of `@TransactionalEventListener` run INSIDE the original transaction? (Note: wrong topic slipped in — let's replace with a relevant question)

Which of the following are correct about `@Valid` vs `@Validated`? (Select ALL that apply)

A) `@Valid` is a standard Jakarta Validation annotation
B) `@Validated` supports validation groups in method parameters
C) `@Valid` on a class enables method-level validation via AOP
D) `@Validated` is Spring-specific
E) Both trigger Bean Validation constraint checking
F) `@Validated(OnCreate.class)` restricts which constraint groups are checked

**Answer: A, B, D, E, F**
C is wrong — `@Validated` on a CLASS enables method-level validation via AOP, not `@Valid`. `@Valid` on method parameters triggers Bean Validation but does NOT enable method-level AOP validation when on the class.

---

**Q6 — MCQ:**
In the error code resolution hierarchy for `errors.rejectValue("email", "email.invalid")` on object `userForm`, which error code has HIGHEST priority?

A) `"email.invalid"`
B) `"email.invalid.email"`
C) `"email.invalid.userForm.email"`
D) `"email.invalid.java.lang.String"`

**Answer: C**
`DefaultMessageCodesResolver` generates codes in order from most specific to least specific:
1. `"email.invalid.userForm.email"` (code + objectName + fieldName) — HIGHEST
2. `"email.invalid.email"` (code + fieldName)
3. `"email.invalid.java.lang.String"` (code + field type)
4. `"email.invalid"` (code only) — LOWEST

Spring checks codes in this order and uses the first one found in `MessageSource`.

---

**Q7 — MCQ:**
When should you use `ConverterFactory<S, R>` instead of `Converter<S, T>`?

A) When the source type is abstract or an interface
B) When you need to convert from one source type to a FAMILY of related target types
C) When conversion requires locale awareness
D) When conversion involves collection types

**Answer: B**
`ConverterFactory<S, R>` is for converting from a single source type `S` to any subtype of `R`. The classic case: `String → Enum<?>` — a single factory handles conversion to ANY specific enum type. If you need just `String → OrderStatus`, use a plain `Converter`. If you need `String → ANY_ENUM`, use `ConverterFactory`.

---

**Q8 — Code Scenario:**

```java
public class OrderForm {
    @NotNull
    @DateTimeFormat(pattern = "dd/MM/yyyy")
    private LocalDate orderDate;

    @NumberFormat(style = NumberFormat.Style.CURRENCY)
    @DecimalMin("0.01")
    private BigDecimal amount;
}
```

Binding request parameter `orderDate = "invalid-date"`. What happens?

A) `ConstraintViolationException` — `@NotNull` is violated
B) `TypeMismatchException` or binding error — type conversion fails before validation
C) `ParseException` propagated to caller
D) `orderDate` is set to `null` silently

**Answer: B**
Type conversion happens BEFORE Bean Validation. "invalid-date" cannot be parsed as `LocalDate` using the given pattern. A binding/type mismatch error is recorded in `BindingResult`. Spring does NOT throw immediately — it records the error and continues. `@NotNull` would be a separate error but in this case the conversion failure is recorded first. If `BindingResult` is present, it captures the error; if not, Spring throws `MethodArgumentNotValidException`.

---

**Q9 — Select all that apply:**
Which approaches correctly register a custom `Converter` with Spring? (Select ALL that apply)

A) Declare a `@Bean` of type `ConversionService` named `"conversionService"` and add converters
B) Implement `WebMvcConfigurer.addFormatters()` in a `@Configuration` class
C) Declare a `@Bean` of type `Converter` — Spring Boot auto-detects and registers it
D) Register via `CustomEditorConfigurer` as a `PropertyEditor`
E) Call `applicationContext.getBeanFactory().getConversionService().addConverter(...)`

**Answer: A, B, C**
D registers `PropertyEditor`, not `Converter`. E is not a standard approach (context's `ConversionService` is typically immutable after setup). A is the explicit approach. B is the Spring MVC approach. C is Spring Boot's auto-detection — Boot detects `Converter`, `GenericConverter`, and `Formatter` beans and registers them automatically.

---

**Q10 — Drag-and-Drop: Match interface to its primary use case:**

Interfaces: `Converter<S,T>`, `ConverterFactory<S,R>`, `GenericConverter`, `Formatter<T>`, `PropertyEditor`

Use cases:
- Stateless, type-safe, any-to-any conversion (single pair)
- Single source type to FAMILY of related target types
- Complex multi-type conversion with full TypeDescriptor access
- Locale-aware String ↔ Type conversion
- Legacy, stateful, String-only conversion (Java Beans)
- Validates collection element types via generics
- Enables `@DateTimeFormat` and `@NumberFormat` processing
- Used when conversion logic depends on annotations on the target field

**Answer:**
- `Converter<S,T>` → Stateless, type-safe, any-to-any conversion (single pair)
- `ConverterFactory<S,R>` → Single source type to FAMILY of related target types
- `GenericConverter` → Complex multi-type conversion with full TypeDescriptor access; Validates collection element types via generics; Used when conversion logic depends on annotations on target field
- `Formatter<T>` → Locale-aware String ↔ Type conversion; Enables `@DateTimeFormat` and `@NumberFormat` processing
- `PropertyEditor` → Legacy, stateful, String-only conversion (Java Beans)

---

## ④ TRICK ANALYSIS

**Trap 1: PropertyEditor is thread-safe because it is a standard Java class**
Wrong. `PropertyEditor` holds state (`setValue()`/`getValue()`). Spring MUST create a new instance per conversion. Never share `PropertyEditor` instances across threads.

**Trap 2: @Valid supports validation groups**
Wrong. `@Valid` (Jakarta) does NOT support groups in method parameter declarations. `@Validated(GroupName.class)` is Spring's annotation for group-specific validation. In nested object cascading (`@Valid` on a field), groups are NOT applied — `Default` group only.

**Trap 3: BindingResult can appear anywhere in the method signature**
Wrong. `BindingResult` (or `Errors`) MUST immediately follow its `@Valid`/`@Validated` parameter. Any parameter between them causes `IllegalStateException` at startup.

**Trap 4: ConversionService converts everything automatically**
Wrong. `ConversionService` must be explicitly used or registered. Not all Spring infrastructure automatically uses it. `@Value` injection uses it via `TypeConverter`. Data binding uses it when configured. It does NOT intercept arbitrary assignments.

**Trap 5: DefaultConversionService handles Java 8 date types**
Wrong. `DefaultConversionService` alone does NOT handle `LocalDate`, `LocalDateTime` etc. You need `DefaultFormattingConversionService` with `DateTimeFormatterRegistrar` (or Spring Boot's `ApplicationConversionService`).

**Trap 6: @Validated on a class enables Bean Validation on all methods**
Nuanced. `@Validated` on a class enables METHOD-LEVEL validation via Spring AOP for ONLY the PUBLIC methods of that class where `@Valid`/`@Validated`/constraint annotations are present on parameters or return values. It does NOT apply Bean Validation to all method invocations automatically.

**Trap 7: Converter and Formatter are interchangeable in all contexts**
Wrong. `Formatter<T>` is specifically for String ↔ T conversion with locale. It can be used anywhere a `Converter` can, but adds locale-awareness. `Converter<S,T>` handles any S→T without locale. Use `Formatter` when locale sensitivity matters (user-facing formatting). Use `Converter` for internal type transformations.

**Trap 8: ValidationUtils.rejectIfEmpty() handles whitespace**
Partially wrong. `rejectIfEmpty()` rejects null AND empty string `""`. It does NOT reject whitespace-only strings. `rejectIfEmptyOrWhitespace()` handles null, `""`, AND whitespace-only strings like `"   "`.

---

## ⑤ SUMMARY SHEET

### Conversion System Comparison

```
┌───────────────────┬──────────────────┬────────────────────┬─────────────────┐
│ Feature           │ PropertyEditor   │ Converter<S,T>     │ Formatter<T>    │
├───────────────────┼──────────────────┼────────────────────┼─────────────────┤
│ Thread-safe       │ NO — stateful    │ YES — stateless    │ YES — stateless │
│ Type support      │ String only      │ Any S → Any T      │ String ↔ T      │
│ Locale support    │ NO               │ NO                 │ YES             │
│ Standard          │ java.beans       │ Spring             │ Spring          │
│ Use case          │ Legacy/internal  │ General conversion │ Presentation    │
│ Registration      │ CustomEditorConf │ ConversionService  │ FormattingConv  │
│ Spring Boot auto  │ NO               │ YES (@Bean)        │ YES (@Bean)     │
└───────────────────┴──────────────────┴────────────────────┴─────────────────┘
```

---

### Converter Selection Guide

```
Need String → specific enum type?           → Converter<String, MyEnum>
Need String → ANY enum type?                → ConverterFactory<String, Enum>
Need locale-sensitive String ↔ Type?        → Formatter<T>
Need collection element conversion?         → GenericConverter
Need annotation-driven conversion?          → ConditionalConverter + GenericConverter
Need @DateTimeFormat / @NumberFormat?       → FormattingConversionService (auto)
Legacy Java Beans compatibility?            → PropertyEditorSupport
```

---

### Validation Quick Reference

```
Spring Validator:
    → Validator.supports(Class) → filter by type
    → Validator.validate(Object, Errors) → record errors
    → ValidationUtils helper methods
    → errors.pushNestedPath() for nested objects

Bean Validation (JSR-380):
    → @NotNull, @NotBlank, @Size, @Min, @Max, @Pattern, @Email
    → @Valid for cascaded validation on nested objects
    → @Validated for group-specific validation
    → Custom: @Constraint(validatedBy=...) + ConstraintValidator<A,T>

@Valid vs @Validated:
    @Valid:     standard, no groups
    @Validated: Spring-specific, supports groups
    @Validated on CLASS: enables method-level validation via AOP

BindingResult placement:
    MUST immediately follow @Valid/@Validated parameter — no gap!
```

---

### Error Code Resolution (DefaultMessageCodesResolver)

```
errors.rejectValue("email", "email.invalid") on object "userForm":

Priority 1: "email.invalid.userForm.email"  (code.objectName.field)
Priority 2: "email.invalid.email"           (code.field)
Priority 3: "email.invalid.java.lang.String" (code.type)
Priority 4: "email.invalid"                 (code — fallback)

First match in MessageSource wins.
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `PropertyEditor` thread safety | NOT thread-safe — new instance per conversion |
| `Converter<S,T>` thread safety | Thread-safe — stateless singleton |
| `ConverterFactory` use case | Source → FAMILY of target types (e.g., String → any Enum) |
| `Formatter<T>` vs `Converter` | Formatter = locale-aware String ↔ T |
| `DefaultConversionService` | Does NOT handle Java 8 dates — need `DefaultFormattingConversionService` |
| `@Valid` groups | NOT supported in method params — use `@Validated(Group.class)` |
| `BindingResult` placement | MUST immediately follow `@Valid`/`@Validated` param |
| `rejectIfEmpty` vs `rejectIfEmptyOrWhitespace` | Former misses whitespace-only strings |
| `@Validated` on class | Enables method-level validation via AOP |
| Spring Boot auto-registration | `Converter`, `Formatter`, `GenericConverter` beans auto-registered |
| Bean named `"conversionService"` | Auto-detected by Spring as the conversion service |

---

### Interview One-Liners

- "`PropertyEditor` is NOT thread-safe because it holds state — Spring creates a new instance per conversion. `Converter` is stateless and thread-safe, registered once as a singleton."
- "`ConverterFactory<S,R>` produces converters for a family of target types — the classic example is `StringToEnumConverterFactory` which handles conversion to ANY enum type."
- "`Formatter<T>` adds locale awareness to String ↔ T conversion — use it for user-facing dates, numbers, and currencies. `Converter<S,T>` is for internal type transformations without locale."
- "`DefaultConversionService` alone does NOT handle Java 8 date types — use `DefaultFormattingConversionService` which includes `DateTimeFormatterRegistrar`."
- "`BindingResult` MUST immediately follow its `@Valid`/`@Validated` parameter — any intervening parameter causes `IllegalStateException` at startup."
- "`@Valid` (Jakarta) does not support validation groups in method parameters. Use Spring's `@Validated(GroupName.class)` for group-targeted validation."
- "Error code resolution goes from most specific to least specific: `code.objectName.field` → `code.field` → `code.fieldType` → `code`. The first match in `MessageSource` is used."
- "Spring Boot auto-detects and registers `Converter`, `Formatter`, and `GenericConverter` beans — no explicit `ConversionService` configuration needed in Spring Boot."

---
