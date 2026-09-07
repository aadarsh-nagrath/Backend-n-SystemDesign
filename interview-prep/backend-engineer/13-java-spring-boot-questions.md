# Java & Spring Boot — Interview Questions

> Sourced/topic-checked against InterviewBit's Spring Boot interview question list. A large share of backend engineering roles are Java/Spring-based — this file complements the language-agnostic backend files with JVM- and framework-specific questions.

---

### 1. What's the difference between JDK, JRE, and JVM?
**JVM** (Java Virtual Machine) — the runtime engine that actually executes compiled Java bytecode, providing platform independence ("write once, run anywhere," since the JVM itself is platform-specific but the bytecode it runs isn't). **JRE** (Java Runtime Environment) — the JVM plus the standard class libraries needed to *run* Java applications, but no development tools. **JDK** (Java Development Kit) — the JRE plus development tools (compiler `javac`, debugger, etc.) needed to *build* Java applications. You need the JDK to develop/compile Java code, but only the JRE (or just the JVM plus your app's dependencies, in modern minimal-runtime setups) to run already-compiled Java applications in production.

### 2. Explain Java garbage collection at a high level, and what "generational" garbage collection means.
The JVM automatically reclaims memory occupied by objects no longer reachable from any live reference, so developers don't manually `free()` memory as in C/C++ (though this doesn't eliminate all memory issues — see the memory-leak discussion in Q3). Most JVM garbage collectors are generational, based on the empirical observation that most objects die young: memory is divided into a small, frequently-collected "young generation" (using a fast, cheap collection algorithm, since most young objects are already garbage by the time it runs) and a larger, less-frequently-collected "old generation" for objects that have survived multiple young-generation collections (using a more thorough, more expensive algorithm, since old objects are more likely to still be genuinely in use) — this generational split is significantly more efficient than treating all memory uniformly.

### 3. Can a Java application still have a "memory leak" despite automatic garbage collection? Give an example.
Yes — a Java memory leak occurs when objects are no longer actually needed by the application logic but are still *reachable* via some reference the developer forgot to clear, so the garbage collector correctly (from its perspective) considers them still live and never reclaims them. Classic example: a `static` collection (like a `List` or `Map`) that objects are added to but never removed from as the application runs — since static fields live for the entire application lifetime, anything referenced from them is never eligible for collection, causing steadily growing memory usage over time that eventually leads to an `OutOfMemoryError`, functionally identical in symptom to a manual-memory-management leak even though the GC is working exactly as designed.

### 4. What is Spring's IoC (Inversion of Control) container, and how does it relate to Dependency Injection?
IoC is the broader principle of inverting control of object creation/wiring away from the application code itself and toward a framework (see also the Dependency Inversion principle in the OOP file). Spring's IoC container (the `ApplicationContext`) is the concrete implementation — it reads configuration (annotations, XML, or Java config classes) describing what "beans" (managed objects) to create and how they depend on each other, then instantiates and wires them together automatically. Dependency Injection is the specific *mechanism* the IoC container uses to fulfill this — injecting a bean's required dependencies (via constructor, setter, or field injection) rather than the bean creating or looking up its own dependencies internally, directly implementing the dependency-injection concept described in the backend fundamentals file.

### 5. What does `@SpringBootApplication` actually do internally?
It's a convenience meta-annotation bundling three separate annotations: `@Configuration` (marks the class as a source of Spring bean definitions), `@EnableAutoConfiguration` (triggers Spring Boot's auto-configuration mechanism — automatically configuring beans based on what's present on the classpath and any explicit properties, e.g., auto-configuring a `DataSource` bean if a JDBC driver and connection URL are detected), and `@ComponentScan` (tells Spring to automatically discover and register other annotated components — `@Service`, `@Repository`, `@Controller` — within the same package and its sub-packages, rather than requiring every bean to be manually declared).

### 6. What is Spring Boot auto-configuration, and how can you disable a specific piece of it if needed?
Auto-configuration inspects the classpath and existing bean definitions to automatically configure sensible-default beans for common needs (an embedded web server, a `DataSource`, a `JPA EntityManagerFactory`) without explicit manual configuration — this is Spring Boot's core value proposition over vanilla Spring (which required much more manual XML/Java configuration). You can exclude a specific auto-configuration class explicitly: `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` — useful when the auto-configured default conflicts with a custom configuration you want full manual control over instead.

### 7. What's the difference between `@Controller` and `@RestController`?
`@Controller` marks a class as a Spring MVC controller whose methods, by default, return *view names* (to be resolved to an HTML template/view) — you'd need to additionally annotate individual methods with `@ResponseBody` to have them return raw data (JSON) directly instead of a view name. `@RestController` is a convenience meta-annotation combining `@Controller` and `@ResponseBody` at the class level, so every method's return value is automatically serialized directly into the HTTP response body (typically as JSON) — the standard choice for building a REST API rather than a server-rendered HTML application.

### 8. What's the difference between `@RequestMapping` and `@GetMapping`/`@PostMapping`?
`@RequestMapping` is the general-purpose, original mapping annotation, requiring an explicit `method = RequestMethod.GET` attribute to restrict it to a specific HTTP method (without it, it matches any method). `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` are more specific, more readable shorthand annotations introduced later, each implicitly restricted to their named HTTP method — functionally equivalent to `@RequestMapping(method = RequestMethod.GET)` but clearer at a glance and the generally preferred style in modern Spring code.

### 9. What is Spring Data JPA, and how does it reduce boilerplate compared to writing raw JPA/Hibernate code?
Spring Data JPA builds on top of JPA/Hibernate (see the databases file's ORM discussion) by letting you define a repository interface (`interface UserRepository extends JpaRepository<User, Long>`) and get a full set of standard CRUD operations (`save`, `findById`, `delete`, etc.) automatically implemented at runtime — with no implementation code written at all. It also supports deriving queries directly from method names (`findByEmailAndActiveTrue(String email)` automatically generates the corresponding query) or custom `@Query` annotations for more complex needs, dramatically reducing the repetitive boilerplate that raw JPA/Hibernate data-access code traditionally required.

### 10. What are Spring Profiles, and what problem do they solve?
Profiles let you define environment-specific configuration/beans (`application-dev.properties`, `application-prod.properties`) and activate the appropriate set at runtime (`spring.profiles.active=prod`, or via an environment variable/command-line flag) — solving the problem of needing different configuration (database URLs, logging levels, feature flags, mock vs. real external service clients) across development, testing, and production environments without hardcoding environment-specific logic into the application code itself, directly paralleling the "build once, deploy many" principle from the CI/CD file (the same artifact, different config per environment).

### 11. What is Spring Boot Actuator, and what are some of its most useful endpoints?
Actuator adds production-ready operational/monitoring endpoints to a Spring Boot application with minimal setup: `/actuator/health` (application health status, integrable with load balancer health checks — see the networking file's readiness/liveness discussion), `/actuator/metrics` (JVM and application metrics, exportable to Prometheus/Micrometer for the observability stack described in the monitoring file), `/actuator/env` (current environment properties/configuration, useful for debugging "why is this config value not what I expect" issues), and `/actuator/beans` (lists every bean currently managed by the application context) — these endpoints should be secured/restricted in production (not exposed publicly without authentication), since they can reveal significant internal application detail.

### 12. How does exception handling work in a Spring Boot REST API, and what is `@ControllerAdvice` used for?
Rather than scattering `try/catch` blocks with manual error-response construction across every controller method, `@ControllerAdvice` (combined with `@ExceptionHandler`) defines centralized, global exception-handling logic — a single class can catch specific exception types thrown anywhere across the application's controllers and translate them into consistent, well-structured error responses (matching the error-response design principles in the API design file) with the appropriate HTTP status code, without duplicating that translation logic in every individual controller.

### 13. What is the default embedded web server in Spring Boot, and can you replace it?
Spring Boot defaults to embedded Apache Tomcat (default port 8080, configurable via `server.port`), meaning a Spring Boot application is a self-contained, runnable JAR with its own web server built in — no separate application server installation/deployment step needed, unlike older-style Java web application deployment (WAR files deployed to an external Tomcat/JBoss instance). You can swap it for Jetty or Undertow instead by excluding the default Tomcat starter dependency and including the alternative's starter — useful if a specific project has performance characteristics or feature needs (e.g., Undertow's lower memory footprint) better served by a different embedded server.

### 14. Can a Spring Boot application be built as a non-web application, and why would you do this?
Yes — omitting the `spring-boot-starter-web` dependency (and not including any web-server-triggering auto-configuration) produces a Spring Boot application with no embedded web server at all, useful for command-line tools, batch/scheduled job processors, or message-queue consumer applications that don't need to expose an HTTP API but still benefit from Spring's dependency injection, configuration management, and the broader Spring ecosystem (Spring Data, Spring Batch, Spring Integration).

### 15. How would you write a unit test for a Spring service class that depends on a repository, using Mockito?
Mock the repository dependency rather than using a real database (directly applying the dependency-injection-enables-testing principle from the OOP file and the mocking discussion in the testing file):
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenFound() {
        User mockUser = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        User result = userService.getUser(1L);

        assertEquals("Alice", result.getName());
    }
}
```
`@Mock` creates a fake `UserRepository` with fully controllable behavior; `@InjectMocks` automatically injects that mock into a real `UserService` instance — the test verifies `UserService`'s own logic in complete isolation, with no real database involved at all, consistent with the unit-vs-integration testing distinction covered in the testing file.

### 16. What's the difference between `@Component`, `@Service`, `@Repository`, and `@Controller` — are they functionally different to Spring?
All four are specializations of the base `@Component` annotation, and Spring's component scanning treats them identically for the purpose of *registering* a bean — the distinction is primarily semantic/documentation-oriented (signaling a class's architectural role to other developers) with one functional exception: `@Repository` additionally enables Spring's automatic exception translation, converting database-specific exceptions (e.g., a raw JDBC `SQLException`) into Spring's unified `DataAccessException` hierarchy, decoupling calling code from needing to know which specific underlying database/driver-specific exception type to catch.

### 17. What is Spring Security, and how does it fit alongside Spring Boot's auto-configuration?
Spring Security is Spring's comprehensive authentication/authorization framework (see the security-authentication file for the general concepts it implements — session/JWT-based auth, OAuth2/OIDC support, CSRF protection, method-level authorization via `@PreAuthorize`). Once the `spring-boot-starter-security` dependency is added, Spring Boot's auto-configuration automatically secures every endpoint by default (requiring authentication) with a generated default password logged at startup — a common "gotcha" for developers new to it who add the dependency expecting no behavior change and are surprised when their previously-open endpoints suddenly require login, until they explicitly configure a `SecurityFilterChain` bean defining their actual desired authorization rules.
