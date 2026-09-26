# Lesson 01-02 — Interfaces and Abstract Classes

## 🎯 Learning Objective

Understand Java interfaces and abstract classes — and more importantly, understand **why Spring Boot is built entirely around interfaces**.

---

## 🤔 Think Before Reading

In Node.js/Express, when you write a service:

```javascript
class UserService {
  constructor(userRepository) {
    this.userRepository = userRepository;
  }
  
  async findById(id) {
    return this.userRepository.findById(id);
  }
}
```

You depend directly on `userRepository`. If you want to swap it with a mock in tests, you just pass a different object — JavaScript is duck-typed.

**Question**: In a statically-typed language like Java, how would you swap implementations without changing all the code that depends on them?

Think about it. Then read on.

---

## 💡 Why Java Needs Interfaces

In Java, types are explicit. If a class depends on `UserRepository`:

```java
public class UserService {
    private UserRepository userRepository; // This is a specific class
}
```

You can **only** pass a `UserRepository` instance — nothing else. You can't pass a mock or a different implementation.

The solution: depend on an **interface** instead of a concrete class.

```java
// Define what a UserRepository CAN do
public interface UserRepository {
    User findById(Long id);
    void save(User user);
    void delete(Long id);
}

// Real implementation
public class JpaUserRepository implements UserRepository {
    @Override
    public User findById(Long id) { /* database query */ }
    
    @Override
    public void save(User user) { /* database save */ }
    
    @Override
    public void delete(Long id) { /* database delete */ }
}

// Test implementation (mock)
public class InMemoryUserRepository implements UserRepository {
    private Map<Long, User> store = new HashMap<>();
    
    @Override
    public User findById(Long id) { return store.get(id); }
    
    @Override
    public void save(User user) { store.put(user.getId(), user); }
    
    @Override
    public void delete(Long id) { store.remove(id); }
}
```

Now `UserService` can depend on the **interface**:

```java
public class UserService {
    private final UserRepository userRepository; // Interface — not a specific class!
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// In production
UserService service = new UserService(new JpaUserRepository());

// In tests
UserService service = new UserService(new InMemoryUserRepository());
```

This is called **programming to an interface** — one of the most important principles in OOP.

---

## 🔄 Node.js vs Java Comparison

| Node.js | Java |
|---------|------|
| Duck typing — any object with the right methods works | Interface — explicit contract |
| No interface needed for swappability | Interface enables swappability in a typed system |
| Mocks created with `jest.fn()` | Mocks implement the interface |
| No compile-time enforcement | Compile-time enforcement |

---

## 💻 Interface Syntax

```java
public interface Printable {
    void print(); // abstract by default — no body
    
    // Java 8+: Default methods (with implementation)
    default void printTwice() {
        print();
        print();
    }
    
    // Java 8+: Static methods
    static Printable noOp() {
        return () -> {}; // lambda!
    }
}
```

Key rules:
- Interface methods are `public` and `abstract` by default
- A class can **implement multiple interfaces**
- Interfaces cannot have instance fields (only `static final` constants)

---

## 🏗️ Abstract Classes

An abstract class is a middle ground: it can have both abstract methods (no body) and concrete methods (with body).

```java
public abstract class Animal {
    private String name;
    
    public Animal(String name) {
        this.name = name;
    }
    
    // Abstract — subclasses MUST implement this
    public abstract String makeSound();
    
    // Concrete — subclasses inherit this
    public void introduce() {
        System.out.println("I am " + name + " and I say: " + makeSound());
    }
}

public class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }
    
    @Override
    public String makeSound() {
        return "Woof!";
    }
}
```

### Interface vs Abstract Class

| | Interface | Abstract Class |
|---|-----------|----------------|
| Methods | All abstract (+ default) | Mix of abstract and concrete |
| Fields | No instance fields | Can have instance fields |
| Multiple inheritance | A class can implement many | A class can extend only ONE |
| Use when | Defining a CONTRACT | Sharing CODE between related classes |

---

## 🔍 How Spring Uses Interfaces (This Is Critical)

Spring Boot is built on interfaces everywhere. Examples:

### 1. Repository Pattern

```java
// Spring Data provides this interface
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring generates ALL the implementation automatically
    // You write the interface. Spring writes the code.
}
```

### 2. Service Interfaces (Best Practice)

```java
public interface UserService {
    UserResponse findById(Long id);
    UserResponse create(CreateUserRequest request);
    void delete(Long id);
}

@Service
public class UserServiceImpl implements UserService {
    @Override
    public UserResponse findById(Long id) { /* implementation */ }
    // ...
}
```

Why? Because Spring creates **proxies** around your beans. A proxy is an object that:
1. Looks like your class (implements the same interface)
2. Wraps your class with extra behavior (logging, transactions, security, caching)

```
Client code → [Spring Proxy] → Your actual UserServiceImpl
                   ↓
            Adds @Transactional
            Adds @Cacheable
            Adds @Secured
```

Spring can only create interface-based proxies if your class implements an interface. (It can also use CGLIB for class-based proxies, but interface proxies are cleaner.)

### 3. ApplicationContext and BeanFactory

```java
// Spring's container is accessed via an interface
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
```

---

## 🧠 Mental Model: Interface as Contract

Think of an interface as a **legal contract**:

```
Interface UserRepository says:
"I promise that any class implementing me will provide:
- findById(Long id) → User
- save(User user) → void
- delete(Long id) → void"

JpaUserRepository signs that contract.
InMemoryUserRepository signs that contract.

UserService doesn't care WHO implements it.
UserService only cares that the contract is fulfilled.
```

---

## 💥 Break It Exercise

What's wrong with this code?

```java
public interface TaskService {
    TaskResponse createTask(CreateTaskRequest request);
    TaskResponse getTask(Long id);
}

@Service
public class TaskServiceImpl implements TaskService {
    
    @Override
    public TaskResponse createTask(CreateTaskRequest request) {
        // implementation
        return new TaskResponse();
    }
    
    // Missing: getTask implementation
}
```

<details>
<summary>Answer</summary>

**Compile error**: `TaskServiceImpl` does not implement all methods from `TaskService`. 

The method `getTask(Long id)` is declared in the interface but not implemented in the class. Java will refuse to compile this.

This is the benefit of interfaces: the compiler enforces the contract.

</details>

---

## 🎯 Practice Exercise

Create:

1. An interface `NotificationService` with methods:
   - `void sendEmail(String to, String subject, String body)`
   - `void sendSms(String phoneNumber, String message)`

2. Two implementations:
   - `EmailNotificationService` — just prints "Sending email to [to]: [subject]"
   - `SmsNotificationService` — just prints "Sending SMS to [phoneNumber]: [message]"

3. A `UserRegistrationService` that depends on `NotificationService` (the interface) and calls `sendEmail` after registration.

4. Show that you can swap implementations without changing `UserRegistrationService`.

<details>
<summary>Solution</summary>

```java
// Interface
public interface NotificationService {
    void sendEmail(String to, String subject, String body);
    void sendSms(String phoneNumber, String message);
}

// Implementation 1
public class EmailNotificationService implements NotificationService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending email to " + to + ": " + subject);
    }
    
    @Override
    public void sendSms(String phoneNumber, String message) {
        System.out.println("SMS via email gateway to " + phoneNumber + ": " + message);
    }
}

// Implementation 2
public class SmsNotificationService implements NotificationService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Email via SMS gateway to " + to + ": " + subject);
    }
    
    @Override
    public void sendSms(String phoneNumber, String message) {
        System.out.println("Sending SMS to " + phoneNumber + ": " + message);
    }
}

// Registration service — depends on the interface
public class UserRegistrationService {
    private final NotificationService notificationService;
    
    public UserRegistrationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
    
    public void register(String email, String phone) {
        System.out.println("User registered: " + email);
        // Calls interface method — doesn't know which implementation
        notificationService.sendEmail(email, "Welcome!", "Thanks for registering.");
    }
}

// Main — swap implementations without changing UserRegistrationService
public class Main {
    public static void main(String[] args) {
        // With email service
        UserRegistrationService service1 = 
            new UserRegistrationService(new EmailNotificationService());
        service1.register("alice@example.com", "+1234567890");
        
        // With SMS service — same service class, different behavior
        UserRegistrationService service2 = 
            new UserRegistrationService(new SmsNotificationService());
        service2.register("bob@example.com", "+0987654321");
    }
}
```

</details>

---

## 🎤 Quick Interview Questions

1. **What is the difference between an interface and an abstract class in Java?**
   > Interface defines a pure contract (no state). Abstract class can have state and partial implementation. Use interface for contracts; abstract class for shared code between related classes.

2. **Why does Spring prefer interface-based injection?**
   > Spring creates proxies around beans. Interface-based proxies are cleaner (JDK dynamic proxies). Without interfaces, Spring must use CGLIB to subclass your bean, which has limitations.

3. **Can a Java class implement multiple interfaces?**
   > Yes. This is how Java achieves multiple "type" inheritance without the diamond problem.

---

## ✅ Completion Checklist

- [ ] Can you explain what an interface is and when to use it?
- [ ] Can you explain the difference between interface and abstract class?
- [ ] Can you explain why Spring uses interfaces for proxies?
- [ ] Did you complete the NotificationService exercise?
- [ ] Can you answer the 3 interview questions above without notes?

---

*Next: [01-03 — Generics and Collections](./03-generics-and-collections.md)*
