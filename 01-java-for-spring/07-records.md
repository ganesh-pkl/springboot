# Lesson 01-07 — Records

## 🎯 Learning Objective

Learn Java Records — the modern, concise way to create immutable data classes, and why they're perfect for DTOs in Spring Boot.

---

## 🤔 Think Before Reading

To create a simple "user data transfer object" in Java (pre-Java 16), you had to write:

```java
public class UserRequest {
    private String name;
    private String email;
    private String password;
    
    public UserRequest() {}
    
    public UserRequest(String name, String email, String password) {
        this.name = name;
        this.email = email;
        this.password = password;
    }
    
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPassword() { return password; }
    
    public void setName(String name) { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setPassword(String password) { this.password = password; }
    
    @Override
    public boolean equals(Object o) { /* boilerplate */ }
    
    @Override
    public int hashCode() { /* boilerplate */ }
    
    @Override
    public String toString() { /* boilerplate */ }
}
```

**That's ~40 lines** to model a simple object with 3 fields.

Now look at the JavaScript equivalent:

```javascript
// JavaScript doesn't need any of that
const userRequest = { name, email, password };
```

Java 16 introduced Records to close this gap.

---

## 💡 Records — The Modern Solution

```java
// Java 16+ — this REPLACES those 40 lines
public record UserRequest(String name, String email, String password) {}
```

One line. That's it. Java automatically generates:
- Constructor with all fields
- Getters (but named `name()`, not `getName()`)
- `equals()` and `hashCode()`
- `toString()`

---

## 💻 Record Syntax and Features

```java
public record UserRequest(
    String name,
    String email,
    String password
) {}

// Usage
UserRequest request = new UserRequest("Alice", "alice@example.com", "secret123");

System.out.println(request.name());     // "Alice" — getter is field name, not getName()
System.out.println(request.email());    // "alice@example.com"
System.out.println(request);            // UserRequest[name=Alice, email=alice@example.com, ...]
```

### Records are immutable

```java
UserRequest request = new UserRequest("Alice", "alice@example.com", "secret");
request.name = "Bob"; // COMPILE ERROR — no setters, fields are final
```

This is intentional and good. DTOs should not be mutated.

---

## 🔍 Adding Methods to Records

```java
public record Money(double amount, String currency) {
    
    // Compact constructor — validates on creation
    public Money {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        if (currency == null || currency.isBlank()) {
            throw new IllegalArgumentException("Currency is required");
        }
    }
    
    // Custom methods
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount + other.amount, this.currency);
    }
    
    public boolean isZero() {
        return this.amount == 0;
    }
    
    public String formatted() {
        return String.format("%.2f %s", amount, currency);
    }
}

// Usage
Money price = new Money(29.99, "USD");
Money tax = new Money(2.40, "USD");
Money total = price.add(tax);
System.out.println(total.formatted()); // 32.39 USD
```

---

## 🏗️ Records in Spring Boot — DTO Pattern

Records are perfect for:
- **Request DTOs** (incoming data from client)
- **Response DTOs** (outgoing data to client)
- **Event objects**
- **Value objects**

```java
// Request DTO — what the client sends
public record CreateTaskRequest(
    String title,
    String description,
    String priority,      // "LOW", "MEDIUM", "HIGH"
    LocalDate dueDate
) {}

// Response DTO — what we send back
public record TaskResponse(
    Long id,
    String title,
    String description,
    String priority,
    String status,
    LocalDate dueDate,
    LocalDateTime createdAt
) {}

// Update DTO — partial updates
public record UpdateTaskRequest(
    String title,
    String description,
    String priority,
    LocalDate dueDate
) {}
```

### In a Controller

```java
@PostMapping("/tasks")
public ResponseEntity<TaskResponse> createTask(@RequestBody CreateTaskRequest request) {
    TaskResponse response = taskService.create(request);
    return ResponseEntity.status(201).body(response);
}

@GetMapping("/tasks/{id}")
public ResponseEntity<TaskResponse> getTask(@PathVariable Long id) {
    TaskResponse response = taskService.findById(id);
    return ResponseEntity.ok(response);
}
```

### In a Service

```java
public TaskResponse create(CreateTaskRequest request) {
    Task task = new Task(
        request.title(),
        request.description(),
        request.priority(),
        request.dueDate()
    );
    Task saved = taskRepository.save(task);
    return mapToResponse(saved);
}

private TaskResponse mapToResponse(Task task) {
    return new TaskResponse(
        task.getId(),
        task.getTitle(),
        task.getDescription(),
        task.getPriority(),
        task.getStatus(),
        task.getDueDate(),
        task.getCreatedAt()
    );
}
```

---

## 🔄 Records with Validation Annotations

Records work with Bean Validation:

```java
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password,
    
    @Min(value = 0, message = "Age cannot be negative")
    @Max(value = 150, message = "Age seems unrealistic")
    Integer age
) {}
```

---

## 🧠 Records vs Regular Classes — When to Use Which

| Scenario | Use |
|----------|-----|
| DTOs (request/response) | `record` |
| Value objects (Money, Email, Address) | `record` |
| Domain entities (User, Order, Product) | Regular class (entities need mutability for JPA) |
| Service classes | Regular class |
| Configuration classes | Regular class |
| Objects that need to change over time | Regular class |

> **Important**: JPA entities cannot be records. JPA needs no-arg constructors and mutable fields. Use records only for DTOs and value objects.

---

## 💥 Break It Exercise

This code won't work. Why?

```java
@Entity
@Table(name = "products")
public record Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id,
    String name,
    double price
) {}
```

<details>
<summary>Answer</summary>

Three problems:

1. **JPA requires a no-arg constructor** — Records don't have one (they have a canonical constructor with all fields)
2. **JPA entities must be mutable** — Records are immutable (fields are `final`)
3. **JPA modifies entities in place** (dirty checking, lazy loading, proxy creation) — impossible with records

Fix: Use a regular class for JPA entities.

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private double price;
    
    // No-arg constructor required by JPA
    protected Product() {}
    
    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }
    
    // Getters and setters...
}
```

Then use a record for the DTO:
```java
public record ProductResponse(Long id, String name, double price) {}
```

</details>

---

## 🎯 Practice Exercise

Create records for a blog API:

1. `CreatePostRequest` — with title (required, 5-200 chars), content (required, min 10 chars), tags (list of strings)
2. `PostResponse` — with id, title, content, tags, authorName, publishedAt
3. `CreateCommentRequest` — with content (required, min 1 char), authorName (required)
4. `CommentResponse` — with id, content, authorName, createdAt

Write a mapper method `toResponse(Post post)` that converts a `Post` entity to a `PostResponse` record.

<details>
<summary>Solution</summary>

```java
// Request DTOs
public record CreatePostRequest(
    @NotBlank @Size(min = 5, max = 200) String title,
    @NotBlank @Size(min = 10) String content,
    List<String> tags
) {}

public record PostResponse(
    Long id,
    String title,
    String content,
    List<String> tags,
    String authorName,
    LocalDateTime publishedAt
) {}

public record CreateCommentRequest(
    @NotBlank @Size(min = 1) String content,
    @NotBlank String authorName
) {}

public record CommentResponse(
    Long id,
    String content,
    String authorName,
    LocalDateTime createdAt
) {}

// Mapper (in service or mapper class)
private PostResponse toResponse(Post post) {
    return new PostResponse(
        post.getId(),
        post.getTitle(),
        post.getContent(),
        post.getTags(),
        post.getAuthor().getName(),
        post.getPublishedAt()
    );
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you create a record with validation annotations?
- [ ] Can you explain why JPA entities cannot be records?
- [ ] Can you write a mapper from entity to record?
- [ ] Did you complete the blog API exercise?
- [ ] Can you explain when to use records vs regular classes?

---

*Next: [01-08 — Annotations](./08-annotations.md)*
