# Lesson 01-06 — Optional

## 🎯 Learning Objective

Master `Optional<T>` — Spring Data returns `Optional` everywhere, and using it incorrectly leads to the exact `NullPointerException` it was designed to prevent.

---

## 🤔 Think Before Reading

In Node.js, you might write:

```javascript
const user = await userRepository.findById(id);
if (!user) {
  throw new Error('User not found');
}
return user;
```

The `if (!user)` check is easy to forget. Developers forget it constantly.

**Question**: What if the type system could FORCE you to handle the "not found" case? How would that work?

---

## 🧠 What Optional Is (and Isn't)

`Optional<T>` is a container that either:
- **Contains** a value of type T
- **Is empty** (no value)

It is **NOT** a replacement for `null` everywhere. It's specifically designed for:
- Return types of methods that might not find a value
- Repository `findById`, `findByEmail` methods
- Optional configuration values

```java
// Without Optional — caller might forget to check
public User findById(Long id) {
    return database.get(id); // Returns null if not found — easy to forget!
}

// With Optional — caller is FORCED to handle the empty case
public Optional<User> findById(Long id) {
    User user = database.get(id);
    return Optional.ofNullable(user); // Wraps null into empty Optional
}
```

---

## 💻 Creating Optionals

```java
// From a value that exists
Optional<String> name = Optional.of("Alice");

// From a value that MIGHT be null
Optional<String> email = Optional.ofNullable(getUserEmail()); // null or value

// Empty Optional
Optional<String> empty = Optional.empty();

// DON'T do this:
Optional<String> bad = Optional.of(null); // NullPointerException!
// Use ofNullable for nullable values
```

---

## 🔍 Using Optionals — The Right Way

### Anti-patterns (defeat the purpose of Optional)

```java
// ❌ Anti-pattern 1: Checking isPresent then get
Optional<User> opt = userRepository.findById(id);
if (opt.isPresent()) {
    User user = opt.get(); // Still verbose, no better than null check
}

// ❌ Anti-pattern 2: get() without checking
User user = userRepository.findById(id).get(); // NoSuchElementException if empty!

// ❌ Anti-pattern 3: Optional as a field
public class Order {
    private Optional<String> notes; // DON'T — use null for optional fields
}
```

### Correct patterns

```java
Optional<User> opt = userRepository.findById(id);

// ✅ Pattern 1: orElseThrow — throw exception if empty
User user = opt.orElseThrow(() -> new UserNotFoundException(id));

// ✅ Pattern 2: orElse — default value
User user = opt.orElse(new User("default@example.com"));

// ✅ Pattern 3: orElseGet — lazy default (supplier)
User user = opt.orElseGet(() -> createDefaultUser());

// ✅ Pattern 4: map — transform if present, stay empty if not
Optional<String> name = opt.map(User::getName);

// ✅ Pattern 5: ifPresent — run side effect if present
opt.ifPresent(user -> sendWelcomeEmail(user));

// ✅ Pattern 6: filter — narrow down
Optional<User> admin = opt.filter(u -> u.getRole().equals("ADMIN"));
```

---

## 🔄 Chaining Optionals — The Pipeline Pattern

```java
// Find user → get their email → check if it's a business email
Optional<String> businessEmail = userRepository.findById(userId)
    .map(User::getEmail)                              // User → String (email)
    .filter(email -> email.endsWith("@company.com")); // Only business emails

// Handle result
String email = businessEmail.orElse("No business email found");
```

Compare with JavaScript optional chaining:
```javascript
const email = userRepository.findById(userId)?.email;
// But JS still doesn't force you to handle the undefined case
```

---

## 🏗️ Optional in Spring Boot — Full Example

```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    // ✅ Throw exception if not found
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(this::toResponse)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // ✅ Return Optional to caller
    public Optional<UserResponse> findByEmail(String email) {
        return userRepository.findByEmail(email)
            .map(this::toResponse);
    }
    
    // ✅ Conditional logic
    public UserResponse findOrCreateByEmail(String email) {
        return userRepository.findByEmail(email)
            .map(this::toResponse)
            .orElseGet(() -> {
                User newUser = userRepository.save(new User(email));
                return toResponse(newUser);
            });
    }
    
    private UserResponse toResponse(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
    }
}
```

---

## 💥 Break It Exercise

Find the bug:

```java
@Service
public class ProductService {
    
    public double getDiscountedPrice(Long productId, Long userId) {
        Optional<Product> product = productRepository.findById(productId);
        Optional<User> user = userRepository.findById(userId);
        
        if (product.isPresent() && user.isPresent()) {
            double basePrice = product.get().getPrice();
            String userTier = user.get().getTier();
            
            if (userTier.equals("PREMIUM")) {
                return basePrice * 0.8; // 20% discount
            }
            return basePrice;
        }
        
        return 0.0; // Silent failure
    }
}
```

**Questions**:
1. What happens when the product is not found? Is this good behavior?
2. What happens when the user is not found? Is this good behavior?
3. How would you fix this?

<details>
<summary>Answer</summary>

**Problems**:
1. Returns `0.0` silently when product not found — caller thinks the product costs nothing
2. Returns `0.0` silently when user not found — same problem
3. The `isPresent() + get()` pattern is verbose and still technically an anti-pattern

**Fix**:
```java
public double getDiscountedPrice(Long productId, Long userId) {
    Product product = productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException(productId));
    
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new UserNotFoundException(userId));
    
    double basePrice = product.getPrice();
    
    return user.getTier().equals("PREMIUM") ? basePrice * 0.8 : basePrice;
}
```

Never return a "magic value" like `0.0` or `null` on failure. Throw a meaningful exception.

</details>

---

## 🎯 Practice Exercise

Given:

```java
public record User(Long id, String name, String email, String tier) {}
public record Order(Long id, Long userId, double amount, String status) {}
```

Write these methods using Optional chains (no `isPresent()` + `get()`):

1. `getOrderAmount(Long orderId)` — returns the amount, throws `OrderNotFoundException` if not found
2. `getOrderStatusForUser(Long userId)` — returns list of statuses for all user's COMPLETED orders
3. `findPremiumUserEmail(Long userId)` — returns `Optional<String>` — the email if user is PREMIUM, empty if not or if user not found

<details>
<summary>Solution</summary>

```java
// 1. Get order amount
public double getOrderAmount(Long orderId) {
    return orderRepository.findById(orderId)
        .map(Order::amount)
        .orElseThrow(() -> new OrderNotFoundException(orderId));
}

// 2. Statuses for completed orders
public List<String> getCompletedOrderStatuses(Long userId) {
    return orderRepository.findByUserId(userId)
        .stream()
        .filter(o -> o.status().equals("COMPLETED"))
        .map(Order::status)
        .collect(Collectors.toList());
}

// 3. Email only if PREMIUM
public Optional<String> findPremiumUserEmail(Long userId) {
    return userRepository.findById(userId)
        .filter(u -> u.tier().equals("PREMIUM"))
        .map(User::email);
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain what `Optional` prevents?
- [ ] Can you use `orElseThrow`, `orElse`, `orElseGet`, `map`, `filter`?
- [ ] Can you write Optional pipelines without `isPresent()` + `get()`?
- [ ] Did you fix the Break It exercise?
- [ ] Did you solve the Practice Exercise?

---

*Next: [01-07 — Records](./07-records.md)*
