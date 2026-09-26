# Lesson 01-03 — Generics and Collections

## 🎯 Learning Objective

Understand Java generics and the Collections framework — and understand why `List<User>`, `Optional<User>`, `Map<String, Object>` appear everywhere in Spring Boot code.

---

## 🤔 Think Before Reading

In JavaScript, arrays are typeless:

```javascript
const users = [];
users.push("Alice");       // Fine
users.push(42);            // Fine (but probably a bug)
users.push({id: 1});       // Fine
```

In Java, arrays and collections have types. **Question**: Why would a statically-typed language want typed collections? What would you lose if collections could hold anything?

---

## 🔄 Node.js vs Java Collections

| JavaScript | Java | Notes |
|-----------|------|-------|
| `[]` / `Array` | `List<T>` | Ordered, indexed |
| `new Set()` | `Set<T>` | Unique elements |
| `new Map()` / `{}` | `Map<K, V>` | Key-value pairs |
| `array.filter()` | `stream().filter()` | Functional operations |
| `array.map()` | `stream().map()` | Transform elements |
| No equivalent | `Queue<T>`, `Deque<T>` | Queue operations |

---

## 💻 Core Collection Types

### List — Ordered, allows duplicates

```java
import java.util.ArrayList;
import java.util.List;

List<String> names = new ArrayList<>();
names.add("Alice");
names.add("Bob");
names.add("Alice"); // Duplicates allowed

System.out.println(names.get(0)); // Alice
System.out.println(names.size());  // 3
System.out.println(names.contains("Bob")); // true

// Iterate
for (String name : names) {
    System.out.println(name);
}
```

**In Spring Boot**: You'll use `List<User>` when returning multiple users from a repository.

### Set — Unordered, no duplicates

```java
import java.util.HashSet;
import java.util.Set;

Set<String> roles = new HashSet<>();
roles.add("ADMIN");
roles.add("USER");
roles.add("ADMIN"); // Ignored — already exists

System.out.println(roles.size()); // 2
```

**In Spring Boot**: `Set<Role>` for user roles, `Set<Permission>` for authorities.

### Map — Key-value pairs

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);
scores.put("Bob", 87);

System.out.println(scores.get("Alice")); // 95
System.out.println(scores.containsKey("Charlie")); // false

// Iterate
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + " → " + entry.getValue());
}
```

**In Spring Boot**: Spring's `@RequestParam Map<String, String>` for query params, validation error messages as `Map<String, String>`.

---

## 🔍 Generics — What Are They?

Generics let you write **type-safe, reusable code**.

Without generics (old Java):
```java
List list = new ArrayList();
list.add("Alice");
list.add(42);

String name = (String) list.get(0); // Manual cast — runtime error risk
String broken = (String) list.get(1); // ClassCastException at runtime!
```

With generics:
```java
List<String> list = new ArrayList<>();
list.add("Alice");
list.add(42); // COMPILE ERROR — caught before runtime!

String name = list.get(0); // No cast needed
```

The compiler catches bugs **before** the program runs.

---

## 🏗️ Writing Generic Classes

```java
// A generic container — T is a type parameter
public class ApiResponse<T> {
    private T data;
    private String message;
    private boolean success;
    
    public ApiResponse(T data, String message, boolean success) {
        this.data = data;
        this.message = message;
        this.success = success;
    }
    
    public T getData() { return data; }
    public String getMessage() { return message; }
    public boolean isSuccess() { return success; }
}

// Usage:
ApiResponse<User> userResponse = new ApiResponse<>(user, "User found", true);
ApiResponse<List<User>> listResponse = new ApiResponse<>(users, "Users found", true);
ApiResponse<String> messageResponse = new ApiResponse<>("Hello", "OK", true);
```

**In Spring Boot**: `ResponseEntity<T>`, `Page<T>`, `Optional<T>` — all generics.

---

## 🔍 Generic Methods

```java
public class CollectionUtils {
    // T is resolved at call site
    public static <T> List<T> filter(List<T> items, java.util.function.Predicate<T> predicate) {
        List<T> result = new ArrayList<>();
        for (T item : items) {
            if (predicate.test(item)) {
                result.add(item);
            }
        }
        return result;
    }
}

// Usage:
List<User> admins = CollectionUtils.filter(users, user -> user.getRole().equals("ADMIN"));
List<String> longNames = CollectionUtils.filter(names, name -> name.length() > 5);
```

---

## 💡 `Optional<T>` — The Null Killer

`Optional` is Spring Boot's favorite return type for "this might not exist":

```java
import java.util.Optional;

// findById returns Optional — the user might not exist
Optional<User> userOptional = userRepository.findById(id);

// Check if present
if (userOptional.isPresent()) {
    User user = userOptional.get();
    // use user
}

// Better: orElseThrow
User user = userOptional.orElseThrow(() -> 
    new RuntimeException("User not found with id: " + id)
);

// Better still: map and transform
String userName = userOptional
    .map(User::getName)
    .orElse("Unknown");

// Even better: orElseGet (lazy)
User user = userOptional.orElseGet(() -> createDefaultUser());
```

### Why Optional Instead of Null?

In JavaScript:
```javascript
const user = await userRepository.findById(id);
if (user) { // Check for null/undefined
  return user;
}
```

In Java without Optional:
```java
User user = userRepository.findById(id); // Could be null!
return user.getName(); // NullPointerException if user is null — no compile warning
```

In Java with Optional:
```java
Optional<User> user = userRepository.findById(id);
return user.map(User::getName).orElseThrow(); // Forced to handle the empty case
```

Optional makes "this might be null" **explicit in the type system**.

---

## 🏗️ Collections in Spring Boot — Real Examples

### Controller returning a list

```java
@GetMapping("/users")
public ResponseEntity<List<UserResponse>> getAllUsers() {
    List<UserResponse> users = userService.findAll();
    return ResponseEntity.ok(users);
}
```

### Repository method

```java
// Spring Data generates this SQL automatically:
// SELECT * FROM users WHERE email = ?
Optional<User> findByEmail(String email);

// SELECT * FROM users WHERE status = ?
List<User> findByStatus(String status);
```

### Service using Optional

```java
public UserResponse findById(Long id) {
    return userRepository.findById(id)
        .map(this::mapToResponse)   // Transform User → UserResponse if present
        .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
}
```

---

## 💥 Break It Exercise

This code compiles but has a runtime bug:

```java
public User getMostActiveUser(List<User> users) {
    Optional<User> first = users.stream()
        .filter(u -> u.getLoginCount() > 100)
        .findFirst();
    
    return first.get(); // Get the user
}
```

**Question**: What happens when no user has more than 100 logins?

<details>
<summary>Answer</summary>

`NoSuchElementException` — calling `.get()` on an empty `Optional` throws an exception.

Fix:
```java
return first.orElseThrow(() -> 
    new RuntimeException("No active user found")
);
// or
return first.orElse(null); // Risky — caller might NPE
```

Never call `.get()` on an `Optional` without checking `isPresent()` first, or use `orElseThrow()` instead.

</details>

---

## 🎯 Practice Exercise

Write a method `groupByStatus(List<Task> tasks)` that returns a `Map<String, List<Task>>` where:
- Key = task status ("TODO", "IN_PROGRESS", "DONE")
- Value = list of tasks with that status

```java
public class Task {
    private Long id;
    private String title;
    private String status; // "TODO", "IN_PROGRESS", "DONE"
    // constructors, getters...
}
```

Try writing it with both a `for` loop AND with Java Streams.

<details>
<summary>Solution</summary>

```java
// With for loop
public Map<String, List<Task>> groupByStatus(List<Task> tasks) {
    Map<String, List<Task>> result = new HashMap<>();
    
    for (Task task : tasks) {
        String status = task.getStatus();
        // computeIfAbsent creates the list if it doesn't exist
        result.computeIfAbsent(status, k -> new ArrayList<>()).add(task);
    }
    
    return result;
}

// With Streams (more idiomatic Java)
import java.util.stream.Collectors;

public Map<String, List<Task>> groupByStatusStream(List<Task> tasks) {
    return tasks.stream()
        .collect(Collectors.groupingBy(Task::getStatus));
}
```

The Streams version is used constantly in Spring Boot code. Get comfortable with it.

</details>

---

## ✅ Completion Checklist

- [ ] Can you declare and use `List<T>`, `Set<T>`, `Map<K,V>`?
- [ ] Can you write a generic class like `ApiResponse<T>`?
- [ ] Do you understand what `Optional<T>` prevents?
- [ ] Can you use `orElseThrow()` instead of `.get()`?
- [ ] Did you complete the `groupByStatus` exercise?

---

*Next: [01-04 — Exceptions](./04-exceptions.md)*
