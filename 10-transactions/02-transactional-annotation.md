# Lesson 10-02 — @Transactional — The Complete Guide

## 🎯 Learning Objective

Understand exactly how `@Transactional` works mechanically — the proxy, the transaction boundary, and the subtle rules that cause silent failures when misunderstood.

---

## 🤔 Think Before Reading

If Spring wraps your service in a proxy for `@Transactional`, and you call one `@Transactional` method from ANOTHER `@Transactional` method in the same class:

```java
@Service
public class PaymentService {
    
    @Transactional
    public void processPayment(Payment payment) {
        deductBalance(payment.getUserId(), payment.getAmount()); // @Transactional too
        recordTransaction(payment); // @Transactional too
    }
    
    @Transactional
    public void deductBalance(Long userId, double amount) { ... }
    
    @Transactional
    public void recordTransaction(Payment payment) { ... }
}
```

**Question**: Are `deductBalance` and `recordTransaction` in SEPARATE transactions or the SAME transaction? Does it matter?

---

## 🔍 How @Transactional Works — The Proxy

When Spring sees `@Transactional`, it creates a **proxy** around your service:

```
What your code sees:          What actually exists:
PaymentService        →       PaymentServiceProxy (extends PaymentService)
                              └── actual PaymentService instance (target)
```

When a method is called on the proxy:
```
Caller → PaymentServiceProxy.processPayment()
              ↓
         Check: is there already an active transaction?
              ↓
         No → BEGIN TRANSACTION
              ↓
         Delegate to real PaymentService.processPayment()
              ↓
         Return normally? → COMMIT
         Exception thrown? → ROLLBACK
```

---

## 🔍 Transaction Propagation

Propagation answers: "What should happen to the transaction when this method is called?"

### REQUIRED (default)

"Reuse an existing transaction if there is one. Otherwise, create a new one."

```java
@Transactional(propagation = Propagation.REQUIRED) // Default
public void processPayment(Payment payment) {
    deductBalance(payment.getUserId(), payment.getAmount());
    // ^ Same transaction — if this fails, processPayment's transaction rolls back too
}
```

**This is why inner method calls within the same transaction share it.**

### REQUIRES_NEW

"Always create a new transaction. Suspend the current one if there is one."

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordAuditLog(String action) {
    // Always in its OWN transaction
    // If the outer transaction rolls back, this one is already committed
    auditRepository.save(new AuditLog(action, LocalDateTime.now()));
}
```

Use `REQUIRES_NEW` for:
- Audit logging (should persist even if the main operation fails)
- Sending notifications (should be independent)

### SUPPORTS

"Join the transaction if there is one. Run without a transaction if there isn't."

### NOT_SUPPORTED

"Suspend the transaction and run without one."

### NEVER

"Throw an exception if called within an active transaction."

### MANDATORY

"Throw an exception if there's NO active transaction."

---

## 🐛 The Self-Invocation Problem — Most Common @Transactional Bug

```java
@Service
public class PaymentService {
    
    @Transactional
    public void processPayment(Payment payment) {
        deductBalance(payment.getUserId(), payment.getAmount());
    }
    
    @Transactional
    public void deductBalance(Long userId, double amount) { ... }
}

// Caller:
paymentService.processPayment(payment); // Goes through proxy ✅

// Inside PaymentService:
public void processPayment(Payment payment) {
    this.deductBalance(...); // ← DIRECT CALL, bypasses proxy! ❌
    // @Transactional on deductBalance is IGNORED
}
```

When you call a method using `this`, you're calling the real object, not the proxy. Spring's proxy never intercepts `this.method()` calls.

**Impact**:
- `@Transactional` on `deductBalance` has NO EFFECT when called from within the same class
- No new transaction is created
- `REQUIRES_NEW` propagation is silently ignored

### How to Fix Self-Invocation

**Option 1**: Extract to another class (cleanest)

```java
@Service
public class PaymentService {
    private final BalanceService balanceService; // Separate class = different proxy
    
    @Transactional
    public void processPayment(Payment payment) {
        balanceService.deductBalance(payment.getUserId(), payment.getAmount()); // Goes through proxy ✅
    }
}

@Service
public class BalanceService {
    @Transactional
    public void deductBalance(Long userId, double amount) { ... }
}
```

**Option 2**: Inject `ApplicationContext` and get self (messy, avoid)

```java
@Service
public class PaymentService implements ApplicationContextAware {
    private ApplicationContext context;
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }
    
    public void processPayment(Payment payment) {
        PaymentService self = context.getBean(PaymentService.class); // Gets the proxy
        self.deductBalance(...); // Goes through proxy
    }
}
```

**Option 3**: `@EnableAspectJAutoProxy(exposeProxy = true)` and `AopContext.currentProxy()` — don't.

---

## 🔍 Rollback Rules

By default, `@Transactional` ONLY rolls back on **unchecked exceptions** (RuntimeException and its subclasses).

```java
@Transactional
public void createUser(CreateUserRequest request) throws IOException {
    // Unchecked exception → ROLLBACK
    throw new IllegalStateException("Something went wrong"); // Rolls back ✅
    
    // Checked exception → COMMITS! (surprising)
    throw new IOException("File not found"); // Does NOT roll back ❌
}
```

To rollback on checked exceptions:
```java
@Transactional(rollbackFor = Exception.class) // Rollback on ANY exception
public void createUser(CreateUserRequest request) throws IOException { ... }

@Transactional(rollbackFor = {IOException.class, SQLException.class}) // Specific
public void processFile() throws IOException { ... }
```

To NOT rollback on a specific exception:
```java
@Transactional(noRollbackFor = BusinessWarningException.class)
public void processOrder(Order order) {
    // BusinessWarningException → commits anyway (it's just a warning)
}
```

---

## 🔍 Read-Only Transactions

```java
@Transactional(readOnly = true) // Performance optimization
public List<UserResponse> findAll() {
    return userRepository.findAll()
        .stream()
        .map(this::mapToResponse)
        .toList();
}
```

`readOnly = true`:
1. Tells Hibernate to skip dirty checking (no need to track changes)
2. Hints the database to use read replicas if configured
3. Slight performance improvement for read-heavy operations

**Note**: If you accidentally modify an entity in a `readOnly = true` transaction, Hibernate may or may not persist it depending on the driver. Don't rely on it as a write guard.

---

## 💻 @Transactional on Controller vs Service

Where should `@Transactional` be placed?

```java
// ❌ On Controller — wrong
@RestController
public class UserController {
    
    @PostMapping("/users")
    @Transactional  // Don't put here
    public ResponseEntity<UserResponse> createUser(...) {
        // HTTP processing is NOT a database concern
    }
}

// ✅ On Service — correct
@Service
public class UserService {
    
    @Transactional  // Here — at the boundary of business operations
    public UserResponse createUser(CreateUserRequest request) {
        // All DB operations in this method share one transaction
    }
}
```

**Why not on Controller?**
- Controllers handle HTTP concerns, not database concerns
- Tests for controllers shouldn't need transaction rollback
- Mixing concerns

---

## 💻 Practical Transaction Pattern

```java
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService; // @Transactional(REQUIRES_NEW)
    
    // Main operation — one transaction for ALL db changes
    @Transactional
    public OrderResponse placeOrder(PlaceOrderRequest request) {
        // All of these are in the SAME transaction
        validateInventory(request);
        Order order = createOrder(request);
        deductInventory(request);
        Payment payment = processPayment(order, request.getPaymentDetails());
        
        // This runs in its OWN transaction (REQUIRES_NEW)
        // Will NOT rollback even if the outer transaction fails
        notificationService.sendOrderConfirmation(order.getId());
        
        return mapToResponse(order, payment);
    }
    
    private void validateInventory(PlaceOrderRequest request) {
        for (OrderItem item : request.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
            
            if (product.getStock() < item.getQuantity()) {
                throw new InsufficientStockException(product.getName(), product.getStock());
            }
        }
    }
    
    private Order createOrder(PlaceOrderRequest request) {
        Order order = new Order(request.getUserId(), "PENDING");
        request.getItems().forEach(item -> order.addItem(new OrderItem(item)));
        return orderRepository.save(order);
    }
    
    private void deductInventory(PlaceOrderRequest request) {
        request.getItems().forEach(item -> {
            Product product = productRepository.findById(item.getProductId()).orElseThrow();
            product.deductStock(item.getQuantity());
            // Dirty checking handles the UPDATE — no explicit save()
        });
    }
    
    private Payment processPayment(Order order, PaymentDetails details) {
        Payment payment = new Payment(order, details.getAmount(), "PENDING");
        // ... payment processing logic
        return paymentRepository.save(payment);
    }
}
```

---

## 💥 Break It Exercise

This code has a transaction bug:

```java
@Service
public class TransferService {
    
    @Transactional
    public void transfer(Long fromId, Long toId, double amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();
        
        from.deduct(amount); // Dirty checking will update DB
        to.add(amount);      // Dirty checking will update DB
        
        sendTransferNotification(fromId, toId, amount);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendTransferNotification(Long fromId, Long toId, double amount) {
        notificationRepository.save(new Notification(fromId, toId, amount));
    }
}
```

**Question**: `sendTransferNotification` uses `REQUIRES_NEW`. Will it run in a separate transaction from `transfer`?

<details>
<summary>Answer</summary>

**No** — because of self-invocation.

`transfer()` calls `this.sendTransferNotification(...)` — directly on `this`, not through the Spring proxy. The proxy is never involved. `REQUIRES_NEW` propagation is completely ignored.

Both operations run in the same transaction as `transfer()`.

**Correct fix**: Extract to another service:

```java
@Service
public class NotificationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendTransferNotification(...) { ... }
}

@Service
public class TransferService {
    private final NotificationService notificationService; // Inject as dependency
    
    @Transactional
    public void transfer(...) {
        // ...
        notificationService.sendTransferNotification(...); // Goes through proxy ✅
    }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain how `@Transactional` uses a proxy?
- [ ] Can you explain REQUIRED vs REQUIRES_NEW propagation?
- [ ] Can you explain the self-invocation problem and fix it?
- [ ] Can you explain why `@Transactional` doesn't roll back on checked exceptions by default?
- [ ] Can you explain when to use `readOnly = true`?

---

*Next: [11-01 — Authentication vs Authorization](../11-security/01-authentication-vs-authorization.md)*
