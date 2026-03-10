# TOPIC 4.7 — Spring Expression Language (SpEL)

---

## ① CONCEPTUAL EXPLANATION

### What is SpEL?

Spring Expression Language (SpEL) is a powerful expression language that supports querying and manipulating an object graph at runtime. It is Spring's own expression language — similar in spirit to OGNL (used in Struts), MVEL, and JBoss EL, but tightly integrated with Spring's type system, conversion service, and bean lifecycle.

```
SpEL is used in:
├── @Value("#{...}")              — bean injection and property binding
├── @Cacheable(key="#{...}")      — dynamic cache key generation
├── @PreAuthorize("#{...}")       — Spring Security method security
├── @ConditionalOnExpression      — Spring Boot conditional beans
├── Spring Data @Query            — dynamic query construction
├── Spring Batch                  — step/flow conditions
├── Spring Integration            — message routing expressions
└── XML bean definitions          — value="#{...}"
```

SpEL is evaluated at runtime by the `ExpressionParser` infrastructure, with the evaluation context providing the variable bindings and bean references.

---

### 4.7.1 — SpEL Architecture — Internal Mechanics

SpEL has three core components:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SpEL Architecture                            │
├──────────────────┬──────────────────┬───────────────────────────┤
│  ExpressionParser│  Expression      │  EvaluationContext         │
│                  │                  │                             │
│  Parses the      │  Compiled AST    │  Provides:                  │
│  expression      │  of the          │  - Root object              │
│  string into     │  expression      │  - Variables (#name)        │
│  an AST          │  Can be          │  - Bean resolver (@beanName)│
│                  │  evaluated       │  - Type converters          │
│                  │  multiple times  │  - Method resolvers         │
│                  │  against         │  - Property accessors       │
│                  │  different       │                             │
│                  │  contexts        │                             │
└──────────────────┴──────────────────┴───────────────────────────┘
```

**Core classes:**

```java
// Parser: converts string → Expression (AST)
ExpressionParser parser = new SpelExpressionParser();

// Expression: the compiled AST — reusable
Expression expr = parser.parseExpression("name.toUpperCase()");

// EvaluationContext: provides the environment for evaluation
EvaluationContext context = new StandardEvaluationContext(rootObject);

// Evaluation: run the expression against a context
String result = expr.getValue(context, String.class);
```

**SpEL Compilation (Spring 4.1+):**

SpEL expressions can be compiled to bytecode for repeated evaluation performance:

```java
SpelExpressionParser parser = new SpelExpressionParser(
    new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE, // compile on first use
                                 this.getClass().getClassLoader()));

Expression expr = parser.parseExpression("payload.amount > 1000");
// First evaluation: interpreted (slow)
// After compilation: bytecode-compiled (fast — comparable to native Java)
```

**SpEL parsing pipeline:**

```
"name.toUpperCase()"
        │
        ▼ Lexer (tokenization)
[name, ., toUpperCase, (, )]
        │
        ▼ Parser (AST construction)
PropertyOrFieldReference("name")
    └── MethodReference("toUpperCase")
        └── []  // no args
        │
        ▼ getValue()
Evaluator → propertyAccessor.read(ctx, rootObject, "name") → "john"
          → "john".toUpperCase() → "JOHN"
```

---

### 4.7.2 — Expression Syntax — Complete Reference

#### Literals

```java
// String literals — single or double quotes
parser.parseExpression("'Hello World'").getValue()          // "Hello World"
parser.parseExpression("\"Hello\"").getValue()              // "Hello"

// Numbers
parser.parseExpression("42").getValue(Integer.class)        // 42
parser.parseExpression("3.14159").getValue(Double.class)    // 3.14159
parser.parseExpression("0xFF").getValue(Integer.class)      // 255 (hex)
parser.parseExpression("1L").getValue(Long.class)           // 1L (long)
parser.parseExpression("1.0E3").getValue(Double.class)      // 1000.0 (scientific)

// Boolean
parser.parseExpression("true").getValue(Boolean.class)      // true
parser.parseExpression("false").getValue(Boolean.class)     // false

// Null
parser.parseExpression("null").getValue()                   // null
```

#### Property Access — dot notation

```java
// Properties accessed via getter
// expr "name" → calls getName() on root object
// expr "address.city" → calls getAddress().getCity()

User user = new User("Alice", new Address("New York"));
StandardEvaluationContext ctx = new StandardEvaluationContext(user);

parser.parseExpression("name").getValue(ctx, String.class)           // "Alice"
parser.parseExpression("address.city").getValue(ctx, String.class)   // "New York"
parser.parseExpression("address.city.length()").getValue(ctx, Integer.class) // 8
```

#### Array, List, Map Access

```java
// Array and List: [index]
parser.parseExpression("numbers[0]").getValue(ctx)    // first element
parser.parseExpression("names[2]").getValue(ctx)      // third element

// Map: [key]
parser.parseExpression("info['name']").getValue(ctx)  // map.get("name")
parser.parseExpression("info[\"age\"]").getValue(ctx) // same with escaped quotes

// Nested
parser.parseExpression("items[0].name").getValue(ctx) // items.get(0).getName()
```

#### Method Invocation

```java
// Instance methods
parser.parseExpression("name.toUpperCase()").getValue(ctx)            // "ALICE"
parser.parseExpression("name.substring(0, 3)").getValue(ctx)          // "Ali"
parser.parseExpression("name.contains('lice')").getValue(ctx, Boolean.class) // true

// Static methods via T() operator
parser.parseExpression("T(java.lang.Math).PI").getValue(Double.class)  // 3.14159...
parser.parseExpression("T(java.lang.Math).abs(-42)").getValue(Integer.class) // 42
parser.parseExpression("T(java.lang.System).currentTimeMillis()").getValue() // millis

// Chaining
parser.parseExpression("name.trim().toUpperCase().substring(0,3)").getValue(ctx)
```

#### Operators — Arithmetic

```java
parser.parseExpression("2 + 3").getValue()     // 5
parser.parseExpression("10 - 4").getValue()    // 6
parser.parseExpression("3 * 4").getValue()     // 12
parser.parseExpression("10 / 3").getValue()    // 3 (integer division)
parser.parseExpression("10.0 / 3").getValue()  // 3.333...
parser.parseExpression("10 % 3").getValue()    // 1 (modulo)
parser.parseExpression("2 ^ 8").getValue()     // 256 (power)

// String concatenation with +
parser.parseExpression("'Hello' + ' ' + 'World'").getValue() // "Hello World"
```

#### Operators — Relational and Equality

```java
parser.parseExpression("2 == 2").getValue(Boolean.class)   // true
parser.parseExpression("2 != 3").getValue(Boolean.class)   // true
parser.parseExpression("5 > 3").getValue(Boolean.class)    // true
parser.parseExpression("5 >= 5").getValue(Boolean.class)   // true
parser.parseExpression("3 < 5").getValue(Boolean.class)    // true
parser.parseExpression("3 <= 3").getValue(Boolean.class)   // true

// Symbolic alternatives (useful in XML where < > conflict with tags):
parser.parseExpression("5 gt 3").getValue(Boolean.class)   // true (greater than)
parser.parseExpression("3 lt 5").getValue(Boolean.class)   // true (less than)
parser.parseExpression("5 ge 5").getValue(Boolean.class)   // true (>=)
parser.parseExpression("3 le 3").getValue(Boolean.class)   // true (<=)
parser.parseExpression("2 eq 2").getValue(Boolean.class)   // true (==)
parser.parseExpression("2 ne 3").getValue(Boolean.class)   // true (!=)

// instanceof
parser.parseExpression("'hello' instanceof T(String)").getValue(Boolean.class) // true
```

#### Operators — Logical

```java
parser.parseExpression("true and false").getValue(Boolean.class) // false
parser.parseExpression("true or false").getValue(Boolean.class)  // true
parser.parseExpression("not true").getValue(Boolean.class)        // false
parser.parseExpression("!true").getValue(Boolean.class)           // false
parser.parseExpression("true && false").getValue(Boolean.class)   // false
parser.parseExpression("true || false").getValue(Boolean.class)   // true
```

#### Ternary and Elvis Operators

```java
// Ternary: condition ? ifTrue : ifFalse
parser.parseExpression("age > 18 ? 'adult' : 'minor'").getValue(ctx) // "adult"

// Elvis operator: value ?: default
// Returns value if not null/empty, otherwise default
// Equivalent to: value != null ? value : default
parser.parseExpression("name ?: 'unknown'").getValue(ctx)  // "Alice" or "unknown"
parser.parseExpression("middleName ?: 'N/A'").getValue(ctx) // "N/A" if middleName null
```

#### Safe Navigation Operator

```java
// Null-safe: obj?.property
// Returns null if obj is null instead of NullPointerException
parser.parseExpression("user?.address?.city").getValue(ctx)
// If user is null → null (no NPE)
// If user.address is null → null (no NPE)
// If both non-null → city value

// Traditional way without ?: → NullPointerException if any null in chain
parser.parseExpression("user.address.city").getValue(ctx) // NPE if user or address is null
```

#### Collection Operators — Selection and Projection

**Selection (`.?[condition]`)** — filter a collection:

```java
// Syntax: collection.?[selectionExpression]
// Returns new collection of elements where expression is true

List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
StandardEvaluationContext ctx = new StandardEvaluationContext();
ctx.setVariable("numbers", numbers);

// Select even numbers
parser.parseExpression("#numbers.?[#this % 2 == 0]").getValue(ctx)
// → [2, 4, 6, 8, 10]

// Select first matching (.^[])
parser.parseExpression("#numbers.^[#this > 5]").getValue(ctx)
// → 6 (first element > 5)

// Select last matching (.$ [])
parser.parseExpression("#numbers.$[#this > 5]").getValue(ctx)
// → 10 (last element > 5)

// With objects:
List<User> users = getUsers(); // users with name, age
ctx.setVariable("users", users);

// Select adult users
parser.parseExpression("#users.?[age >= 18]").getValue(ctx)
// → List<User> with only adults

// Select users with names starting with 'A'
parser.parseExpression("#users.?[name.startsWith('A')]").getValue(ctx)
```

**Projection (`.![expression]`)** — transform a collection:

```java
// Syntax: collection.![projectionExpression]
// Returns new collection with each element transformed

// Get all user names from a list of users
parser.parseExpression("#users.![name]").getValue(ctx)
// → List<String> ["Alice", "Bob", "Charlie"]

// Get all email addresses uppercase
parser.parseExpression("#users.![email.toUpperCase()]").getValue(ctx)
// → List<String>

// Combine selection and projection:
// Names of adult users
parser.parseExpression("#users.?[age >= 18].![name]").getValue(ctx)
// → List<String> of names of adults only
```

---

### 4.7.3 — Variables in SpEL

Variables are registered in the `EvaluationContext` and accessed with `#variableName`:

```java
StandardEvaluationContext ctx = new StandardEvaluationContext(rootObject);

// Register variables
ctx.setVariable("maxSize", 100);
ctx.setVariable("prefix", "PROD_");
ctx.setVariable("currentUser", getCurrentUser());

// Access in expressions
parser.parseExpression("#maxSize * 2").getValue(ctx)        // 200
parser.parseExpression("#prefix + name").getValue(ctx)      // "PROD_Alice"
parser.parseExpression("#currentUser.role").getValue(ctx)   // user's role
```

**Special built-in variables:**

```java
// #this — refers to the current element (in collection operations)
parser.parseExpression("#numbers.?[#this > 5]")    // #this = each number

// #root — refers to the root evaluation object
parser.parseExpression("#root.name")                // same as just "name"
```

---

### 4.7.4 — Bean References in SpEL

In an `ApplicationContext`, beans can be referenced directly using `@beanName`:

```java
// In @Value annotation:
@Value("#{userService.findAdmin().name}")
private String adminName;
// Calls ctx.getBean("userService").findAdmin().getName()

@Value("#{@userService.findAdmin().name}")
private String adminName2;  // @ prefix is explicit bean reference

// In EvaluationContext with BeanFactory:
StandardEvaluationContext ctx = new StandardEvaluationContext();
ctx.setBeanResolver(new BeanFactoryResolver(beanFactory));

parser.parseExpression("@myService.doWork()").getValue(ctx);
// Resolves @myService by calling beanFactory.getBean("myService")
```

**The `@` prefix resolves the expression as a bean name from the `ApplicationContext`.**

This is how `@Value("#{someBean.property}")` works — Spring sets up the evaluation context with a `BeanFactoryResolver` so bean names are resolvable.

---

### 4.7.5 — Type References — T() Operator

The `T()` operator provides access to types (classes) — enabling static method calls and type checking:

```java
// Static field access
parser.parseExpression("T(java.lang.Math).PI").getValue()           // 3.14159...
parser.parseExpression("T(Integer).MAX_VALUE").getValue()            // 2147483647
parser.parseExpression("T(java.util.Collections).EMPTY_LIST").getValue() // []

// Static method call
parser.parseExpression("T(java.lang.Math).sqrt(16)").getValue()     // 4.0
parser.parseExpression("T(java.lang.System).currentTimeMillis()").getValue()

// Type checking
parser.parseExpression("'hello' instanceof T(String)").getValue(Boolean.class)  // true
parser.parseExpression("42 instanceof T(Integer)").getValue(Boolean.class)      // true

// Class access
parser.parseExpression("T(String)").getValue()                      // class java.lang.String

// For non-java.lang classes, must use fully qualified name
parser.parseExpression("T(java.util.List)").getValue()              // interface java.util.List
parser.parseExpression("T(com.example.MyEnum).VALUE_ONE").getValue() // enum value
```

**`T()` with enum values:**

```java
public enum Status { ACTIVE, INACTIVE, PENDING }

// In @Value or @PreAuthorize:
@Value("#{T(com.example.Status).ACTIVE}")
private Status defaultStatus;

// In SpEL expression:
parser.parseExpression("status == T(com.example.Status).ACTIVE").getValue(ctx, Boolean.class)
```

---

### 4.7.6 — Collection Construction in SpEL

SpEL can construct collections inline:

```java
// List construction
parser.parseExpression("{1, 2, 3, 4, 5}").getValue()
// → [1, 2, 3, 4, 5]

// List of strings
parser.parseExpression("{'a', 'b', 'c'}").getValue()
// → ["a", "b", "c"]

// Map construction — {key:value, key:value}
parser.parseExpression("{name:'Alice', age:30}").getValue()
// → {name=Alice, age=30}

// Array construction
parser.parseExpression("new int[]{1,2,3}").getValue()        // [1, 2, 3]
parser.parseExpression("new String[]{'a','b'}").getValue()   // ["a", "b"]

// Multidimensional array
parser.parseExpression("new int[4][5]").getValue()           // 4x5 int array

// Object construction
parser.parseExpression("new com.example.User('Alice', 30)").getValue()
// Calls User constructor with args
```

---

### 4.7.7 — Assignment in SpEL

SpEL expressions can SET values (not just get):

```java
StandardEvaluationContext ctx = new StandardEvaluationContext(user);

// Assignment via setValue()
parser.parseExpression("name").setValue(ctx, "Bob");
// Calls user.setName("Bob")

// Assignment expression with =
parser.parseExpression("name = 'Charlie'").getValue(ctx);
// Same effect

// Can be used in complex expressions
parser.parseExpression("address.city = 'Boston'").getValue(ctx);
// Calls user.getAddress().setCity("Boston")
```

---

### 4.7.8 — SpEL in @Value Annotations

In `@Value`, SpEL uses `#{...}` delimiters (as opposed to `${...}` for property placeholders):

```java
@Component
public class SpELValueExamples {

    // Bean property access
    @Value("#{userService.defaultUser.name}")
    private String defaultUserName;

    // Static method call
    @Value("#{T(java.lang.Math).random()}")
    private double randomValue;  // NEW value each time bean is created (not each access!)

    // Arithmetic
    @Value("#{2 * T(java.lang.Math).PI}")
    private double twoPi;

    // Conditional based on property AND bean
    @Value("#{environment['app.env'] == 'prod' ? prodService.endpoint : devService.endpoint}")
    private String serviceEndpoint;

    // System property
    @Value("#{systemProperties['user.home']}")
    private String userHome;

    // Environment variable
    @Value("#{systemEnvironment['PATH']}")
    private String path;

    // Current date via Java 8 time
    @Value("#{T(java.time.LocalDate).now().toString()}")
    private String today;

    // Combine ${} and #{}:
    // ${} is resolved FIRST by PropertySourcesPlaceholderConfigurer
    // Then #{} is evaluated by SpEL
    @Value("#{${app.max-count:10} * 2}")
    private int doubleMaxCount;
    // ${app.max-count:10} → resolved to "10" → #{10 * 2} → 20

    // List of values from property
    @Value("#{'${app.servers}'.split(',')}")
    private List<String> servers;
    // ${app.servers} = "server1,server2,server3"
    // → 'server1,server2,server3'.split(',') → ["server1", "server2", "server3"]
}
```

---

### 4.7.9 — EvaluationContext — StandardEvaluationContext vs SimpleEvaluationContext

**`StandardEvaluationContext` — full-featured, potentially unsafe:**

```java
StandardEvaluationContext ctx = new StandardEvaluationContext(rootObject);
// Supports:
//   - All SpEL features
//   - Bean references (@beanName)
//   - T() type references (can instantiate ANY class)
//   - Constructor invocation
//   - Static method calls
//   - Variable assignment

// SECURITY RISK: If expression comes from user input,
// T(java.lang.Runtime).getRuntime().exec('rm -rf /') could execute!
```

**`SimpleEvaluationContext` — restricted, safe for user input:**

```java
// Spring 5.0+
EvaluationContext ctx = SimpleEvaluationContext
    .forReadOnlyDataBinding()   // or forReadWriteDataBinding()
    .withInstanceMethods()      // allow instance methods
    .build();

// SimpleEvaluationContext does NOT support:
//   - T() type references
//   - Constructor invocation (new ...)
//   - Bean references (@beanName)
//   - Static method invocation (only some allowed)
// Safer for processing user-provided expressions
```

**When to use which:**

```
StandardEvaluationContext:
    → Expressions written by developers (in @Value, @PreAuthorize, etc.)
    → Trusted expressions in configuration files
    → All built-in Spring annotations use this

SimpleEvaluationContext:
    → Expressions entered by end users (search filters, custom formulas)
    → Anywhere input comes from potentially untrusted sources
    → Data binding scenarios where type system access is not needed
```

---

### 4.7.10 — SpEL in Spring Framework Integrations

#### @Cacheable with SpEL

```java
@Service
public class ProductService {

    @Cacheable(
        value = "products",
        key = "#id",                              // simple variable
        condition = "#id > 0",                   // cache only for positive IDs
        unless = "#result == null"               // don't cache null results
    )
    public Product findById(long id) { ... }

    @Cacheable(
        value = "products",
        key = "#root.methodName + '_' + #category + '_' + #page"
    )
    public Page<Product> findByCategory(String category, int page) { ... }
    // #root.methodName → "findByCategory"
    // key → "findByCategory_electronics_0"

    @CacheEvict(
        value = "products",
        key = "#product.id",
        condition = "#product.id > 0"
    )
    public void updateProduct(Product product) { ... }
}
```

**SpEL root variables available in cache annotations:**

```
#root.methodName      → name of the cached method
#root.method          → the method itself
#root.target          → the target object
#root.targetClass     → class of the target object
#root.args            → array of method arguments
#root.caches          → array of caches targeted
#result               → return value (in @CacheEvict unless, @Cacheable unless)
#argumentName         → method argument by name (e.g., #id, #category)
```

#### Spring Security @PreAuthorize / @PostAuthorize

```java
@Service
public class DocumentService {

    @PreAuthorize("hasRole('ADMIN') or hasRole('MODERATOR')")
    public void deleteDocument(Long id) { ... }

    @PreAuthorize("hasPermission(#document, 'WRITE')")
    public void updateDocument(Document document) { ... }

    @PreAuthorize("#username == authentication.principal.username")
    public UserProfile getUserProfile(String username) { ... }
    // Only allows users to see their own profile

    @PostAuthorize("returnObject.owner == authentication.principal.username")
    public Document getDocument(Long id) { ... }
    // Returns document only if current user is the owner

    @PreFilter("filterObject.owner == authentication.principal.username")
    public void saveDocuments(List<Document> documents) { ... }
    // Filters input list to only docs owned by current user

    @PostFilter("filterObject.owner == authentication.principal.username")
    public List<Document> getAllDocuments() { ... }
    // Filters returned list to only docs owned by current user
}
```

**Available SpEL variables in Security expressions:**

```
authentication         → current Authentication object
principal             → authentication.principal
hasRole('ROLE')       → checks if user has role
hasAnyRole('R1','R2') → checks if user has any of the roles
isAuthenticated()     → true if user is authenticated
isAnonymous()         → true if user is anonymous
permitAll             → always true
denyAll               → always false
#paramName            → method parameters by name
returnObject          → return value (@PostAuthorize/@PostFilter)
filterObject          → current element in list (@Pre/PostFilter)
```

---

### 4.7.11 — SpEL Compilation Modes

```java
// MIXED (default): interpreted, compiled when stable
SpelParserConfiguration config = new SpelParserConfiguration(
    SpelCompilerMode.MIXED,
    getClass().getClassLoader());

// OFF: always interpreted (safest, slowest for hot paths)
SpelParserConfiguration config = new SpelParserConfiguration(
    SpelCompilerMode.OFF, null);

// IMMEDIATE: compile on first call (may fail if types change)
SpelParserConfiguration config = new SpelParserConfiguration(
    SpelCompilerMode.IMMEDIATE,
    getClass().getClassLoader());

SpelExpressionParser parser = new SpelExpressionParser(config);
Expression expr = parser.parseExpression("payload.amount * 0.9");
// In IMMEDIATE mode: compiled to bytecode after first evaluation
// Subsequent evaluations: JVM-compiled speed
```

**When compilation helps:**
- High-volume event processing
- Message routing in Spring Integration
- Repeated evaluation of same expression with different data

**When compilation cannot be used:**
- Expressions with `T()` type references (not yet compilable in some cases)
- Expressions with collection projections/selections
- Null-safe navigation operator

---

### Common Misconceptions

**Misconception 1:** "`${...}` and `#{...}` are the same thing in @Value."
Wrong. `${...}` is property placeholder syntax — resolved by `PropertySourcesPlaceholderConfigurer` against `PropertySources`. `#{...}` is SpEL syntax — evaluated by `ExpressionParser` against the Spring context. They are completely different mechanisms. `${...}` is resolved FIRST, then `#{...}` is evaluated.

**Misconception 2:** "SpEL property access requires explicit getter calls."
Wrong. SpEL uses Spring's `PropertyAccessor` which automatically discovers getters. `user.name` in SpEL calls `user.getName()` automatically. You do NOT write `user.getName()` — you just write `user.name`.

**Misconception 3:** "`@beanName` in SpEL requires the full class name."
Wrong. `@beanName` refers to the bean's NAME in the container (same as the name used with `getBean()`). It does NOT use class names. `@userService` resolves to `ctx.getBean("userService")`.

**Misconception 4:** "SpEL is always safe to use with user input."
Critically wrong. `StandardEvaluationContext` allows arbitrary class instantiation via `T()` and `new`. User-provided expressions evaluated with `StandardEvaluationContext` can execute arbitrary code. Always use `SimpleEvaluationContext` for user-provided expressions.

**Misconception 5:** "`T(Math).PI` requires `java.lang.Math` in `T()`."
Half wrong. For classes in `java.lang` package, you can use the simple name: `T(Math).PI` works because `java.lang` is auto-imported. For all other packages, fully qualified name is required: `T(java.util.List)`, `T(com.example.MyClass)`.

**Misconception 6:** "SpEL `.?[]` (selection) modifies the original collection."
Wrong. SpEL selection returns a NEW collection. The original collection is not modified.

**Misconception 7:** "The Elvis operator `?:` checks for null only."
Wrong. Elvis operator returns the left side if it is NOT null AND NOT empty string. An empty string `""` evaluates to the right side (default) in some cases depending on the operand type.

**Misconception 8:** "@Value SpEL is evaluated every time the value is accessed."
Wrong. `@Value` fields are injected ONCE during `postProcessProperties()` (at bean initialization). The SpEL expression is evaluated ONCE at startup. Subsequent reads of the field return the cached value. For dynamic evaluation, use `SpelExpressionParser` directly.

---

## ② CODE EXAMPLES

### Example 1 — Complete SpEL Parser Usage

```java
@Component
public class SpELDemo {

    private final ExpressionParser parser = new SpelExpressionParser();

    public void demonstrateBasicExpressions() {
        // String manipulation
        Expression expr = parser.parseExpression("'Hello World'.toUpperCase()");
        System.out.println(expr.getValue(String.class)); // "HELLO WORLD"

        // Arithmetic
        System.out.println(parser.parseExpression("10 + 5 * 2").getValue()); // 20

        // Boolean
        System.out.println(parser.parseExpression("10 > 5 && 3 < 4")
            .getValue(Boolean.class)); // true

        // Ternary
        System.out.println(parser.parseExpression("10 > 5 ? 'big' : 'small'")
            .getValue()); // "big"

        // Elvis
        String name = null;
        StandardEvaluationContext ctx = new StandardEvaluationContext();
        ctx.setVariable("name", name);
        System.out.println(parser.parseExpression("#name ?: 'anonymous'")
            .getValue(ctx)); // "anonymous"

        // Safe navigation
        User user = null;
        ctx.setVariable("user", user);
        System.out.println(parser.parseExpression("#user?.address?.city")
            .getValue(ctx)); // null (no NPE)
    }
}
```

---

### Example 2 — Root Object and Property Access

```java
public class Person {
    private String firstName;
    private String lastName;
    private int age;
    private Address address;
    private List<String> hobbies;
    // getters and setters...
}

public class SpELPropertyAccessDemo {

    private final ExpressionParser parser = new SpelExpressionParser();

    public void demonstrate() {
        Person person = new Person();
        person.setFirstName("John");
        person.setLastName("Doe");
        person.setAge(30);
        person.setAddress(new Address("New York", "NY"));
        person.setHobbies(Arrays.asList("reading", "coding", "gaming"));

        StandardEvaluationContext ctx = new StandardEvaluationContext(person);

        // Property access via getter
        System.out.println(parser.parseExpression("firstName")
            .getValue(ctx));                // "John"

        // Nested property
        System.out.println(parser.parseExpression("address.city")
            .getValue(ctx));                // "New York"

        // Method call on property
        System.out.println(parser.parseExpression("firstName.toUpperCase()")
            .getValue(ctx));                // "JOHN"

        // Concatenation
        System.out.println(parser.parseExpression("firstName + ' ' + lastName")
            .getValue(ctx));                // "John Doe"

        // Collection access
        System.out.println(parser.parseExpression("hobbies[0]")
            .getValue(ctx));                // "reading"

        // Collection size
        System.out.println(parser.parseExpression("hobbies.size()")
            .getValue(ctx));                // 3

        // Boolean on property
        System.out.println(parser.parseExpression("age >= 18 ? 'adult' : 'minor'")
            .getValue(ctx));                // "adult"

        // Setting value
        parser.parseExpression("firstName").setValue(ctx, "Jane");
        System.out.println(person.getFirstName()); // "Jane"
    }
}
```

---

### Example 3 — Collection Selection and Projection

```java
public class CollectionOperationsDemo {

    private final ExpressionParser parser = new SpelExpressionParser();

    public void demonstrate() {
        List<Person> people = Arrays.asList(
            new Person("Alice", 17, "F"),
            new Person("Bob", 25, "M"),
            new Person("Charlie", 15, "M"),
            new Person("Diana", 30, "F"),
            new Person("Eve", 22, "F")
        );

        StandardEvaluationContext ctx = new StandardEvaluationContext();
        ctx.setVariable("people", people);

        // Select adults (age >= 18)
        List<Person> adults = (List<Person>)
            parser.parseExpression("#people.?[age >= 18]").getValue(ctx);
        System.out.println("Adults: " + adults.size()); // 3

        // Select females only
        List<Person> females = (List<Person>)
            parser.parseExpression("#people.?[gender == 'F']").getValue(ctx);
        System.out.println("Females: " + females.size()); // 3

        // First adult
        Person firstAdult = (Person)
            parser.parseExpression("#people.^[age >= 18]").getValue(ctx);
        System.out.println("First adult: " + firstAdult.getName()); // "Bob"

        // Last adult
        Person lastAdult = (Person)
            parser.parseExpression("#people.$[age >= 18]").getValue(ctx);
        System.out.println("Last adult: " + lastAdult.getName()); // "Eve"

        // Project names of all adults
        List<String> adultNames = (List<String>)
            parser.parseExpression("#people.?[age >= 18].![name]").getValue(ctx);
        System.out.println("Adult names: " + adultNames); // [Bob, Diana, Eve]

        // Projection: get all names uppercase
        List<String> upperNames = (List<String>)
            parser.parseExpression("#people.![name.toUpperCase()]").getValue(ctx);
        System.out.println("Upper names: " + upperNames);
        // [ALICE, BOB, CHARLIE, DIANA, EVE]

        // Complex: names of adult females sorted
        List<String> adultFemaleNames = (List<String>)
            parser.parseExpression("#people.?[age >= 18 && gender == 'F'].![name]")
                .getValue(ctx);
        System.out.println("Adult females: " + adultFemaleNames); // [Diana, Eve]
    }
}
```

---

### Example 4 — @Value with SpEL — All Patterns

```java
@Service
public class ConfigurationBean {

    // Basic bean property
    @Value("#{configService.maxRetries}")
    private int maxRetries;

    // Method call on bean
    @Value("#{configService.getProperty('timeout')}")
    private String timeout;

    // Static constant
    @Value("#{T(java.lang.Integer).MAX_VALUE}")
    private int maxInt;

    // Current time (evaluated ONCE at startup)
    @Value("#{T(java.lang.System).currentTimeMillis()}")
    private long startupTime;

    // System property
    @Value("#{systemProperties['java.io.tmpdir']}")
    private String tempDir;

    // Environment variable with default
    @Value("#{systemEnvironment['JAVA_HOME'] ?: '/usr/java/default'}")
    private String javaHome;

    // Property + SpEL combination
    @Value("#{${app.pool.min:5} + ${app.pool.extra:2}}")
    private int poolSize; // e.g., 5 + 2 = 7

    // Split string property into list
    @Value("#{'${app.allowed.origins:localhost}'.split(',')}")
    private List<String> allowedOrigins;

    // Environment-based selection
    @Value("#{environment['spring.profiles.active'] != null && "
         + "environment['spring.profiles.active'].contains('prod') "
         + "? 'STRICT' : 'RELAXED'}")
    private String securityMode;

    // Conditional with null-safe
    @Value("#{featureFlags?.isEnabled('newUI') ?: false}")
    private boolean newUiEnabled;
}
```

---

### Example 5 — Bean References in SpEL

```java
@Service("pricingService")
public class PricingService {
    public double getDiscount(String tier) {
        return switch (tier) {
            case "GOLD"     -> 0.20;
            case "SILVER"   -> 0.10;
            case "BRONZE"   -> 0.05;
            default         -> 0.0;
        };
    }

    public double applyDiscount(double price, double discount) {
        return price * (1 - discount);
    }
}

@Component
public class OrderConfig {

    // Access another Spring bean's method via @beanName
    @Value("#{pricingService.getDiscount('GOLD')}")
    private double goldDiscount; // 0.20

    // Chain calls on bean
    @Value("#{pricingService.applyDiscount(100.0, pricingService.getDiscount('SILVER'))}")
    private double discountedPrice; // 100.0 * (1 - 0.10) = 90.0

    // Check bean property
    @Value("#{@configService.featureEnabled ? @premiumService.endpoint : @basicService.endpoint}")
    private String activeEndpoint;
    // @configService → beanFactory.getBean("configService")
}
```

---

### Example 6 — Collection Construction

```java
@Component
public class DataInitializer {

    // Inline list construction
    @Value("#{{'dev', 'test', 'staging'}}")
    private List<String> nonProdEnvironments;

    // Inline map construction
    @Value("#{{server1: 8080, server2: 8081, server3: 8082}}")
    private Map<String, Integer> serverPorts;

    // Filtered collection
    @Value("#{@userRepository.findAll().?[active == true].![username]}")
    private List<String> activeUsernames;

    public void demonstrateConstruction() {
        ExpressionParser parser = new SpelExpressionParser();

        // Construct list
        List<Integer> nums = (List<Integer>)
            parser.parseExpression("{1, 2, 3, 4, 5}").getValue();
        System.out.println(nums); // [1, 2, 3, 4, 5]

        // Construct map
        Map<String, Integer> map = (Map<String, Integer>)
            parser.parseExpression("{a:1, b:2, c:3}").getValue();
        System.out.println(map); // {a=1, b=2, c=3}

        // Construct new object
        Object user = parser.parseExpression(
            "new com.example.User('Alice', 25)").getValue();
        System.out.println(user); // User{name=Alice, age=25}

        // New array
        int[] arr = (int[])
            parser.parseExpression("new int[]{10, 20, 30}").getValue();
        System.out.println(Arrays.toString(arr)); // [10, 20, 30]
    }
}
```

---

### Example 7 — SpEL in Caching

```java
@Service
public class ProductCatalogService {

    @Cacheable(
        cacheNames = "catalog",
        key = "'product_' + #productId",
        condition = "#productId > 0 && #includeInactive == false",
        unless = "#result?.discontinued == true"
    )
    public Product getProduct(long productId, boolean includeInactive) {
        return productRepository.findById(productId);
    }

    @Cacheable(
        cacheNames = "catalog",
        key = "#root.methodName + '_' + #category + '_' + #page + '_' + #size"
    )
    public Page<Product> getProductsByCategory(String category, int page, int size) {
        return productRepository.findByCategory(category, PageRequest.of(page, size));
    }

    @CachePut(
        cacheNames = "catalog",
        key = "'product_' + #product.id"
    )
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(
        cacheNames = "catalog",
        key = "'product_' + #productId",
        condition = "#productId > 0"
    )
    public void deleteProduct(long productId) {
        productRepository.deleteById(productId);
    }

    // Evict ALL entries in catalog cache
    @CacheEvict(cacheNames = "catalog", allEntries = true)
    public void refreshCatalog() {
        // triggers full cache eviction
    }
}
```

---

### Example 8 — SimpleEvaluationContext for User Input

```java
@Service
public class SafeExpressionService {

    // UNSAFE: StandardEvaluationContext — never use with user input
    public Object evaluateUnsafe(String userExpression, Object data) {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext ctx = new StandardEvaluationContext(data);
        return parser.parseExpression(userExpression).getValue(ctx);
        // User could type: T(java.lang.Runtime).getRuntime().exec('malicious')
        // CRITICAL SECURITY VULNERABILITY
    }

    // SAFE: SimpleEvaluationContext — for user-provided expressions
    public Object evaluateSafe(String userExpression, Object data) {
        ExpressionParser parser = new SpelExpressionParser();
        EvaluationContext ctx = SimpleEvaluationContext
            .forReadOnlyDataBinding()  // read-only: no setValue()
            .withInstanceMethods()     // allow instance method calls
            .withRootObject(data)
            .build();

        try {
            return parser.parseExpression(userExpression).getValue(ctx);
        } catch (EvaluationException | ParseException e) {
            throw new InvalidExpressionException("Invalid expression: " + userExpression, e);
        }
    }

    // Example: user-defined filter expressions for reports
    public List<Order> filterOrders(List<Order> orders, String filterExpression) {
        ExpressionParser parser = new SpelExpressionParser();
        EvaluationContext ctx = SimpleEvaluationContext
            .forReadOnlyDataBinding()
            .withInstanceMethods()
            .build();

        return orders.stream()
            .filter(order -> {
                try {
                    EvaluationContext orderCtx = SimpleEvaluationContext
                        .forReadOnlyDataBinding()
                        .withInstanceMethods()
                        .withRootObject(order)
                        .build();
                    return Boolean.TRUE.equals(
                        parser.parseExpression(filterExpression).getValue(orderCtx, Boolean.class));
                } catch (Exception e) {
                    return false; // invalid expression → exclude
                }
            })
            .collect(Collectors.toList());
    }
}
```

---

### Example 9 — SpEL Template Expressions

```java
// Template expressions mix literal text with SpEL blocks
ExpressionParser parser = new SpelExpressionParser();
ParserContext templateContext = new TemplateParserContext(); // default: #{...} delimiters

Expression template = parser.parseExpression(
    "Hello #{name}, you have #{messages.size()} messages. " +
    "Your score is #{score > 100 ? 'excellent' : 'good'}.",
    templateContext);

User user = new User("Alice", Arrays.asList("msg1", "msg2", "msg3"), 150);
StandardEvaluationContext ctx = new StandardEvaluationContext(user);

String result = template.getValue(ctx, String.class);
System.out.println(result);
// "Hello Alice, you have 3 messages. Your score is excellent."

// Custom delimiters
ParserContext customContext = new ParserContext() {
    @Override public String getExpressionPrefix() { return "${"; }
    @Override public String getExpressionSuffix() { return "}"; }
    @Override public boolean isTemplate()         { return true; }
};

Expression customTemplate = parser.parseExpression(
    "User: ${firstName} ${lastName}, Age: ${age}",
    customContext);
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
What is the difference between `${app.timeout}` and `#{...}` in `@Value`?

A) They are identical — both resolve property values
B) `${...}` resolves property placeholders from `PropertySources`; `#{...}` is SpEL evaluated against the Spring context
C) `${...}` is SpEL; `#{...}` is for property placeholders
D) `${...}` works in Java config only; `#{...}` works in XML only

**Answer: B**
`${...}` uses `PropertySourcesPlaceholderConfigurer` to look up keys in `PropertySources` (property files, env vars, etc.). `#{...}` is SpEL — evaluated by `ExpressionParser` against the Spring application context.

---

**Q2 — Code Output Prediction:**

```java
ExpressionParser parser = new SpelExpressionParser();

System.out.println(parser.parseExpression("2 + 3 * 4").getValue());
System.out.println(parser.parseExpression("{1,2,3}.size()").getValue());
System.out.println(parser.parseExpression("T(Math).PI > 3").getValue());
System.out.println(parser.parseExpression("'hello' ?: 'world'").getValue());
```

What is the output?

A)
```
14
3
true
hello
```
B)
```
20
3
true
world
```
C)
```
14
3
true
world
```
D)
```
14
3
false
hello
```

**Answer: A**
`2 + 3 * 4` → operator precedence: `*` before `+` → `2 + 12 = 14`. `{1,2,3}.size()` → 3. `T(Math).PI > 3` → 3.14... > 3 → true. `'hello' ?: 'world'` → `'hello'` is not null → returns `'hello'`.

---

**Q3 — Select all that apply:**
Which of the following SpEL expressions are syntactically VALID? (Select ALL that apply)

A) `T(java.lang.Math).sqrt(16.0)`
B) `{'key1' : 'val1', 'key2' : 'val2'}`
C) `users.?[age > 18].![name]`
D) `#list.select[#this > 5]`
E) `name?.toUpperCase()`
F) `T(Math).PI`

**Answer: A, B, C, E, F**
D is wrong — the correct selection syntax is `.?[...]`, not `.select[...]`. A: valid T() static call. B: valid map construction. C: valid selection + projection chain. E: valid safe navigation. F: valid T() with java.lang auto-import.

---

**Q4 — Trick Question:**

```java
@Value("#{T(java.lang.System).currentTimeMillis()}")
private long startupTime;
```

If `getStartupTime()` is called 1 hour after bean creation, what value does it return?

A) The current timestamp at the time of the call (updated every call)
B) The timestamp at the time the bean was created (SpEL evaluated once)
C) Zero — long fields default to 0
D) `IllegalStateException` — T() cannot call System.currentTimeMillis()

**Answer: B**
`@Value` is resolved ONCE during `postProcessProperties()` at bean initialization. The SpEL expression is evaluated once at startup. The `long` field stores the result — it is NOT a live expression. Calling `getStartupTime()` always returns the value captured at startup, regardless of when it is called.

---

**Q5 — MCQ:**
What does the expression `#users.?[age >= 18].![email]` return when applied to a `List<User>`?

A) The first user with age >= 18
B) A boolean — true if any user has age >= 18
C) A list of email addresses of users where age >= 18
D) A list of User objects filtered to age >= 18

**Answer: C**
`.?[age >= 18]` selects users where age >= 18 (returns `List<User>`). `.![email]` projects the `email` property from each user in the list (returns `List<String>`). Combined: list of email strings for adult users.

---

**Q6 — Security Scenario:**

```java
@PostMapping("/filter")
public List<Order> filterOrders(@RequestBody FilterRequest request) {
    ExpressionParser parser = new SpelExpressionParser();
    StandardEvaluationContext ctx = new StandardEvaluationContext(orderService);
    List<Order> result = (List<Order>)
        parser.parseExpression(request.getFilterExpression()).getValue(ctx);
    return result;
}
```

A malicious user sends: `T(java.lang.Runtime).getRuntime().exec('rm -rf /')`

What happens?

A) SpEL sanitizes the expression and ignores dangerous calls
B) Spring Security blocks the expression
C) The expression executes — arbitrary command execution occurs on the server
D) `AccessDeniedException` — T() not allowed without explicit permission

**Answer: C**
`StandardEvaluationContext` allows full SpEL evaluation including `T()` type references and system command execution. This is a CRITICAL security vulnerability — arbitrary code execution. Always use `SimpleEvaluationContext` for user-provided expressions.

---

**Q7 — Select all that apply:**
Which SpEL features are NOT available in `SimpleEvaluationContext`? (Select ALL that apply)

A) Property access (e.g., `user.name`)
B) `T()` type references
C) Instance method calls (e.g., `name.toUpperCase()`)
D) `new` object construction
E) Arithmetic operators
F) `@beanName` bean references

**Answer: B, D, F**
`SimpleEvaluationContext` allows: property access (A), instance method calls (C), arithmetic (E), and basic collection operations. It does NOT allow: `T()` type references (B), `new` construction (D), or bean references (F). These restrictions prevent arbitrary code execution.

---

**Q8 — Code Output Prediction:**

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext ctx = new StandardEvaluationContext();
ctx.setVariable("numbers", numbers);

Object first  = parser.parseExpression("#numbers.^[#this > 5]").getValue(ctx);
Object last   = parser.parseExpression("#numbers.$[#this > 5]").getValue(ctx);
Object select = parser.parseExpression("#numbers.?[#this % 2 == 0]").getValue(ctx);
Object project = parser.parseExpression("#numbers.![#this * 2]").getValue(ctx);

System.out.println(first);
System.out.println(last);
System.out.println(select);
System.out.println(project);
```

What is the output?

A)
```
6
10
[2, 4, 6, 8, 10]
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```
B)
```
5
10
[2, 4, 6, 8, 10]
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```
C)
```
6
10
[1, 3, 5, 7, 9]
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```
D)
```
true
true
[2, 4, 6, 8, 10]
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

**Answer: A**
`.^[#this > 5]` = first element > 5 = 6. `.$[#this > 5]` = last element > 5 = 10. `.?[#this % 2 == 0]` = filter evens = [2,4,6,8,10]. `.![#this * 2]` = project each * 2 = [2,4,6,8,10,12,14,16,18,20].

---

**Q9 — MCQ:**
In `@Cacheable(key = "#root.methodName + '_' + #id")`, what does `#root.methodName` resolve to?

A) The class name containing the method
B) The name of the cached method as a String
C) The full qualified method signature
D) The cache name

**Answer: B**
`#root.methodName` is a built-in SpEL variable in cache annotations that resolves to the name of the method being cached (e.g., `"findById"`). This allows the cache key to incorporate the method name for uniqueness.

---

**Q10 — Drag-and-Drop: Match SpEL syntax to description:**

Syntax: `?.`, `.?[]`, `.![]`, `.^[]`, `.$[]`, `?:`, `T()`, `@bean`

Descriptions:
- Type reference for static access
- Bean reference from ApplicationContext
- Null-safe property navigation
- Collection selection (filter)
- Collection projection (transform)
- First matching element
- Last matching element
- Elvis operator (null-safe default)

**Answer:**
- `T()` → Type reference for static access
- `@bean` → Bean reference from ApplicationContext
- `?.` → Null-safe property navigation
- `.?[]` → Collection selection (filter)
- `.![]` → Collection projection (transform)
- `.^[]` → First matching element
- `.$[]` → Last matching element
- `?:` → Elvis operator (null-safe default)

---

## ④ TRICK ANALYSIS

**Trap 1: ${} and #{} are interchangeable**
Completely wrong and dangerous. `${key}` = property placeholder (resolved by `PropertySourcesPlaceholderConfigurer` against `PropertySources`). `#{expression}` = SpEL (evaluated against Spring context). `${key}` is processed FIRST during BFPP phase. `#{expression}` is processed during `postProcessProperties()`. You CAN nest them: `#{${app.count:5} * 2}` — property resolved first, then SpEL evaluated.

**Trap 2: @Value SpEL is evaluated on every field access**
Dead wrong. `@Value` fields are injected ONCE at bean initialization time. The SpEL expression is evaluated ONCE. The result is stored in the field. `@Value("#{T(System).currentTimeMillis()}")` gives you startup time, not current time on every call. For dynamic evaluation, use `ExpressionParser` programmatically.

**Trap 3: T(Math) requires java.lang.Math**
`T(Math)` works because `java.lang` is auto-imported. `T(Math)` and `T(java.lang.Math)` are equivalent. But `T(List)` does NOT work — must use `T(java.util.List)` for non-`java.lang` classes.

**Trap 4: .?[selection] modifies original collection**
Wrong. Selection creates a NEW collection. The original `List` is not mutated. `#list.?[#this > 5]` returns a new `ArrayList` containing filtered elements. The original `#list` is unchanged.

**Trap 5: Elvis operator checks null only**
The Elvis operator `?:` is defined as returning the left side if it evaluates to NOT null. An empty string `""` is not null — it returns `""`, not the default. However, `Boolean.FALSE` is not null — it returns `false`, not the default. `null ?: "default"` returns `"default"`. `"" ?: "default"` returns `""`. `false ?: true` returns `false`.

**Trap 6: StandardEvaluationContext is always safe**
CRITICAL security trap. `StandardEvaluationContext` with user-provided expressions enables Remote Code Execution (RCE). `T(java.lang.Runtime).getRuntime().exec('command')` executes OS commands. Always use `SimpleEvaluationContext` for user input. Spring had CVE-2022-22950 related to SpEL injection.

**Trap 7: Safe navigation .? prevents all NPEs**
Only prevents NPEs in the chain WHERE it is used. `user?.address.city` — if `user` is null: returns null (no NPE). If `address` is null: NPE! Must use `user?.address?.city` to protect each level separately.

**Trap 8: @beanName requires the class name**
Wrong. `@beanName` resolves the bean NAME (as registered in the container, e.g., `"orderService"` for `@Service("orderService")` or `"orderServiceImpl"` for the default name). It is NOT the class name. `@OrderService` would fail if the bean name is `"orderServiceImpl"`.

**Trap 9: Collection selection .?[] returns same type as input**
Nuanced. SpEL selection on a `List` returns an `ArrayList`. Selection on a `Map` (`map.?[value > 5]`) filters the map entries (key-value pairs where value > 5) and returns a new `Map`. The return type depends on the source type.

**Trap 10: T() works for any class**
Almost true but with limitations. `T()` can reference any class accessible in the ClassLoader. However, SpEL compilation does NOT support all T() usages. In `SimpleEvaluationContext`, T() is completely blocked. In restricted environments (OSGi, etc.), class accessibility may prevent T() from working.

---

## ⑤ SUMMARY SHEET

### SpEL Syntax Quick Reference

```
LITERALS:
'string'           → String literal
42, 3.14, 0xFF     → Numeric literals
true, false        → Boolean
null               → Null

OPERATORS:
+, -, *, /, %      → Arithmetic (^ = power)
==, !=, <, >, <=, >= → Comparison
eq, ne, lt, gt, le, ge → Symbolic comparison (XML-safe)
&&, ||, !           → Logical (also: and, or, not)
?:                  → Elvis (null-safe default)
?:                  → Ternary condition (condition ? x : y)
=                   → Assignment
instanceof T(Type)  → Type checking

NAVIGATION:
obj.property       → Property (calls getter)
obj?.property      → Null-safe navigation
obj[index]         → Array/List index
obj['key']         → Map key

COLLECTION:
{1, 2, 3}          → List literal
{key:val}          → Map literal
.?[expr]           → Selection (filter)
.^[expr]           → First matching element
.$[expr]           → Last matching element
.![expr]           → Projection (transform)
#this              → Current element in collection operations

TYPE SYSTEM:
T(FullyQualifiedClass) → Type reference (static methods/fields)
new ClassName(args)    → Constructor call
'x' instanceof T(String) → instanceof check

VARIABLES:
#variableName      → User-defined variable
#this              → Current object in collection operations
#root              → Root evaluation object

BEAN REFERENCES:
@beanName          → Spring bean from ApplicationContext

@VALUE SPECIFIC:
${key}             → Property placeholder (PropertySources)
${key:default}     → Property with default
#{expression}      → SpEL expression
#{${key} + 1}      → Property inside SpEL
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `${}` vs `#{}` | `${}` = property placeholder; `#{}` = SpEL expression |
| `@Value` evaluation timing | Once at bean initialization — NOT on every field access |
| `T(Math)` | java.lang auto-imported — works without full package |
| `T(java.util.List)` | Non-java.lang REQUIRES fully qualified name |
| Collection selection | Returns NEW collection — original unchanged |
| Elvis `?:` | Returns left if NOT null — empty string is NOT null |
| Safe navigation `?.` | Each level needs its own `?.` for full protection |
| `@beanName` | Resolves by bean NAME in context — not class name |
| `StandardEvaluationContext` | Full-featured but UNSAFE for user input |
| `SimpleEvaluationContext` | Restricted — safe for user input (no T(), no new, no @bean) |
| `#root.methodName` in cache | Name of the cached method |
| `#result` in cache | Return value (in @Cacheable unless, @CacheEvict condition) |
| `${} inside #{}` | Property resolved first, then SpEL evaluated |

---

### EvaluationContext Comparison

```
┌─────────────────────────────┬──────────────────┬──────────────────┐
│ Feature                     │ StandardCtx      │ SimpleCtx        │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ Property access             │ YES              │ YES              │
│ Method calls                │ YES              │ YES (instance)   │
│ Arithmetic operators        │ YES              │ YES              │
│ T() type references         │ YES              │ NO               │
│ new object construction     │ YES              │ NO               │
│ @beanName references        │ YES (with resolver│ NO              │
│ Assignment (setValue)       │ YES              │ Optional         │
│ Static method calls         │ YES              │ NO               │
│ User input safe?            │ NO — RCE risk    │ YES              │
└─────────────────────────────┴──────────────────┴──────────────────┘
```

---

### Interview One-Liners

- "SpEL uses `#{...}` delimiters in `@Value`, while property placeholders use `${...}`. `${...}` is resolved first by `PropertySourcesPlaceholderConfigurer`; `#{...}` is evaluated by `ExpressionParser` against the Spring context."
- "`@Value` fields are injected ONCE at bean initialization — the SpEL expression is evaluated once and the result stored. It is NOT a live expression evaluated on every field access."
- "`T(Math).PI` works because `java.lang` is auto-imported in SpEL. For other packages, use the fully qualified name: `T(java.util.Collections).EMPTY_LIST`."
- "SpEL collection selection `.?[condition]` filters and returns a NEW collection; `.![expression]` projects each element into a new collection. Original collection is never mutated."
- "The Elvis operator `?:` returns the left operand if it is NOT null — an empty string `\"\"` is not null and IS returned. Only `null` triggers the right-hand default."
- "NEVER evaluate user-provided SpEL expressions with `StandardEvaluationContext` — `T(java.lang.Runtime).getRuntime().exec()` enables Remote Code Execution. Use `SimpleEvaluationContext` for user input."
- "`@beanName` in SpEL resolves the bean by its registered NAME in the `ApplicationContext` (same as `ctx.getBean('beanName')`), not by class name."
- "SpEL in `@Cacheable` key expressions has built-in root variables: `#root.methodName`, `#root.target`, `#root.args`, and method parameters by name (e.g., `#id`, `#category`)."

---
