# Lesson 01-04 — Exceptions in Java

## 🎯 Learning Objective

Understand Java's exception system — and specifically why Spring Boot uses **unchecked exceptions** everywhere while the Java language also supports checked exceptions.

---

## 🤔 Think Before Reading

In Node.js, error handling looks like:

```javascript
try {
  const user = await userService.findById(id);
  return user;
} catch (error) {
  if (error.code === 'NOT_FOUND') {
    return res.status(404).json({ error: 'User not found' });
  }
  throw error;
}
```

**Question**: Java has two types of exceptions: checked and unchecked. What might be the difference? Why would you ever want the compiler to FORCE you to handle an exception?

---

## 📋 Exception Hierarchy

```
Throwable
├── Error (JVM-level, don't catch: OutOfMemoryError, StackOverflowError)
└── Exception
    ├── IOException (checked)
    ├── SQLException (checked)
    ├── RuntimeException (unchecked)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   └── (Your custom exceptions)
    └── (Other checked exceptions)
```

---

## 🔍 Checked vs Unchecked Exceptions

### Checked Exceptions

The compiler **forces** you to handle them. If you don't, your code won't compile.

```java
// FileNotFoundException is checked
public void readFile(String path) throws IOException { // Must declare!
    FileReader reader = new FileReader(path); // Throws IOException
}

// Callers must handle it:
try {
    readFile("config.txt");
} catch (IOException e) {
    System.out.println("File not found: " + e.getMessage());
}
```

### Unchecked Exceptions (RuntimeException)

The compiler does NOT force you to handle them.

```java
public void processUser(User user) {
    // NullPointerException is unchecked — no "throws" declaration needed
    String upper = user.getName().toUpperCase(); // NPE if user is null
}
```

---

## 🧠 Why Spring Boot Uses Unchecked Exceptions

Here's the problem with checked exceptions in a web application:

```java
// Repository layer
public User findById(Long id) throws UserNotFoundException { // checked

// Service layer — must handle or declare it
public UserResponse getUser(Long id) throws UserNotFoundException { // propagated

// Controller layer — must handle or declare it
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) 
    throws UserNotFoundException { // propagated again
```

Every layer has to declare the exception. This creates:
1. Boilerplate code everywhere
2. Tight coupling (every layer knows about database exceptions)
3. Harder-to-read code

Spring Boot's pattern: throw unchecked exceptions and catch them centrally:

```java
// Custom unchecked exception
public class UserNotFoundException extends RuntimeException { // extends RuntimeException!
    public UserNotFoundException(Long id) {
        super("User not found with id: " + id);
    }
}

// Repository/Service — just throw it, no "throws" declaration
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// Catch it ONCE, globally
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse(ex.getMessage()));
    }
}
```

This is the pattern you'll use throughout the course.

---

## 💻 Creating Custom Exceptions

```java
// Base custom exception
public class AppException extends RuntimeException {
    private final int statusCode;
    
    public AppException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }
    
    public int getStatusCode() {
        return statusCode;
    }
}

// Specific exceptions
public class UserNotFoundException extends AppException {
    public UserNotFoundException(Long id) {
        super("User not found with id: " + id, 404);
    }
}

public class DuplicateEmailException extends AppException {
    public DuplicateEmailException(String email) {
        super("Email already exists: " + email, 409);
    }
}

public class InsufficientFundsException extends AppException {
    public InsufficientFundsException(double balance, double required) {
        super("Insufficient funds. Balance: " + balance + ", Required: " + required, 400);
    }
}
```

---

## 🔄 Java Exception Flow vs Node.js

**Node.js (Express)**:
```javascript
// Error middleware at the end of the chain
app.use((err, req, res, next) => {
  if (err.status === 404) {
    return res.status(404).json({ error: err.message });
  }
  res.status(500).json({ error: 'Internal Server Error' });
});
```

**Spring Boot**:
```java
// @ControllerAdvice — same concept, cleaner syntax
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }
    
    @ExceptionHandler(Exception.class) // Catch-all
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}
```

The mental model is identical. The syntax is different.

---

## 💻 try/catch/finally in Java

```java
public void processPayment(Long orderId) {
    Connection connection = null;
    
    try {
        connection = database.getConnection();
        // Process payment
        connection.commit();
        
    } catch (InsufficientFundsException e) {
        // Specific business logic exception
        System.out.println("Cannot process: " + e.getMessage());
        if (connection != null) connection.rollback();
        throw e; // Re-throw to let caller handle it
        
    } catch (Exception e) {
        // Catch all other exceptions
        System.out.println("Unexpected error: " + e.getMessage());
        if (connection != null) connection.rollback();
        
    } finally {
        // ALWAYS runs, even if exception is thrown
        if (connection != null) {
            connection.close();
        }
    }
}
```

**In Spring Boot**: `@Transactional` handles commit/rollback automatically. You rarely write manual try/catch/finally for transactions.

---

## 🔍 try-with-resources (Modern Java)

For resources that must be closed:

```java
// Old way (resource might not close if exception occurs)
FileReader reader = new FileReader("file.txt");
try {
    // use reader
} finally {
    reader.close();
}

// Modern way — automatically closes resource
try (FileReader reader = new FileReader("file.txt")) {
    // use reader
    // reader.close() called automatically
}
```

---

## 💥 Break It Exercise

This code has a subtle exception handling bug:

```java
@Service
public class TransferService {
    
    public void transfer(Long fromAccountId, Long toAccountId, double amount) {
        try {
            Account from = accountRepository.findById(fromAccountId)
                .orElseThrow(() -> new AccountNotFoundException(fromAccountId));
                
            Account to = accountRepository.findById(toAccountId)
                .orElseThrow(() -> new AccountNotFoundException(toAccountId));
            
            from.deduct(amount);
            to.add(amount);
            
            accountRepository.save(from);
            accountRepository.save(to);
            
        } catch (Exception e) {
            System.out.println("Transfer failed: " + e.getMessage());
        }
    }
}
```

**Question**: What happens when this fails halfway through (after `from.deduct` but before `to.add`)? What's the bug with the exception handling?

<details>
<summary>Answer</summary>

Two bugs:

1. **Swallowed exception**: The `catch (Exception e)` swallows all exceptions. The caller has no way to know the transfer failed. The method returns normally even on failure.

2. **No transaction**: If `save(from)` succeeds but `save(to)` fails, the money is deducted but never received. This is a data integrity bug.

Fix:
```java
@Transactional // Let Spring handle rollback
public void transfer(Long fromAccountId, Long toAccountId, double amount) {
    Account from = accountRepository.findById(fromAccountId)
        .orElseThrow(() -> new AccountNotFoundException(fromAccountId));
        
    Account to = accountRepository.findById(toAccountId)
        .orElseThrow(() -> new AccountNotFoundException(toAccountId));
    
    if (from.getBalance() < amount) {
        throw new InsufficientFundsException(from.getBalance(), amount);
    }
    
    from.deduct(amount);
    to.add(amount);
    
    accountRepository.save(from);
    accountRepository.save(to);
    // If any exception throws here, @Transactional rolls back BOTH saves
}
```

Never swallow exceptions silently. Always either handle them properly or let them propagate.

</details>

---

## 🎯 Practice Exercise

Create a hierarchy of custom exceptions for a task management API:

```
AppException (base, unchecked)
├── ResourceNotFoundException
│   ├── TaskNotFoundException
│   └── UserNotFoundException  
├── ValidationException
│   └── DuplicateTitleException
└── AuthorizationException
    └── UnauthorizedAccessException
```

Each exception should:
1. Accept a meaningful message
2. Have a `statusCode` field (HTTP status code)
3. Be throwable from service methods without `throws` declarations

<details>
<summary>Solution</summary>

```java
// Base
public class AppException extends RuntimeException {
    private final int statusCode;
    
    public AppException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }
    
    public int getStatusCode() { return statusCode; }
}

// Resource exceptions (404)
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String message) {
        super(message, 404);
    }
}

public class TaskNotFoundException extends ResourceNotFoundException {
    public TaskNotFoundException(Long id) {
        super("Task not found with id: " + id);
    }
}

public class UserNotFoundException extends ResourceNotFoundException {
    public UserNotFoundException(Long id) {
        super("User not found with id: " + id);
    }
    
    public UserNotFoundException(String email) {
        super("User not found with email: " + email);
    }
}

// Validation exceptions (400/409)
public class ValidationException extends AppException {
    public ValidationException(String message) {
        super(message, 400);
    }
}

public class DuplicateTitleException extends AppException {
    public DuplicateTitleException(String title) {
        super("Task with title already exists: " + title, 409);
    }
}

// Authorization exceptions (403)
public class AuthorizationException extends AppException {
    public AuthorizationException(String message) {
        super(message, 403);
    }
}

public class UnauthorizedAccessException extends AuthorizationException {
    public UnauthorizedAccessException(Long resourceId) {
        super("You are not authorized to access resource: " + resourceId);
    }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain the difference between checked and unchecked exceptions?
- [ ] Can you explain why Spring Boot uses unchecked exceptions?
- [ ] Can you write a custom exception hierarchy?
- [ ] Do you understand why swallowing exceptions is dangerous?
- [ ] Can you explain how `@ControllerAdvice` relates to Express error middleware?

---

*Next: [01-05 — Lambdas and Streams](./05-lambdas-and-streams.md)*
