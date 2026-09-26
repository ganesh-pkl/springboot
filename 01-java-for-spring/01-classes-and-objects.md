# Lesson 01-01 — Classes and Objects in Java

## 🎯 Learning Objective

Understand how Java classes and objects work, how they differ from JavaScript objects, and why these differences matter when reading Spring Boot code.

---

## 🤔 Think Before Reading

You've written this in JavaScript:

```javascript
const user = {
  name: "Alice",
  email: "alice@example.com",
  greet() {
    return `Hello, ${this.name}`;
  }
};
```

**Question**: In Java, how would you create a `user` with the same structure? Before reading on — think about what might be different.

---

## 🔄 Node.js vs Java Objects

In JavaScript, objects are dynamic dictionaries. You can add properties at any time:

```javascript
const user = {};
user.name = "Alice";    // Fine in JS
user.role = "admin";    // Fine in JS, added dynamically
```

In Java, objects are **instances of classes**, and the structure is fixed at compile time:

```java
// You CANNOT add properties to a Java object at runtime
// Every property must be declared in the class
```

This is not a limitation — it's a design choice that gives you:

- Compile-time error detection
- Better tooling (IntelliJ knows everything about your class)
- Better performance (JVM can optimize known structures)

---

## 💻 Your First Java Class

```java
public class User {
    // Fields (properties)
    private String name;
    private String email;
    private int age;
  
    // Constructor (like a factory function in JS)
    public User(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
  
    // Methods
    public String greet() {
        return "Hello, " + this.name;
    }
  
    // Getters (Spring needs these to serialize/deserialize)
    public String getName() {
        return name;
    }
  
    public String getEmail() {
        return email;
    }
  
    public int getAge() {
        return age;
    }
}
```

Creating an instance:

```java
User user = new User("Alice", "alice@example.com", 30);
System.out.println(user.greet()); // Hello, Alice
```

---

# 🔍 Access Modifiers — Why They Matter in Spring

| Modifier      | Accessible from            |
| ------------- | -------------------------- |
| `public`    | Anywhere                   |
| `private`   | Only within the same class |
| `protected` | Same class + subclasses    |
| (default)     | Same package only          |

In Spring Boot:

- **Fields** are almost always `private` (encapsulation)
- **Methods** that Spring needs to call (e.g., `@Bean` methods) must be `public`
- **Service methods** called by controllers must be `public`

```java
@Service
public class UserService {
    private final UserRepository userRepository; // private — Spring injects via constructor
  
    public UserService(UserRepository userRepository) { // public — Spring calls this
        this.userRepository = userRepository;
    }
  
    public User findById(Long id) { // public — Controller calls this
        // ...
    }
  
    private void validateUser(User user) { // private — internal helper
        // ...
    }
}
```

---

## 🧠 The `this` Keyword

`this` in Java refers to the current instance — just like JavaScript.

```java
public class User {
    private String name;
  
    public User(String name) {
        this.name = name; // "this.name" = field, "name" = parameter
    }
}
```

One critical difference from JavaScript: in Java, `this` always refers to the current instance — it cannot be reassigned or lost like in JavaScript callbacks.

---

## 🏗️ Constructors

Java constructors are called with `new`:

```java
// No-arg constructor
public User() {
    this.name = "Unknown";
}

// All-args constructor
public User(String name, String email) {
    this.name = name;
    this.email = email;
}
```

### Why constructors matter deeply in Spring Boot

Spring Boot's recommended way to inject dependencies is **constructor injection**:

```java
@Service
public class OrderService {
    private final UserService userService;
    private final ProductService productService;
  
    // Spring calls this constructor and provides the dependencies
    public OrderService(UserService userService, ProductService productService) {
        this.userService = userService;
        this.productService = productService;
    }
}
```

This pattern ensures:

1. Dependencies are **required** (can't create OrderService without them)
2. Dependencies are **immutable** (declared `final`)
3. It's **testable** (you can pass mocks in tests)

---

## 🔄 Static vs Instance Members

```java
public class MathUtils {
    // Static — belongs to the CLASS, not an instance
    public static int add(int a, int b) {
        return a + b;
    }
  
    // Instance — belongs to an INSTANCE
    public int multiply(int a, int b) {
        return a * b;
    }
}

// Usage
MathUtils.add(2, 3);          // Called on class — no instance needed
new MathUtils().multiply(2, 3); // Called on instance
```

**Spring Boot Rule**: Spring manages instances (beans). Static methods cannot be overridden and cannot be proxied by Spring. Avoid using `static` for business logic.

---

## 💥 Break It Exercise

This code has a problem:

```java
@Service
public class UserService {
    private UserRepository userRepository;
  
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
  
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

**Question**: What happens if `findById` is called before `setUserRepository`? What's the error?

<details>
<summary>Answer</summary>

`NullPointerException` — `userRepository` was never set, so it's `null`.

This is why **constructor injection** is preferred over setter injection. With constructor injection, the object cannot exist without its dependencies.

</details>

---

## 🎯 Practice Exercise

Write a `Product` class with:

- `id` (Long)
- `name` (String)
- `price` (double)
- `inStock` (boolean)

Requirements:

1. All fields `private`
2. An all-args constructor
3. A no-arg constructor
4. Getters for all fields
5. A method `isAffordable(double budget)` that returns `true` if price <= budget

Don't look at the solution until you've tried.

<details>
<summary>Solution</summary>

```java
public class Product {
    private Long id;
    private String name;
    private double price;
    private boolean inStock;
  
    // No-arg constructor
    public Product() {}
  
    // All-args constructor
    public Product(Long id, String name, double price, boolean inStock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.inStock = inStock;
    }
  
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public double getPrice() { return price; }
    public boolean isInStock() { return inStock; }
  
    // Business method
    public boolean isAffordable(double budget) {
        return this.price <= budget;
    }
}
```

Notice `isInStock()` not `getInStock()` — Java convention for booleans is `is` prefix.

</details>

---

## ✅ Completion Checklist

Before moving on, confirm you can:

- [ ] Write a Java class with fields, constructor, and methods
- [ ] Explain the difference between `static` and instance members
- [ ] Explain why `private` fields with a public constructor is good practice
- [ ] Explain why constructor injection is preferred in Spring Boot
- [ ] Explain the key difference between Java objects and JavaScript objects

---

*Next: [01-02 — Interfaces and Abstract Classes](./02-interfaces-and-abstract.md)*
