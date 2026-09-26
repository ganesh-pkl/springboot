# Lesson 09-01 — Spring Data JPA Repositories

## 🎯 Learning Objective

Understand Spring Data JPA's repository abstraction. Understand what `JpaRepository` provides, how query derivation works, and when to write custom JPQL/native queries.

---

## 🤔 Think Before Reading

In Node.js with Prisma or TypeORM, you have to write methods for each query. Spring Data JPA claims it can generate query implementations from method names alone.

**Question**: If you write `List<User> findByEmail(String email)` with NO implementation, how does Spring know what SQL to run? What's the mechanism?

---

## 🏗️ The Repository Hierarchy

```
Repository<T, ID>               ← Marker interface
    └── CrudRepository<T, ID>   ← Basic CRUD
        └── PagingAndSortingRepository<T, ID>  ← + pagination
            └── JpaRepository<T, ID>           ← + JPA-specific features (flush, batch, etc.)
```

In almost all Spring Boot apps, extend `JpaRepository`:

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // T = User, ID = Long
}
```

This gives you for FREE:

```java
// From CrudRepository:
userRepository.save(user);
userRepository.saveAll(users);
userRepository.findById(id);
userRepository.existsById(id);
userRepository.findAll();
userRepository.count();
userRepository.deleteById(id);
userRepository.delete(user);
userRepository.deleteAll();

// From JpaRepository:
userRepository.flush();
userRepository.saveAndFlush(user);
userRepository.findAll(Sort.by("name"));
userRepository.findAll(PageRequest.of(0, 20));
```

No SQL. No implementation. Spring Data generates all of it.

---

## 🔍 Query Derivation — The Magic Explained

When Spring starts, it scans all repository interfaces. For methods like `findByEmail(String email)`, it:

1. Parses the method name using a grammar
2. Builds a query based on the parsed structure
3. Stores the generated query for execution at runtime

**Grammar examples**:

```java
// findBy[Field][Condition]
findByEmail(String email)
// → SELECT * FROM users WHERE email = ?

findByNameAndEmail(String name, String email)
// → SELECT * FROM users WHERE name = ? AND email = ?

findByAgeGreaterThan(int age)
// → SELECT * FROM users WHERE age > ?

findByNameContaining(String keyword)
// → SELECT * FROM users WHERE name LIKE %?%

findByActiveTrue()
// → SELECT * FROM users WHERE active = true

findByCreatedAtBetween(LocalDateTime start, LocalDateTime end)
// → SELECT * FROM users WHERE created_at BETWEEN ? AND ?

findByStatusOrderByCreatedAtDesc(String status)
// → SELECT * FROM users WHERE status = ? ORDER BY created_at DESC

// Count queries
long countByStatus(String status);
// → SELECT COUNT(*) FROM users WHERE status = ?

// Existence queries  
boolean existsByEmail(String email);
// → SELECT COUNT(*) > 0 FROM users WHERE email = ?

// Delete queries
void deleteByStatus(String status);
// → DELETE FROM users WHERE status = ?
```

---

## 🔍 Supported Keywords

| Keyword | SQL |
|---------|-----|
| `And` | `WHERE ... AND ...` |
| `Or` | `WHERE ... OR ...` |
| `Not` | `WHERE NOT ...` |
| `IsNull` / `Null` | `WHERE ... IS NULL` |
| `IsNotNull` / `NotNull` | `WHERE ... IS NOT NULL` |
| `Like` | `WHERE ... LIKE ?` |
| `NotLike` | `WHERE ... NOT LIKE ?` |
| `StartingWith` | `WHERE ... LIKE ?%` |
| `EndingWith` | `WHERE ... LIKE %?` |
| `Containing` | `WHERE ... LIKE %?%` |
| `Before` | `WHERE date < ?` |
| `After` | `WHERE date > ?` |
| `Between` | `WHERE date BETWEEN ? AND ?` |
| `In` | `WHERE id IN (...)` |
| `NotIn` | `WHERE id NOT IN (...)` |
| `True` | `WHERE field = true` |
| `False` | `WHERE field = false` |
| `IgnoreCase` | `WHERE LOWER(field) = LOWER(?)` |
| `OrderBy[Field]Asc/Desc` | `ORDER BY field ASC/DESC` |

---

## 💻 Your Task Repository

```java
package com.example.taskapi.repository;

import com.example.taskapi.entity.Task;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;

@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    
    // Simple derived queries
    List<Task> findByStatus(String status);
    List<Task> findByUserId(Long userId);
    List<Task> findByUserIdAndStatus(Long userId, String status);
    boolean existsByTitleAndUserId(String title, Long userId);
    long countByStatus(String status);
    
    // Ordered
    List<Task> findByStatusOrderByCreatedAtDesc(String status);
    
    // With pagination
    Page<Task> findByUserId(Long userId, Pageable pageable);
    Page<Task> findByUserIdAndStatus(Long userId, String status, Pageable pageable);
    
    // Date range
    List<Task> findByDueDateBetween(LocalDateTime start, LocalDateTime end);
    List<Task> findByDueDateBeforeAndStatusNot(LocalDateTime date, String status);
}
```

---

## 💻 Custom JPQL Queries with @Query

When derived queries aren't enough:

```java
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    
    // JPQL — uses entity/field names, not table/column names
    @Query("SELECT t FROM Task t WHERE t.user.id = :userId AND t.status != 'DONE'")
    List<Task> findActiveTasksByUser(@Param("userId") Long userId);
    
    // JPQL with JOIN FETCH (solves N+1)
    @Query("SELECT t FROM Task t JOIN FETCH t.user WHERE t.id = :id")
    Optional<Task> findByIdWithUser(@Param("id") Long id);
    
    // JPQL projection — select specific fields
    @Query("SELECT new com.example.dto.TaskSummary(t.id, t.title, t.status) FROM Task t WHERE t.user.id = :userId")
    List<TaskSummary> findSummaryByUser(@Param("userId") Long userId);
    
    // JPQL with aggregation
    @Query("SELECT COUNT(t) FROM Task t WHERE t.user.id = :userId AND t.status = :status")
    long countByUserAndStatus(@Param("userId") Long userId, @Param("status") String status);
    
    // Native SQL — when JPQL isn't enough
    @Query(value = "SELECT * FROM tasks WHERE user_id = :userId AND created_at > NOW() - INTERVAL '7 days'",
           nativeQuery = true)
    List<Task> findRecentTasksByUser(@Param("userId") Long userId);
    
    // Native SQL for pagination
    @Query(value = "SELECT * FROM tasks WHERE user_id = :userId",
           countQuery = "SELECT COUNT(*) FROM tasks WHERE user_id = :userId",
           nativeQuery = true)
    Page<Task> findByUserIdNative(@Param("userId") Long userId, Pageable pageable);
}
```

---

## 💻 Modifying Queries

```java
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    
    // Update with JPQL
    @Modifying
    @Transactional
    @Query("UPDATE Task t SET t.status = :status WHERE t.user.id = :userId AND t.status = 'TODO'")
    int updateAllTodoToStatus(@Param("userId") Long userId, @Param("status") String status);
    
    // Bulk delete
    @Modifying
    @Transactional
    @Query("DELETE FROM Task t WHERE t.status = 'DONE' AND t.user.id = :userId")
    int deleteCompletedTasksForUser(@Param("userId") Long userId);
}
```

`@Modifying` is required for UPDATE/DELETE. `@Transactional` ensures the modification is committed.

---

## 💻 Pagination and Sorting

```java
// Pagination
PageRequest pageRequest = PageRequest.of(0, 20); // page 0, 20 items per page
Page<Task> taskPage = taskRepository.findByUserId(userId, pageRequest);

// Pagination with sorting
PageRequest sortedPageRequest = PageRequest.of(0, 20, Sort.by("createdAt").descending());
Page<Task> sorted = taskRepository.findByUserId(userId, sortedPageRequest);

// Using the Page<T> object
System.out.println(taskPage.getContent());     // List<Task>
System.out.println(taskPage.getTotalElements()); // Total count in DB
System.out.println(taskPage.getTotalPages());    // Total pages
System.out.println(taskPage.getNumber());        // Current page number
System.out.println(taskPage.hasNext());          // More pages?
System.out.println(taskPage.hasPrevious());      // Previous pages?
```

### In Controller

```java
@GetMapping
public ResponseEntity<Page<TaskResponse>> getTasks(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {
    
    Sort.Direction sortDirection = direction.equalsIgnoreCase("asc") 
        ? Sort.Direction.ASC 
        : Sort.Direction.DESC;
    
    PageRequest pageRequest = PageRequest.of(page, size, Sort.by(sortDirection, sortBy));
    Page<Task> taskPage = taskRepository.findByUserId(getCurrentUserId(), pageRequest);
    
    Page<TaskResponse> responsePage = taskPage.map(this::mapToResponse);
    return ResponseEntity.ok(responsePage);
}
```

---

## 🔍 JPQL vs Native SQL — When to Use Each

| Use JPQL when | Use Native SQL when |
|---------------|---------------------|
| Standard queries | DB-specific functions (PostgreSQL `ARRAY`, `JSON`) |
| Portability matters | Complex joins with CTEs |
| Working with entity relationships | Full-text search (`tsvector`, `plainto_tsquery`) |
| Simple aggregations | Window functions, lateral joins |

---

## 💥 Break It Exercise

This repository method won't work. Why?

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<User> findByName(String name);          // Works
    List<User> findByEmailAndActive(String email, boolean active); // Works
    List<User> findByTasksStatus(String status); // ??? Works?
}
```

<details>
<summary>Answer</summary>

`findByTasksStatus(String status)` navigates `User.tasks` (a collection) and then `.status` on each element.

Spring Data JPA **can** handle this — it generates:
```sql
SELECT DISTINCT u.* FROM users u
JOIN tasks t ON t.user_id = u.id
WHERE t.status = ?
```

So it DOES work, but it returns users who HAVE at least one task with that status. The user may have other tasks too.

The more subtle issue: if a user has 5 tasks with `status='TODO'` and 2 with `status='DONE'`, querying `findByTasksStatus("TODO")` returns the user once (DISTINCT), but their full task list (via lazy loading) still has all 7 tasks.

**This often confuses people**: filtering users by task status doesn't filter the tasks returned in the user's collection.

</details>

---

## 🎯 Practice Exercise

Build a complete repository for the Task API:

```java
public interface TaskRepository extends JpaRepository<Task, Long> {
    // Write these:
    
    // 1. Find all tasks by user ID ordered by createdAt descending
    
    // 2. Find all tasks by user ID and status
    
    // 3. Check if a task with the given title already exists for a user
    
    // 4. Count tasks by status for a user
    
    // 5. Find tasks due before a date that are NOT done
    
    // 6. Custom JPQL: Find all tasks with their user's name using JOIN FETCH
    
    // 7. Paginated tasks for a user
}
```

<details>
<summary>Solution</summary>

```java
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    
    // 1.
    List<Task> findByUserIdOrderByCreatedAtDesc(Long userId);
    
    // 2.
    List<Task> findByUserIdAndStatus(Long userId, String status);
    
    // 3.
    boolean existsByTitleAndUserId(String title, Long userId);
    
    // 4.
    @Query("SELECT COUNT(t) FROM Task t WHERE t.user.id = :userId AND t.status = :status")
    long countByUserIdAndStatus(@Param("userId") Long userId, @Param("status") String status);
    
    // 5.
    List<Task> findByDueDateBeforeAndStatusNot(LocalDateTime date, String status);
    
    // 6.
    @Query("SELECT t FROM Task t JOIN FETCH t.user WHERE t.status != 'DONE'")
    List<Task> findAllActiveWithUser();
    
    // 7.
    Page<Task> findByUserId(Long userId, Pageable pageable);
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain what `JpaRepository` provides out of the box?
- [ ] Can you write 5 derived query methods?
- [ ] Can you write a custom JPQL query with `@Query`?
- [ ] Can you use pagination with `PageRequest` and `Page<T>`?
- [ ] Can you write a `@Modifying` query?
- [ ] Can you explain when to use native SQL vs JPQL?

---

*Next: [10-01 — ACID and Transactions](../10-transactions/01-acid-and-transactions.md)*
