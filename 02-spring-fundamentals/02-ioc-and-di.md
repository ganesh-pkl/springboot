# Lesson 02-02 — IoC and Dependency Injection

## 🎯 Learning Objective

Understand Inversion of Control and Dependency Injection at a mechanical level — not just what they are, but **exactly how Spring implements them**.

---

## 🤔 Think Before Reading

Here are three ways to give `UserService` access to `UserRepository`:

**Option A — Service creates its own dependency:**
```java
public class UserService {
    private UserRepository userRepository = new JpaUserRepository(); // creates it
}
```

**Option B — Dependency is passed in:**
```java
public class UserService {
    private UserRepository userRepository;
    
    public UserService(UserRepository userRepository) { // receives it
        this.userRepository = userRepository;
    }
}
```

**Option C — Spring provides it:**
```java
@Service
public class UserService {
    private final UserRepository userRepository; // Spring provides it
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**Question**: What are the concrete differences between A, B, and C? When would you prefer each?

---

## 🔍 Deep Dive: Option A — The Dependency Creates Itself

```java
public class UserService {
    // Creates its own dependency
    private UserRepository userRepository = new JpaUserRepository();
}
```

**Problems**:
1. `UserService` is tightly coupled to `JpaUserRepository` — can't swap to `MongoUserRepository` without changing this class
2. Can't mock `userRepository` in unit tests — you'd always hit the real database
3. Can't control the `JpaUserRepository` initialization (connection pool, etc.)
4. If 10 services do this, you might create 10 separate repository instances instead of sharing one

---

## 🔍 Option B — Constructor Injection (DI without Spring)

```java
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// Caller provides the dependency
UserService service = new UserService(new JpaUserRepository(dataSource));

// Or in tests:
UserService service = new UserService(new InMemoryUserRepository());
```

This IS dependency injection. Spring didn't invent this — it automated it.

**Benefits**:
- Decoupled from specific implementation
- Testable (pass mock in tests)
- Explicit dependencies (you can see what a class needs just by reading its constructor)

**Remaining problem**: Someone still has to do all the wiring. That's what Spring's container handles.

---

## 🧠 Inversion of Control — Explained Precisely

"Normal" control:

```
YOUR CODE calls the framework/library

Example:
  myCode → new FileReader("file.txt")  // You control when FileReader is created
  myCode → httpClient.get(url)         // You control when request is made
```

"Inverted" control:

```
THE FRAMEWORK calls YOUR CODE

Example:
  Spring → calls UserController.getUser() when HTTP request arrives
  Spring → calls UserService(userRepo) when it needs a UserService
  Spring → calls @PostConstruct method after bean is initialized
```

**The control of "when things happen" is inverted** — you declare what should happen, the framework decides when and how.

This is why Spring calls the pattern "IoC" — **you** stop calling `new`, **Spring** calls `new` on your behalf.

---

## 💻 Three Forms of Dependency Injection

### 1. Constructor Injection (RECOMMENDED)

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    // Spring detects: one constructor = inject all parameters
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

**Why it's recommended**:
- Dependencies are required (can't create object without them)
- Dependencies are immutable (`final`)
- Easy to test (just call `new UserService(mockRepo, mockEmail)`)
- Makes circular dependencies fail fast (Spring detects at startup)

### 2. Field Injection (AVOID in production code)

```java
@Service
public class UserService {
    
    @Autowired  // Spring injects directly into field via reflection
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
}
```

**Why it's problematic**:
- Can't write `new UserService()` in tests — fields are `null` without Spring
- Dependencies are not visible from outside (hidden coupling)
- Makes circular dependencies harder to catch
- Relies on Spring's reflection to set private fields

### 3. Setter Injection (for optional dependencies only)

```java
@Service
public class UserService {
    private UserRepository userRepository;
    private AuditService auditService; // Optional — might not exist
    
    @Autowired // Required dependency
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Autowired(required = false) // Optional dependency
    public void setAuditService(AuditService auditService) {
        this.auditService = auditService;
    }
}
```

---

## 🔍 How Spring Implements Constructor Injection

When Spring starts and encounters:

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Spring does this (simplified):

```
1. Spring sees @Service → register UserService as a bean definition

2. Spring inspects the constructor: 
   "UserService needs a UserRepository"

3. Spring looks in its container for a bean of type UserRepository

4. Spring finds JpaUserRepository (which is @Repository, a subtype)

5. Spring calls: new UserService(jpaUserRepositoryInstance)
   (using Java reflection: constructor.newInstance(args))

6. Spring stores the created UserService instance in its container
```

The actual Spring code doing this is in `ConstructorResolver` and `AutowiredAnnotationBeanPostProcessor`.

---

## 📊 IoC Container — The Big Picture

```
Your Application Code
    ↓ (annotated with @Component, @Service, etc.)
ApplicationContext (IoC Container)
    ├── Scans your code (ClassPathBeanDefinitionScanner)
    ├── Reads bean definitions (what beans exist, what they need)
    ├── Resolves dependency order (who needs whom)
    ├── Creates beans in dependency order
    ├── Injects dependencies via constructors
    ├── Runs BeanPostProcessors (add proxy for @Transactional, etc.)
    └── Stores all beans in a Map<String, Object>
```

The container is essentially a `HashMap`:
- Key: bean name (usually the class name with lowercase first letter)
- Value: the actual singleton bean instance

```java
// Internally, Spring's container is roughly:
Map<String, Object> beans = new HashMap<>();
beans.put("userService", new UserService(beans.get("userRepository")));
beans.put("orderService", new OrderService(beans.get("userRepository"), ...));
// etc.
```

---

## 🔄 Node.js vs Spring DI

In **Node.js**, the module system gives you a limited form of DI:

```javascript
// user-repository.js — creates one instance
const db = require('./db');
const UserRepository = class {
    findById(id) { return db.query(`SELECT * FROM users WHERE id = $1`, [id]); }
};

module.exports = new UserRepository(db); // Singleton via module cache
```

```javascript
// user-service.js — receives the repository
const userRepository = require('./user-repository');

class UserService {
    async findById(id) {
        return userRepository.findById(id);
    }
}

module.exports = new UserService();
```

The Node.js module system acts as a primitive IoC container. But:
- No lifecycle management
- No interface swapping
- No proxy wrapping for cross-cutting concerns
- No profiles or conditional beans

Spring's container is far more capable.

---

## 🏗️ Real Spring Example — Tracing DI

Let's trace a complete DI chain:

```java
@Repository
public class JpaUserRepository implements UserRepository {
    // No dependencies
}

@Service
public class EmailService {
    // No dependencies
}

@Service
public class AuditService {
    // No dependencies
}

@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}

@RestController
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

**Spring's creation order**:

```
Step 1: Create JpaUserRepository() — no dependencies
Step 2: Create EmailService() — no dependencies
Step 3: Create AuditService() — no dependencies
Step 4: Create UserService(jpaUserRepository, emailService) — depends on steps 1 & 2
Step 5: Create UserController(userService) — depends on step 4
```

Spring figures this out automatically using dependency graph analysis (topological sort).

---

## 💥 Break It Exercise

This code will cause an error at application startup:

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;
    
    public ServiceA(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;
    
    public ServiceA(ServiceA serviceA) {
        this.serviceA = serviceA;
    }
}
```

**Question**: What error will Spring throw? Why? How do you fix it?

<details>
<summary>Answer</summary>

**Error**: `BeanCurrentlyInCreationException` — circular dependency detected.

**Why**: 
- To create `ServiceA`, Spring needs `ServiceB`
- To create `ServiceB`, Spring needs `ServiceA`
- Infinite loop — Spring detects this and fails at startup

**How to fix**: Redesign your architecture. One of these services shouldn't depend on the other. A common fix is to extract shared logic into a third service that both depend on.

```java
// Instead of A depends on B depends on A:
@Service
public class SharedService { /* shared logic */ }

@Service  
public class ServiceA {
    private final SharedService sharedService;
    public ServiceA(SharedService sharedService) { ... }
}

@Service
public class ServiceB {
    private final SharedService sharedService;
    public ServiceB(SharedService sharedService) { ... }
}
```

> Note: With field injection (`@Autowired` on fields), Spring can sometimes work around circular dependencies by using a "partially created" proxy. This is another reason field injection is considered bad practice — it can hide architectural problems.

</details>

---

## 🎤 Interview Questions

**Easy**:
1. What is Inversion of Control?
   > The framework controls object creation and lifecycle, not the application code.

2. What are the three types of Dependency Injection?
   > Constructor, field, and setter injection. Constructor is recommended.

**Intermediate**:
3. Why is constructor injection preferred over field injection?
   > Constructor injection makes dependencies explicit, mandatory, and immutable. Field injection hides dependencies and makes testing without Spring harder.

4. How does Spring resolve which implementation to inject when an interface has multiple implementations?
   > Spring throws `NoUniqueBeanDefinitionException`. You resolve it with `@Primary` (one implementation is default) or `@Qualifier("beanName")` (specify which one to use).

**Advanced**:
5. How does Spring detect circular dependencies?
   > Spring tracks beans "currently being created" in a set. If a bean's dependency is already in that set, it's a circular dependency. With constructor injection, this fails immediately. With field injection, Spring may try to break the cycle with early references.

---

## ✅ Completion Checklist

- [ ] Can you explain IoC in one sentence without jargon?
- [ ] Can you explain the difference between the 3 forms of DI?
- [ ] Can you explain WHY constructor injection is preferred?
- [ ] Can you trace Spring's object creation order for a dependency chain?
- [ ] Can you explain what a circular dependency is and why it happens?

---

*Next: [02-03 — ApplicationContext](./03-application-context.md)*
