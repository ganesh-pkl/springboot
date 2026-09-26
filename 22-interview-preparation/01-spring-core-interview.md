# Interview Preparation — Spring Core, JPA, Security, and Scenario Questions

## 🎯 Purpose

This module tests UNDERSTANDING, not memorization. Every question here has been asked in real Spring Boot interviews at companies from startups to FAANG.

Work through each section without looking at answers first.

---

## PART 1 — Spring Core

### ⭐ Easy

**Q1**: What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

<details>
<summary>Answer</summary>

All four are specializations of `@Component` — they all mark a class as a Spring bean. The differences:

- `@Component`: Generic, catch-all stereotype
- `@Service`: Marks business logic layer (semantic clarity only, no functional difference from @Component)
- `@Repository`: Marks data access layer + enables automatic exception translation (JPA/JDBC exceptions → Spring DataAccessException)
- `@Controller`: Marks web layer + enables Spring MVC to detect and register routes
- `@RestController`: `@Controller` + `@ResponseBody` — all methods return JSON

</details>

---

**Q2**: What is the difference between IoC and Dependency Injection?

<details>
<summary>Answer</summary>

**IoC** (Inversion of Control) is the principle: the framework controls object creation and lifecycle, not your application code.

**Dependency Injection** is one implementation of IoC: objects receive their dependencies from the outside (constructor, setter, field) rather than creating them internally.

IoC is the concept. DI is one way to implement it.

Other implementations of IoC: Service Locator pattern, Factory pattern, Template Method pattern.

</details>

---

**Q3**: What are the three ways to inject dependencies in Spring? Which is recommended and why?

<details>
<summary>Answer</summary>

1. **Constructor injection** (recommended)
2. **Field injection** (`@Autowired` on fields)
3. **Setter injection** (`@Autowired` on setters)

Constructor injection is recommended because:
- Dependencies are explicit and required (can't instantiate without them)
- Dependencies can be `final` (immutable)
- Easier to test (just call `new MyService(mockDep)`)
- Circular dependencies fail at startup (detectable early)
- No Spring needed in unit tests

</details>

---

**Q4**: What is the default scope of a Spring bean? What other scopes exist?

<details>
<summary>Answer</summary>

Default: **Singleton** — one instance per ApplicationContext.

Other scopes:
- **Prototype**: New instance every time the bean is requested
- **Request**: One instance per HTTP request (web apps)
- **Session**: One instance per HTTP session (web apps)
- **Application**: One instance per ServletContext

For singletons: they must be **stateless** — no instance variables that change per request, as they're shared across all threads.

</details>

---

**Q5**: What happens if two beans implement the same interface and you try to inject the interface?

<details>
<summary>Answer</summary>

Spring throws `NoUniqueBeanDefinitionException`: "expected single matching bean but found 2."

Resolution options:
1. `@Primary` — mark one implementation as the default
2. `@Qualifier("beanName")` — specify which one to inject at the injection point
3. Inject `List<InterfaceType>` — Spring injects ALL implementations
4. `@ConditionalOn*` — conditionally create only one implementation

</details>

---

### ⭐⭐ Intermediate

**Q6**: Explain what a `BeanPostProcessor` does. Give an example of what Spring uses it for.

<details>
<summary>Answer</summary>

`BeanPostProcessor` is a hook that runs before and after each bean's initialization:

- `postProcessBeforeInitialization()` — runs before `@PostConstruct`
- `postProcessAfterInitialization()` — runs after `@PostConstruct`

It can replace the bean with a different object — typically a **proxy**.

Spring uses it for:
- `AutowiredAnnotationBeanPostProcessor` — processes `@Autowired` field/setter injection
- `TransactionProxyCreator` — wraps `@Transactional` beans in transaction proxies
- `AsyncAnnotationBeanPostProcessor` — wraps `@Async` beans in async execution proxies
- `CachingInterceptor` — wraps `@Cacheable` beans in caching proxies

This is why you must be careful with method visibility and self-invocation when using these annotations.

</details>

---

**Q7**: What is component scanning? What package does `@SpringBootApplication` scan by default?

<details>
<summary>Answer</summary>

Component scanning is the process of Spring reading `.class` metadata to find beans (classes annotated with `@Component` and its specializations).

`@SpringBootApplication` includes `@ComponentScan` which scans:
- The package where the main class is located
- ALL sub-packages of that package

Classes in other packages are NOT scanned unless explicitly configured.

Common mistake: main class at `com.example` scans `com.example.*` but not `com.other.*`.

</details>

---

**Q8**: What is the bean lifecycle? Describe at least 5 steps in order.

<details>
<summary>Answer</summary>

1. Bean definition loaded (from component scan or @Bean method)
2. Bean instantiated (constructor called)
3. Dependencies injected (constructor or field injection)
4. Aware interfaces called (`BeanNameAware`, `ApplicationContextAware`, etc.)
5. `BeanPostProcessor.postProcessBeforeInitialization()` called
6. `@PostConstruct` methods called
7. `InitializingBean.afterPropertiesSet()` called (if implemented)
8. `BeanPostProcessor.postProcessAfterInitialization()` — **proxies created here**
9. Bean is in service (used by application)
10. `@PreDestroy` methods called (on shutdown)
11. `DisposableBean.destroy()` called (if implemented)

</details>

---

### ⭐⭐⭐ Advanced

**Q9**: Explain how `@Transactional` works internally. Include the proxy mechanism.

<details>
<summary>Answer</summary>

1. When Spring starts and finds a `@Transactional` annotated bean, `BeanPostProcessor` creates a **proxy** around the bean
2. The proxy is stored in the container — not the original bean
3. When a method is called through the proxy, the proxy's transaction advice runs:
   - Checks if there's an existing transaction (propagation rules)
   - Starts a transaction if needed (BEGIN)
   - Calls the original method
   - Commits if no exception, rolls back if RuntimeException
4. Rollback only happens for unchecked exceptions (RuntimeException) by default

**Why self-invocation doesn't work**: `this.method()` goes directly to the real object, not the proxy. The proxy is never involved, so transaction advice never runs.

**Why private methods don't work**: JDK dynamic proxies can only proxy interface methods. CGLIB can proxy non-final public/protected methods. Private methods are never proxied.

</details>

---

**Q10**: What is the difference between `@Configuration` and `@Component` when used with `@Bean` methods?

<details>
<summary>Answer</summary>

`@Configuration` uses CGLIB to create a **subclass proxy** of the configuration class. This proxy intercepts method calls to `@Bean` methods.

When `@Bean` method A calls another `@Bean` method B:
- With `@Configuration`: the proxy intercepts the call, returns the existing singleton from the container
- With `@Component` ("Lite Mode"): the proxy doesn't exist, each call creates a NEW instance

```java
@Configuration
public class Config {
    @Bean DataSource dataSource() { return new HikariDataSource(); }
    @Bean UserRepo userRepo() { return new Repo(dataSource()); }  // Same DataSource
    @Bean AuditRepo auditRepo() { return new Repo(dataSource()); } // Same DataSource
}

@Component // Lite mode — two different DataSources created!
public class Config2 {
    @Bean DataSource dataSource() { return new HikariDataSource(); }
    @Bean UserRepo userRepo() { return new Repo(dataSource()); }  // DataSource #1
    @Bean AuditRepo auditRepo() { return new Repo(dataSource()); } // DataSource #2 — DIFFERENT!
}
```

</details>

---

## PART 2 — JPA / Hibernate

### ⭐ Easy

**Q11**: What is the difference between JPA and Hibernate?

<details>
<summary>Answer</summary>

JPA is a specification (interface + annotations) — it defines what an ORM should do, but contains no implementation.

Hibernate is an implementation of JPA — it provides the actual ORM engine that implements the JPA spec.

You write JPA code (`@Entity`, `EntityManager`, `JpaRepository`). Hibernate executes it. Spring Boot uses Hibernate as the default JPA provider.

</details>

---

**Q12**: What is the persistence context and what is dirty checking?

<details>
<summary>Answer</summary>

**Persistence context**: A first-level cache and change tracker. When you load an entity within a transaction, it's "managed" by the persistence context. JPA takes a snapshot of its state at load time.

**Dirty checking**: At transaction commit, JPA compares each managed entity's current state to its snapshot. If anything changed, JPA generates UPDATE SQL automatically. No explicit `save()` needed for changes to managed entities.

Dirty checking only works within an active transaction. After the transaction closes, entities become "detached" and changes are not tracked.

</details>

---

**Q13**: Explain the N+1 problem and how to fix it.

<details>
<summary>Answer</summary>

N+1: When loading a list of N entities, and then accessing a lazy-loaded relationship on each one, you end up with 1 + N queries instead of 1 or 2.

Example: Load 100 users (1 query), then access `user.getTasks()` for each (100 more queries) = 101 queries.

Fixes:
1. **JOIN FETCH**: `SELECT u FROM User u JOIN FETCH u.tasks WHERE...` — one query
2. **@EntityGraph**: `@EntityGraph(attributePaths = {"tasks"})` — tells Spring Data to join
3. **@BatchSize(size=20)**: Hibernate fetches in batches — reduces to ceil(N/20) queries
4. **Projection queries**: `SELECT new UserStats(u.id, COUNT(t))` — aggregate in DB

</details>

---

### ⭐⭐⭐ Advanced

**Q14**: What is the difference between `CascadeType.REMOVE` and `orphanRemoval = true`?

<details>
<summary>Answer</summary>

**CascadeType.REMOVE**: When you call `delete(parent)`, all children are also deleted.

**orphanRemoval = true**: When you remove an entity from its parent's collection, it is deleted from the database (it becomes an "orphan" — no parent references it anymore).

```java
// CascadeType.REMOVE:
userRepository.delete(user); // Also deletes all user's tasks

// orphanRemoval:
user.getTasks().remove(task); // Also deletes task from DB (orphan)
```

They're different triggers. You can have orphanRemoval without CascadeType.REMOVE.

Use orphanRemoval when children cannot exist without their parent (composition relationship).

</details>

---

**Q15**: Explain `@Transactional` rollback behavior for checked vs unchecked exceptions.

<details>
<summary>Answer</summary>

By default, `@Transactional` only rolls back for **unchecked exceptions** (RuntimeException and its subclasses).

Checked exceptions (IOException, SQLException, etc.) do NOT trigger rollback by default — the transaction commits.

This surprises many developers. The historical reason: checked exceptions represent "expected" failure conditions that the caller might handle. Unchecked exceptions represent "unexpected" failures.

To change behavior:
```java
@Transactional(rollbackFor = Exception.class)     // Rollback on ANY exception
@Transactional(rollbackFor = IOException.class)   // Rollback on IOException specifically
@Transactional(noRollbackFor = ValidationException.class) // Don't rollback on this one
```

</details>

---

## PART 3 — Security

### ⭐⭐ Intermediate

**Q16**: Describe the Spring Security filter chain. What happens when a request arrives?

<details>
<summary>Answer</summary>

Spring Security is implemented as a chain of servlet filters running before your application code:

1. Request arrives at Tomcat
2. Passes through SecurityFilterChain (multiple filters in sequence)
3. Key filters: CorsFilter → CsrfFilter → AuthenticationFilter (your JwtFilter) → ExceptionTranslationFilter → AuthorizationFilter
4. Each filter can pass to the next or terminate the chain
5. If all security checks pass → DispatcherServlet → Controller

JWT flow specifically:
1. JwtAuthFilter reads `Authorization: Bearer <token>` header
2. Validates the token (signature, expiry)
3. Loads user from DB via UserDetailsService
4. Sets `Authentication` in `SecurityContextHolder`
5. Subsequent AuthorizationFilter reads SecurityContext to check permissions

</details>

---

**Q17**: What is the `SecurityContextHolder`? How is it thread-safe?

<details>
<summary>Answer</summary>

`SecurityContextHolder` stores the current user's authentication information, accessible from anywhere in the application.

Thread safety: By default, it uses `ThreadLocal` storage — each thread has its own `SecurityContext`. So different HTTP request threads have different security contexts.

Access pattern:
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
UserDetails user = (UserDetails) auth.getPrincipal();
```

Limitation: In async methods (`@Async`), a different thread runs the code, so it has a different `ThreadLocal` and loses the `SecurityContext`. Fix: Configure `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` or use Spring Security's async support.

</details>

---

## PART 4 — Scenario Questions

**Q18**: You're seeing intermittent `LazyInitializationException` in production. What are the likely causes and how do you fix them?

<details>
<summary>Answer</summary>

**Cause**: Code is accessing a lazy-loaded relationship (e.g., `user.getTasks()`) AFTER the transaction (and therefore Hibernate session) has closed. The proxy can't load the data because there's no active session.

Common sources:
1. Service method without `@Transactional` calls `entity.getLazyCollection()`
2. Entity is returned from `@Transactional` method, then used in controller or view
3. Serialization (Jackson converting entity to JSON) triggers lazy load outside transaction
4. Async method tries to access lazy relation after original transaction closed

**Fixes**:
1. Add `@Transactional` to the method that accesses the lazy data
2. Use `JOIN FETCH` or `@EntityGraph` to eagerly load needed data
3. Map to DTOs INSIDE the `@Transactional` boundary (before the session closes)
4. Never serialize entities directly — always map to DTOs

**Anti-pattern to avoid**: `spring.jpa.open-in-view=true` (default enabled!) creates a transaction for the entire request-response cycle. This hides LazyInitializationException but causes N+1 in controllers/views and is terrible for performance.

</details>

---

**Q19**: A user reports that their account balance was charged but the order wasn't created. How do you investigate and prevent this?

<details>
<summary>Answer</summary>

This is a **partial failure** — a transaction integrity issue.

**Investigation**:
1. Check application logs for the timestamp of the incident
2. Look for any exception in the order creation process
3. Check if charge and order creation were in the SAME database transaction
4. Check if a caught exception swallowed the error

**Root cause** (likely): 
```java
public void placeOrder(Order order) {
    paymentService.charge(order.getAmount()); // Committed!
    try {
        orderService.create(order);
    } catch (Exception e) {
        log.error("Order creation failed"); // Swallowed! Charge already committed.
    }
}
```

**Fix**:
```java
@Transactional // Both operations in same transaction
public void placeOrder(Order order) {
    paymentService.charge(order.getAmount());
    orderService.create(order);
    // If either fails → both roll back
}
```

**Additional protection**: Idempotency keys, compensation transactions, saga pattern for distributed systems.

</details>

---

**Q20**: A Spring Boot application starts taking 2x longer to respond after a certain time. No code changes were deployed. What do you check?

<details>
<summary>Answer</summary>

Systematic investigation:

1. **Connection pool exhaustion**: Check `spring.datasource.hikari.active` via Actuator. If connections are maxed, queries wait for available connections.

2. **Memory/GC**: Check GC logs or JVM metrics. If heap is full, GC pauses cause slowdowns.

3. **N+1 queries appearing**: Check if new data was added that triggers worse query patterns. 1 user → 1 extra query becomes 10,000 users → 10,000 extra queries.

4. **Missing database index**: As data grows, unindexed queries degrade. `EXPLAIN ANALYZE` the slow queries.

5. **External service**: Is a third-party API call taking longer? Check response times.

6. **Lock contention**: Long-running transactions holding row locks, causing other transactions to wait.

7. **Cache degradation**: If using caching, check hit rate. If cache expired or evicted, all requests hit DB.

Use Spring Actuator metrics, slow query logs (`spring.jpa.show-sql`), and APM tools.

</details>

---

## PART 5 — Design and Architecture

**Q21**: How would you implement rate limiting in Spring Boot?

<details>
<summary>Answer</summary>

Multiple approaches:

1. **Spring Filter**: Create a `RateLimitFilter` that extends `OncePerRequestFilter`. Track requests per IP/user in a `ConcurrentHashMap<String, AtomicInteger>`. Return 429 if limit exceeded.

2. **AOP Aspect**: Create a `@RateLimit` annotation. Implement an `@Around` aspect that checks a Redis counter.

3. **Redis + Lua script**: Atomic counter in Redis. Lua script checks and increments atomically. Best for distributed systems.

4. **Bucket4j library**: Token bucket algorithm. Spring Boot integration available.

5. **API Gateway**: Handle at infrastructure level (Kong, AWS API Gateway) — no application code needed.

For production, Redis-based solutions are preferred (work across multiple instances).

</details>

---

**Q22**: How do you handle database migrations in a Spring Boot application?

<details>
<summary>Answer</summary>

Use **Flyway** or **Liquibase** — migration tools that track which SQL scripts have been applied.

With Flyway:
1. Create SQL files in `src/main/resources/db/migration/` named `V1__init.sql`, `V2__add_user_table.sql`, etc.
2. Add dependency: `spring-boot-starter-flyway`
3. Set `spring.jpa.hibernate.ddl-auto=validate` (Hibernate validates, doesn't create)
4. On startup, Flyway runs any pending migrations in order

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Never use** `ddl-auto=create` or `update` in production — it can destroy data.

</details>

---

## 📋 Final Self-Assessment

Rate yourself honestly (1-5) on each:

```
[ ] 1-5  Spring IoC / DI understanding
[ ] 1-5  Bean lifecycle
[ ] 1-5  @Transactional internals
[ ] 1-5  JPA persistence context and dirty checking
[ ] 1-5  N+1 problem detection and fixes
[ ] 1-5  Spring Security filter chain
[ ] 1-5  JWT implementation
[ ] 1-5  AOP and proxy mechanism
[ ] 1-5  Production debugging ability
```

If anything is below 3, go back to that module.

---

*Congratulations! Now build the capstone project → [Project 4: Production-Grade Backend](../projects/04-production-grade-backend/README.md)*
