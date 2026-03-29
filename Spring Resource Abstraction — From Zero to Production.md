# Spring Resource Abstraction — From Zero to Production

---

## 🧩 1. Requirement Understanding — Feel the Pain First

You just joined a team. Your first ticket:

```
TICKET-447: Load SQL seed data on app startup
- Execute /db/seed.sql from the classpath on startup
- Also load /etc/myapp/overrides.properties from the server filesystem
- Also fetch a config JSON from http://config-server/app/config.json
- Works in local dev, Docker, and production JAR deployment
```

You sit down and write the obvious code:

```java
@PostConstruct
public void init() throws Exception {

    // Load classpath file
    File sqlFile = new File("db/seed.sql");                          // ← dev machine works
    String sql = new String(Files.readAllBytes(sqlFile.toPath()));   // ← Docker: FileNotFoundException
    jdbcTemplate.execute(sql);

    // Load filesystem file
    File propsFile = new File("/etc/myapp/overrides.properties");
    Properties props = new Properties();
    props.load(new FileInputStream(propsFile));                      // ← dev: file doesn't exist

    // Load from HTTP
    URL url = new URL("http://config-server/app/config.json");
    InputStream is = url.openStream();                               // ← completely different API
    String json = new String(is.readAllBytes());
}
```

You test it locally. It works. You deploy to Docker. It explodes:

```
FileNotFoundException: db/seed.sql (No such file or directory)
```

Because in a packaged JAR, `db/seed.sql` isn't a file on disk. It's **inside the JAR archive**. `new File(...)` has no idea how to reach inside a ZIP archive.

So you try the classpath way:

```java
InputStream is = getClass().getResourceAsStream("/db/seed.sql");
```

That works in the JAR. But now your three resources need **three completely different APIs**:

```
Classpath resource  → getClass().getResourceAsStream()
Filesystem file     → new FileInputStream(new File(...))
HTTP URL            → new URL(...).openStream()
```

Every time you want to read a resource, you need to know WHERE it lives first, and use the matching API. If the location changes (moves from classpath to filesystem), you rewrite code. If you're in a library that accepts "a resource location", you can't write one method that handles all three.

**That's the pain.** Three different things that all fundamentally mean: *"give me the bytes at this location."*

---

## 🧠 2. Design Thinking — Naive Approaches First

**Naive Approach 1: Just use `String` paths everywhere**

```java
public void loadResource(String path) throws Exception {
    if (path.startsWith("http")) {
        // HTTP logic
    } else if (path.startsWith("/")) {
        // filesystem logic  
    } else {
        // classpath logic
    }
}
```

**Why this fails:** Every method that touches resources now has this `if/else` chain. You paste it in 15 places. Then someone adds FTP support and you have to find all 15 locations. This is the classic "stringly typed" problem — you're encoding behavior in string parsing.

---

**Naive Approach 2: Make an interface, write three implementations yourself**

```java
interface MyResource {
    InputStream open() throws IOException;
    boolean exists();
}

class ClasspathMyResource implements MyResource { ... }
class FilesystemMyResource implements MyResource { ... }
class HttpMyResource implements MyResource { ... }
```

**Why this fails:** You just reinvented a wheel. You still need to write the factory that parses `"classpath:db/seed.sql"` → `ClasspathMyResource`. You need to handle edge cases (JARs, encoding, relative paths). And every library that doesn't use YOUR interface is incompatible. You need a standard.

---

**Naive Approach 3: Just use `java.net.URL` for everything**

```java
// URL can represent classpath, file, http — it's already standard!
URL url = getClass().getResource("/db/seed.sql");  // classpath
URL url2 = new URL("file:/etc/myapp/overrides.properties");
URL url3 = new URL("http://config-server/app/config.json");
```

**Why this fails:** `URL` is read-only and clunky. You can't ask "does this exist?" reliably. You can't ask "when was it last modified?" You can't ask "what's the filename?" You can't write to it. `URL.openStream()` can't tell you content length without downloading everything. And `URL` inside a JAR returns a `jar:file://...` URL that many utilities choke on.

---

**The correct solution emerges:** We need a standard interface that wraps any location type, speaks a common language (`exists()`, `getInputStream()`, `getFilename()`, `lastModified()`), and lets the caller stay completely ignorant of WHERE the resource actually lives. Spring calls it `Resource`.

---

## 🧠 3. Concept Introduction — Plain English First

Imagine a **universal USB adapter**.

Your devices have different plugs — UK plug, EU plug, US plug. Instead of buying a different charger for each country, you carry one adapter that accepts any plug on one side and outputs standard USB on the other. The device charging from the USB port doesn't care which plug type fed it.

```
[ ClassPath file ] ──┐
[ Filesystem file ] ──┤──► [ Resource (standard interface) ] ──► your code reads bytes
[ HTTP URL ] ─────────┘
[ byte[] in memory ] ┘
```

Your code only ever talks to the right side — the standard `Resource` interface. It calls `resource.getInputStream()` and gets bytes. It doesn't know or care that one Resource is inside a JAR, one is on a server, one is in memory.

**Under the hood — how this actually works:**

```
You write:
    resource.getInputStream()

Spring internally does:
    if this is ClassPathResource:
        → classLoader.getResourceAsStream(path)    ← handles JARs
    if this is FileSystemResource:
        → new FileInputStream(file)
    if this is UrlResource:
        → url.openConnection().getInputStream()
    if this is ByteArrayResource:
        → new ByteArrayInputStream(data)
```

The complexity is hidden inside each implementation. You write once, it works everywhere.

**The second part — how does `"classpath:db/seed.sql"` become a `Resource` object?**

Something needs to parse that string and pick the right implementation. That something is called a `ResourceLoader`. The Spring ApplicationContext IS a ResourceLoader — when you say `"classpath:..."` it knows to hand you a `ClassPathResource`. When you say `"file:..."`, a `FileSystemResource`. The string prefix is the routing key.

```
"classpath:db/seed.sql"   →  ResourceLoader  →  ClassPathResource
"file:/etc/app/cfg.props" →  ResourceLoader  →  FileSystemResource
"http://server/cfg.json"  →  ResourceLoader  →  UrlResource
```

---

## 🏗 4. Task Breakdown — Plan Before Coding

```
TICKET-447 implementation plan:

TODO 1: Inject a ResourceLoader into our bean so we can load resources by string
TODO 2: Load the classpath SQL file using "classpath:" prefix — read bytes — execute
TODO 3: Load the filesystem properties file using "file:" prefix — load into Properties
TODO 4: Load the HTTP JSON using "http://" prefix — read bytes — parse
TODO 5: Handle the "what if the file doesn't exist" case gracefully
TODO 6: Replace manual string loading with @Value injection (cleaner version)
TODO 7: Load ALL sql files from a directory using wildcards (bonus requirement)
```

---

## 💻 5. Step-by-Step Implementation

### TODO 1: Get a ResourceLoader

```java
@Service
public class AppInitializer {

    // ResourceLoader: the thing that converts a location string like
    // "classpath:db/seed.sql" into an actual Resource object.
    // ApplicationContext implements this interface — Spring injects it automatically.
    @Autowired
    private ResourceLoader resourceLoader;
```

**What Spring does internally when it sees `@Autowired ResourceLoader`:**
The ApplicationContext registers itself as a `ResourceLoader` bean. So Spring injects the context itself here. You're getting the full ApplicationContext, just typed as `ResourceLoader`.

---

### TODO 2: Load the classpath SQL file

```java
    @PostConstruct
    public void init() throws IOException {

        // Resource: a uniform handle to any location.
        // getResource() parses the prefix and creates the right implementation.
        // "classpath:" prefix → Spring creates a ClassPathResource internally.
        Resource sqlResource = resourceLoader.getResource("classpath:db/seed.sql");

        // COMMON MISTAKE — the silent failure version:
        // boolean exists = new File("db/seed.sql").exists();   ← always false in JAR
        //
        // Correct way — ask the Resource itself:
        if (!sqlResource.exists()) {
            throw new IllegalStateException("seed.sql not found on classpath");
        }

        // getInputStream(): opens the resource for reading.
        // Works whether the file is on disk OR inside a JAR.
        // Each call returns a FRESH stream — you can call this multiple times.
        String sql;
        try (InputStream is = sqlResource.getInputStream()) {
            sql = new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }

        jdbcTemplate.execute(sql);
        System.out.println("Seed SQL loaded from: " + sqlResource.getDescription());
        // prints something like: "class path resource [db/seed.sql]"
```

**What Spring does internally for ClassPathResource.getInputStream():**

```
classLoader.getResourceAsStream("db/seed.sql")
    ↓
JVM scans the classpath (including inside JAR files)
    ↓
Finds db/seed.sql inside myapp.jar!/db/seed.sql
    ↓
Returns an InputStream reading directly from the JAR entry
```

**The silent failure trap with `getFile()`:**

```java
// THIS COMPILES AND RUNS FINE IN DEV. BLOWS UP IN PRODUCTION:
File f = sqlResource.getFile();  // ← throws FileNotFoundException when inside JAR
// "cannot be resolved to absolute file path because it does not reside
//  in the file system: jar:file:/app/myapp.jar!/db/seed.sql"

// WHY: JAR entries have no java.io.File representation. They live inside
// a ZIP archive. There is no filesystem path you can point to.
// RULE: Always use getInputStream() — never getFile() for classpath resources.
```

---

### TODO 3: Load the filesystem properties file

```java
        // "file:" prefix → Spring creates a FileSystemResource internally.
        // This one reads from the actual server filesystem — NOT the classpath.
        Resource propsResource = resourceLoader.getResource("file:/etc/myapp/overrides.properties");

        Properties overrides = new Properties();
        if (propsResource.exists()) {
            try (InputStream is = propsResource.getInputStream()) {
                overrides.load(is);
            }
            System.out.println("Loaded " + overrides.size() + " overrides");
            System.out.println("Last modified: " + propsResource.lastModified());
        }
        // If file doesn't exist in dev — that's fine, we just skip it.
        // In prod the ops team puts it there.
```

**What's different about FileSystemResource internally:**
```
propsResource.getInputStream()
    →  new FileInputStream(new File("/etc/myapp/overrides.properties"))
```
Straightforward. No JAR magic needed. `getFile()` DOES work here — the file is a real filesystem file.

---

### TODO 4: Load from HTTP

```java
        // "http://" → Spring creates a UrlResource internally.
        Resource configResource = resourceLoader.getResource("http://config-server/app/config.json");

        // getURL(): gives you back the java.net.URL this resource wraps.
        System.out.println("Fetching from: " + configResource.getURL());

        String json;
        try (InputStream is = configResource.getInputStream()) {
            // Internally: url.openConnection().getInputStream()
            json = new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }

        parseAndApplyConfig(json);
    }
```

**All three cases now use identical code patterns:**
```
Resource r = resourceLoader.getResource("classpath:db/seed.sql");  ─┐
Resource r = resourceLoader.getResource("file:/etc/app/cfg.props"); ─┤─► r.getInputStream()
Resource r = resourceLoader.getResource("http://server/cfg.json");  ─┘   r.exists()
                                                                          r.getFilename()
```

---

### TODO 5: Silent failures to watch for — "no prefix" trap

```java
// DANGEROUS — no prefix:
Resource r = resourceLoader.getResource("db/seed.sql");   // no "classpath:" prefix!

// What happens depends on which ApplicationContext type is running:
//   ClassPathXmlApplicationContext → ClassPathResource (probably what you want)
//   FileSystemXmlApplicationContext → FileSystemResource (NOT what you want)
//   AnnotationConfigApplicationContext → ClassPathResource (OK in modern Spring)

// The code works on your machine, breaks in a different deployment.
// RULE: Always use an explicit prefix. Never rely on context-type defaults.

// Correct:
Resource r = resourceLoader.getResource("classpath:db/seed.sql");   // always classpath
Resource r = resourceLoader.getResource("file:/etc/app/config.xml"); // always filesystem
```

---

### TODO 6: Cleaner version — inject Resources directly with @Value

Instead of calling `resourceLoader.getResource(...)` manually, Spring can inject `Resource` objects directly into fields. The string-to-Resource conversion happens automatically.

```java
@Service
public class AppInitializer {

    // @Value with a Resource field: Spring sees the target type is Resource,
    // not String. It automatically runs the location string through a
    // ResourceEditor (a converter) and hands you the Resource object.
    // No explicit ResourceLoader needed.
    @Value("classpath:db/seed.sql")
    private Resource seedSql;

    @Value("file:/etc/myapp/overrides.properties")
    private Resource overridesFile;

    // The location itself can come from a property file:
    @Value("${config.server.url:http://localhost:8080/config.json}")
    private Resource configJson;

    @PostConstruct
    public void init() throws IOException {

        if (seedSql.exists()) {
            try (InputStream is = seedSql.getInputStream()) {
                jdbcTemplate.execute(new String(is.readAllBytes(), StandardCharsets.UTF_8));
            }
        }

        // The overrides file might not exist in dev — handle gracefully
        if (overridesFile.exists()) {
            Properties p = new Properties();
            try (InputStream is = overridesFile.getInputStream()) {
                p.load(is);
            }
        }
    }
}
```

**What Spring does internally for `@Value("classpath:db/seed.sql") Resource field`:**
```
Bean creation time:
  1. Spring sees @Value("classpath:db/seed.sql") on a Resource-typed field
  2. Finds ResourceEditor registered by ResourceEditorRegistrar
  3. ResourceEditor calls: applicationContext.getResource("classpath:db/seed.sql")
  4. Returns ClassPathResource — injected into the field
  5. All this happens before @PostConstruct runs
```

---

### TODO 7: Load ALL sql files from a directory — Wildcards

New requirement: instead of one `seed.sql`, load all `V001__init.sql`, `V002__users.sql` etc. in order.

```java
@Service
public class MigrationRunner {

    // ResourcePatternResolver: an extension of ResourceLoader that can
    // return MULTIPLE resources matching a wildcard pattern.
    // ApplicationContext implements this — so we can @Autowired it.
    @Autowired
    private ResourcePatternResolver resolver;

    // OR: inject an array directly via @Value with a wildcard pattern:
    @Value("classpath:db/migrations/*.sql")
    private Resource[] migrations;

    @PostConstruct
    public void runMigrations() throws IOException {

        // Sort by filename so V001 runs before V002
        Arrays.sort(migrations, Comparator.comparing(Resource::getFilename));

        for (Resource migration : migrations) {
            System.out.println("Running migration: " + migration.getFilename());
            try (InputStream is = migration.getInputStream()) {
                String sql = new String(is.readAllBytes(), StandardCharsets.UTF_8);
                jdbcTemplate.execute(sql);
            }
        }
    }
}
```

**The critical `classpath:` vs `classpath*:` distinction:**

```java
// classpath: — finds the FIRST match and stops.
// If your migrations are inside a JAR AND also on the filesystem,
// it only returns from whichever comes first in the classpath.
Resource[] r1 = resolver.getResources("classpath:db/migrations/*.sql");

// classpath*: — scans EVERY jar and directory in the classpath.
// Returns ALL matches from everywhere.
// Essential when your app is made of multiple JARs (microservice modules,
// plugin architectures, Spring Boot fat JAR with embedded dependencies).
Resource[] r2 = resolver.getResources("classpath*:db/migrations/*.sql");
```

**Silent failure with `classpath:` in multi-JAR setups:**
```
You have:
  app.jar  containing: db/migrations/V001__base.sql
  module.jar containing: db/migrations/V002__users.sql

classpath:db/migrations/*.sql  →  only finds V001__base.sql (first JAR wins)
classpath*:db/migrations/*.sql →  finds V001 AND V002 (all JARs scanned)

classpath: silently drops V002. No error. Your DB is half-migrated.
```

---

### Bonus: Writing to a Resource (not just reading)

```java
@Service
public class ReportWriter {

    public void writeReport(String content, String outputPath) throws IOException {

        // WritableResource: an extension of Resource that also lets you write.
        // Only FileSystemResource and PathResource implement this.
        // ClassPathResource, UrlResource, ByteArrayResource do NOT.
        WritableResource output = new FileSystemResource(outputPath);

        if (!output.isWritable()) {
            throw new IllegalStateException("Cannot write to: " + outputPath);
        }

        try (OutputStream os = output.getOutputStream()) {
            os.write(content.getBytes(StandardCharsets.UTF_8));
        }

        System.out.println("Wrote " + output.contentLength() + " bytes");
        System.out.println("To: " + output.getFile().getAbsolutePath());
    }
}
```

**Common mistake — trying to write to a ClassPathResource:**
```java
WritableResource wr = (WritableResource) new ClassPathResource("output.txt");
// → ClassCastException: ClassPathResource is not a WritableResource
// ClassPathResource is read-only by design — classpath is for reading app resources,
// not for writing runtime output.
```

---

### Understanding `isOpen()` — the "one-time use" flag

```java
// isOpen() = false means: getInputStream() gives you a FRESH stream every time.
// isOpen() = true means: there is ONE stream, and once you read it, it's gone.

Resource cp = new ClassPathResource("config.xml");
cp.isOpen()  // → false  — safe to call getInputStream() multiple times

Resource is = new InputStreamResource(someExistingStream);
is.isOpen()  // → true — this wraps an ALREADY OPEN stream

// The InputStreamResource silent failure:
is.getInputStream();  // works — reads the stream
is.getInputStream();  // returns same exhausted stream — reads ZERO bytes
                      // No exception. Just empty data.

// Fix: use ByteArrayResource for in-memory data you might read multiple times:
byte[] data = fetchSomeBytes();
Resource safe = new ByteArrayResource(data);
safe.isOpen()  // → false — creates new ByteArrayInputStream each time
safe.getInputStream();  // fresh stream
safe.getInputStream();  // another fresh stream — same underlying data
```

---

## ⚠️ 6. Developer Decisions & Tradeoffs

**Decision 1: `@Value` injection vs `ResourceLoader.getResource()`**

```
Use @Value when:
  - The resource location is known at compile time (or comes from a property)
  - You just need the resource injected, no dynamic construction

Use ResourceLoader when:
  - Resource location is determined at runtime (user input, database value)
  - You're building a component that accepts arbitrary locations
  - You need to create resources programmatically in a loop
```

**Decision 2: `classpath:` vs `classpath*:`**

```
Use classpath:   when the resource is in ONE known location,
                 you want fast lookup, not multi-JAR scanning.

Use classpath*:  when you need to aggregate resources from multiple JARs —
                 plugin configs, distributed migration files, META-INF scanning.
                 It's slower (scans everything) — don't use it for single known files.
```

**Decision 3: `getInputStream()` vs `getFile()`**

```
ALWAYS use getInputStream() unless you specifically need a java.io.File object
(e.g. passing to a library that requires File, not InputStream).

getFile() throws for JAR-embedded resources.
getInputStream() works everywhere.

The cost: with getInputStream() you manage the stream lifecycle yourself.
The benefit: it works in dev, test, Docker, JAR — everywhere.
```

---

## 🧪 7. Edge Cases & What Silently Breaks

```java
@SpringBootTest
class ResourceLoadingTest {

    @Autowired
    private ResourceLoader resourceLoader;

    @Test
    void classpathResourceExistsInJar() throws IOException {
        Resource r = resourceLoader.getResource("classpath:db/seed.sql");

        // This is what you actually want to test:
        assertTrue(r.exists(), "seed.sql must exist on classpath");
        assertTrue(r.isReadable(), "seed.sql must be readable");

        // DO NOT test getFile() — it will pass in IDE, fail in CI with fat JAR:
        // assertNotNull(r.getFile()); ← WRONG, don't do this
    }

    @Test
    void inputStreamResourceIsDestroyedAfterOneRead() throws IOException {
        byte[] data = "test content".getBytes();
        Resource r = new InputStreamResource(new ByteArrayInputStream(data));

        // First read — works:
        byte[] first = r.getInputStream().readAllBytes();
        assertEquals("test content", new String(first));

        // Second read — silently returns empty or wrong data:
        byte[] second = r.getInputStream().readAllBytes();
        assertEquals(0, second.length,
            "InputStreamResource is exhausted after first read — isOpen()=true");

        // Fix: use ByteArrayResource instead
        Resource safe = new ByteArrayResource(data);
        byte[] a = safe.getInputStream().readAllBytes();
        byte[] b = safe.getInputStream().readAllBytes();
        assertArrayEquals(a, b, "ByteArrayResource gives fresh stream each time");
    }

    @Test
    void noPrefixIsDangerousInTests() {
        // In a test context (AnnotationConfigApplicationContext), no-prefix
        // resolves to classpath — but don't rely on this:
        Resource dangerous = resourceLoader.getResource("db/seed.sql"); // no prefix
        Resource safe = resourceLoader.getResource("classpath:db/seed.sql");

        // They might be equal here, but in FileSystemXmlApplicationContext they diverge.
        // Always use the explicit one.
        assertEquals(safe.getDescription(),
            resourceLoader.getResource("classpath:db/seed.sql").getDescription());
    }

    @Test
    void wildcardClasspathFindsAllFiles() throws IOException {
        PathMatchingResourcePatternResolver resolver =
            new PathMatchingResourcePatternResolver();

        Resource[] files = resolver.getResources("classpath*:db/migrations/*.sql");

        // Verify count — if this is 0, your patterns are wrong or files are missing:
        assertTrue(files.length > 0, "Should find at least one migration file");

        // Verify ordering after sort:
        Arrays.sort(files, Comparator.comparing(Resource::getFilename));
        assertTrue(files[0].getFilename().compareTo(files[files.length-1].getFilename()) < 0);
    }
}
```

---

## 🔄 8. Refactoring — What a Senior Dev Does Next

**First version — what we wrote above — has one problem:** every resource user writes the same `try (InputStream is = ...) { bytes = is.readAllBytes() }` boilerplate, and the error handling is scattered everywhere.

**Senior dev refactoring:** Extract a `ResourceReader` utility and use `EncodedResource` for character encoding correctness.

```java
// EncodedResource: a wrapper that pairs a Resource with a specific character
// encoding, so you get a correctly-decoded Reader instead of raw bytes.
// Spring's XML and properties loaders use this internally.
@Component
public class ResourceReader {

    @Autowired
    private ResourceLoader resourceLoader;

    public String readText(String location) {
        return readText(location, StandardCharsets.UTF_8);
    }

    public String readText(String location, Charset charset) {
        Resource resource = resourceLoader.getResource(location);

        if (!resource.exists()) {
            throw new IllegalArgumentException(
                "Resource not found: " + location + " (" + resource.getDescription() + ")");
        }

        // EncodedResource: wraps a Resource with encoding info.
        // Gives you a Reader that handles charset conversion correctly.
        // Avoids: new InputStreamReader(is, charset) scattered everywhere.
        EncodedResource encoded = new EncodedResource(resource, charset);

        try (Reader reader = encoded.getReader()) {
            return FileCopyUtils.copyToString(reader);
            // FileCopyUtils.copyToString: reads entire Reader to String, closes it.
            // Part of Spring's utility package — no external dependencies.
        } catch (IOException e) {
            throw new UncheckedIOException(
                "Failed to read resource: " + resource.getDescription(), e);
        }
    }

    public Properties readProperties(String location) {
        Resource resource = resourceLoader.getResource(location);
        Properties props = new Properties();
        if (!resource.exists()) return props;  // empty props if missing — non-fatal

        try (InputStream is = resource.getInputStream()) {
            props.load(is);
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to load properties from: " + location, e);
        }
        return props;
    }
}

// Usage is now clean everywhere:
@Service
public class AppInitializer {

    @Autowired private ResourceReader resourceReader;

    @PostConstruct
    public void init() {
        String sql = resourceReader.readText("classpath:db/seed.sql");
        jdbcTemplate.execute(sql);

        Properties overrides = resourceReader.readProperties("file:/etc/myapp/overrides.properties");
        configStore.applyOverrides(overrides);
    }
}
```

**What improved:**
- Error message includes the resource description (tells you exactly what path failed)
- Encoding handled correctly in one place
- `exists()` check before reading — no cryptic stream errors
- Callers don't manage streams at all

---

## 🧠 9. Mental Model Summary

### Every term, redefined in plain English:

| Term | What it actually is |
|---|---|
| **Resource** | A uniform handle to "bytes at some location" — like a remote control that works regardless of which TV it's pointed at |
| **ClassPathResource** | A Resource that reads from inside your JAR or classpath directories |
| **FileSystemResource** | A Resource that reads (and writes) from the server's actual filesystem |
| **UrlResource** | A Resource that fetches from an HTTP, FTP, or file URL |
| **ByteArrayResource** | A Resource backed by a byte array in memory — useful for tests and in-memory data |
| **InputStreamResource** | A Resource wrapping an already-open stream — single-use only |
| **WritableResource** | A Resource you can also write to — only FileSystemResource and PathResource |
| **ResourceLoader** | The factory that converts a location string (`"classpath:..."`) into a Resource object |
| **ResourcePatternResolver** | A ResourceLoader that also supports wildcards — returns many Resources at once |
| **isOpen()** | When `true`: stream is single-use (InputStreamResource). When `false`: fresh stream every call |
| **classpath:** | Prefix meaning "search the classpath, return the FIRST match" |
| **classpath*:** | Prefix meaning "search ALL JARs and directories, return EVERY match" |
| **EncodedResource** | A Resource + character encoding pair — gives you a Reader with correct charset |

---

### The decision rules:

```
Which prefix should I use?
  → Known single location: classpath: or file: with full path
  → Aggregating from multiple JARs: classpath*:
  → External server file: file:
  → Never: no prefix (context-type-dependent)

How should I read from a Resource?
  → ALWAYS getInputStream()
  → NEVER getFile() for classpath resources (blows up in JAR)

Which in-memory Resource type?
  → Data read once: InputStreamResource is OK, but risky
  → Data read multiple times: ByteArrayResource (always safe)

When to use @Value vs ResourceLoader.getResource()?
  → Fixed/configured location: @Value
  → Runtime-determined location: ResourceLoader

When to use classpath* in wildcards?
  → If your app has multiple JARs contributing the same resource path: classpath*:
  → Single JAR, known path: classpath: (faster)
```

### The one-page mental picture:

```
Your code
    │
    ▼
ResourceLoader.getResource("PREFIX:path")
    │
    ├── "classpath:"  ──►  ClassPathResource  ──►  getInputStream() ──► JVM classpath (incl. JARs)
    ├── "file:"       ──►  FileSystemResource ──►  getInputStream() ──► OS filesystem
    ├── "http:"       ──►  UrlResource        ──►  getInputStream() ──► HTTP connection
    └── byte[]        ──►  ByteArrayResource  ──►  getInputStream() ──► in-memory
                                  │
                          common interface:
                           .exists()
                           .isReadable()
                           .getFilename()
                           .lastModified()
                           .contentLength()
                           .getDescription()    ← great for error messages
```

Everything above the ResourceLoader line is your code — uniform, location-agnostic. Everything below is Spring's problem.
