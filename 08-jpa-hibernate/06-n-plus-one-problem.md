# Lesson 08-06 — The N+1 Problem

## 🎯 Learning Objective

Understand the most common and damaging JPA performance problem. Learn to detect it, understand why it happens, and fix it with fetch joins and EntityGraphs.

---

## 🤔 Think Before Reading

You have 100 users. Each user has tasks. You want to show all users with their task counts.

Naive approach: Load all users, then for each user load their tasks.

**Question**: How many database queries does that require? Is there a better way?

---

## 🔢 What Is N+1?

The N+1 problem occurs when loading a collection triggers one query per parent entity.

```
1 query → Get all users (returns 100 users)
+
100 queries → Get tasks for user 1, user 2, ... user 100
=
101 queries total (1 + N)
```

With 1000 users: 1001 queries. With 10,000 users: 10,001 queries.

This is catastrophic for performance.

---

## 💻 Reproducing the N+1 Problem

```java
@Service
public class UserReportService {
    
    private final UserRepository userRepository;
    
    public List<UserReport> generateReport() {
        List<User> users = userRepository.findAll();
        // Query 1: SELECT * FROM users
        
        return users.stream()
            .map(user -> {
                // For each user, accessing tasks triggers a lazy load:
                // Query 2, 3, 4, ... N+1: SELECT * FROM tasks WHERE user_id = ?
                int taskCount = user.getTasks().size(); 
                return new UserReport(user.getId(), user.getName(), taskCount);
            })
            .collect(Collectors.toList());
        // Total queries: 1 + number of users
    }
}
```

With lazy loading (default), `user.getTasks()` is NOT loaded when you load the user. It's loaded when you first access the collection.

This is called **lazy initialization** — good for avoiding unnecessary loads, but a trap when you actually need the related data.

---

## 🔍 Detecting N+1

Add to `application.properties`:
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

You'll see:
```sql
SELECT u.* FROM users u;

SELECT t.* FROM tasks t WHERE t.user_id = 1;
SELECT t.* FROM tasks t WHERE t.user_id = 2;
SELECT t.* FROM tasks t WHERE t.user_id = 3;
-- ... 97 more
```

In production, use **Hibernate statistics** or tools like **p6spy** or **DataDog APM**.

---

## ✅ Fix 1 — JPQL Fetch Join

```java
// In your repository:
public interface UserRepository extends JpaRepository<User, Long> {
    
    // JPQL JOIN FETCH loads users AND their tasks in ONE query
    @Query("SELECT u FROM User u JOIN FETCH u.tasks WHERE u.id IN :ids")
    List<User> findAllWithTasks();
}
```

Generated SQL:
```sql
SELECT u.*, t.*
FROM users u
INNER JOIN tasks t ON t.user_id = u.id
```

**One query** — not 101.

### Limitation of JOIN FETCH with pagination

```java
// ❌ This causes a HUGE problem
@Query("SELECT u FROM User u JOIN FETCH u.tasks")
Page<User> findAllWithTasksPaged(Pageable pageable);
// Hibernate warning: HHH90003004: firstResult/maxResults specified with collection fetch; 
// applying in memory! → Loads ALL data into memory, paginates in Java
```

**Never JOIN FETCH a collection with pagination.** This loads the entire dataset into memory.

Fix for pagination + collections:
```java
// Step 1: Get user IDs with pagination
@Query("SELECT u.id FROM User u")
Page<Long> findUserIds(Pageable pageable);

// Step 2: Load users with tasks by those IDs
@Query("SELECT u FROM User u JOIN FETCH u.tasks WHERE u.id IN :ids")
List<User> findByIdsWithTasks(@Param("ids") List<Long> ids);
```

---

## ✅ Fix 2 — @EntityGraph

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Declare which associations to fetch eagerly for this query
    @EntityGraph(attributePaths = {"tasks"})
    List<User> findAll();
    
    // Multiple associations
    @EntityGraph(attributePaths = {"tasks", "profile"})
    Optional<User> findById(Long id);
}
```

`@EntityGraph` is cleaner than JPQL JOIN FETCH when you need dynamic loading decisions.

---

## ✅ Fix 3 — Batch Fetching (Hibernate-specific)

```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
@BatchSize(size = 20)  // Hibernate fetches in batches of 20
private List<Task> tasks;
```

Instead of N queries, Hibernate batches: `SELECT * FROM tasks WHERE user_id IN (1, 2, 3, ..., 20)`.

Reduces N+1 to ceil(N/batchSize) queries.

---

## 🔍 When to Use Each Fix

| Situation | Solution |
|-----------|---------|
| Always need the association | Make it `EAGER` (careful with collections) |
| Usually don't need it, sometimes do | `LAZY` + `JOIN FETCH` when needed |
| Need to paginate with collection data | Two-query approach (page IDs, then fetch with join) |
| Dynamic per-request control | `@EntityGraph` |
| Many associations, some rarely used | `@BatchSize` as optimization |

---

## 🔍 Eager vs Lazy — Default Behavior

| Relationship | Default FetchType |
|-------------|-------------------|
| `@ManyToOne` | `EAGER` |
| `@OneToOne` | `EAGER` |
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |

The `EAGER` defaults for `@ManyToOne` mean: every time you load a Task, its User is ALSO loaded. This can cause unexpected queries.

**Best practice**: Override defaults to `LAZY` for ALL relationships. Fetch eagerly only when you explicitly need it.

```java
@ManyToOne(fetch = FetchType.LAZY)  // Override the EAGER default
@JoinColumn(name = "user_id")
private User user;
```

---

## 💥 Break It — LazyInitializationException

```java
@Service
public class TaskService {
    
    public TaskResponse getTaskWithUser(Long taskId) {
        Task task = taskRepository.findById(taskId).orElseThrow();
        // task.getUser() is LAZY
        return new TaskResponse(
            task.getId(),
            task.getTitle(),
            task.getUser().getName() // ← Problem here
        );
    }
}
```

If this method is NOT annotated with `@Transactional`, what happens when `task.getUser().getName()` is called?

<details>
<summary>Answer</summary>

**`LazyInitializationException`**: "could not initialize proxy - no Session"

**Why**: 
1. `findById()` runs a query and the transaction (if any) ends
2. `task.getUser()` is a lazy proxy — it needs an active Hibernate Session to load
3. No active session → exception

**Fixes**:
1. Add `@Transactional` to the service method
2. Use `JOIN FETCH` in the repository query to load user eagerly
3. Access `task.getUser()` within the transaction boundary

```java
// Fix 1: Keep transaction open
@Transactional
public TaskResponse getTaskWithUser(Long taskId) {
    Task task = taskRepository.findById(taskId).orElseThrow();
    return new TaskResponse(task.getId(), task.getTitle(), task.getUser().getName());
}

// Fix 2: Fetch join
@Query("SELECT t FROM Task t JOIN FETCH t.user WHERE t.id = :id")
Optional<Task> findByIdWithUser(@Param("id") Long id);
```

`LazyInitializationException` is one of the most common Spring Boot errors. Knowing this fix is essential.

</details>

---

## 🎯 Practice Exercise

Given:
- `Author` → has many `Books`
- `Book` → has many `Reviews`

You need to display: all authors, with the count of books and average review rating.

1. Write a JPQL query that avoids N+1
2. Consider: should you load all Reviews for every Book? What's a smarter approach?

<details>
<summary>Solution</summary>

```java
// DTO for the projection
public record AuthorStats(Long authorId, String authorName, Long bookCount, Double avgRating) {}

// Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {
    
    // Use aggregate query — no N+1, no loading entire entities
    @Query("""
        SELECT new com.example.dto.AuthorStats(
            a.id,
            a.name,
            COUNT(DISTINCT b.id),
            AVG(r.rating)
        )
        FROM Author a
        LEFT JOIN a.books b
        LEFT JOIN b.reviews r
        GROUP BY a.id, a.name
        """)
    List<AuthorStats> findAuthorStats();
}
```

This is ONE query with aggregation — no N+1, no loading full entities or their reviews.

**Key insight**: For reporting queries, use aggregate JPQL projections (or native SQL) instead of loading entity graphs. Load only the data you actually need.

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain the N+1 problem in plain English?
- [ ] Can you reproduce it in code and detect it from SQL logs?
- [ ] Can you fix it with `JOIN FETCH`?
- [ ] Can you explain why `JOIN FETCH` with pagination is dangerous?
- [ ] Can you fix `LazyInitializationException`?
- [ ] Do you know the default fetch types for each relationship annotation?

---

*Next: [09-01 — Spring Data JPA Repositories](../09-postgresql/01-spring-data-jpa.md)*
