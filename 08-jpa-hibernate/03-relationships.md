# Lesson 08-03 — JPA Relationships

## 🎯 Learning Objective

Implement entity relationships correctly. Understand the difference between the owning side and the inverse side. Understand why incorrectly mapped relationships are a source of subtle bugs.

---

## 🤔 Think Before Reading

In SQL, you model a one-to-many relationship with a foreign key:

```sql
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name VARCHAR);
CREATE TABLE tasks (id BIGSERIAL PRIMARY KEY, title VARCHAR, user_id BIGINT REFERENCES users(id));
```

**Question**: In JPA, how do you represent this? If `Task` has a `user_id`, which class holds the relationship — `Task` or `User`? Both? What does it mean to "own" a relationship?

---

## 🏗️ The Four Relationship Types

| Relationship | Example | Annotation |
|---|---|---|
| Many-to-One | Many tasks → One user | `@ManyToOne` |
| One-to-Many | One user → Many tasks | `@OneToMany` |
| One-to-One | One user → One profile | `@OneToOne` |
| Many-to-Many | Many tasks ↔ Many tags | `@ManyToMany` |

---

## 🔍 The Owning Side — Critical Concept

JPA uses the concept of **owning side** vs **inverse side**.

**Rule**: The owning side is where the foreign key column lives in the database.

In SQL:
```sql
tasks.user_id → users.id  ← Foreign key is in tasks table
```

Therefore, `Task` is the **owning side** of the User-Task relationship.

```java
// OWNING SIDE — Task has the foreign key
@Entity
public class Task {
    @ManyToOne
    @JoinColumn(name = "user_id")  // ← This is the FK column
    private User user;
}

// INVERSE SIDE — User is "mapped by" Task
@Entity
public class User {
    @OneToMany(mappedBy = "user")  // ← "user" = field name in Task class
    private List<Task> tasks;
}
```

**Only the owning side** matters for persistence. If you add a task to `user.getTasks()` but don't set `task.setUser(user)`, **the database won't have the foreign key**.

---

## 💻 Many-to-One (@ManyToOne)

This is the most common relationship. A task belongs to a user.

```java
@Entity
@Table(name = "tasks")
public class Task {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private String status;
    
    @ManyToOne(fetch = FetchType.LAZY)       // Lazy loading (recommended)
    @JoinColumn(name = "user_id", nullable = false)  // FK column
    private User user;
    
    protected Task() {}
    
    public Task(String title, User user) {
        this.title = title;
        this.user = user;
        this.status = "TODO";
    }
    
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

Generated SQL:
```sql
CREATE TABLE tasks (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255),
    status VARCHAR(255),
    user_id BIGINT NOT NULL REFERENCES users(id)
);
```

---

## 💻 One-to-Many (@OneToMany)

The inverse side. A user has many tasks.

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @OneToMany(
        mappedBy = "user",          // Field name in Task class
        cascade = CascadeType.ALL,  // Operations cascade to tasks
        fetch = FetchType.LAZY,     // Don't load tasks unless asked
        orphanRemoval = true        // Delete tasks if removed from list
    )
    private List<Task> tasks = new ArrayList<>();
    
    // Helper methods to maintain BOTH sides of the relationship
    public void addTask(Task task) {
        tasks.add(task);
        task.setUser(this); // ← Must set owning side!
    }
    
    public void removeTask(Task task) {
        tasks.remove(task);
        task.setUser(null);
    }
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public List<Task> getTasks() { return tasks; }
}
```

### ⚠️ Common Mistake

```java
// ❌ This does NOT persist the relationship
user.getTasks().add(task); // Only sets inverse side — mappedBy

// ✅ This persists the relationship  
task.setUser(user);         // Sets owning side — has the FK column

// ✅ Or use the helper method
user.addTask(task);         // Sets BOTH sides
```

---

## 🔍 Cascade Types

Cascade means: "when I do X to the parent, do X to the children too."

| CascadeType | Meaning |
|------------|---------|
| `PERSIST` | Saving parent also saves children |
| `MERGE` | Merging parent also merges children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children |
| `DETACH` | Detaching parent also detaches children |
| `ALL` | All of the above |

```java
// Without cascade:
User user = new User("Alice", "alice@example.com");
Task task = new Task("Learn JPA", user);

userRepository.save(user); // Saves user
taskRepository.save(task); // Must separately save task

// With CascadeType.PERSIST:
User user = new User("Alice", "alice@example.com");
Task task = new Task("Learn JPA", user);
user.addTask(task);

userRepository.save(user); // Saves user AND task (cascade)
```

**Be careful with `CascadeType.REMOVE`** — deleting a user also deletes all their tasks. This might not always be what you want (e.g., audit history).

---

## 🔍 orphanRemoval

When an entity is removed from a collection, should it be deleted from the database?

```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Task> tasks;

// Usage:
@Transactional
public void removeTaskFromUser(Long userId, Long taskId) {
    User user = userRepository.findById(userId).orElseThrow();
    user.getTasks().removeIf(t -> t.getId().equals(taskId));
    // With orphanRemoval = true: JPA deletes the task from DB
    // Without orphanRemoval: task is just removed from the List, still in DB
}
```

---

## 💻 One-to-One (@OneToOne)

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id", referencedColumnName = "id")
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String bio;
    private String avatarUrl;
    
    @OneToOne(mappedBy = "profile")  // Inverse side
    private User user;
}
```

---

## 💻 Many-to-Many (@ManyToMany)

Tags and tasks: one task can have many tags, one tag can belong to many tasks.

```java
@Entity
public class Task {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "task_tags",                        // Join table name
        joinColumns = @JoinColumn(name = "task_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();
    
    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getTasks().add(this);
    }
    
    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getTasks().remove(this);
    }
}

@Entity
public class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany(mappedBy = "tags")  // Inverse side
    private Set<Task> tasks = new HashSet<>();
    
    // Getters
    public Set<Task> getTasks() { return tasks; }
}
```

Generated SQL:
```sql
CREATE TABLE task_tags (
    task_id BIGINT REFERENCES tasks(id),
    tag_id BIGINT REFERENCES tags(id),
    PRIMARY KEY (task_id, tag_id)
);
```

### ⚠️ @ManyToMany pitfall

Avoid `CascadeType.REMOVE` on `@ManyToMany`. If you delete a Task, you don't want all Tags deleted — other tasks might use them.

---

## 💥 Break It Exercise

This code has a subtle relationship bug:

```java
@Service
@Transactional
public class TaskService {
    
    public void assignTaskToUser(Long taskId, Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        Task task = taskRepository.findById(taskId).orElseThrow();
        
        // Add task to user's list
        user.getTasks().add(task);
        
        userRepository.save(user);
    }
}
```

**Question**: After this runs, what is `task.getUser()`? Does the database have a `user_id` in the tasks table?

<details>
<summary>Answer</summary>

**Problem**: Only the inverse side was set (`user.getTasks().add(task)`). The owning side (`task.setUser(user)`) was NOT set.

Since `@OneToMany(mappedBy = "user")` means `user.tasks` is the INVERSE side, adding to it doesn't affect the FK column.

Result:
- `task.getUser()` → null (field not set)
- `tasks.user_id` in database → NULL or unchanged

Fix:
```java
user.getTasks().add(task);  // Set inverse side (for in-memory consistency)
task.setUser(user);          // Set owning side (this updates the FK column!)

// Or use the helper method:
user.addTask(task); // which does both internally
```

This is the #1 most common JPA relationship bug.

</details>

---

## 🎯 Practice Exercise

Model a blog system:

- `Post`: id, title, content, author (User)
- `Comment`: id, content, author (User), post (Post)
- `User`: id, name, email

Requirements:
1. User has many Posts (one-to-many)
2. Post has many Comments (one-to-many)
3. Each Comment belongs to a User (many-to-one)
4. Add helper methods on `Post` to add/remove comments
5. Add helper methods on `User` to add posts

Which side is the "owning" side in each relationship? Where are the FK columns?

<details>
<summary>Solution</summary>

```java
// FK analysis:
// posts.user_id → users.id        → Post is owning side for User-Post
// comments.post_id → posts.id      → Comment is owning side for Post-Comment
// comments.user_id → users.id      → Comment is owning side for User-Comment

@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Post> posts = new ArrayList<>();
    
    public void addPost(Post post) {
        posts.add(post);
        post.setAuthor(this);
    }
}

@Entity
public class Post {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String content;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User author;
    
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, fetch = FetchType.LAZY, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
    
    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }
    
    public void removeComment(Comment comment) {
        comments.remove(comment);
        comment.setPost(null);
    }
    
    public void setAuthor(User author) { this.author = author; }
}

@Entity
public class Comment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String content;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User author;
    
    public void setPost(Post post) { this.post = post; }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain "owning side" vs "inverse side"?
- [ ] Can you implement `@ManyToOne` with `@OneToMany(mappedBy=...)`?
- [ ] Can you explain what `cascade` does?
- [ ] Can you explain what `orphanRemoval` does?
- [ ] Can you implement `@ManyToMany` with a join table?
- [ ] Can you explain why you must set BOTH sides of a relationship?

---

*Next: [08-04 — Lazy vs Eager Loading](./04-lazy-vs-eager.md)*
