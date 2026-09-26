# Lesson 01-05 — Lambdas and Streams

## 🎯 Learning Objective

Understand Java lambdas and the Stream API — you'll use these constantly when working with collections, JPA results, DTOs, and functional patterns in Spring Boot.

---

## 🤔 Think Before Reading

JavaScript has always been functional-friendly:

```javascript
const activeUsers = users
  .filter(u => u.isActive)
  .map(u => ({ id: u.id, name: u.name }))
  .sort((a, b) => a.name.localeCompare(b.name));
```

**Question**: Before lambdas existed in Java (pre-2014), how would you implement `filter` on a list? Think about what you'd have to write manually.

---

## 📜 Before Lambdas — Anonymous Classes (Java 7 and earlier)

```java
// Sorting a list of users by name — pre-Java 8
List<User> users = getUserList();

Collections.sort(users, new Comparator<User>() {
    @Override
    public int compare(User a, User b) {
        return a.getName().compareTo(b.getName());
    }
});
```

That's 5 lines to say "sort by name." Painful.

---

## ⚡ Lambda Expressions

A lambda is a shorthand for a single-method (functional) interface:

```java
// Before:
Comparator<User> comparator = new Comparator<User>() {
    @Override
    public int compare(User a, User b) {
        return a.getName().compareTo(b.getName());
    }
};

// After (lambda):
Comparator<User> comparator = (a, b) -> a.getName().compareTo(b.getName());

// Using it:
users.sort(comparator);
// or inline:
users.sort((a, b) -> a.getName().compareTo(b.getName()));
```

Lambda syntax:
```
(parameters) -> expression
(parameters) -> { statements; return result; }
```

---

## 🔍 Functional Interfaces

A functional interface has **exactly one abstract method**. Lambdas implement them.

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);  // Single abstract method
}

@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);  // Single abstract method
}

@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);  // Single abstract method
}

@FunctionalInterface
public interface Supplier<T> {
    T get();  // Single abstract method
}
```

Built-in functional interfaces in `java.util.function`:

| Interface | Method | Use |
|-----------|--------|-----|
| `Predicate<T>` | `boolean test(T)` | Filter conditions |
| `Function<T,R>` | `R apply(T)` | Transform T → R |
| `Consumer<T>` | `void accept(T)` | Side effects |
| `Supplier<T>` | `T get()` | Factory / lazy values |
| `BiFunction<T,U,R>` | `R apply(T,U)` | Two-arg transform |

---

## 🔄 JavaScript vs Java Lambdas

```javascript
// JavaScript
const isActive = user => user.isActive;
const toDto = user => ({ id: user.id, name: user.name });
const printUser = user => console.log(user.name);
const createDefault = () => new User("default@example.com");
```

```java
// Java equivalents
Predicate<User> isActive = user -> user.isActive();
Function<User, UserDTO> toDto = user -> new UserDTO(user.getId(), user.getName());
Consumer<User> printUser = user -> System.out.println(user.getName());
Supplier<User> createDefault = () -> new User("default@example.com");
```

---

## 🌊 Java Streams

Streams let you process collections with a pipeline of operations.

```java
import java.util.stream.Collectors;

List<User> users = getUsers();

List<String> activeUserNames = users.stream()           // Create stream
    .filter(u -> u.isActive())                          // Filter (like .filter())
    .sorted((a, b) -> a.getName().compareTo(b.getName())) // Sort
    .map(User::getName)                                 // Transform (like .map())
    .collect(Collectors.toList());                      // Collect result
```

Compare with JavaScript:
```javascript
const activeUserNames = users
  .filter(u => u.isActive)
  .sort((a, b) => a.name.localeCompare(b.name))
  .map(u => u.name);
```

Almost identical mental model. Different syntax.

---

## 💻 Stream Operations Reference

### Terminal Operations (produce a result, end the stream)

```java
List<User> users = getUsers();

// collect — gather into a collection
List<User> activeUsers = users.stream()
    .filter(User::isActive)
    .collect(Collectors.toList());

// count — count elements
long activeCount = users.stream()
    .filter(User::isActive)
    .count();

// findFirst — first matching element
Optional<User> firstAdmin = users.stream()
    .filter(u -> u.getRole().equals("ADMIN"))
    .findFirst();

// anyMatch / allMatch / noneMatch
boolean hasAdmin = users.stream()
    .anyMatch(u -> u.getRole().equals("ADMIN"));

// reduce — aggregate to single value
int totalAge = users.stream()
    .mapToInt(User::getAge)
    .sum();

// forEach — side effects
users.stream()
    .forEach(u -> System.out.println(u.getName()));
```

### Intermediate Operations (return a new stream)

```java
// filter — keep matching elements
.filter(u -> u.getAge() >= 18)

// map — transform each element
.map(u -> new UserDTO(u.getId(), u.getName()))

// flatMap — flatten nested collections
.flatMap(u -> u.getOrders().stream())

// sorted — sort elements
.sorted(Comparator.comparing(User::getName))

// distinct — remove duplicates
.distinct()

// limit — take first N
.limit(10)

// skip — skip first N
.skip(5)
```

---

## 🔍 Method References

Instead of writing the lambda explicitly, you can reference a method:

```java
// Lambda
.map(user -> user.getName())
// Method reference — same thing
.map(User::getName)

// Lambda
.filter(user -> user.isActive())
// Method reference
.filter(User::isActive)

// Lambda  
.forEach(user -> System.out.println(user))
// Method reference
.forEach(System.out::println)

// Lambda (calling instance method on a specific object)
.filter(emailValidator::validate)
// Same as:
.filter(email -> emailValidator.validate(email))
```

---

## 🏗️ Collectors — Grouping Results

```java
// Group users by role
Map<String, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole));

// Count by role
Map<String, Long> countByRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole, Collectors.counting()));

// Join strings
String names = users.stream()
    .map(User::getName)
    .collect(Collectors.joining(", "));
// → "Alice, Bob, Charlie"

// Convert to map
Map<Long, User> userById = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));
```

---

## 🔍 Streams in Spring Boot — Real Usage

### Convert entities to DTOs

```java
@Service
public class UserService {
    
    public List<UserResponse> findAll() {
        return userRepository.findAll()      // List<User> from DB
            .stream()
            .map(this::mapToResponse)        // User → UserResponse
            .collect(Collectors.toList());
    }
    
    private UserResponse mapToResponse(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
    }
}
```

### Filter and transform results

```java
public List<TaskResponse> findActiveTasksByUser(Long userId) {
    return taskRepository.findByUserId(userId)
        .stream()
        .filter(task -> !task.isCompleted())
        .sorted(Comparator.comparing(Task::getDueDate))
        .map(task -> new TaskResponse(task.getId(), task.getTitle(), task.getDueDate()))
        .collect(Collectors.toList());
}
```

### Exception in orElseThrow

```java
// Supplier<T> — zero arg function
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserNotFoundException(id));
//                ↑ This is a Supplier<RuntimeException>
```

---

## 💥 Break It Exercise

What's wrong here?

```java
public List<UserResponse> getActiveAdmins() {
    List<User> users = userRepository.findAll();
    
    users.stream()
        .filter(u -> u.isActive() && u.getRole().equals("ADMIN"))
        .map(u -> new UserResponse(u.getId(), u.getName(), u.getEmail()));
    
    return users.stream()
        .map(u -> new UserResponse(u.getId(), u.getName(), u.getEmail()))
        .collect(Collectors.toList());
}
```

**Question**: What's the bug? What does this actually return?

<details>
<summary>Answer</summary>

The first stream pipeline is **discarded**. Streams are lazy — no work is done until a terminal operation is called. There's no terminal operation (like `collect`) on the first pipeline, so all that filtering is lost.

The method returns ALL users (unfiltered), not just active admins.

Fix:
```java
public List<UserResponse> getActiveAdmins() {
    return userRepository.findAll()
        .stream()
        .filter(u -> u.isActive() && u.getRole().equals("ADMIN"))
        .map(u -> new UserResponse(u.getId(), u.getName(), u.getEmail()))
        .collect(Collectors.toList());
}
```

One pipeline, one collect.

</details>

---

## 🎯 Practice Exercise

Given this data:

```java
public record Order(Long id, Long userId, double amount, String status) {}

List<Order> orders = List.of(
    new Order(1L, 1L, 150.0, "COMPLETED"),
    new Order(2L, 2L, 50.0, "PENDING"),
    new Order(3L, 1L, 200.0, "COMPLETED"),
    new Order(4L, 3L, 75.0, "CANCELLED"),
    new Order(5L, 2L, 300.0, "COMPLETED")
);
```

Write Stream pipelines to:

1. Get total revenue from COMPLETED orders
2. Group orders by userId
3. Get top 2 orders by amount (descending)
4. Check if any order exceeds $250

<details>
<summary>Solution</summary>

```java
// 1. Total revenue from COMPLETED orders
double revenue = orders.stream()
    .filter(o -> o.status().equals("COMPLETED"))
    .mapToDouble(Order::amount)
    .sum();
System.out.println("Revenue: " + revenue); // 650.0

// 2. Group by userId
Map<Long, List<Order>> byUser = orders.stream()
    .collect(Collectors.groupingBy(Order::userId));

// 3. Top 2 orders by amount
List<Order> top2 = orders.stream()
    .sorted(Comparator.comparingDouble(Order::amount).reversed())
    .limit(2)
    .collect(Collectors.toList());

// 4. Any order > $250
boolean hasHighValue = orders.stream()
    .anyMatch(o -> o.amount() > 250);
System.out.println("Has high value: " + hasHighValue); // true
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you write a lambda for `Predicate`, `Function`, `Consumer`, `Supplier`?
- [ ] Can you chain `filter`, `map`, `collect` correctly?
- [ ] Do you understand why you need a terminal operation?
- [ ] Can you use method references (`::`) instead of verbose lambdas?
- [ ] Did you solve the Practice Exercise?

---

*Next: [01-06 — Optional](./06-optional.md)*
