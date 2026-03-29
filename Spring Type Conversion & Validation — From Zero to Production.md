# Spring Type Conversion & Validation — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

New ticket lands on your desk:

```
TICKET-512: Build an order submission API endpoint
- Accept POST /orders with JSON body
- Fields: customerId (string), amount ("USD 99.99"), 
          orderDate ("25/12/2024"), status ("pending")
- Validate: amount > 0, date not in past, customerId not blank
- Return 400 with field-level errors if invalid
- Works for both Create and Update (different validation rules)
```

You write the obvious first version:

```java
@RestController
public class OrderController {

    @PostMapping("/orders")
    public ResponseEntity<?> createOrder(@RequestBody Map<String, String> body) {

        // Manual type conversion — string to your domain types:
        String amountStr = body.get("amount");           // "USD 99.99"
        String[] parts = amountStr.split(" ");
        Currency currency = Currency.getInstance(parts[0]);
        BigDecimal amount = new BigDecimal(parts[1]);
        Money money = new Money(currency, amount);       // your domain object

        String dateStr = body.get("orderDate");          // "25/12/2024"
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        LocalDate orderDate = LocalDate.parse(dateStr, fmt);

        String statusStr = body.get("status");           // "pending"
        OrderStatus status = OrderStatus.valueOf(statusStr.toUpperCase());

        // Manual validation — scattered everywhere:
        List<String> errors = new ArrayList<>();

        if (body.get("customerId") == null || body.get("customerId").isBlank()) {
            errors.add("customerId: is required");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            errors.add("amount: must be positive");
        }
        if (orderDate.isBefore(LocalDate.now())) {
            errors.add("orderDate: cannot be in the past");
        }

        if (!errors.isEmpty()) {
            return ResponseEntity.badRequest().body(errors);
        }

        // Finally — save the order
        orderService.save(new Order(money, orderDate, status));
        return ResponseEntity.ok().build();
    }
}
```

This works. For two weeks. Then:

- A second endpoint needs the same `"USD 99.99"` → `Money` conversion. You copy-paste.
- A third endpoint parses `"25/12/2024"` differently. Bugs appear.
- QA finds that `"   "` (whitespace) passes the blank check on one endpoint but not another.
- You add an Update endpoint — it needs DIFFERENT validation rules (some fields become optional).
- The `status.toUpperCase()` crashes with `NullPointerException` when status is missing from JSON.

Your controller is now 150 lines of conversion + validation spaghetti. None of it is reusable. None of it is testable in isolation.

**The core problems:**

```
Problem 1: Type conversion is inlined everywhere
           "USD 99.99" → Money  written 6 times across 6 controllers

Problem 2: Validation logic has no home
           if/else error accumulation duplicated everywhere

Problem 3: No way to say "use THESE rules for Create, THOSE for Update"
           without ugly if(isUpdate) branches inside the same validator

Problem 4: No standard error format
           one endpoint returns List<String>, another throws Exception,
           a third returns Map<String, String> — all for the same concept: "validation failed"
```

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Put conversion and validation in a utility class**

```java
public class OrderConverter {
    public static Money parseMoney(String s) { ... }
    public static LocalDate parseDate(String s) { ... }
}

public class OrderValidator {
    public static List<String> validate(Order o) { ... }
}
```

**Why this fails:** Static utility methods can't be Spring beans — they can't use `@Autowired`, can't be mocked in tests, can't be swapped per environment. You also still call them manually in every controller. There's no standard contract — every utility class looks different. And you still have no solution for Create vs Update having different rules.

---

**Naive Approach 2: Use a base controller with shared parsing methods**

```java
public abstract class BaseController {
    protected Money parseMoney(String s) { ... }
    protected LocalDate parseDate(String s) { ... }
    protected void validateNotBlank(String field, String value, List<String> errors) { ... }
}

@RestController
public class OrderController extends BaseController {
    // inherits parsing methods
}
```

**Why this fails:** Java single inheritance — your controller can't extend both `BaseController` and some other useful base. Parsing is still controller-specific, not reusable by services or batch jobs. Validation logic is still tangled with HTTP concerns. The "Create vs Update" problem is completely unsolved.

---

**Naive Approach 3: Annotate the DTO directly with validation rules**

```java
public class OrderRequest {
    @NotBlank
    private String customerId;

    @Positive
    private BigDecimal amount;   // but amount comes in as "USD 99.99"...
                                 // how does "USD 99.99" get converted to BigDecimal first?
}
```

**Why this fails on its own:** You still need something to convert `"USD 99.99"` → `Money` before validation even runs. Annotations alone don't solve conversion. And if you just make everything `String`, you lose type safety entirely. Also: `@Positive` on a BigDecimal doesn't know about your `Create vs Update` distinction.

---

**The correct solution emerges:**

We need three separate, composable things:

```
1. A CONVERSION SYSTEM — converts strings/values to domain types, registered once,
   usable everywhere. Spring calls this: ConversionService + Converter

2. A VALIDATION SYSTEM — expresses rules on the object itself OR programmatically,
   separate from HTTP code. Spring provides two: its own Validator interface
   AND standard Bean Validation annotations (@NotNull, @Size etc.)

3. A WAY TO SAY "WHICH RULES APPLY NOW" — for Create vs Update scenarios.
   Spring calls this: Validation Groups
```

---

## 🧠 3. Concept Introduction — Plain English First

### Conversion: The Passport Control Analogy

Imagine an international airport. Travelers arrive from different countries (different types: `String`, `Integer`, `Long`). They all want to enter the same destination country (your domain type: `Money`, `OrderStatus`, `LocalDate`).

Instead of having each gate agent (controller method) personally handle every passport format from every country, you have a central **passport control office** — one place that knows how to verify and process every type of traveler. Any gate agent just sends the traveler there.

```
"USD 99.99" ──┐
42            ──┤──► [ ConversionService ] ──► Money / Integer / LocalDate
"2024-12-25"  ──┤     (central office)          (domain type)
"PENDING"     ──┘
```

Each specific conversion rule ("how to convert a US passport") is called a **Converter**. You write one Converter per type pair, register it once in the office, and forget it.

### Validation: The Customs Inspector Analogy

After passport control (conversion), the traveler goes through customs (validation). The customs inspector checks: "Is everything in order? Any prohibited items?"

But here's the key insight: **the inspector looks at the OBJECT, not the raw string**. By the time validation runs, you already have a proper `Money` object, a proper `LocalDate` — not raw strings. Validation operates on domain types, not input strings.

```
CONVERSION (happens first):
  "USD -5.00" → Money(USD, -5.00)   ← type is now correct

VALIDATION (happens second, on the domain object):
  Money(USD, -5.00) → REJECTED: "amount must be positive"
```

### The Two Validation Systems

Spring ships with two separate validation systems that work together:

```
System 1 — Spring's own Validator interface:
  You write: class OrderValidator implements Validator { ... }
  Best for: complex business rules, cross-field checks, nested object validation
  How: programmatic — you write Java code that calls errors.rejectValue(...)

System 2 — Bean Validation (standard annotations):
  You write: @NotNull, @Size, @Min on your fields
  Best for: simple structural rules that belong ON the object itself
  How: declarative — annotations on the class
```

They're not competitors — you mix them. Annotations for simple rules, programmatic Validator for complex business logic.

### The "Passport Type" System (Validation Groups)

Create and Update operations need different rules. Bean Validation solves this with **groups** — think of them as "passport types": tourist vs business visa. Same person, different rules apply based on which passport they're carrying.

```
@NotNull(groups = OnCreate.class)   ← "required when creating"
@Null(groups = OnUpdate.class)      ← "must be absent when updating"
```

You tell Spring which "passport type" is in play: `@Validated(OnCreate.class)`.

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TODO 1: Write a Converter<String, Money> — the "USD 99.99" → Money conversion, registered once
TODO 2: Write a ConverterFactory<String, Enum> — handles ALL enums, not just OrderStatus
TODO 3: Register both with ConversionService — the central office
TODO 4: Write Bean Validation annotations on the DTO for simple rules
TODO 5: Write a Spring Validator for complex cross-field business rules
TODO 6: Wire validation into the controller — @Valid, BindingResult
TODO 7: Add validation groups for Create vs Update difference
TODO 8: Handle the "type conversion fails BEFORE validation" edge case
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: Write a Converter

```java
// Converter<S, T>: a stateless, thread-safe object that knows how to turn
// one specific type S into one specific type T.
// "Stateless" means: it stores no data between calls. It's safe to use as a singleton.
public class StringToMoneyConverter implements Converter<String, Money> {

    // THE SILENT FAILURE VERSION — what a junior dev writes first:
    // public Money convert(String source) {
    //     String[] parts = source.split(" ");
    //     return new Money(Currency.getInstance(parts[0]), new BigDecimal(parts[1]));
    // }
    // Silent failure: "USD99.99" (no space) → ArrayIndexOutOfBoundsException at runtime
    // "INVALID 99.99" → IllegalArgumentException with no context
    // These exceptions bubble up as 500 errors, not 400 validation errors.

    @Override
    public Money convert(String source) {
        // Fail fast with a clear message — Spring will catch this and
        // record it as a binding error, not a 500 server error
        if (source == null || source.isBlank()) {
            throw new IllegalArgumentException("Money value cannot be blank");
        }

        String[] parts = source.trim().split("\\s+");  // handles multiple spaces
        if (parts.length != 2) {
            throw new IllegalArgumentException(
                "Money must be in format 'CURRENCY AMOUNT', e.g. 'USD 99.99'. Got: " + source);
        }

        try {
            Currency currency = Currency.getInstance(parts[0].toUpperCase());
            BigDecimal amount = new BigDecimal(parts[1]);
            return new Money(currency, amount);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException(
                "Invalid currency code '" + parts[0] + "' or amount '" + parts[1] + "'", e);
        }
    }
}
```

**What Spring does internally when this converter is registered and `"USD 99.99"` needs to become `Money`:**

```
1. DataBinder receives the raw string "USD 99.99" for a field of type Money
2. It asks ConversionService: "can you convert String → Money?"
3. ConversionService checks its registry: yes, StringToMoneyConverter handles this
4. Calls converter.convert("USD 99.99")
5. Returns Money(USD, 99.99) — stored in the field
6. If the converter throws, DataBinder records a TypeMismatch error in BindingResult
   — NOT a 500 error, a 400-level binding error
```

---

### TODO 2: Write a ConverterFactory for ALL Enums

```java
// ConverterFactory<S, R>: instead of writing one Converter per enum type
// (StringToOrderStatusConverter, StringToPaymentStatusConverter, etc.),
// you write ONE factory that produces the right Converter for ANY enum.
//
// S = the source type (String)
// R = the "family" of target types (Enum — all enums are subtypes of Enum)
public class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

    @Override
    // <T extends Enum> means: T can be OrderStatus, PaymentStatus, or any enum
    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {

        // This returns a Converter tailored to the specific enum class requested
        return source -> {
            if (source == null || source.isBlank()) {
                return null;  // let @NotNull handle the null case
            }

            String normalized = source.trim().toUpperCase().replace("-", "_");

            try {
                //noinspection unchecked
                return (T) Enum.valueOf(targetType, normalized);
            } catch (IllegalArgumentException e) {
                // Build a helpful error message listing valid values
                String validValues = Arrays.stream(targetType.getEnumConstants())
                    .map(Enum::name)
                    .collect(Collectors.joining(", "));
                throw new IllegalArgumentException(
                    "'" + source + "' is not valid for " + targetType.getSimpleName()
                    + ". Valid values: " + validValues);
            }
        };
    }
}
```

**Why ConverterFactory instead of multiple Converters:**

```
Without factory — you'd need to write:
  StringToOrderStatusConverter
  StringToPaymentStatusConverter
  StringToShippingMethodConverter
  StringToUserRoleConverter
  ... (one per enum in your codebase)

With factory — one class handles ALL of them.
The factory gets called with the TARGET type at runtime:
  "pending"  → factory.getConverter(OrderStatus.class)  → OrderStatus.PENDING
  "failed"   → factory.getConverter(PaymentStatus.class) → PaymentStatus.FAILED
```

---

### TODO 3: Register both with ConversionService

```java
// ConversionService: the central registry that holds all converters.
// Think of it as the directory of the passport control office.
// Spring auto-detects a bean named exactly "conversionService".
@Configuration
public class ConversionConfig {

    @Bean
    // THIS NAME MATTERS: Spring looks for a bean named "conversionService"
    // A different name = Spring ignores it for automatic wiring
    public ConversionService conversionService() {

        // DefaultFormattingConversionService: the standard implementation.
        // "Formatting" = it also handles @DateTimeFormat and @NumberFormat annotations.
        // DefaultConversionService alone does NOT handle LocalDate — this one does.
        DefaultFormattingConversionService service =
            new DefaultFormattingConversionService();

        // Register our custom converters:
        service.addConverter(new StringToMoneyConverter());
        service.addConverterFactory(new StringToEnumConverterFactory());

        return service;
    }
}
```

**The silent failure trap — wrong name:**

```java
// WRONG — Spring will NOT auto-detect this for @Value and DataBinder:
@Bean
public ConversionService myConversionService() { ... }  // name doesn't matter? WRONG.

// The automatic wiring only happens for the name "conversionService".
// With a different name, your converters silently don't apply to @Value injection
// and DataBinder. No error — just your "USD 99.99" stops being converted.

// Fix:
@Bean
public ConversionService conversionService() { ... }  // exact name required
// OR: in Spring Boot, just register Converter as a @Bean — Boot handles registration:
@Bean
public Converter<String, Money> stringToMoneyConverter() {
    return new StringToMoneyConverter();  // Spring Boot auto-registers this
}
```

**What `DefaultFormattingConversionService` gives you out of the box:**

```
String → Integer, Long, Double, Float (and primitives)
String → Boolean
String → BigDecimal, BigInteger
String → enum (basic — our factory replaces this with a better version)
Integer → Long (number widening/narrowing)
LocalDate, LocalDateTime → String (when @DateTimeFormat is present)
... and about 50 more built-in pairs
```

---

### TODO 4: Bean Validation annotations on the DTO

```java
// The DTO (Data Transfer Object) — the shape of the incoming request body.
// Bean Validation annotations express rules that are ALWAYS true about this object
// regardless of context. They live ON the class because they're properties of it.
public class OrderRequest {

    // @NotBlank: rejects null, "", and "   " (whitespace-only).
    // Different from @NotEmpty which allows "   " (whitespace).
    // Different from @NotNull which allows "".
    @NotBlank(message = "Customer ID is required")
    private String customerId;

    // @NotNull: the field must be present (not null).
    // Our StringToMoneyConverter handles the "USD 99.99" → Money conversion BEFORE
    // this annotation is checked. Validation runs on the Money object, not the string.
    @NotNull(message = "Amount is required")
    private Money amount;

    // @DateTimeFormat: tells Spring HOW to parse the string "25/12/2024" → LocalDate.
    // This is NOT validation — it's a conversion hint.
    // @FutureOrPresent: THIS is validation — runs after conversion succeeds.
    @NotNull
    @FutureOrPresent(message = "Order date cannot be in the past")
    @DateTimeFormat(pattern = "dd/MM/yyyy")
    private LocalDate orderDate;

    @NotNull
    private OrderStatus status;  // "pending" → OrderStatus.PENDING via our factory

    // @Valid on a nested object: tells Bean Validation to also validate
    // the Address object's own annotations (cascade into it)
    @Valid
    @NotNull
    private Address shippingAddress;

    // getters and setters...
}
```

**The ordering that matters — conversion happens BEFORE validation:**

```
Request body: { "amount": "INVALID", "orderDate": "not-a-date" }

Step 1 — CONVERSION (DataBinder runs StringToMoneyConverter):
  "INVALID" → converter throws IllegalArgumentException
  → DataBinder records: TypeMismatch error on field "amount"
  → orderDate fails date parsing: TypeMismatch error on "orderDate"

Step 2 — VALIDATION (@NotNull, @FutureOrPresent etc.):
  These run ONLY on fields that converted successfully
  Fields with TypeMismatch errors: validation is SKIPPED for them

Result in BindingResult:
  field "amount": typeMismatch — "INVALID" is not a valid Money value
  field "orderDate": typeMismatch — "not-a-date" is not a valid date

The @FutureOrPresent never even runs on orderDate because it never became a LocalDate.
```

---

### TODO 5: Spring Validator for complex business rules

Some rules can't be expressed as annotations. "Ship date must be after order date" involves TWO fields. "Amount above $10,000 requires manager approval" involves business logic. That's what Spring's `Validator` interface is for.

```java
// Validator: a Spring interface for programmatic validation.
// "supports()" filters which objects this validator handles.
// "validate()" inspects the object and records errors in the Errors object.
@Component
public class OrderValidator implements Validator {

    // supports(): Spring calls this to check if the validator can handle a given type.
    // isAssignableFrom() means: handle OrderRequest AND any subclass of it.
    @Override
    public boolean supports(Class<?> clazz) {
        return OrderRequest.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        // Errors: the object where you RECORD problems — not where you throw them.
        // Recording an error here does NOT stop execution. All errors accumulate.
        // Spring returns all of them at once.
        OrderRequest order = (OrderRequest) target;

        // Simple field rule — ValidationUtils handles the null check for you:
        // rejectIfEmptyOrWhitespace: rejects null, "", AND "   "
        // rejectIfEmpty: rejects null and "" but NOT "   " (whitespace passes!)
        ValidationUtils.rejectIfEmptyOrWhitespace(
            errors, "customerId", "field.required", "Customer ID is required");

        // Amount validation — amount is already a Money object here (conversion happened)
        if (order.getAmount() != null) {
            if (order.getAmount().value().compareTo(BigDecimal.ZERO) <= 0) {
                // rejectValue(fieldName, errorCode, errorArgs[], defaultMessage)
                // errorCode: used to look up message in messages.properties
                // {0} in the message gets replaced with errorArgs[0]
                errors.rejectValue("amount", "amount.positive",
                    new Object[]{order.getAmount()},
                    "Amount must be positive, was: {0}");
            }
        }

        // Cross-field validation — this can't be expressed as a single-field annotation
        if (order.getShipDate() != null && order.getOrderDate() != null) {
            if (order.getShipDate().isBefore(order.getOrderDate())) {
                // reject() with no field name = "global error" (not tied to one field)
                errors.reject("dates.invalid",
                    "Ship date cannot be before order date");
            }
        }

        // Nested object validation — push into the nested path so field names
        // appear as "shippingAddress.street" instead of just "street"
        if (order.getShippingAddress() != null) {
            // pushNestedPath: all subsequent rejectValue calls get this prefix
            errors.pushNestedPath("shippingAddress");
            ValidationUtils.invokeValidator(
                new AddressValidator(), order.getShippingAddress(), errors);
            // popNestedPath: go back to the parent object's context
            errors.popNestedPath();
        }
    }
}
```

**What `errors.rejectValue()` does internally:**

```
errors.rejectValue("amount", "amount.positive", ...)

Spring's DefaultMessageCodesResolver generates FOUR error codes, most-specific first:
  1. "amount.positive.orderRequest.amount"  (code.objectName.fieldName)
  2. "amount.positive.amount"               (code.fieldName)
  3. "amount.positive.com.example.Money"    (code.fieldType)
  4. "amount.positive"                      (code alone)

Spring checks messages.properties in this order.
First match wins. If none match, the defaultMessage parameter is used.

This means in messages.properties you can override at any level:
  amount.positive=Amount must be greater than zero
  amount.positive.amount=The order amount must be positive     ← more specific, wins
```

---

### TODO 6: Wire validation into the controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private OrderValidator orderValidator;

    // @InitBinder: called before EACH request handler in this controller.
    // WebDataBinder: the component that binds request data to objects AND runs validation.
    // We register our custom validator here so it runs alongside Bean Validation.
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.addValidators(orderValidator);
    }

    // @Valid: tells Spring to run Bean Validation (@NotBlank, @NotNull etc.)
    //         on the OrderRequest object before the method body runs.
    // BindingResult: MUST immediately follow the @Valid parameter — no exceptions.
    //                Captures all errors (both Bean Validation AND our OrderValidator).
    //                If absent, Spring throws MethodArgumentNotValidException on first error.
    @PostMapping
    public ResponseEntity<?> createOrder(
            @Valid @RequestBody OrderRequest request,
            BindingResult result) {           // ← MUST be right here, no gap

        // THE SILENT FAILURE TRAP — wrong parameter order:
        // createOrder(@Valid @RequestBody OrderRequest request,
        //             String someOtherParam,   ← gap here!
        //             BindingResult result)    ← Spring throws IllegalStateException
        //
        // Spring can't figure out WHICH @Valid this BindingResult belongs to.
        // Fails at startup, not at request time.

        if (result.hasErrors()) {
            // getFieldErrors(): errors tied to specific fields
            // getGlobalErrors(): errors not tied to any field (cross-field rules)
            Map<String, List<String>> errorMap = new HashMap<>();

            result.getFieldErrors().forEach(error ->
                errorMap.computeIfAbsent(error.getField(), k -> new ArrayList<>())
                    .add(error.getDefaultMessage())
            );
            result.getGlobalErrors().forEach(error ->
                errorMap.computeIfAbsent("_global", k -> new ArrayList<>())
                    .add(error.getDefaultMessage())
            );

            return ResponseEntity.badRequest().body(errorMap);
        }

        // request.getAmount() is already a Money object here — not a string
        // request.getOrderDate() is already a LocalDate — not a string
        // request.getStatus() is already an OrderStatus enum — not a string
        orderService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

---

### TODO 7: Validation Groups for Create vs Update

```java
// Groups are just empty interfaces — they're marker types, like tags.
// They have no methods. They exist purely so annotations can reference them.
public interface OnCreate {}
public interface OnUpdate {}

public class OrderRequest {

    // groups = OnCreate.class: this @NotNull constraint is checked ONLY
    // when the caller specifies the OnCreate group.
    // When the OnUpdate group is active, this constraint is ignored.
    @NotNull(groups = OnCreate.class, message = "Customer ID required for new orders")
    // @Null on Update means: if customerId IS provided in an update request, reject it.
    // You can't change the customer on an existing order.
    @Null(groups = OnUpdate.class, message = "Customer ID cannot be changed on update")
    private String customerId;

    // groups = {OnCreate.class, OnUpdate.class}: always validated, in both scenarios
    @NotNull(groups = {OnCreate.class, OnUpdate.class}, message = "Amount is required")
    private Money amount;

    // No groups specified = Default group = validated by plain @Valid
    // @Valid does NOT support groups — it always runs the Default group
    @FutureOrPresent(message = "Order date cannot be in the past")
    private LocalDate orderDate;
}

@RestController
@RequestMapping("/orders")
public class OrderController {

    // @Validated(OnCreate.class): Spring's annotation (not Jakarta's @Valid).
    // Runs ONLY the constraints that have groups = OnCreate.class.
    // Constraints with no groups (Default group) are also included.
    @PostMapping
    public ResponseEntity<?> create(
            @Validated(OnCreate.class) @RequestBody OrderRequest request,
            BindingResult result) {
        // Runs: @NotNull(groups=OnCreate), @NotNull(groups={OnCreate,OnUpdate}), @FutureOrPresent
        // Skips: @Null(groups=OnUpdate)
        if (result.hasErrors()) return ResponseEntity.badRequest().body(result.getAllErrors());
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<?> update(
            @PathVariable String id,
            @Validated(OnUpdate.class) @RequestBody OrderRequest request,
            BindingResult result) {
        // Runs: @Null(groups=OnUpdate), @NotNull(groups={OnCreate,OnUpdate}), @FutureOrPresent
        // Skips: @NotNull(groups=OnCreate)
        if (result.hasErrors()) return ResponseEntity.badRequest().body(result.getAllErrors());
        return ResponseEntity.ok().build();
    }
}
```

**The `@Valid` vs `@Validated` decision:**

```
@Valid (jakarta.validation.Valid):
  ✓ Standard, works everywhere
  ✗ No group support in method parameters
  Use when: simple validation, no Create/Update distinction needed

@Validated (org.springframework.validation.annotation.Validated):
  ✓ Spring-specific
  ✓ Supports groups: @Validated(OnCreate.class)
  ✓ On CLASS: enables method-level validation via AOP (for @Service methods)
  Use when: you need Create/Update distinction, or method-level validation on services
```

---

### TODO 8: Build a custom Bean Validation constraint for business-specific rules

For rules that are reusable across multiple DTOs (not just `OrderRequest`):

```java
// Step 1: Define the annotation.
// @Constraint(validatedBy=...): links this annotation to its implementation class.
// @Target: where this annotation can be placed (on fields and method parameters here)
// @Retention(RUNTIME): must be RUNTIME so the JVM can see it when validating
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PositiveMoneyValidator.class)
@Documented
public @interface PositiveMoney {
    // These three are REQUIRED by the Bean Validation spec — don't skip them:
    String message() default "Amount must be positive";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Step 2: Implement the validator.
// ConstraintValidator<A, T>: A = the annotation type, T = the field type being validated
public class PositiveMoneyValidator
        implements ConstraintValidator<PositiveMoney, Money> {

    // initialize(): called once when the validator is created.
    // Use it if you need to read annotation attributes.
    @Override
    public void initialize(PositiveMoney annotation) {
        // nothing needed here — our rule is fixed: amount > 0
    }

    // isValid(): the actual check.
    // IMPORTANT: return true for null values — let @NotNull handle null.
    // Mixing null checking here creates confusing error messages.
    @Override
    public boolean isValid(Money value, ConstraintValidatorContext context) {
        if (value == null) return true;  // @NotNull handles null separately

        boolean valid = value.amount().compareTo(BigDecimal.ZERO) > 0;

        if (!valid) {
            // Custom message with the actual bad value embedded:
            context.disableDefaultConstraintViolation();  // suppress default message
            context.buildConstraintViolationWithTemplate(
                "Amount " + value.amount() + " " + value.currency() + " must be positive")
                .addConstraintViolation();
        }

        return valid;
    }
}

// Step 3: Use it just like any built-in annotation:
public class OrderRequest {
    @NotNull
    @PositiveMoney  // reusable across ANY DTO that has a Money field
    private Money amount;
}

public class InvoiceRequest {
    @NotNull
    @PositiveMoney  // same annotation, same validator, different DTO
    private Money totalDue;
}
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: Spring Validator vs Bean Validation annotations**

```
Use Bean Validation (@NotNull, @Size etc.) when:
  → The rule is a structural property of the field itself
  → "email must be a valid email format" — this is always true
  → Simple, single-field, stateless checks
  → The rule makes sense on a database entity too

Use Spring Validator (implements Validator) when:
  → The rule involves multiple fields ("ship date after order date")
  → The rule requires injected services ("sku must exist in the database")
  → The rule is only valid in a specific context (not always)
  → You need to validate collections element-by-element with index-aware errors
```

**Decision 2: Converter vs PropertyEditor**

```
Always use Converter for new code. PropertyEditor is legacy.

Converter wins because:
  ✓ Thread-safe singleton — one instance shared everywhere
  ✓ Handles any type pair, not just String
  ✓ Testable in isolation: converter.convert("USD 99.99")
  ✗ Slightly more setup (need ConversionService registration)

PropertyEditor exists for:
  → Internal Spring machinery (XML bean definitions)
  → Legacy codebases
  → Spring MVC @InitBinder registration (still works but not recommended)
```

**Decision 3: ConverterFactory vs individual Converters**

```
Write individual Converter when:
  → Only one or two specific type pairs
  → The conversion logic differs significantly per target type

Write ConverterFactory when:
  → You have a "family" of target types (all enums, all Number subtypes)
  → The conversion algorithm is the same, only the target class differs
  → Adding a new enum/subtype should NOT require a new Converter
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
@SpringBootTest
class ValidationAndConversionTest {

    @Autowired
    private ConversionService conversionService;

    @Autowired
    private javax.validation.Validator beanValidator;

    // TRAP 1: rejectIfEmpty vs rejectIfEmptyOrWhitespace
    @Test
    void whitespacePassesRejectIfEmpty() {
        DataBinder binder = new DataBinder(new OrderRequest(), "orderRequest");
        binder.addValidators(new OrderValidator());

        MutablePropertyValues pvs = new MutablePropertyValues();
        pvs.add("customerId", "   ");  // whitespace only
        binder.bind(pvs);
        binder.validate();

        // If your validator uses rejectIfEmpty() instead of rejectIfEmptyOrWhitespace():
        // "   " passes the empty check and customerId = "   " — NOT rejected!
        // Your service gets a whitespace customerId and queries the DB with it.
        // Fix: always use rejectIfEmptyOrWhitespace for string fields.
        assertTrue(binder.getBindingResult().hasFieldErrors("customerId"),
            "Whitespace-only customerId must be rejected");
    }

    // TRAP 2: Conversion failure records as TypeMismatch, not your custom message
    @Test
    void conversionFailureBeforeValidation() {
        Set<ConstraintViolation<OrderRequest>> violations =
            beanValidator.validate(new OrderRequest() {{
                // Amount is null because "GARBAGE" failed conversion
                // before bean validation even ran
            }});

        // The @NotNull message says "Amount is required"
        // but the actual error shown to the user says "Failed to convert value of type..."
        // The TYPE MISMATCH error replaces your validation error.
        // User sees a technical message, not your friendly one.
        // Fix: in the controller, handle TypeMismatch errors separately in BindingResult.
    }

    // TRAP 3: @Valid does NOT support groups — silently applies Default group
    @Test
    void atValidIgnoresGroups() {
        // This is legal Java but groups are SILENTLY IGNORED:
        // @Valid @RequestBody OrderRequest request  ← no groups
        // If OrderRequest has @NotNull(groups=OnCreate.class),
        // that constraint is NEVER checked when using plain @Valid.
        // Use @Validated(OnCreate.class) for group-specific validation.
    }

    // TRAP 4: DefaultConversionService missing LocalDate
    @Test
    void defaultConversionServiceMissesLocalDate() {
        DefaultConversionService basic = new DefaultConversionService();

        // This FAILS silently — returns null or throws:
        assertThrows(ConversionFailedException.class, () ->
            basic.convert("2024-12-25", LocalDate.class)
        );

        // Fix: use DefaultFormattingConversionService instead:
        DefaultFormattingConversionService formatting = new DefaultFormattingConversionService();
        LocalDate date = formatting.convert("2024-12-25", LocalDate.class);
        assertNotNull(date);  // works!
    }

    // TRAP 5: BindingResult gap causes startup failure, not runtime failure
    @Test
    void bindingResultGapFailsAtStartup() {
        // You can't unit test this — Spring detects the gap during context loading.
        // Your entire application fails to start with:
        // "An Errors/BindingResult argument is expected to be declared
        //  immediately after the model attribute"
        //
        // This means a bad @Valid + BindingResult placement is caught in CI
        // (context loads in tests) not in production. Your test suite protects you.
    }

    // TRAP 6: Null safety in isValid()
    @Test
    void constraintValidatorMustHandleNull() {
        // If your ConstraintValidator.isValid() doesn't handle null:
        // @PositiveMoney Money amount — when amount IS null:
        //   isValid(null, context) → NullPointerException inside your validator
        //   → 500 error instead of a clean "amount is required" message
        //
        // Fix: always return true for null in isValid(). Let @NotNull do its job.
        PositiveMoneyValidator validator = new PositiveMoneyValidator();
        assertTrue(validator.isValid(null, null),
            "Validator must return true for null — let @NotNull handle it");
    }
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version problem:** error responses are inconsistent. Each controller builds its own `Map<String, List<String>>` from `BindingResult`. 10 controllers = 10 slightly different error formats. Frontend team is annoyed.

**Senior dev refactoring:** extract a `@RestControllerAdvice` that handles validation errors globally, and standardize the error shape:

```java
// @RestControllerAdvice: a global exception handler for all @RestController classes.
// Any exception thrown (or not caught) in any controller reaches here.
@RestControllerAdvice
public class ValidationExceptionHandler {

    // Handles the case where @Valid is used WITHOUT BindingResult present —
    // Spring throws this exception when validation fails and there's no BindingResult to catch it.
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidationFailed(
            MethodArgumentNotValidException ex) {
        return buildErrorResponse(ex.getBindingResult());
    }

    // Handles method-level validation failures (from @Validated on a @Service class)
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleConstraintViolation(
            ConstraintViolationException ex) {
        List<FieldErrorDetail> errors = ex.getConstraintViolations().stream()
            .map(v -> new FieldErrorDetail(
                v.getPropertyPath().toString(),
                v.getMessage()))
            .collect(Collectors.toList());
        return new ValidationErrorResponse("Validation failed", errors);
    }

    private ValidationErrorResponse buildErrorResponse(BindingResult result) {
        List<FieldErrorDetail> fieldErrors = result.getFieldErrors().stream()
            .map(e -> new FieldErrorDetail(e.getField(), e.getDefaultMessage()))
            .collect(Collectors.toList());

        List<FieldErrorDetail> globalErrors = result.getGlobalErrors().stream()
            .map(e -> new FieldErrorDetail("_global", e.getDefaultMessage()))
            .collect(Collectors.toList());

        fieldErrors.addAll(globalErrors);
        return new ValidationErrorResponse("Validation failed", fieldErrors);
    }
}

// Standardized error response shape — same format from every endpoint:
public record ValidationErrorResponse(
    String message,
    List<FieldErrorDetail> errors
) {}

public record FieldErrorDetail(
    String field,
    String message
) {}

// NOW controllers become clean — no BindingResult needed:
@RestController
@RequestMapping("/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<Order> createOrder(
            @Validated(OnCreate.class) @RequestBody OrderRequest request) {
        // No BindingResult. If validation fails, MethodArgumentNotValidException
        // is thrown automatically and caught by our global handler above.
        // The controller only contains happy-path logic.
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(orderService.create(request));
    }
}
```

**What improved:**
- Every endpoint returns the same error JSON shape
- Controllers contain zero validation-handling boilerplate
- Frontend team has one contract to handle
- Adding a new endpoint automatically gets validation error handling
- Changing the error format requires one change in one place

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **Converter\<S,T\>** | A stateless, thread-safe recipe for turning one specific type into another. Write once, registered once, called automatically. |
| **ConverterFactory\<S,R\>** | A factory that produces Converters for an entire family of target types. One factory handles String→ANY enum instead of one Converter per enum. |
| **GenericConverter** | The most powerful converter — gets full type information including generics (like "List of Strings" vs "List of Integers"). Use when Converter isn't enough. |
| **ConversionService** | The central registry of all Converters. The "passport control office directory." |
| **DefaultFormattingConversionService** | The standard ConversionService implementation — includes built-in converters AND support for @DateTimeFormat/@NumberFormat. Use this, not DefaultConversionService. |
| **PropertyEditor** | Legacy Java Beans way of converting strings. Stateful, NOT thread-safe. Still used internally but don't write new ones. |
| **Formatter\<T\>** | Like a Converter, but specifically for String↔T and locale-aware. Use when display format changes by country (dates, currencies). |
| **Validator** | Spring's interface for programmatic validation. `supports()` filters which types. `validate()` records errors in an Errors object. |
| **Errors** | The accumulator of validation problems. You record into it; you don't throw from it. Records field-level and global errors. |
| **BindingResult** | Extends Errors. Attached to a specific bound object in a controller. Must immediately follow its @Valid parameter. |
| **Bean Validation** | The Jakarta standard (@NotNull, @Size, @Email etc.). Declarative — you put annotations on fields. Spring integrates this. |
| **ConstraintValidator\<A,T\>** | The implementation class behind a custom @Annotation. `isValid()` contains the actual check. |
| **@Valid** | Standard Jakarta annotation. Triggers Bean Validation. No group support. |
| **@Validated** | Spring's annotation. Like @Valid but adds group support. On a class: enables method-level validation via AOP. |
| **Validation Groups** | Empty marker interfaces (OnCreate, OnUpdate). Let you tag constraints with "only check this in scenario X." |
| **@InitBinder** | Method in a controller that registers PropertyEditors or Validators for that controller's request binding. |
| **ValidationUtils** | Helper methods — `rejectIfEmptyOrWhitespace()` (handles null + "" + "   ") vs `rejectIfEmpty()` (handles only null + ""). |

---

### The decision flowchart:

```
Need to convert a type?
  ├── One specific pair (String → Money)?              → Converter<String, Money>
  ├── One source → all subtypes (String → any Enum)?  → ConverterFactory<String, Enum>
  ├── Needs locale (display "$1,234" vs "1.234,56€")? → Formatter<Money>
  └── Needs generics info (List<String> → Set<Long>)? → GenericConverter

Need to validate?
  ├── Simple field rule (not null, min length, pattern)?    → Bean Validation annotation
  ├── Cross-field rule (ship date after order date)?        → Spring Validator
  ├── Rule needs @Autowired service (check DB)?             → Spring Validator
  └── Rule is reusable across multiple DTOs?               → Custom @Constraint annotation

Create vs Update need different rules?
  → Define interface groups: OnCreate, OnUpdate
  → Tag constraints: @NotNull(groups = OnCreate.class)
  → Apply in controller: @Validated(OnCreate.class)

@Valid or @Validated?
  → No group distinction needed: @Valid (standard, simpler)
  → Need groups: @Validated(GroupName.class) (Spring-specific)
  → Enabling method-level validation on @Service: @Validated on the class
```

### The execution timeline every request follows:

```
HTTP Request arrives
       ↓
DataBinder reads request body/params as Strings
       ↓
CONVERSION runs (ConversionService):
  "USD 99.99" → Money     ← your StringToMoneyConverter
  "pending"   → OrderStatus  ← your StringToEnumConverterFactory
  "25/12/2024" → LocalDate  ← DateTimeFormatter from @DateTimeFormat
  If any conversion fails: TypeMismatch error recorded, field skipped in validation
       ↓
VALIDATION runs (only on successfully converted fields):
  Bean Validation annotations: @NotNull, @Size, @PositiveMoney...
  Spring Validator: OrderValidator.validate(order, errors)
  If errors: BindingResult captures them (no exception yet)
       ↓
Controller method runs:
  If BindingResult present: YOU check result.hasErrors()
  If BindingResult absent: Spring throws MethodArgumentNotValidException automatically
       ↓
Response returned (200 OK or 400 Bad Request)
```
