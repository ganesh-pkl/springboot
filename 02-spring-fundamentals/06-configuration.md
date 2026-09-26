# Lesson 02-06 — Configuration (@Configuration and @Bean)

## 🎯 Learning Objective

Understand how to manually define beans when annotations alone aren't enough — and understand when to use `@Configuration` vs stereotype annotations.

---

## 🤔 Think Before Reading

You're integrating with a third-party library for sending emails:

```java
// This is from a JAR file — you can't modify it
public class SendgridEmailClient {
    public SendgridEmailClient(String apiKey, int timeout) { ... }
    public void send(Email email) { ... }
}
```

**Question**: How do you make Spring manage this object as a bean? You can't add `@Service` to it — it's not your code.

---

## 💡 The @Configuration + @Bean Solution

```java
@Configuration
public class EmailConfig {
    
    @Value("${sendgrid.api-key}") // Reads from application.properties
    private String apiKey;
    
    @Bean
    public SendgridEmailClient sendgridEmailClient() {
        return new SendgridEmailClient(apiKey, 30);
        // Spring manages this object — it becomes a bean
    }
}
```

Now `SendgridEmailClient` is a Spring bean. Any class can inject it:

```java
@Service
public class EmailService {
    private final SendgridEmailClient emailClient;
    
    public EmailService(SendgridEmailClient emailClient) {
        this.emailClient = emailClient; // Spring injects the bean from EmailConfig
    }
}
```

---

## 🔍 What @Configuration Does

`@Configuration` marks a class as a source of bean definitions.

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        ds.setUsername("user");
        ds.setPassword("pass");
        ds.setMaximumPoolSize(10);
        return ds;
    }
    
    @Bean
    public UserRepository userRepository(DataSource dataSource) {
        // Spring sees the DataSource parameter and injects the bean above
        return new JpaUserRepository(dataSource);
    }
}
```

**Important**: Spring calls `@Bean` methods once and stores the result. Singleton by default — just like stereotype annotations.

---

## 🔍 @Configuration vs @Component — The Proxy Difference

`@Configuration` is special. Spring subclasses it with CGLIB to add proxy behavior:

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DatabaseConfig databaseConfig() {
        return new DatabaseConfig("jdbc:postgresql://...");
    }
    
    @Bean
    public UserRepository userRepository() {
        // Calls databaseConfig()
        DatabaseConfig config = databaseConfig(); // Is this a new object?
    }
    
    @Bean
    public AuditRepository auditRepository() {
        // Also calls databaseConfig()
        DatabaseConfig config = databaseConfig(); // Same object or new?
    }
}
```

**Without `@Configuration`** (using `@Component` instead): Each call to `databaseConfig()` would create a NEW object. You'd get two different `DatabaseConfig` instances.

**With `@Configuration`**: Spring's CGLIB proxy intercepts the method call to `databaseConfig()`. If the bean already exists in the container, it returns the existing singleton. Both `userRepository` and `auditRepository` get the SAME `DatabaseConfig` instance.

This is called **"Full" Configuration Mode**.

---

## 💻 Common @Bean Patterns

### Pattern 1: Simple bean registration

```java
@Configuration
public class AppConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.registerModule(new JavaTimeModule());
        return mapper;
    }
}
```

### Pattern 2: Bean with dependencies

```java
@Configuration
public class ServiceConfig {
    
    @Bean
    public UserService userService(
            UserRepository userRepository,      // Injected from another @Bean
            EmailService emailService,          // Injected from another @Bean
            @Value("${app.user.max-login-attempts}") int maxAttempts) {
        return new UserService(userRepository, emailService, maxAttempts);
    }
}
```

### Pattern 3: Conditional beans

```java
@Configuration
public class StorageConfig {
    
    // Only creates this bean if "storage.type=s3" in properties
    @Bean
    @ConditionalOnProperty(name = "storage.type", havingValue = "s3")
    public StorageService s3StorageService(@Value("${aws.s3.bucket}") String bucket) {
        return new S3StorageService(bucket);
    }
    
    // Default fallback
    @Bean
    @ConditionalOnMissingBean(StorageService.class)
    public StorageService localStorageService(@Value("${storage.local.path}") String path) {
        return new LocalStorageService(path);
    }
}
```

### Pattern 4: Named beans for multiple implementations

```java
@Configuration
public class NotificationConfig {
    
    @Bean("emailNotificationService")
    public NotificationService emailService(EmailClient emailClient) {
        return new EmailNotificationService(emailClient);
    }
    
    @Bean("smsNotificationService")
    public NotificationService smsService(TwilioClient twilioClient) {
        return new SmsNotificationService(twilioClient);
    }
}
```

---

## 🔄 @Configuration vs Stereotype Annotations — When to Use Which

| Situation | Use |
|-----------|-----|
| Your own class (service, repository) | `@Service`, `@Repository`, etc. |
| Third-party class | `@Configuration` + `@Bean` |
| Need complex initialization logic | `@Configuration` + `@Bean` |
| Conditional creation | `@Configuration` + `@Bean` + `@Conditional` |
| Multiple instances of same type | `@Configuration` + `@Bean` with names |

---

## 🏗️ @Value — Reading Configuration

```java
@Configuration
public class AppConfig {
    
    // Read from application.properties / environment variables
    @Value("${server.port}")
    private int serverPort;
    
    @Value("${app.name:MyApp}") // Default value after the colon
    private String appName;
    
    @Value("${app.allowed-origins}") // Can be a list
    private List<String> allowedOrigins;
    
    @Bean
    public AppSettings appSettings() {
        return new AppSettings(serverPort, appName, allowedOrigins);
    }
}
```

---

## 🔍 @ConfigurationProperties — Better for Multiple Properties

```java
// application.yml
app:
  email:
    host: smtp.gmail.com
    port: 587
    username: app@example.com
    password: secret
    timeout: 30
```

```java
// Bind the entire namespace to a class
@ConfigurationProperties(prefix = "app.email")
@Configuration
public class EmailProperties {
    private String host;
    private int port;
    private String username;
    private String password;
    private int timeout;
    
    // Getters and setters (required for binding)
    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    // ... etc
}

// Usage
@Service
public class EmailService {
    private final EmailProperties emailProperties;
    
    public EmailService(EmailProperties emailProperties) {
        this.emailProperties = emailProperties;
    }
    
    public void connect() {
        System.out.println("Connecting to: " + emailProperties.getHost() + ":" + emailProperties.getPort());
    }
}
```

`@ConfigurationProperties` is better than multiple `@Value` annotations for related properties.

---

## 💥 Break It Exercise

What's wrong with this configuration?

```java
@Component  // ← Note: @Component, not @Configuration
public class AppConfig {
    
    @Bean
    public DataSource primaryDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://primary-host/db");
        return ds;
    }
    
    @Bean
    public UserRepository userRepository() {
        return new JpaUserRepository(primaryDataSource()); // Direct method call
    }
    
    @Bean
    public AuditRepository auditRepository() {
        return new JpaAuditRepository(primaryDataSource()); // Direct method call
    }
}
```

<details>
<summary>Answer</summary>

Using `@Component` instead of `@Configuration` disables Spring's CGLIB proxy (this is called "Lite Mode").

In Lite Mode, method calls to `@Bean` methods are **not intercepted** by Spring. So:
- `userRepository()` calls `primaryDataSource()` → creates a new `HikariDataSource` instance #1
- `auditRepository()` calls `primaryDataSource()` → creates ANOTHER new `HikariDataSource` instance #2

You now have TWO connection pools pointing to the same database, with TWO sets of connections. This wastes connections and may cause issues.

Fix: Change `@Component` to `@Configuration`.

This is a subtle but important difference.

</details>

---

## 🎯 Practice Exercise

Create a configuration class for a Redis connection:

Given these properties in `application.yml`:
```yaml
redis:
  host: localhost
  port: 6379
  password: mypassword
  timeout: 2000
  max-connections: 10
```

1. Create a `RedisProperties` class with `@ConfigurationProperties(prefix = "redis")`
2. Create a `RedisConfig` class with a `@Bean` method that returns a `RedisConnectionConfig` object initialized from the properties
3. Create a simple `CacheService` that takes `RedisConnectionConfig` via constructor injection and has a `connect()` method that prints the host and port

<details>
<summary>Solution</summary>

```java
// Properties class
@ConfigurationProperties(prefix = "redis")
public class RedisProperties {
    private String host;
    private int port;
    private String password;
    private int timeout;
    private int maxConnections;
    
    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public int getTimeout() { return timeout; }
    public void setTimeout(int timeout) { this.timeout = timeout; }
    public int getMaxConnections() { return maxConnections; }
    public void setMaxConnections(int maxConnections) { this.maxConnections = maxConnections; }
}

// Config class
@Configuration
@EnableConfigurationProperties(RedisProperties.class)
public class RedisConfig {
    
    @Bean
    public RedisConnectionConfig redisConnectionConfig(RedisProperties props) {
        return new RedisConnectionConfig(
            props.getHost(),
            props.getPort(),
            props.getPassword(),
            props.getTimeout(),
            props.getMaxConnections()
        );
    }
}

// Simple domain class (not a bean — just data)
public class RedisConnectionConfig {
    private final String host;
    private final int port;
    private final String password;
    private final int timeout;
    private final int maxConnections;
    
    public RedisConnectionConfig(String host, int port, String password, int timeout, int maxConnections) {
        this.host = host;
        this.port = port;
        this.password = password;
        this.timeout = timeout;
        this.maxConnections = maxConnections;
    }
    
    public String getHost() { return host; }
    public int getPort() { return port; }
}

// Service
@Service
public class CacheService {
    private final RedisConnectionConfig config;
    
    public CacheService(RedisConnectionConfig config) {
        this.config = config;
    }
    
    public void connect() {
        System.out.println("Connecting to Redis at " + config.getHost() + ":" + config.getPort());
    }
}
```

</details>

---

## 🏁 Level 1 Checkpoint

**Before moving to Level 2**, can you answer these without notes?

```
[ ] What problem did Spring solve?
[ ] What is IoC? What is DI?
[ ] What is the ApplicationContext?
[ ] What is a BeanDefinition vs a Bean instance?
[ ] What are the 5 stereotype annotations?
[ ] What is the bean lifecycle in 5 key steps?
[ ] When do you use @Configuration + @Bean vs @Service?
[ ] What is the difference between singleton and prototype scope?
[ ] Why must singleton beans be stateless?
[ ] What is BeanPostProcessor and what does Spring use it for?
```

If you can't answer any of these, go back to the relevant lesson.

**Build Challenge**: Without looking at your notes, write a `UserService` that:
- Takes `UserRepository` and `AuditService` via constructor injection
- Has `@PostConstruct` to print "UserService ready"
- Has a `findById(Long id)` method that throws `UserNotFoundException` if not found
- Has `@PreDestroy` to print "UserService shutting down"

---

*Next: [Level 2 → Why Spring Boot Exists](../03-spring-boot-fundamentals/01-why-spring-boot.md)*
