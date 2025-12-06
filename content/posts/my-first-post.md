+++
date = '2025-12-05T19:18:58+01:00'
draft = true
title = 'My First Post'
+++
# Testing Spring Boot with PGlite: A Docker-Free Alternative to Testcontainers

## The Database Testing Dilemma

When building Spring Boot applications with PostgreSQL, you face a common challenge: how do you test your data layer effectively? The traditional approaches each have their drawbacks:

**H2 In-Memory Database**: Fast and easy, but it's not PostgreSQL. Subtle differences in SQL dialects, data types, and features can lead to bugs that only appear in production.

**Testcontainers**: Provides real PostgreSQL via Docker, giving you production-like testing. However, it requires Docker to be installed and running, adds significant overhead to test execution, and can be tricky in CI/CD environments.

**Docker Compose**: Similar benefits and drawbacks to Testcontainers, with the added complexity of managing infrastructure files.

What if there was a way to test against a real PostgreSQL-compatible database without Docker, while maintaining the speed of in-memory databases?

## Enter PGlite

[PGlite](https://pglite.dev/) is a WebAssembly build of PostgreSQL packaged as a TypeScript library. It provides a lightweight, embeddable PostgreSQL implementation that runs entirely in-process. The [`@electric-sql/pglite-socket`](https://www.npmjs.com/package/@electric-sql/pglite-socket) package exposes this via the PostgreSQL wire protocol, allowing any PostgreSQL client (including JDBC) to connect to it.

### Why PGlite for Testing?

- **No Docker required**: Just Node.js/npx, which most development machines already have
- **True PostgreSQL compatibility**: Uses actual PostgreSQL code (version 17.5), not a compatible alternative
- **Fast startup**: Initializes in 1-2 seconds vs. 10-15 seconds for Docker containers
- **Lightweight**: WebAssembly-based, minimal resource footprint
- **Isolated**: Each test class can have its own database instance

### The Trade-offs

PGlite isn't perfect for all scenarios:

- **Single connection limit**: Only supports one connection at a time (fine for most tests with proper pool configuration)
- **In-memory only**: Data doesn't persist between runs (which is actually ideal for tests)
- **Requires Node.js**: Adds a runtime dependency beyond the JVM
- **Less mature**: Newer technology compared to Testcontainers

## Implementation: A JUnit 5 Extension Approach

Rather than manually starting and stopping PGlite for each test, we can create a reusable JUnit 5 extension that manages the lifecycle automatically. Here's how:

### Step 1: Create the JUnit 5 Extension

```java
@ExtendWith(PgLiteExtension.class)
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles({"postgres", "pglite"})
@DisabledInNativeImage
public @interface PgLiteTest {
}
```

The extension handles:
1. Starting `npx @electric-sql/pglite-socket` before tests
2. Finding an available port to avoid conflicts
3. Verifying the server is ready via JDBC health check
4. Stopping the server gracefully after tests complete

### Step 2: Configure Spring Boot for PGlite

Create `src/test/resources/application-pglite.properties`:

```properties
# PGlite uses "template1" as the default database name
spring.datasource.url=jdbc:postgresql://localhost:${pglite.port:5432}/template1?sslmode=disable
spring.datasource.username=postgres
spring.datasource.password=postgres

# CRITICAL: PGlite only supports ONE connection at a time
spring.datasource.hikari.maximum-pool-size=1
spring.datasource.hikari.minimum-idle=1

# Use SQL scripts for schema and data initialization
spring.jpa.hibernate.ddl-auto=none
spring.sql.init.mode=always
```

### Step 3: Write Your Tests

```java
@PgLiteTest
public class PgLiteIntegrationTests {

    @Autowired
    private VetRepository vets;

    @Test
    void testFindAll() {
        List<Vet> result = vets.findAll();
        assertThat(result).isNotEmpty();
    }
}
```

That's it! The extension handles all the infrastructure.

## Under the Hood: The Extension Implementation

The key parts of the `PgLiteExtension`:

### Starting the Server

```java
@Override
public void beforeAll(ExtensionContext context) throws Exception {
    int port = findAvailablePort();

    ProcessBuilder processBuilder = new ProcessBuilder(
        "npx", "@electric-sql/pglite-socket",
        "--port=" + port
    );

    Process process = processBuilder.start();

    // Check for immediate startup failure
    Thread.sleep(100);
    if (!process.isAlive()) {
        throw new RuntimeException(
            "pglite-server failed to start (exit code: " +
            process.exitValue() + ")"
        );
    }

    waitForServerReady(port);
    System.setProperty("pglite.port", String.valueOf(port));
}
```

### Health Check with JDBC

Instead of just checking if the port is open, we verify PostgreSQL protocol readiness:

```java
private boolean canConnectJdbc(int port) {
    String url = "jdbc:postgresql://localhost:" + port +
                 "/template1?sslmode=disable";
    try (Connection conn = DriverManager.getConnection(
            url, "postgres", "postgres");
         Statement stmt = conn.createStatement()) {
        stmt.execute("SELECT 1");
        return true;
    } catch (Exception e) {
        return false;
    }
}
```

### Graceful Shutdown

```java
@Override
public void afterAll(ExtensionContext context) throws Exception {
    Process process = store.remove(PROCESS_KEY, Process.class);
    if (process != null && process.isAlive()) {
        process.destroy();

        boolean terminated = process.waitFor(5, TimeUnit.SECONDS);
        if (!terminated) {
            process.destroyForcibly();
            process.waitFor(5, TimeUnit.SECONDS);
        }
    }

    System.clearProperty("pglite.port");
}
```

## Real-World Results

Testing the Spring PetClinic application with PGlite:

```
22:35:23.821 INFO  Starting PgLiteIntegrationTests
22:35:23.886 INFO  HikariPool-1 - Added connection
22:35:25.378 INFO  Initialized JPA EntityManagerFactory
22:35:25.931 INFO  Started PgLiteIntegrationTests in 2.236 seconds
```

**2.2 seconds** from start to finish, including:
- PGlite server startup
- Spring Boot context initialization
- Database schema creation
- Data loading
- Two integration tests

Compare this to typical Testcontainers startup (10-15 seconds) or full Docker Compose (15-30 seconds).

## When to Use PGlite vs. Testcontainers

### Choose PGlite when:
- You want faster test execution
- Docker isn't available in your environment
- You're testing standard PostgreSQL features
- You want to reduce infrastructure dependencies
- Single connection per test is acceptable

### Choose Testcontainers when:
- You need multiple concurrent connections
- You're testing PostgreSQL extensions
- You need persistent data between test runs
- You want to test the exact production PostgreSQL version
- Docker is already part of your infrastructure

## Lessons Learned

### Database Name Matters
PGlite uses `template1` as the default database, not `postgres`. Make sure your JDBC URL reflects this.

### Schema Strategy
We found that using SQL scripts (`schema.sql`/`data.sql`) works better than Hibernate's `ddl-auto=create` because the column ordering in Hibernate-generated schemas can differ from your data scripts.

### Single Connection Limitation
Setting `hikari.maximum-pool-size=1` is essential. The single connection limit sounds restrictive, but for most integration tests where operations are sequential, it's not an issue.

### Process Management
Always check if the process is alive immediately after starting. PGlite might fail instantly if npx can't find the package, and you don't want to wait 30 seconds for a timeout.

## Conclusion

PGlite provides a compelling middle ground between H2's speed and Testcontainers' accuracy. It's not a silver bullet - there are valid use cases for all three approaches - but it's a valuable addition to your testing toolkit.

The combination of true PostgreSQL compatibility, fast startup, and zero Docker dependency makes it particularly attractive for:
- Local development where Docker might be cumbersome
- CI/CD pipelines in restricted environments
- Teams wanting faster feedback loops
- Projects where infrastructure simplicity is valued

The JUnit 5 extension pattern makes it trivial to adopt: just add `@PgLiteTest` to your test class and you're done.

## Try It Yourself

The complete implementation is available in the [Spring PetClinic repository](https://github.com/spring-projects/spring-petclinic). Look for:
- `src/test/java/org/springframework/samples/petclinic/pglite/PgLiteExtension.java`
- `src/test/java/org/springframework/samples/petclinic/pglite/PgLiteTest.java`
- `src/test/java/org/springframework/samples/petclinic/PgLiteIntegrationTests.java`

**Prerequisites**: Node.js/npx installed (`node --version` should work)

Run the tests:
```bash
./mvnw test -Dtest=PgLiteIntegrationTests
```

## Resources

- [PGlite Documentation](https://pglite.dev/)
- [pglite-socket on npm](https://www.npmjs.com/package/@electric-sql/pglite-socket)
- [JUnit 5 Extensions Guide](https://junit.org/junit5/docs/current/user-guide/#extensions)
- [Spring Boot Testing Documentation](https://docs.spring.io/spring-boot/reference/testing/index.html)

---

*Have you tried PGlite in your projects? I'd love to hear about your experiences in the comments below.*
