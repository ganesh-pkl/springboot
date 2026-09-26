# Lesson 02-04 — Beans and Bean Lifecycle

## 🎯 Learning Objective

Understand what a Spring bean is, what scopes are available, and the complete lifecycle of a bean from definition to destruction. Know exactly when your code runs.

---

## 🤔 Think Before Reading

You have a `UserService` that opens a connection pool when it starts and must close it cleanly when the application shuts down.

**Question**: With plain Java, how would you handle this? With Spring, where would you put initialization and cleanup code?

---

## 🏷️ What Is a Spring Bean?

A **bean** is any object that is:
1. Created by the Spring container
2. Managed by the Spring container (lifecycle, dependencies)
3. Stored in the Spring container (can be retrieved by name or type)

Not every object in your application is a bean. `new String("hello")` is not a bean. `new UserRequest("Alice", ...)` is not a bean. Only objects that Spring creates and manages are beans.

---

## 🏗️ Declaring Beans

### Method 1: Stereotype Annotations (most common)

```java
@Component      // Generic component
@Service        // Business logic (same as @Component, semantically richer)
@Repository     // Data access (same as @Component + adds exception translation)
@Controller     // Web layer (same as @Component + Spring MVC)
@RestController // @Controller + @ResponseBody
```

```java
@Service
public class UserService {
    // Spring creates and manages this
}
```

### Method 2: @Bean in @Configuration

```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserRepository userRepository(DataSource dataSource) {
        return new JpaUserRepository(dataSource);
    }
    
    @Bean
    public EmailService emailService() {
        SmtpConfig config = new SmtpConfig("smtp.gmail.com", 587);
        return new EmailService(config);
    }
}
```

Use `@Bean` when:
- You don't own the class (third-party library)
- You need to configure the object with custom logic
- You need conditional bean creation

---

## 🔍 Bean Scopes

By default, Spring beans are **singletons**. But there are other scopes:

```java
// Singleton (default) — one instance per ApplicationContext
@Service
@Scope("singleton")
public class UserService { ... }

// Prototype — new instance every time the bean is requested
@Service
@Scope("prototype")
public class UserDraftService { ... }

// Request — one instance per HTTP request (web apps only)
@Service
@Scope("request")
public class RequestContextService { ... }

// Session — one instance per HTTP session (web apps only)
@Service
@Scope("session")
public class UserSessionService { ... }
```

### Singleton vs Prototype

```java
ApplicationContext context = ...;

// Singleton: both variables point to the same object
UserService s1 = context.getBean(UserService.class);
UserService s2 = context.getBean(UserService.class);
System.out.println(s1 == s2); // true

// Prototype: new object each time
UserDraftService d1 = context.getBean(UserDraftService.class);
UserDraftService d2 = context.getBean(UserDraftService.class);
System.out.println(d1 == d2); // false
```

### ⚠️ The Singleton Statelessness Rule

If a bean is singleton, it's shared by ALL threads handling ALL requests.

```java
// DANGEROUS — stateful singleton
@Service
public class UserService {
    private User currentUser; // Shared across ALL requests — race condition!
    
    public void setCurrentUser(User user) {
        this.currentUser = user; // Thread A sets it, Thread B reads it: WRONG USER!
    }
}

// CORRECT — stateless singleton
@Service
public class UserService {
    private final UserRepository userRepository; // Immutable dependency — safe
    
    public User findById(Long id) { // State is local to the method
        return userRepository.findById(id).orElseThrow();
    }
}
```

**Rule**: Singleton beans must be stateless. All state should be in method-local variables or in the database.

---

## 🔄 Bean Lifecycle — Complete Picture

```
1. BeanDefinition created (class detected during scan)
          ↓
2. BeanFactory post-processing (property resolution)
          ↓
3. Bean instantiation (constructor called)
          ↓
4. Dependency injection (dependencies set)
          ↓
5. Aware interfaces (BeanNameAware, ApplicationContextAware)
          ↓
6. BeanPostProcessor.postProcessBeforeInitialization()
          ↓
7. @PostConstruct methods called
          ↓
8. InitializingBean.afterPropertiesSet() (if implemented)
          ↓
9. @Bean(initMethod = "...") called (if specified)
          ↓
10. BeanPostProcessor.postProcessAfterInitialization()
    (Proxies created here for @Transactional, @Cacheable, etc.)
          ↓
11. Bean is READY — stored in context, used by application
          ↓
      ... application runs ...
          ↓
12. @PreDestroy methods called (on shutdown)
          ↓
13. DisposableBean.destroy() (if implemented)
          ↓
14. @Bean(destroyMethod = "...") called (if specified)
          ↓
15. Bean is destroyed
```

The critical ones to remember: steps 3, 4, 7, 10, 12.

---

## 💻 Lifecycle Callbacks in Practice

```java
@Service
public class DatabaseConnectionService {
    
    private Connection connection;
    
    // Step 7: Called after all dependencies are injected
    @PostConstruct
    public void initialize() {
        System.out.println("Initializing database connection...");
        this.connection = createConnection();
        System.out.println("Database connection ready");
    }
    
    // Step 12: Called before the bean is destroyed
    @PreDestroy
    public void cleanup() {
        System.out.println("Closing database connection...");
        if (connection != null) {
            connection.close();
        }
    }
    
    private Connection createConnection() {
        // ... create connection
        return new Connection();
    }
}
```

### Why not do initialization in the constructor?

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    // Called BEFORE dependencies are injected
    public UserService() {
        // userRepository is null here! Can't use it.
        userRepository.findAll(); // NullPointerException!
    }
}
```

Use `@PostConstruct` — it runs AFTER dependencies are injected.

---

## 🔍 BeanPostProcessor — The Most Powerful Spring Hook

`BeanPostProcessor` runs code before and after each bean is initialized. This is how Spring adds proxy wrapping.

```java
// Spring's internal mechanism (simplified)
public interface BeanPostProcessor {
    
    // Called BEFORE @PostConstruct
    Object postProcessBeforeInitialization(Object bean, String beanName);
    
    // Called AFTER @PostConstruct — THIS is where proxies are created
    Object postProcessAfterInitialization(Object bean, String beanName);
}
```

When Spring's `TransactionProxyCreator` runs `postProcessAfterInitialization`:

```
Input:  your UserService instance
Output: a Proxy object that wraps UserService
        (the proxy intercepts method calls and adds transaction management)
```

The proxy is stored in the container, not your original bean. When code asks for `UserService`, it gets the proxy.

```
@Autowired UserService userService; 
// This is actually a TransactionProxy{target=UserService}
// Calls go through proxy → transaction management → your code
```

---

## 🏗️ Lazy Initialization

By default, singleton beans are created at startup. This can slow startup time.

```java
// Only created when first requested
@Service
@Lazy
public class HeavyService {
    // Takes 3 seconds to initialize
    @PostConstruct
    public void init() { /* slow initialization */ }
}
```

In Spring Boot, you can make ALL beans lazy:

```yaml
# application.yml
spring:
  main:
    lazy-initialization: true
```

> Trade-off: Faster startup, but first request to each bean is slower. Also, errors in bean creation only surface on first use, not at startup.

---

## 💥 Break It Exercise

What's wrong with this code?

```java
@Service
public class CacheService {
    
    private Map<String, Object> cache = new HashMap<>();
    
    public void put(String key, Object value) {
        cache.put(key, value);
    }
    
    public Object get(String key) {
        return cache.get(key);
    }
    
    public void clear() {
        cache.clear();
    }
}
```

No Spring annotation issue — the Spring setup is fine. The problem is in the design.

<details>
<summary>Answer</summary>

`HashMap` is **not thread-safe**.

`CacheService` is a singleton shared across all threads. If two threads call `put()` simultaneously on a regular `HashMap`, you can get data corruption.

Fix:
```java
// Option 1: ConcurrentHashMap
private Map<String, Object> cache = new ConcurrentHashMap<>();

// Option 2: Use Spring's built-in cache abstraction (@Cacheable)
// Option 3: Use Redis for distributed caching
```

This is a very common singleton bug. Singleton beans must be thread-safe.

</details>

---

## 🎯 Practice Exercise

Create a `MetricsService` that:

1. Has a `Map<String, Long>` to store request counts per endpoint
2. Uses `@PostConstruct` to print "MetricsService started" and seed 3 default counters
3. Has a method `increment(String endpoint)` that adds 1 to the counter
4. Has a method `getCount(String endpoint)` that returns the count (0 if not found)
5. Uses `@PreDestroy` to print all final counter values before shutdown
6. Uses `ConcurrentHashMap` (thread safety!)

<details>
<summary>Solution</summary>

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Service;

import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Service
public class MetricsService {
    
    private Map<String, Long> counters;
    
    @PostConstruct
    public void initialize() {
        System.out.println("MetricsService started");
        counters = new ConcurrentHashMap<>();
        // Seed default counters
        counters.put("/api/users", 0L);
        counters.put("/api/products", 0L);
        counters.put("/api/orders", 0L);
    }
    
    public void increment(String endpoint) {
        counters.merge(endpoint, 1L, Long::sum);
        // merge: if key exists, apply the function; if not, put the initial value
    }
    
    public long getCount(String endpoint) {
        return counters.getOrDefault(endpoint, 0L);
    }
    
    @PreDestroy
    public void shutdown() {
        System.out.println("=== Final Metrics ===");
        counters.forEach((endpoint, count) -> 
            System.out.printf("  %s: %d requests%n", endpoint, count)
        );
    }
}
```

</details>

---

## 🎤 Interview Questions

**Easy**:
1. What is the default scope of a Spring bean?
   > Singleton — one instance per ApplicationContext.

2. When does `@PostConstruct` run?
   > After the bean is created and all dependencies are injected, before the bean is put into service.

**Intermediate**:
3. Why should singleton beans be stateless?
   > Singletons are shared across all threads. Instance variables in a singleton can be read/written by multiple threads simultaneously, causing race conditions.

4. What is `BeanPostProcessor` and what is it used for?
   > An interface that lets you run custom code before and after each bean's initialization. Spring uses it to create proxies for `@Transactional`, `@Cacheable`, `@Async` beans.

**Advanced**:
5. What's the difference between `@PostConstruct` and `InitializingBean.afterPropertiesSet()`?
   > Both run after DI, but `@PostConstruct` is a standard Java annotation (not Spring-specific), making your code less coupled to Spring. Prefer `@PostConstruct`. `afterPropertiesSet` is Spring-specific.

---

## ✅ Completion Checklist

- [ ] Can you name all 4 stereotype annotations and their differences?
- [ ] Can you explain singleton vs prototype scope?
- [ ] Can you explain the bean lifecycle in 6-7 key steps?
- [ ] Can you explain why `@PostConstruct` is needed (vs constructor)?
- [ ] Can you explain what `BeanPostProcessor` does?
- [ ] Do you know why singletons must be stateless?

---

*Next: [02-05 — Component Scanning](./05-component-scanning.md)*
