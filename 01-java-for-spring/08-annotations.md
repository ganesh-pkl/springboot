# Lesson 01-08 — Annotations

## 🎯 Learning Objective

Understand what Java annotations **actually are** mechanically — because Spring Boot is built on annotations, and developers who think they're "magic" eventually hit walls they can't debug.

---

## 🤔 Think Before Reading

You'll write this constantly in Spring Boot:

```java
@RestController
@RequestMapping("/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        // ...
    }
}
```

**Question**: What does `@RestController` actually DO? When you add it, what mechanism causes Spring to treat this class as an HTTP controller? Is it magic? Is the compiler handling it?

Think about this. Then read.

---

## 🧠 What Annotations Are Mechanically

Annotations are **metadata attached to code elements** (classes, methods, fields, parameters).

They are **not code**. They don't do anything by themselves.

Something else has to:
1. **Read** the annotation
2. **Decide** what to do based on it
3. **Act** on it

This is the fundamental truth that most Spring Boot tutorials skip.

---

## 💻 Defining a Custom Annotation

```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)           // Can be applied to methods
@Retention(RetentionPolicy.RUNTIME)   // Available at runtime via reflection
@Documented                           // Appears in Javadoc
public @interface LogExecutionTime {
    String level() default "INFO";    // Optional attribute with default
}
```

### Meta-annotations explained

| Meta-annotation | Purpose |
|----------------|---------|
| `@Target` | Where the annotation can be placed |
| `@Retention` | How long the annotation is kept |
| `@Documented` | Include in Javadoc |
| `@Inherited` | Subclasses inherit the annotation |

### @Target values

```java
ElementType.TYPE          // Class, interface, enum
ElementType.METHOD        // Method
ElementType.FIELD         // Field (instance variable)
ElementType.PARAMETER     // Method parameter
ElementType.CONSTRUCTOR   // Constructor
ElementType.ANNOTATION_TYPE // Another annotation
```

### @Retention values

```java
RetentionPolicy.SOURCE    // Discarded after compilation (e.g., @Override)
RetentionPolicy.CLASS     // In .class file, NOT available at runtime
RetentionPolicy.RUNTIME   // Available at runtime via reflection (Spring uses this)
```

**Critical**: Spring annotations use `RUNTIME` retention. This is what allows Spring to read them via Java reflection at startup.

---

## 🔍 Reading Annotations via Reflection

This is what Spring actually does:

```java
// Reading annotations at runtime
Class<?> clazz = UserController.class;

// Check if @RestController is present
boolean isController = clazz.isAnnotationPresent(RestController.class);

// Get the annotation and its attributes
RequestMapping mapping = clazz.getAnnotation(RequestMapping.class);
String[] paths = mapping.value(); // e.g., ["/users"]
```

```java
// Reading method annotations
Method method = clazz.getMethod("getUser", Long.class);
GetMapping getMapping = method.getAnnotation(GetMapping.class);
String[] methodPaths = getMapping.value(); // e.g., ["/{id}"]
```

This is how Spring builds its routing table — it scans your classes, reads annotations, and constructs a map of `HTTP method + path → controller method`.

---

## 🏗️ Creating a Working Custom Annotation

Let's build a simple `@LogExecutionTime` annotation that logs how long a method takes:

**Step 1**: Define the annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {
    String value() default ""; // Optional description
}
```

**Step 2**: Write a processor (in Spring, this would be an AOP aspect)

```java
// Simplified — not Spring AOP yet
public class AnnotationProcessor {
    
    public Object process(Object target, String methodName, Object[] args) throws Exception {
        Method method = target.getClass().getMethod(methodName);
        
        if (method.isAnnotationPresent(LogExecutionTime.class)) {
            LogExecutionTime annotation = method.getAnnotation(LogExecutionTime.class);
            long start = System.currentTimeMillis();
            
            Object result = method.invoke(target, args);
            
            long duration = System.currentTimeMillis() - start;
            System.out.println("[" + annotation.value() + "] Execution time: " + duration + "ms");
            return result;
        }
        
        return method.invoke(target, args);
    }
}
```

**Step 3**: Use the annotation

```java
public class UserService {
    
    @LogExecutionTime("Find all users")
    public List<User> findAll() {
        // ... database query
    }
}
```

In Spring Boot, Step 2 is handled by **AOP (Aspect-Oriented Programming)**. We'll cover this in detail in Level 10.

---

## 📋 How Spring Processes Core Annotations

Let's trace what Spring does with `@RestController`:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller              // ← @RestController is composed of @Controller
@ResponseBody            // ← and @ResponseBody
public @interface RestController {
    String value() default "";
}
```

`@RestController` is a **composed annotation** — it combines `@Controller` and `@ResponseBody`.

When Spring starts:
1. **Component scan** — Spring scans classes in your package
2. **Reads `@Controller`** — marks this class as a controller bean
3. **Reads `@ResponseBody`** — all methods should serialize return values to JSON/XML
4. **Reads `@RequestMapping`** — registers the URL prefix
5. **Reads method annotations** (`@GetMapping`, etc.) — registers individual routes
6. **Creates a `RequestMappingHandlerMapping`** — a lookup table from URL patterns to methods

This is not magic. It's reflection + annotation processing + a lot of well-organized code.

---

## 🧠 Mental Model: Annotation Processing in Spring

```
Application starts
      ↓
Spring creates ApplicationContext
      ↓
Component scan: Spring reads all .class files in your package
      ↓
For each class:
  - Is @Component (or @Service, @Repository, @Controller) present?
  - If yes: register as a bean definition
  - Read other annotations to configure behavior
      ↓
BeanPostProcessors run after bean creation:
  - @Transactional → wrap in transaction proxy
  - @Cacheable → wrap in cache proxy
  - @Async → wrap in async proxy
      ↓
Application is ready
```

Each annotation triggers a specific piece of Spring infrastructure.

---

## 🔍 Key Spring Annotations and What Processes Them

| Annotation | Processed By | What Happens |
|------------|-------------|--------------|
| `@Component` | `ClassPathBeanDefinitionScanner` | Registered as bean |
| `@Autowired` | `AutowiredAnnotationBeanPostProcessor` | Dependency injected |
| `@Transactional` | `TransactionInterceptor` + `BeanFactoryTransactionAttributeSource` | Wrapped in transaction proxy |
| `@Cacheable` | `CacheInterceptor` | Wrapped in cache proxy |
| `@Async` | `AsyncAnnotationAdvisor` | Wrapped in async executor proxy |
| `@Valid` | `MethodValidationInterceptor` | Triggers Bean Validation |
| `@GetMapping` | `RequestMappingHandlerMapping` | Registered as HTTP endpoint |

You don't need to memorize this table — but understanding the pattern is essential.

---

## 💥 Break It Exercise

This annotation won't work as intended. Why?

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS) // ← Notice this
public @interface Auditable {
    String action();
}

@Service
public class UserService {
    
    @Auditable(action = "CREATE_USER")
    public User createUser(CreateUserRequest request) {
        // ...
    }
}
```

And the processor:

```java
// In some Spring AOP aspect
Method method = joinPoint.getSignature() as MethodSignature .getMethod();
Auditable auditable = method.getAnnotation(Auditable.class); // Returns null!
```

<details>
<summary>Answer</summary>

`@Retention(RetentionPolicy.CLASS)` means the annotation is stored in the `.class` file but **NOT available at runtime via reflection**.

Spring uses reflection at runtime to process annotations. With `CLASS` retention, `method.getAnnotation(Auditable.class)` returns `null`.

Fix: Change to `@Retention(RetentionPolicy.RUNTIME)`.

This is one of the most common annotation gotchas.

</details>

---

## 🎯 Practice Exercise

Create a `@RateLimit` annotation:

```java
@RateLimit(maxRequests = 10, windowSeconds = 60)
public void sendEmail(String to, String content) {
    // ...
}
```

Requirements:
1. Define the annotation with `maxRequests` (default 100) and `windowSeconds` (default 60) attributes
2. It should be applicable to methods only
3. It should be available at runtime
4. Write a simple processor that reads the annotation from a method and prints the limits

Don't implement actual rate limiting — just read and print the annotation values.

<details>
<summary>Solution</summary>

```java
// Annotation definition
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimit {
    int maxRequests() default 100;
    int windowSeconds() default 60;
}

// Usage
public class EmailService {
    
    @RateLimit(maxRequests = 10, windowSeconds = 60)
    public void sendEmail(String to, String content) {
        System.out.println("Sending email to " + to);
    }
    
    @RateLimit // uses defaults: 100 requests per 60 seconds
    public void sendBulkEmail(List<String> recipients, String content) {
        System.out.println("Sending bulk email to " + recipients.size() + " recipients");
    }
}

// Simple processor (reads and prints limits)
public class RateLimitProcessor {
    
    public void logRateLimits(Class<?> clazz) {
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isAnnotationPresent(RateLimit.class)) {
                RateLimit limit = method.getAnnotation(RateLimit.class);
                System.out.printf(
                    "Method %s: max %d requests per %d seconds%n",
                    method.getName(),
                    limit.maxRequests(),
                    limit.windowSeconds()
                );
            }
        }
    }
}

// Test it
public class Main {
    public static void main(String[] args) {
        new RateLimitProcessor().logRateLimits(EmailService.class);
        // Output:
        // Method sendEmail: max 10 requests per 60 seconds
        // Method sendBulkEmail: max 100 requests per 60 seconds
    }
}
```

</details>

---

## 🎤 Interview Questions

1. **What is an annotation in Java? What does it actually do on its own?**
   > An annotation is metadata attached to code elements. By itself, it does nothing — something must read it via reflection and act on it.

2. **What is the difference between `@Retention(RUNTIME)` and `@Retention(CLASS)`?**
   > RUNTIME annotations are available via reflection while the program runs. CLASS annotations are in the .class file but not readable at runtime. Spring requires RUNTIME.

3. **How does Spring know what to do when it sees `@Transactional`?**
   > Spring's `BeanPostProcessor` infrastructure reads the annotation at startup, and at runtime, Spring creates a proxy around the bean that intercepts method calls, begins/commits/rolls back transactions.

---

## ✅ Completion Checklist

- [ ] Can you create a custom annotation with attributes?
- [ ] Can you explain what `@Target`, `@Retention`, `@Documented` mean?
- [ ] Can you explain how Spring reads annotations (reflection)?
- [ ] Can you explain why `@Retention(RUNTIME)` is required for Spring?
- [ ] Did you solve the `@RateLimit` exercise?

---

## 🏁 Level 0 Complete!

**Level 0 Checkpoint**: Before moving to Spring Fundamentals, make sure you can:

```
[ ] Write Java classes with proper encapsulation
[ ] Define and implement interfaces
[ ] Use List, Set, Map, Optional fluently
[ ] Handle exceptions (checked vs unchecked)
[ ] Write lambda expressions and Stream pipelines
[ ] Create records for DTOs
[ ] Understand what annotations are and how they work
```

If you checked all boxes: **Let's go to Level 1 — the heart of Spring**.

---

*Next: [Level 1 → Why Spring Exists](../02-spring-fundamentals/01-why-spring-exists.md)*
