# Lesson 08-01 — JPA and Hibernate Deep Dive

## 🎯 Learning Objective

Understand what JPA IS vs what Hibernate IS. Understand the persistence context, dirty checking, entity lifecycle. Build a foundation that prevents the most common JPA bugs.

---

## 🤔 Think Before Reading

In Node.js, you might use Prisma, TypeORM, or Sequelize. With Prisma:

```javascript
const user = await prisma.user.create({
  data: { name: "Alice", email: "alice@example.com" }
});

const foundUser = await prisma.user.findUnique({
  where: { id: 1 }
});
```

**Question**: In Spring Boot, when you do:
```java
User user = userRepository.findById(1L).orElseThrow();
user.setName("New Name"); // Just changing an object
```

No explicit `save()` call. Will this update the database? How? Why?

---

## 🔍 JPA vs Hibernate — Critical Distinction

These are NOT the same thing:

| | JPA | Hibernate |
|--|-----|-----------|
| What is it? | **Specification** (a standard) | **Implementation** (actual code) |
| Who defines it? | Jakarta (formerly Java EE) | Red Hat / JBoss |
| Is it code? | No — it's a contract (interfaces, annotations) | Yes — it's the actual ORM engine |

**Analogy**:
- JPA = JDBC's interface standard
- Hibernate = PostgreSQL driver (an implementation)

```java
// You write code against JPA interfaces:
import jakarta.persistence.EntityManager;  // JPA interface
import jakarta.persistence.Entity;         // JPA annotation

// Hibernate provides the implementation:
// EntityManager → SessionImpl (Hibernate's implementation)
```

In Spring Boot, when you add `spring-boot-starter-data-jpa`, you get Hibernate as the default JPA implementation. You write JPA code; Hibernate does the work.

---

## 🧠 The Core Concept: Persistence Context

This is the most important concept in JPA. If you understand this, everything else makes sense.

The **Persistence Context** is a first-level cache. It's a temporary "workspace" that tracks all the entities you've touched in the current operation.

```
Persistence Context
┌─────────────────────────────────────────┐
│  Entity Cache (first-level cache)        │
│  ┌─────────────────────────────────┐    │
│  │ ID 1 → User{name="Alice"} ✓     │    │
│  │ ID 2 → User{name="Bob"}   ✓     │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Change Tracker                         │
│  ┌─────────────────────────────────┐    │
│  │ User{id=1} → "Alice" (original) │    │
│  │ User{id=2} → "Bob" (original)   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

When you load an entity:
1. JPA checks the persistence context first
2. If found → returns cached version (no database hit)
3. If not found → hits the database, stores result in context

---

## 🔍 Dirty Checking — How Auto-Updates Work

This is the answer to the "Think Before Reading" question.

```java
@Transactional
public void updateUserName(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow(); // Loaded into persistence context
    user.setName(newName); // ← Changed in memory. No save() called!
    // Method ends → transaction commits → JPA compares "current" vs "original"
    // Detects: name changed from "Alice" to "New Name"
    // Automatically generates: UPDATE users SET name = 'New Name' WHERE id = 1
}
```

**Dirty checking** is JPA comparing the current state of entities to their "snapshot" taken when they were loaded. If anything changed, JPA generates UPDATE statements automatically on transaction commit.

**This ONLY works within a transaction**. Outside a transaction, there's no persistence context, so no dirty checking.

```java
// ❌ No dirty checking outside @Transactional
public void badUpdate(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName); // Does NOTHING to the database!
    // No transaction → no persistence context → no dirty checking
}

// ✅ Dirty checking works
@Transactional
public void goodUpdate(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName); // Will be flushed to DB on commit
}
```

---

## 💻 Entity Lifecycle States

JPA entities have four states:

```
New/Transient → Managed → Detached → Removed
```

### 1. New/Transient
Entity is created but not associated with any persistence context:

```java
User user = new User("Alice", "alice@example.com");
// state: Transient (no ID, not tracked)
```

### 2. Managed
Entity is in the persistence context — changes are tracked:

```java
@Transactional
public void example() {
    User user = userRepository.findById(1L).orElseThrow();
    // state: Managed (tracked by persistence context)
    
    user.setName("New Name"); // Change tracked!
    // At commit: UPDATE users SET name = 'New Name' WHERE id = 1
}
```

```java
@Transactional
public void example2() {
    User user = new User("Bob", "bob@example.com");
    userRepository.save(user); // save() transitions from Transient → Managed
    // state: Managed (has ID assigned)
    
    user.setEmail("new@example.com"); // Also tracked!
}
```

### 3. Detached
Entity was managed but the persistence context closed (transaction ended):

```java
// After @Transactional method returns, persistence context closes
// All entities become Detached

User user = userService.findById(1L); // Gets detached entity
user.setName("Changed"); // Change NOT tracked — no persistence context!
```

### 4. Removed

```java
@Transactional
public void deleteUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    userRepository.delete(user);
    // state: Removed — DELETE SQL will run on commit
}
```

---

## 💻 Creating Your First Entity

```java
package com.example.taskapi.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity                              // Marks this as a JPA entity
@Table(name = "tasks")               // Maps to "tasks" table (default: class name)
public class Task {
    
    @Id                              // Primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY) // AUTO_INCREMENT / SERIAL
    private Long id;
    
    @Column(name = "title", nullable = false, length = 200)
    private String title;
    
    @Column(name = "description", columnDefinition = "TEXT")
    private String description;
    
    @Column(name = "status", nullable = false, length = 20)
    private String status;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // JPA REQUIRES a no-arg constructor (even if you have others)
    // Make it protected so your code can't accidentally call it
    protected Task() {}
    
    // Business constructor
    public Task(String title, String description) {
        this.title = title;
        this.description = description;
        this.status = "TODO";
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }
    
    // Getters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    
    // Setters (only for mutable fields)
    public void setStatus(String status) {
        this.status = status;
        this.updatedAt = LocalDateTime.now();
    }
    
    public void setTitle(String title) {
        this.title = title;
        this.updatedAt = LocalDateTime.now();
    }
}
```

### @GeneratedValue Strategies

```java
// IDENTITY — use DB auto-increment (PostgreSQL SERIAL, MySQL AUTO_INCREMENT)
@GeneratedValue(strategy = GenerationType.IDENTITY)

// SEQUENCE — use DB sequence (more efficient for batch inserts)
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "task_seq")
@SequenceGenerator(name = "task_seq", sequenceName = "task_sequence", allocationSize = 1)

// UUID
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

---

## 🔍 @Column — Mapping Fields

```java
@Column(
    name = "email_address",    // Column name in DB (default: field name)
    nullable = false,          // NOT NULL constraint
    unique = true,             // UNIQUE constraint
    length = 255,              // VARCHAR(255)
    updatable = false,         // Exclude from UPDATE statements
    insertable = true,         // Include in INSERT statements
    columnDefinition = "TEXT"  // Raw SQL type (overrides length, etc.)
)
private String email;
```

If you don't add `@Column`, JPA uses the field name as the column name and applies defaults.

---

## 🏗️ @PrePersist and @PreUpdate

Instead of setting timestamps in the constructor:

```java
@Entity
public class Task {
    
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    @PrePersist // Called before INSERT
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate // Called before UPDATE
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

Or use Spring Data's `@CreatedDate` / `@LastModifiedDate` with `@EnableJpaAuditing`.

---

## 💥 Break It Exercise

What SQL does JPA generate for this?

```java
@Transactional
public User findAndReturn(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    return user; // Return the managed entity
}

// Caller:
User user = userService.findAndReturn(1L);
user.setName("Hacked!"); // After @Transactional method returned
```

Does `user.setName("Hacked!")` update the database?

<details>
<summary>Answer</summary>

**No** — no UPDATE runs.

When `findAndReturn()` returns, the `@Transactional` boundary ends, the transaction commits, and the persistence context closes. The `user` object becomes **detached**.

After the method returns, `user` is a detached entity. Setting its name is just modifying a Java object — no persistence context is tracking it.

To actually save changes after the method returns, you'd need to:
1. Call `userRepository.save(user)` explicitly
2. Or do the modification within the `@Transactional` method

**This is one of the most common JPA bugs**: assuming that changes to objects persist when there's no active transaction.

</details>

---

## 🎯 Practice Exercise

Create a `User` entity with:

1. `id` — Long, auto-generated with IDENTITY
2. `name` — String, not null, max 100 chars
3. `email` — String, not null, unique
4. `passwordHash` — String, not null (column name: `password_hash`)
5. `role` — String, not null, default "USER" (length 20)
6. `active` — boolean, not null, default true
7. `createdAt` — LocalDateTime, not null, not updatable
8. `updatedAt` — LocalDateTime, updated automatically

Use `@PrePersist` and `@PreUpdate` for timestamps.
Use a protected no-arg constructor and a public constructor for creation.

<details>
<summary>Solution</summary>

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    private String passwordHash;
    
    @Column(name = "role", nullable = false, length = 20)
    private String role;
    
    @Column(name = "active", nullable = false)
    private boolean active;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    protected User() {}
    
    public User(String name, String email, String passwordHash) {
        this.name = name;
        this.email = email;
        this.passwordHash = passwordHash;
        this.role = "USER";
        this.active = true;
    }
    
    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPasswordHash() { return passwordHash; }
    public String getRole() { return role; }
    public boolean isActive() { return active; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    
    // Setters (only mutable fields)
    public void setName(String name) { this.name = name; }
    public void setRole(String role) { this.role = role; }
    public void setActive(boolean active) { this.active = active; }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain the difference between JPA and Hibernate?
- [ ] Can you explain what the persistence context is?
- [ ] Can you explain dirty checking (how does `setName()` update the DB)?
- [ ] Can you name the 4 entity lifecycle states?
- [ ] Can you write an entity with all JPA annotations?
- [ ] Do you understand why entity changes outside `@Transactional` don't persist?

---

*Next: [08-02 — Entity Relationships](./02-entities-and-mapping.md)*
