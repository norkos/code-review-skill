# Java Code Review Guide

Java review focus: Java 17/21 features, Spring Boot 3 best practices, concurrency (virtual threads), JPA performance, and code maintainability.

> For Java 8 / Spring Boot 2 / `javax.*` legacy stacks, use the [Java 8 Guide](java8.md).

## Table of Contents

- [Modern Java Features (17/21+)](#modern-java-features-1721)
- [Stream API & Optional](#stream-api--optional)
- [Spring Boot Best Practices](#spring-boot-best-practices)
- [JPA and Database Performance](#jpa-and-database-performance)
- [Concurrency and Virtual Threads](#concurrency-and-virtual-threads)
- [Lombok Usage Guidelines](#lombok-usage-guidelines)
- [Exception Handling](#exception-handling)
- [Testing Guidelines](#testing-guidelines)
- [Review Checklist](#review-checklist)

---

## Modern Java Features (17/21+)

### Records

```java
// ❌ Traditional POJO/DTO: lots of boilerplate
public class UserDto {
    private final String name;
    private final int age;

    public UserDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
    // getters, equals, hashCode, toString...
}

// ✅ Use a record: concise, immutable, clear semantics
public record UserDto(String name, int age) {
    // Compact constructor for validation
    public UserDto {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

### Switch Expressions and Pattern Matching

```java
// ❌ Traditional switch: easy to forget break; verbose and error-prone
String type = "";
switch (obj) {
    case Integer i: // Java 16+
        type = String.format("int %d", i);
        break;
    case String s:
        type = String.format("string %s", s);
        break;
    default:
        type = "unknown";
}

// ✅ Switch expression: no fall-through risk, value must be returned
String type = switch (obj) {
    case Integer i -> "int %d".formatted(i);
    case String s  -> "string %s".formatted(s);
    case null      -> "null value"; // Java 21 handles null
    default        -> "unknown";
};
```

### Text Blocks

```java
// ❌ Concatenating SQL/JSON strings
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 20\n" +
              "}";

// ✅ Use a text block: what you see is what you get
String json = """
    {
      "name": "Alice",
      "age": 20
    }
    """;
```

---

## Stream API & Optional

### Don't Overuse Streams

```java
// ❌ A simple loop doesn't need a Stream (overhead + worse readability)
items.stream().forEach(item -> {
    process(item);
});

// ✅ Use for-each for simple cases
for (var item : items) {
    process(item);
}

// ❌ Extremely complex Stream chain
List<Dto> result = list.stream()
    .filter(...)
    .map(...)
    .peek(...)
    .sorted(...)
    .collect(...); // hard to debug

// ✅ Split into meaningful steps
var filtered = list.stream().filter(...).toList();
// ...
```

### Using Optional Correctly

```java
// ❌ Using Optional as a parameter or field (serialization issues, more complex call sites)
public void process(Optional<String> name) { ... }
public class User {
    private Optional<String> email; // not recommended
}

// ✅ Use Optional only for return values
public Optional<User> findUser(String id) { ... }

// ❌ Using Optional but still calling isPresent() + get()
Optional<User> userOpt = findUser(id);
if (userOpt.isPresent()) {
    return userOpt.get().getName();
} else {
    return "Unknown";
}

// ✅ Use the functional API
return findUser(id)
    .map(User::getName)
    .orElse("Unknown");
```

---

## Spring Boot Best Practices

### Dependency Injection (DI)

```java
// ❌ Field injection (@Autowired)
// Downsides: hard to test (needs reflection), hides too many dependencies, poor immutability
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
}

// ✅ Constructor injection
// Upsides: explicit dependencies, easy to unit test (mocks), fields can be final
@Service
public class UserService {
    private final UserRepository userRepo;

    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// 💡 Tip: Lombok @RequiredArgsConstructor simplifies this, but watch out for circular dependencies
```

### Configuration Management

```java
// ❌ Hardcoded configuration values
@Service
public class PaymentService {
    private String apiKey = "sk_live_12345";
}

// ❌ @Value scattered throughout the code
@Value("${app.payment.api-key}")
private String apiKey;

// ✅ Use @ConfigurationProperties for type-safe configuration
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String apiKey, int timeout, String url) {}
```

---

## JPA and Database Performance

### N+1 Query Problem

> 📖 For general principles and cross-language solutions, see the [N+1 Queries Cross-Language Guide](cross-cutting/n-plus-one-queries.md)

```java
// ❌ FetchType.EAGER or lazy loading triggered inside a loop
// Entity definition
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER) // Dangerous!
    private List<Order> orders;
}

// Business code
List<User> users = userRepo.findAll(); // 1 SQL statement
for (User user : users) {
    // If Lazy, this triggers N SQL statements
    System.out.println(user.getOrders().size());
}

// ✅ Use @EntityGraph or JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### Transaction Management

```java
// ❌ Opening transactions in the Controller layer (holds DB connections too long)
// ❌ @Transactional on a private method (AOP doesn't apply)
@Transactional
private void saveInternal() { ... }

// ✅ @Transactional on public Service-layer methods
// ✅ Explicitly mark reads with readOnly = true (performance optimization)
@Service
public class UserService {
    @Transactional(readOnly = true)
    public User getUser(Long id) { ... }

    @Transactional
    public void createUser(UserDto dto) { ... }
}
```

### Entity Design

```java
// ❌ Using Lombok @Data on an Entity
// @Data's equals/hashCode includes all fields and may trigger lazy loading, causing performance issues or exceptions
@Entity
@Data
public class User { ... }

// ✅ Use only @Getter, @Setter
// ✅ Custom equals/hashCode (usually based on ID)
@Entity
@Getter
@Setter
public class User {
    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id != null && id.equals(((User) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

---

## Concurrency and Virtual Threads

### Virtual Threads (Java 21+)

```java
// ❌ Traditional thread pool for many blocking I/O tasks (resource exhaustion)
ExecutorService executor = Executors.newFixedThreadPool(100);

// ✅ Use virtual threads for I/O-bound tasks (high throughput)
// Enable in Spring Boot 3.2+: spring.threads.virtual.enabled=true
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// On virtual threads, blocking operations (DB queries, HTTP requests) barely consume OS thread resources
```

### Thread Safety

```java
// ❌ SimpleDateFormat is not thread-safe
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// ✅ Use DateTimeFormatter (Java 8+)
private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// ❌ HashMap loses data under concurrency (in Java 7 and earlier, resize could also loop forever; Java 8 fixed the infinite loop but it's still not thread-safe)
// ✅ Use ConcurrentHashMap
Map<String, String> cache = new ConcurrentHashMap<>();
```

---

## Lombok Usage Guidelines

```java
// ❌ Overusing @Builder makes it impossible to enforce required fields
@Builder
public class Order {
    private String id; // required
    private String note; // optional
}
// Callers may omit id: Order.builder().note("hi").build();

// ✅ For key business objects, hand-write the Builder or constructor to enforce invariants
// Or add validation logic in build() (Lombok @Builder.Default, etc.)
```

---

## Exception Handling

### Global Exception Handling

```java
// ❌ try-catch everywhere that swallows exceptions or only logs them
try {
    userService.create(user);
} catch (Exception e) {
    e.printStackTrace(); // shouldn't be used in production
    // return null; // swallows the exception; callers don't know what happened
}

// ✅ Custom exceptions + @ControllerAdvice (Spring Boot 3 ProblemDetail)
public class UserNotFoundException extends RuntimeException { ... }

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

---

## Testing Guidelines

### Unit Tests vs Integration Tests

```java
// ❌ Unit tests depending on a real database or external services
@SpringBootTest // boots the whole Context; slow
public class UserServiceTest { ... }

// ✅ Unit tests use Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void shouldCreateUser() { ... }
}

// ✅ Integration tests use Testcontainers
@Testcontainers
@SpringBootTest
class UserRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // ...
}
```

---

## Review Checklist

### Fundamentals and Conventions
- [ ] Uses Java 17/21 features (switch expressions, records, text blocks)
- [ ] Avoids outdated classes (Date, Calendar, SimpleDateFormat)
- [ ] Do collection operations prefer the Stream API or Collections methods?
- [ ] Optional is used only for return values, not fields or parameters

### Spring Boot
- [ ] Uses constructor injection instead of @Autowired field injection
- [ ] Configuration properties use @ConfigurationProperties
- [ ] Controllers have a single responsibility; business logic lives in Services
- [ ] Global exception handling uses @ControllerAdvice / ProblemDetail

### Database & Transactions
- [ ] Read-only transactions are marked `@Transactional(readOnly = true)`
- [ ] Checked for N+1 queries (EAGER fetch or calls inside loops)
- [ ] Entity classes don't use @Data and implement equals/hashCode correctly
- [ ] Database indexes cover the query conditions

### Concurrency and Performance
- [ ] Were virtual threads considered for I/O-bound tasks?
- [ ] Are thread-safe classes used correctly (ConcurrentHashMap vs HashMap)?
- [ ] Is lock granularity reasonable? Avoid I/O while holding a lock

### Maintainability
- [ ] Key business logic has adequate unit tests
- [ ] Logging is appropriate (use Slf4j, avoid System.out)
- [ ] Magic values are extracted into constants or enums
