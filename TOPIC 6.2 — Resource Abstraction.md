# TOPIC 6.2 — Resource Abstraction

---

## ① CONCEPTUAL EXPLANATION

### What is the Resource Abstraction?

Spring's `Resource` abstraction provides a unified interface for accessing low-level resources — files, classpath entries, URLs, servlet context paths — regardless of their physical location or access mechanism.

```
Without Resource abstraction:
    File file = new File("config.xml");               // only works for filesystem
    URL url = new URL("http://server/config.xml");    // only works for URLs
    InputStream is = getClass().getResourceAsStream("config.xml"); // only classpath

With Resource abstraction:
    Resource r1 = new ClassPathResource("config.xml");
    Resource r2 = new FileSystemResource("/etc/app/config.xml");
    Resource r3 = new UrlResource("http://server/config.xml");
    Resource r4 = new ServletContextResource(ctx, "/WEB-INF/config.xml");
    // All use the same Resource interface — uniform API
```

---

### 6.2.1 — The Resource Interface

```java
public interface Resource extends InputStreamSource {

    // Core capability — from InputStreamSource:
    InputStream getInputStream() throws IOException;

    // Existence check
    boolean exists();

    // Readability (exists AND is readable)
    boolean isReadable();

    // Open stream check (true = getInputStream() returns a FRESH stream each time)
    boolean isOpen();         // UrlResource for file:// → false, InputStreamResource → true

    // File-based checks
    boolean isFile();         // true if backed by java.io.File
    File getFile() throws IOException;  // throws if not file-backed

    // URL/URI representations
    URL getURL() throws IOException;
    URI getURI() throws IOException;

    // NIO support
    ReadableByteChannel readableChannel() throws IOException;

    // Metadata
    long contentLength() throws IOException;
    long lastModified() throws IOException;
    String getFilename();                    // "config.xml", null for some types
    String getDescription();                 // human-readable description

    // Relative resource creation
    Resource createRelative(String relativePath) throws IOException;
}
```

**Key design note:** `Resource` does NOT define write capabilities. It is read-oriented. For writing, use `WritableResource` (extends `Resource`).

---

### 6.2.2 — Resource Implementations — Complete Taxonomy

```
Resource (interface)
├── AbstractResource (base class — provides default implementations)
│   ├── ClassPathResource          → classpath: resources
│   ├── FileSystemResource         → filesystem files (implements WritableResource)
│   ├── UrlResource                → any URL (http:, file:, ftp:, etc.)
│   ├── ServletContextResource     → web application resources
│   ├── InputStreamResource        → wraps existing InputStream
│   ├── ByteArrayResource          → wraps byte[] in memory
│   ├── PathResource               → java.nio.file.Path (implements WritableResource)
│   └── VfsResource                → JBoss VFS (virtual filesystem)
│
└── WritableResource (extends Resource)
    ├── FileSystemResource
    └── PathResource
```

#### ClassPathResource

```java
// Loads from classpath
Resource r = new ClassPathResource("data/config.xml");
// Equivalent: classpath:data/config.xml

// With specific ClassLoader
Resource r2 = new ClassPathResource("config.xml", MyClass.class.getClassLoader());

// With anchor class (relative to class's package)
Resource r3 = new ClassPathResource("config.xml", MyService.class);
// Resolves relative to: com/example/service/config.xml

// Key characteristics:
r.exists()      // true if resource on classpath
r.isFile()      // true if classpath entry is a filesystem file (not in JAR)
r.getFile()     // works if not in JAR — throws if inside JAR
r.getURL()      // always works — jar:file://... or file://...
r.isOpen()      // false — each getInputStream() returns a fresh stream
```

#### FileSystemResource

```java
// Absolute path
Resource r = new FileSystemResource("/etc/myapp/config.properties");

// Relative path (relative to JVM working directory)
Resource r2 = new FileSystemResource("config/settings.xml");

// From File object
Resource r3 = new FileSystemResource(new File("/etc/myapp/config.properties"));

// WritableResource — can write
WritableResource wr = (WritableResource) r;
OutputStream os = wr.getOutputStream();
// write to file...

// Key characteristics:
r.isFile()      // true — always backed by java.io.File
r.getFile()     // always works
r.isOpen()      // false
```

#### UrlResource

```java
// HTTP URL
Resource r1 = new UrlResource("http://config-server/app/config.json");

// File URL (alternative to FileSystemResource)
Resource r2 = new UrlResource("file:/etc/myapp/config.properties");

// FTP URL
Resource r3 = new UrlResource("ftp://server/config.xml");

// Key characteristics:
r1.isFile()     // false for http:// URLs
r2.isFile()     // true for file:// URLs
r1.getURL()     // returns the URL
r1.isOpen()     // false for most URL types
```

#### InputStreamResource

```java
// Wraps an ALREADY OPEN InputStream
// isOpen() returns TRUE — stream can only be read ONCE
Resource r = new InputStreamResource(someInputStream);

// Key limitation: can only be read ONCE
r.isOpen()      // TRUE — indicates single-use
r.getInputStream() // works first time
r.getInputStream() // MAY fail on second call (stream may be exhausted)

// contentLength() and isFile() throw UnsupportedOperationException
// Use sparingly — prefer ByteArrayResource for in-memory data
```

#### ByteArrayResource

```java
// In-memory resource from byte array
byte[] data = "Hello World".getBytes(StandardCharsets.UTF_8);
Resource r = new ByteArrayResource(data);

// Optional description
Resource r2 = new ByteArrayResource(data, "Configuration data");

// Key characteristics:
r.exists()      // true
r.isOpen()      // false — each getInputStream() wraps data in ByteArrayInputStream
r.contentLength() // data.length
r.getFile()     // throws FileNotFoundException — not file-backed
```

#### PathResource (NIO)

```java
// From java.nio.file.Path
Path path = Paths.get("/etc/myapp/config.properties");
Resource r = new PathResource(path);

// From String path
Resource r2 = new PathResource("/etc/myapp/config.properties");

// Implements WritableResource
WritableResource wr = (WritableResource) r;
OutputStream os = wr.getOutputStream();

// NIO channel support
ReadableByteChannel channel = r.readableChannel();
```

---

### 6.2.3 — ResourceLoader Interface

`ResourceLoader` is the interface for loading `Resource` objects from location strings:

```java
public interface ResourceLoader {

    String CLASSPATH_URL_PREFIX = "classpath:";

    // Load resource from location string
    Resource getResource(String location);

    // The ClassLoader used for loading
    ClassLoader getClassLoader();
}
```

`ApplicationContext` extends `ResourceLoader` — the context IS a resource loader.

**ResourceLoader resolution rules:**

```
Location string prefix → Resource type created:
    "classpath:data/config.xml"    → ClassPathResource
    "file:/etc/app/config.xml"     → FileSystemResource (via UrlResource)
    "http://server/config.xml"     → UrlResource
    "https://server/config.xml"    → UrlResource
    "/WEB-INF/config.xml"          → in web context → ServletContextResource
                                     in non-web context → ClassPathResource or FileSystemResource
    "data/config.xml" (no prefix)  → depends on ApplicationContext type:
        ClassPathXmlApplicationContext → ClassPathResource
        FileSystemXmlApplicationContext → FileSystemResource
        AnnotationConfigApplicationContext → ClassPathResource
```

**No-prefix resolution depends on context type:**

```java
// ClassPathXmlApplicationContext:
ctx.getResource("config.xml")  // → ClassPathResource("config.xml")

// FileSystemXmlApplicationContext:
ctx.getResource("config.xml")  // → FileSystemResource("config.xml")

// AnnotationConfigApplicationContext:
ctx.getResource("config.xml")  // → ClassPathResource("config.xml")

// ALWAYS explicit with prefix:
ctx.getResource("classpath:config.xml")   // → ClassPathResource (always)
ctx.getResource("file:config.xml")        // → FileSystemResource (always)
```

---

### 6.2.4 — ResourcePatternResolver — Wildcard Support

`ResourcePatternResolver` extends `ResourceLoader` to support Ant-style wildcard patterns:

```java
public interface ResourcePatternResolver extends ResourceLoader {

    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    // Returns ALL matching resources
    Resource[] getResources(String locationPattern) throws IOException;
}
```

**`classpath:` vs `classpath*:`:**

```
classpath:com/example/**/*.xml
    → Finds resources in the FIRST classpath entry that matches
    → Will NOT search ALL jars/directories
    → Useful when resource is in a known single location

classpath*:com/example/**/*.xml
    → Finds resources across ALL classpath entries
    → Searches all JARs, all directories
    → Essential for plugin architectures, multi-jar scanning
    → Significantly slower (scans everything)
```

**Ant-style patterns:**

```
?   → matches exactly one character
*   → matches zero or more characters within path segment
**  → matches zero or more path segments (directories)

Examples:
classpath:com/example/service/*Service.xml   → all *Service.xml in service/
classpath:com/example/**/*.xml               → all .xml in example and subdirs
classpath*:META-INF/spring/*.xml             → all spring/*.xml in ALL jars
classpath*:com/example/**/config-*.properties → all config-*.properties
```

**`PathMatchingResourcePatternResolver` — the standard implementation:**

```java
// Programmatic usage:
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

// Load all XML configs from service package
Resource[] resources = resolver.getResources("classpath*:com/example/service/**/*.xml");

// Load all spring.factories files from all JARs
Resource[] factories = resolver.getResources("classpath*:META-INF/spring.factories");

// Load all properties files
Resource[] props = resolver.getResources("classpath*:/**/*.properties");
```

---

### 6.2.5 — ResourceLoaderAware

Beans that need a `ResourceLoader` can implement `ResourceLoaderAware`:

```java
@Component
public class ConfigLoader implements ResourceLoaderAware {

    private ResourceLoader resourceLoader;

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
        // Called during Aware processing (before @PostConstruct)
        // The ResourceLoader is the ApplicationContext itself
    }

    public Properties loadConfig(String location) {
        Resource resource = resourceLoader.getResource(location);
        Properties props = new Properties();
        try (InputStream is = resource.getInputStream()) {
            props.load(is);
        } catch (IOException e) {
            throw new RuntimeException("Cannot load: " + location, e);
        }
        return props;
    }
}

// Alternative: inject ResourceLoader via @Autowired
@Component
public class ConfigLoader {

    @Autowired
    private ResourceLoader resourceLoader; // ApplicationContext injected

    // OR inject ApplicationContext directly:
    @Autowired
    private ApplicationContext ctx; // also implements ResourceLoader
}
```

---

### 6.2.6 — @Value with Resource Injection

`Resource` objects can be injected directly via `@Value`:

```java
@Component
public class DataLoader {

    // Inject Resource directly
    @Value("classpath:data/initial-data.sql")
    private Resource sqlScript;

    @Value("classpath:templates/email-welcome.html")
    private Resource emailTemplate;

    @Value("file:/etc/myapp/external-config.properties")
    private Resource externalConfig;

    @Value("${config.location:classpath:default-config.properties}")
    private Resource dynamicConfig;  // location from property

    @PostConstruct
    public void load() throws IOException {
        // Use the injected resource
        String sql = new String(sqlScript.getInputStream().readAllBytes(),
            StandardCharsets.UTF_8);
        jdbcTemplate.execute(sql);

        String html = new String(emailTemplate.getInputStream().readAllBytes(),
            StandardCharsets.UTF_8);
        emailTemplateCache.put("welcome", html);
    }
}

// Array injection — multiple resources via pattern
@Value("classpath:migrations/*.sql")
private Resource[] migrations; // resolved via ResourcePatternResolver
```

**How `@Value` Resource injection works internally:**

Spring's `ResourceEditorRegistrar` registers a `ResourceEditor` with the `ConversionService`. When `@Value("classpath:...")` is processed and the target type is `Resource`, the `ResourceEditor` converts the string to a `Resource` by calling `resourceLoader.getResource(locationString)`.

---

### 6.2.7 — Resource in XML Configuration

```xml
<!-- Inject resource via property -->
<bean id="dataLoader" class="com.example.DataLoader">
    <property name="sqlScript" value="classpath:data/init.sql"/>
</bean>

<!-- PropertySourcesPlaceholderConfigurer with resources -->
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:app.properties</value>
            <value>classpath:database.properties</value>
            <value>file:/etc/myapp/override.properties</value>
        </list>
    </property>
    <property name="ignoreResourceNotFound" value="true"/>
</bean>

<!-- Import another XML config -->
<import resource="classpath:context/database-context.xml"/>
<import resource="classpath*:META-INF/spring/module-*.xml"/>
```

---

### 6.2.8 — WritableResource

For resources that support writing:

```java
public interface WritableResource extends Resource {
    boolean isWritable();
    OutputStream getOutputStream() throws IOException;
    WritableByteChannel writableChannel() throws IOException;
}

// Usage:
WritableResource wr = new FileSystemResource("/tmp/output.txt");
if (wr.isWritable()) {
    try (OutputStream os = wr.getOutputStream()) {
        os.write("Hello".getBytes(StandardCharsets.UTF_8));
    }
}

// PathResource also supports writing:
WritableResource pr = new PathResource(Paths.get("/tmp/output.txt"));
try (OutputStream os = pr.getOutputStream()) {
    os.write("Hello".getBytes());
}
```

---

### 6.2.9 — EncodedResource — Character Encoding Support

```java
// Wrap Resource with character encoding information
EncodedResource encodedResource =
    new EncodedResource(new ClassPathResource("config.xml"), StandardCharsets.UTF_8);

// Provides Reader with correct encoding
Reader reader = encodedResource.getReader();

// Used internally by XmlBeanDefinitionReader, PropertiesLoaderSupport, etc.
// Ensures correct character set when reading text resources
```

---

### 6.2.10 — Resource Utilities

```java
// ResourceUtils — utility methods
URL url = ResourceUtils.getURL("classpath:config.xml");
File file = ResourceUtils.getFile("classpath:config.xml");
boolean isJarURL = ResourceUtils.isJarURL(url);
boolean isFileURL = ResourceUtils.isFileURL(url);

// FileCopyUtils — copy resource content
byte[] bytes = FileCopyUtils.copyToByteArray(resource.getInputStream());
String content = FileCopyUtils.copyToString(
    new InputStreamReader(resource.getInputStream(), StandardCharsets.UTF_8));

// StreamUtils
byte[] bytes2 = StreamUtils.copyToByteArray(resource.getInputStream());
String content2 = StreamUtils.copyToString(
    resource.getInputStream(), StandardCharsets.UTF_8);
```

---

### Common Misconceptions

**Misconception 1:** "`ClassPathResource.getFile()` always works."
Wrong. `ClassPathResource.getFile()` throws `FileNotFoundException` when the resource is inside a JAR file (packaged application). It only works when the classpath resource is a plain filesystem file. Always prefer `getInputStream()` over `getFile()` for portability.

**Misconception 2:** "`classpath:` and `classpath*:` are equivalent."
Wrong. `classpath:` finds the FIRST matching resource in the classpath. `classpath*:` finds ALL matching resources across ALL classpath entries (all JARs and directories). For wildcard patterns in multi-jar setups, `classpath*:` is essential.

**Misconception 3:** "No-prefix resource location always resolves to classpath."
Wrong. No-prefix resolution depends on the `ApplicationContext` type. `ClassPathXmlApplicationContext` → classpath. `FileSystemXmlApplicationContext` → filesystem. `AnnotationConfigApplicationContext` → classpath (in modern Spring). Always use explicit prefixes for clarity.

**Misconception 4:** "`InputStreamResource` is equivalent to `ByteArrayResource` for in-memory data."
Wrong. `InputStreamResource.isOpen()` returns `true` — the stream can only be read ONCE. `ByteArrayResource` wraps the data in a new `ByteArrayInputStream` each time `getInputStream()` is called — multiple reads work. Use `ByteArrayResource` for in-memory data that may be read multiple times.

**Misconception 5:** "`ResourceLoader` and `ResourcePatternResolver` are the same."
Wrong. `ResourceLoader` loads a SINGLE resource from an exact location. `ResourcePatternResolver` extends `ResourceLoader` to load MULTIPLE resources using Ant-style patterns. `ApplicationContext` implements `ResourcePatternResolver` (the full interface), not just `ResourceLoader`.

**Misconception 6:** "`@Value` can only inject strings — not `Resource` objects."
Wrong. Spring's `ResourceEditor` converts string locations to `Resource` objects automatically when the `@Value` target field type is `Resource`.

---

## ② CODE EXAMPLES

### Example 1 — Resource Loading Patterns

```java
@Service
public class ResourceLoadingService {

    @Autowired
    private ResourceLoader resourceLoader;

    public void demonstrateLoading() throws IOException {

        // ClassPath resource
        Resource cp = resourceLoader.getResource("classpath:templates/email.html");
        System.out.println("Classpath exists: " + cp.exists());
        System.out.println("Filename: " + cp.getFilename());
        System.out.println("Description: " + cp.getDescription());

        // Read content safely — works whether in JAR or filesystem
        if (cp.exists() && cp.isReadable()) {
            String content = new String(
                cp.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
            System.out.println("Content length: " + content.length());
        }

        // File resource
        Resource fs = resourceLoader.getResource("file:/etc/myapp/config.properties");
        if (fs.exists()) {
            Properties props = new Properties();
            try (InputStream is = fs.getInputStream()) {
                props.load(is);
            }
            System.out.println("Properties loaded: " + props.size());
        }

        // URL resource
        Resource url = resourceLoader.getResource("https://config-server/app/config.json");
        System.out.println("URL resource: " + url.getURL());

        // Relative resource creation
        Resource base = resourceLoader.getResource("classpath:config/");
        Resource relative = base.createRelative("database.properties");
        System.out.println("Relative: " + relative.getDescription());
    }
}
```

---

### Example 2 — Pattern-based Resource Loading

```java
@Configuration
public class MultiResourceConfig {

    @Autowired
    private ResourcePatternResolver resolver;  // ApplicationContext implements this

    @Bean
    public void loadAllMigrations() throws IOException {
        // Find all SQL migration files in any JAR/directory
        Resource[] migrations = resolver.getResources(
            "classpath*:db/migrations/*.sql");

        // Sort by filename (V001__init.sql, V002__add_users.sql, etc.)
        Arrays.sort(migrations, Comparator.comparing(Resource::getFilename));

        for (Resource migration : migrations) {
            System.out.println("Migration: " + migration.getFilename());
            String sql = new String(
                migration.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
            jdbcTemplate.execute(sql);
        }
    }

    @Bean
    public List<Properties> loadAllModuleConfigs() throws IOException {
        // Load all module configuration files from all JARs
        Resource[] configs = resolver.getResources(
            "classpath*:META-INF/module-config.properties");

        List<Properties> allConfigs = new ArrayList<>();
        for (Resource config : configs) {
            Properties props = new Properties();
            try (InputStream is = config.getInputStream()) {
                props.load(is);
            }
            allConfigs.add(props);
            System.out.println("Loaded module config from: " + config.getDescription());
        }
        return allConfigs;
    }
}
```

---

### Example 3 — @Value Resource Injection

```java
@Component
public class ApplicationDataInitializer {

    // Single resource injection
    @Value("classpath:data/seed-data.sql")
    private Resource seedDataSql;

    // Resource from property value
    @Value("${app.template.location:classpath:templates/default.html}")
    private Resource htmlTemplate;

    // Array from pattern — all SQL files
    @Value("classpath:migrations/*.sql")
    private Resource[] sqlMigrations;

    // Multiple specific resources
    @Value("classpath:config/base.properties")
    private Resource baseConfig;

    @Value("classpath:config/security.properties")
    private Resource securityConfig;

    @PostConstruct
    public void initialize() throws IOException {
        // Read and execute seed SQL
        if (seedDataSql.exists()) {
            String sql = StreamUtils.copyToString(
                seedDataSql.getInputStream(), StandardCharsets.UTF_8);
            jdbcTemplate.execute(sql);
        }

        // Load HTML template
        String template = StreamUtils.copyToString(
            htmlTemplate.getInputStream(), StandardCharsets.UTF_8);
        templateEngine.registerTemplate("default", template);

        // Execute all migrations in order
        Arrays.sort(sqlMigrations, Comparator.comparing(Resource::getFilename));
        for (Resource migration : sqlMigrations) {
            String migrationSql = StreamUtils.copyToString(
                migration.getInputStream(), StandardCharsets.UTF_8);
            jdbcTemplate.execute(migrationSql);
        }

        // Load combined properties
        Properties combined = new Properties();
        try (InputStream base = baseConfig.getInputStream()) {
            combined.load(base);
        }
        try (InputStream sec = securityConfig.getInputStream()) {
            combined.load(sec); // security overrides base
        }
        configStore.init(combined);
    }
}
```

---

### Example 4 — Writing Resources (WritableResource)

```java
@Service
public class ReportExportService {

    public void exportReport(Report report, String outputPath) throws IOException {
        WritableResource outputResource = new FileSystemResource(outputPath);

        if (!outputResource.isWritable()) {
            throw new IllegalStateException("Cannot write to: " + outputPath);
        }

        try (OutputStream os = outputResource.getOutputStream();
             OutputStreamWriter writer = new OutputStreamWriter(os, StandardCharsets.UTF_8)) {

            writer.write("# Report: " + report.getTitle() + "\n");
            writer.write("Generated: " + LocalDateTime.now() + "\n\n");

            for (ReportLine line : report.getLines()) {
                writer.write(line.format() + "\n");
            }
        }

        System.out.println("Report written to: " + outputResource.getFile().getAbsolutePath());
        System.out.println("File size: " + outputResource.contentLength() + " bytes");
        System.out.println("Last modified: " + outputResource.lastModified());
    }

    public void appendToLog(String logPath, String message) throws IOException {
        PathResource logResource = new PathResource(Paths.get(logPath));

        // PathResource supports NIO channel operations
        try (WritableByteChannel channel = logResource.writableChannel()) {
            ByteBuffer buffer = ByteBuffer.wrap(
                (message + "\n").getBytes(StandardCharsets.UTF_8));
            channel.write(buffer);
        }
    }
}
```

---

### Example 5 — Custom ResourceLoader

```java
// Custom ResourceLoader that loads from database
public class DatabaseResourceLoader implements ResourceLoader {

    private final JdbcTemplate jdbcTemplate;
    private final ClassLoader classLoader;

    public DatabaseResourceLoader(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        this.classLoader = getClass().getClassLoader();
    }

    @Override
    public Resource getResource(String location) {
        if (location.startsWith("db:")) {
            // db:config/templates/email.html
            String key = location.substring(3);
            String content = jdbcTemplate.queryForObject(
                "SELECT content FROM resources WHERE resource_key = ?",
                String.class, key);

            if (content != null) {
                return new ByteArrayResource(
                    content.getBytes(StandardCharsets.UTF_8),
                    "Database resource: " + key);
            }
            throw new ResourceNotFoundException("Not found in DB: " + key);
        }

        // Delegate to default Spring resource loading for other prefixes
        return new DefaultResourceLoader(classLoader).getResource(location);
    }

    @Override
    public ClassLoader getClassLoader() {
        return classLoader;
    }
}

// Usage:
@Configuration
public class ResourceConfig {

    @Bean
    public DatabaseResourceLoader databaseResourceLoader(JdbcTemplate jdbcTemplate) {
        return new DatabaseResourceLoader(jdbcTemplate);
    }

    @Bean
    public EmailTemplateEngine emailTemplateEngine(
            DatabaseResourceLoader resourceLoader) {
        Resource template = resourceLoader.getResource("db:templates/welcome-email");
        return new EmailTemplateEngine(template);
    }
}
```

---

### Example 6 — Resource Metadata and Inspection

```java
@Component
public class ResourceInspector {

    public void inspect(Resource resource) throws IOException {
        System.out.println("=== Resource Inspection ===");
        System.out.println("Description:   " + resource.getDescription());
        System.out.println("Filename:      " + resource.getFilename());
        System.out.println("Exists:        " + resource.exists());
        System.out.println("Is readable:   " + resource.isReadable());
        System.out.println("Is open:       " + resource.isOpen());
        System.out.println("Is file:       " + resource.isFile());

        if (resource.exists()) {
            System.out.println("Content length: " + resource.contentLength());
            System.out.println("Last modified:  " + resource.lastModified());
            System.out.println("URL:            " + resource.getURL());
            System.out.println("URI:            " + resource.getURI());
        }

        if (resource.isFile()) {
            File file = resource.getFile();
            System.out.println("Absolute path:  " + file.getAbsolutePath());
            System.out.println("Can read:       " + file.canRead());
            System.out.println("Can write:      " + file.canWrite());
        }

        // Type identification
        System.out.println("Resource type: " +
            switch (resource) {
                case ClassPathResource r   -> "ClassPath";
                case FileSystemResource r  -> "FileSystem";
                case UrlResource r         -> "URL";
                case ByteArrayResource r   -> "ByteArray";
                case InputStreamResource r -> "InputStream";
                default -> resource.getClass().getSimpleName();
            });
    }
}
```

---

## ③ EXAM-STYLE QUESTIONS

**Q1 — MCQ:**
Which method on `Resource` returns a new `InputStream` each time it is called?

A) Only `ClassPathResource` — other types return the same stream
B) All `Resource` implementations where `isOpen()` returns `false`
C) Only `FileSystemResource` and `PathResource`
D) `InputStreamResource` — it wraps an existing stream and returns it each time

**Answer: B**
When `isOpen()` returns `false`, the contract guarantees that `getInputStream()` returns a FRESH stream on each call. When `isOpen()` is `true` (like `InputStreamResource`), the stream can only be read once. Most implementations (`ClassPathResource`, `FileSystemResource`, `ByteArrayResource`, `UrlResource`) return `false` from `isOpen()`.

---

**Q2 — Select all that apply:**
Which of the following Resource implementations support writing via `WritableResource`? (Select ALL that apply)

A) `ClassPathResource`
B) `FileSystemResource`
C) `PathResource`
D) `UrlResource`
E) `ByteArrayResource`
F) `InputStreamResource`

**Answer: B, C**
Only `FileSystemResource` and `PathResource` implement `WritableResource`. `ClassPathResource`, `UrlResource`, `ByteArrayResource`, and `InputStreamResource` are read-only.

---

**Q3 — Code Output Prediction:**

```java
Resource r1 = new ClassPathResource("config.xml");
Resource r2 = new InputStreamResource(
    new ByteArrayInputStream("test".getBytes()));
Resource r3 = new ByteArrayResource("test".getBytes());

System.out.println(r1.isOpen());  // ?
System.out.println(r2.isOpen());  // ?
System.out.println(r3.isOpen());  // ?
```

A) `false, false, false`
B) `false, true, false`
C) `true, false, true`
D) `false, true, true`

**Answer: B**
`ClassPathResource.isOpen()` → `false` (fresh stream each call). `InputStreamResource.isOpen()` → `true` (wraps EXISTING stream — single use). `ByteArrayResource.isOpen()` → `false` (creates new `ByteArrayInputStream` each call).

---

**Q4 — MCQ:**
What is the difference between `classpath:META-INF/services/*.factories` and `classpath*:META-INF/services/*.factories`?

A) They are identical — both search the full classpath
B) `classpath:` finds the first match; `classpath*:` finds ALL matches across all JARs and directories
C) `classpath*:` is for XML config only; `classpath:` is for annotation config
D) `classpath:` supports wildcards; `classpath*:` does not

**Answer: B**
`classpath:` finds the FIRST matching resource in the classpath and stops. `classpath*:` finds ALL matching resources across ALL classpath entries (every JAR and directory). For multi-JAR applications where resources exist in multiple JARs, `classpath*:` is required.

---

**Q5 — Trick Question:**

```java
Resource r = new ClassPathResource("data/file.xml");
// Assuming this resource exists inside a JAR (packaged application)

try {
    File f = r.getFile();
    System.out.println(f.getAbsolutePath());
} catch (Exception e) {
    System.out.println("Error: " + e.getClass().getSimpleName());
}
```

What is the output?

A) The absolute file path of the resource inside the JAR
B) `Error: IOException` — cannot get File reference for JAR resources
C) `Error: FileNotFoundException` — JAR-embedded resources have no File representation
D) `null` — `getFile()` returns null for JAR resources

**Answer: C**
`ClassPathResource.getFile()` throws `FileNotFoundException` with message "cannot be resolved to absolute file path because it does not reside in the file system: it is inside a JAR". JAR entries have no `java.io.File` representation. Always use `getInputStream()` or `getURL()` for portability.

---

**Q6 — Select all that apply:**
Which statements about `ResourceLoader` are TRUE? (Select ALL that apply)

A) `ApplicationContext` implements `ResourceLoader`
B) `ResourceLoader.getResource()` supports Ant-style wildcard patterns
C) `ResourcePatternResolver` extends `ResourceLoader` with wildcard support
D) An un-prefixed location resolves differently depending on the `ApplicationContext` type
E) `ResourceLoaderAware.setResourceLoader()` injects the `ApplicationContext` itself

**Answer: A, C, D, E**
B is wrong — plain `ResourceLoader` does NOT support wildcards. `ResourcePatternResolver` (C) adds that capability. D is correct — no-prefix resolution is context-type-dependent. E is correct — the injected `ResourceLoader` IS the `ApplicationContext`, which also implements `ResourcePatternResolver`.

---

**Q7 — MCQ:**
A Spring bean is injected with `@Value("classpath:config/*.properties")` into a `Resource[]` field. What Spring infrastructure resolves this?

A) `PropertySourcesPlaceholderConfigurer` + `ResourceEditor`
B) `ResourcePatternResolver` via `ResourceArrayPropertyEditor`
C) `AutowiredAnnotationBeanPostProcessor` + `ConversionService`
D) `PathMatchingResourcePatternResolver` directly in `@Value` processing

**Answer: B**
`@Value` injection into `Resource[]` uses `ResourceArrayPropertyEditor` which delegates to `PathMatchingResourcePatternResolver` (the `ApplicationContext` implementation) to resolve the wildcard pattern into an array of matching `Resource` objects.

---

**Q8 — MCQ:**
What does `Resource.createRelative("../sibling/other.xml")` do?

A) Throws `UnsupportedOperationException` — relative paths not supported
B) Creates a new `Resource` relative to the current resource's location
C) Creates an absolute path — ignores the current resource's location
D) Returns null if the relative path doesn't exist

**Answer: B**
`createRelative()` creates a new `Resource` interpreted relative to the current resource's location. For example, if the current resource is `classpath:config/database.xml`, then `createRelative("../security.xml")` creates a `Resource` for `classpath:security.xml`. The behavior is implementation-specific but all built-in implementations support it.

---

**Q9 — Select all that apply:**
Which of the following correctly describe `ByteArrayResource` vs `InputStreamResource`? (Select ALL that apply)

A) `ByteArrayResource.isOpen()` returns `false` — multiple reads supported
B) `InputStreamResource.isOpen()` returns `true` — single read only
C) Both support `getFile()` for accessing as a `java.io.File`
D) `ByteArrayResource` is preferred for in-memory data that may be read multiple times
E) `InputStreamResource.contentLength()` returns the byte count accurately

**Answer: A, B, D**
C is wrong — neither supports `getFile()` (not file-backed). E is wrong — `InputStreamResource.contentLength()` throws `UnsupportedOperationException` (stream length unknown without reading).

---

**Q10 — Drag-and-Drop: Match prefix to Resource type created:**

Prefixes: `classpath:`, `file:`, `http:`, `classpath*:` (in `getResources()`), no prefix (AnnotationConfigApplicationContext)

Resource types: `ClassPathResource`, `FileSystemResource`, `UrlResource`, `multiple ClassPathResource[]`, `ClassPathResource (default)`

**Answer:**
- `classpath:` → `ClassPathResource`
- `file:` → `FileSystemResource` (via `UrlResource` for `file://` URLs, or direct `FileSystemResource`)
- `http:` → `UrlResource`
- `classpath*:` (in `getResources()`) → multiple `Resource[]` from all classpath entries
- no prefix (in `AnnotationConfigApplicationContext`) → `ClassPathResource` (default)

---

## ④ TRICK ANALYSIS

**Trap 1: `ClassPathResource.getFile()` works in all environments**
Wrong. Works ONLY when the classpath resource is a plain filesystem file. Throws `FileNotFoundException` when inside a JAR. Always use `getInputStream()` for safe cross-environment code.

**Trap 2: `classpath:` and `classpath*:` are interchangeable**
Wrong. `classpath:` finds FIRST match (stops after first JAR/directory that has the resource). `classpath*:` finds ALL matches. In Spring Boot fat JARs with multiple classpath entries, only `classpath*:` ensures all matching resources are found.

**Trap 3: `InputStreamResource` can be read multiple times**
Wrong. `InputStreamResource.isOpen()` returns `true` — the wrapped `InputStream` is consumed on first read. Subsequent reads may return empty data or throw. Use `ByteArrayResource` for reusable in-memory resources.

**Trap 4: No-prefix always resolves to classpath**
Wrong. No-prefix resolution is context-type-dependent. `FileSystemXmlApplicationContext` resolves no-prefix as filesystem. `ClassPathXmlApplicationContext` as classpath. Always use explicit prefixes (`classpath:`, `file:`) for predictable behavior.

**Trap 5: `ResourceLoader` supports wildcard patterns**
Wrong. `ResourceLoader` handles EXACT locations only. `ResourcePatternResolver` (extending `ResourceLoader`) adds Ant-style wildcard support via `getResources()`. ApplicationContext implements `ResourcePatternResolver`, not just `ResourceLoader`.

**Trap 6: `@Value("classpath:...")` on `Resource` field needs special configuration**
Wrong. Spring auto-registers `ResourceEditor` via `ResourceEditorRegistrar`. No explicit configuration needed — `@Value` with `Resource` target type works out of the box in any Spring context.

**Trap 7: `FileSystemResource` path is always absolute**
Wrong. `FileSystemResource` accepts both absolute (`/etc/app/config.xml`) and relative paths (`config/settings.xml`). Relative paths are resolved against the JVM's working directory (`System.getProperty("user.dir")`), NOT the application classpath.

**Trap 8: All Resource types support `lastModified()` and `contentLength()`**
Wrong. `InputStreamResource` throws `UnsupportedOperationException` for both. `ByteArrayResource.contentLength()` works (returns array length). `ClassPathResource.lastModified()` works for filesystem files but may not work reliably for JAR entries.

---

## ⑤ SUMMARY SHEET

### Resource Implementation Reference

```
┌──────────────────────────┬───────────┬─────────┬──────────┬────────────┐
│ Implementation           │ isOpen()  │ isFile()│ Writable │ getFile()  │
├──────────────────────────┼───────────┼─────────┼──────────┼────────────┤
│ ClassPathResource        │ false     │ varies* │ NO       │ JARs: fail │
│ FileSystemResource       │ false     │ true    │ YES      │ always OK  │
│ PathResource             │ false     │ true    │ YES      │ always OK  │
│ UrlResource (http:)      │ false     │ false   │ NO       │ fails      │
│ UrlResource (file:)      │ false     │ true    │ NO       │ OK         │
│ InputStreamResource      │ TRUE      │ false   │ NO       │ fails      │
│ ByteArrayResource        │ false     │ false   │ NO       │ fails      │
│ ServletContextResource   │ false     │ varies  │ NO       │ varies     │
└──────────────────────────┴───────────┴─────────┴──────────┴────────────┘
* isFile() = true if not inside JAR
```

---

### Location Prefix Reference

```
Prefix          → Resource type         → Notes
classpath:      → ClassPathResource     → first match only
classpath*:     → Resource[]            → ALL matches (wildcard)
file:           → FileSystemResource    → filesystem path
http:/https:    → UrlResource           → HTTP resource
ftp:            → UrlResource           → FTP resource
(no prefix)     → context-dependent     → depends on ApplicationContext type
```

---

### Key Rules to Memorize

| Rule | Detail |
|---|---|
| `getInputStream()` vs `getFile()` | Always prefer `getInputStream()` — portable across JARs |
| `isOpen() = true` | Single-read only — `InputStreamResource` |
| `classpath*:` | ALL classpath entries — essential for multi-JAR |
| `classpath:` | FIRST match only — stops after finding one |
| `WritableResource` | Only `FileSystemResource` and `PathResource` |
| No-prefix resolution | Depends on `ApplicationContext` type |
| `ByteArrayResource` | Reusable — new `ByteArrayInputStream` each `getInputStream()` |
| `createRelative()` | Creates resource relative to current resource location |
| `@Value` → `Resource` | Auto-converted via `ResourceEditor` — no extra config |
| `@Value` → `Resource[]` | Resolved via `ResourceArrayPropertyEditor` + `ResourcePatternResolver` |
| `ResourceLoaderAware` | Injects the `ApplicationContext` (which implements `ResourcePatternResolver`) |

---

### Interface Hierarchy

```
InputStreamSource
    └── Resource
            ├── WritableResource
            │   ├── FileSystemResource
            │   └── PathResource
            └── (all others — read-only)

ResourceLoader
    └── ResourcePatternResolver
            └── ApplicationContext (implements both)
```

---

### Interview One-Liners

- "`ClassPathResource.getFile()` throws `FileNotFoundException` when the resource is inside a JAR — always use `getInputStream()` for portable resource reading."
- "`classpath:` finds the FIRST matching resource; `classpath*:` finds ALL matching resources across ALL classpath entries — essential for plugin/multi-JAR architectures."
- "`InputStreamResource.isOpen()` returns `true` — it wraps an existing stream that can only be read once. Use `ByteArrayResource` for in-memory data that needs multiple reads."
- "No-prefix location resolution is context-type-dependent: `ClassPathXmlApplicationContext` → classpath, `FileSystemXmlApplicationContext` → filesystem. Use explicit prefixes for clarity."
- "`WritableResource` is only implemented by `FileSystemResource` and `PathResource` — all other built-in implementations are read-only."
- "`@Value('classpath:...')` on a `Resource` field works automatically via `ResourceEditor` registered by `ResourceEditorRegistrar` — no explicit configuration needed."
- "`ApplicationContext` implements `ResourcePatternResolver` (which extends `ResourceLoader`) — it can load both single resources and collections of resources via Ant-style wildcards."

---
