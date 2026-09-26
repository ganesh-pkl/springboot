# Lesson 04-01 — Controllers & Request Mapping

## 🎯 Learning Objective

Build Spring Boot REST controllers. Map your Express router + handler knowledge to Spring Boot's controller model. Understand exactly what happens when an HTTP request hits your app.

---

## 🤔 Think Before Reading

In Express, you write:

```javascript
const router = express.Router();

router.get('/users/:id', async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.status(200).json(user);
});

router.post('/users', async (req, res) => {
  const user = await userService.create(req.body);
  res.status(201).json(user);
});
```

**Question**: In Spring Boot, where does the "routing table" live? Is it central (like Express `app.use()`) or distributed across classes?

---

## 🔄 Express vs Spring Boot Routing

| Express | Spring Boot |
|---------|------------|
| `express.Router()` | `@RestController` class |
| `router.get('/path', handler)` | `@GetMapping("/path")` method |
| `router.post('/path', handler)` | `@PostMapping("/path")` method |
| `app.use('/users', userRouter)` | `@RequestMapping("/users")` on controller class |
| `req.params.id` | `@PathVariable Long id` |
| `req.query.name` | `@RequestParam String name` |
| `req.body` | `@RequestBody UserRequest request` |
| `res.status(200).json(user)` | `ResponseEntity.ok(user)` |

---

## 💻 Your First Controller

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import java.util.List;

@RestController                   // Mark as controller + serialize responses to JSON
@RequestMapping("/api/users")     // Base path for all methods in this controller
public class UserController {
    
    // GET /api/users
    @GetMapping
    public ResponseEntity<List<UserResponse>> getAllUsers() {
        List<UserResponse> users = List.of(
            new UserResponse(1L, "Alice", "alice@example.com"),
            new UserResponse(2L, "Bob", "bob@example.com")
        );
        return ResponseEntity.ok(users);
    }
    
    // GET /api/users/42
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        UserResponse user = new UserResponse(id, "Alice", "alice@example.com");
        return ResponseEntity.ok(user);
    }
    
    // POST /api/users
    @PostMapping
    public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest request) {
        UserResponse created = new UserResponse(1L, request.name(), request.email());
        return ResponseEntity.status(201).body(created);
    }
    
    // PUT /api/users/42
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @RequestBody UpdateUserRequest request) {
        UserResponse updated = new UserResponse(id, request.name(), request.email());
        return ResponseEntity.ok(updated);
    }
    
    // DELETE /api/users/42
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        // Delete logic
        return ResponseEntity.noContent().build(); // 204 No Content
    }
}
```

---

## 🔍 What @RestController Actually Does

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller      // ← marks as MVC controller (component scan picks it up)
@ResponseBody    // ← ALL methods return JSON/XML instead of view names
public @interface RestController { ... }
```

`@ResponseBody` tells Spring: "Don't look for a template file to render. Serialize the return value directly to the HTTP response body using Jackson."

Without `@ResponseBody`:
- Spring MVC would try to find a view template named after the return value
- `return "users"` → tries to render `users.html`

With `@ResponseBody` (or `@RestController`):
- Spring serializes the return value as JSON
- `return user` → `{"id": 1, "name": "Alice", ...}`

---

## 🔍 HTTP Request Lifecycle in Spring Boot

When `GET /api/users/42` arrives:

```
1. Tomcat receives HTTP request

2. DispatcherServlet (Spring's "front controller") intercepts

3. HandlerMapping finds the right method:
   - Which class has @RequestMapping("/api/users")?
   - Within that class, which method has @GetMapping("/{id}")?
   - Found: UserController.getUserById()

4. HandlerAdapter calls the method:
   - Resolves @PathVariable: "42" → Long 42
   - Resolves @RequestBody: JSON → Java object
   - Resolves @RequestParam: query string → Java values

5. Method executes, returns ResponseEntity<UserResponse>

6. MessageConverter (Jackson) serializes UserResponse → JSON string

7. HTTP response is sent back with status 200 and JSON body
```

---

## 💻 Request Parameters — All Forms

### @PathVariable — URL segment

```java
// GET /api/tasks/123
@GetMapping("/{taskId}")
public ResponseEntity<TaskResponse> getTask(@PathVariable Long taskId) { ... }

// Multiple path variables
// GET /api/users/42/tasks/99
@GetMapping("/users/{userId}/tasks/{taskId}")
public ResponseEntity<TaskResponse> getUserTask(
        @PathVariable Long userId,
        @PathVariable Long taskId) { ... }

// When variable name differs from parameter name
@GetMapping("/{task-id}")
public ResponseEntity<TaskResponse> getTask(@PathVariable("task-id") Long taskId) { ... }
```

### @RequestParam — Query string

```java
// GET /api/tasks?status=OPEN&priority=HIGH&page=0&size=20
@GetMapping
public ResponseEntity<Page<TaskResponse>> getTasks(
        @RequestParam(required = false) String status,         // Optional
        @RequestParam(required = false) String priority,       // Optional
        @RequestParam(defaultValue = "0") int page,            // Default 0
        @RequestParam(defaultValue = "20") int size) { ... }   // Default 20

// GET /api/tasks?tags=java&tags=spring&tags=backend
@GetMapping
public ResponseEntity<List<TaskResponse>> getByTags(
        @RequestParam List<String> tags) { ... } // Multiple values → List
```

### @RequestBody — Request body (JSON)

```java
// POST /api/users
// Body: {"name": "Alice", "email": "alice@example.com", "password": "secret"}
@PostMapping
public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest request) {
    // Jackson deserializes the JSON body to CreateUserRequest
    System.out.println(request.name());  // "Alice"
    System.out.println(request.email()); // "alice@example.com"
    // ...
}
```

### @RequestHeader — HTTP headers

```java
@GetMapping("/profile")
public ResponseEntity<UserResponse> getProfile(
        @RequestHeader("Authorization") String authHeader,
        @RequestHeader(value = "X-Request-ID", required = false) String requestId) { ... }
```

---

## 💻 ResponseEntity — Full HTTP Response Control

```java
// 200 OK with body
return ResponseEntity.ok(user);
return ResponseEntity.status(200).body(user);

// 201 Created with Location header and body
URI location = URI.create("/api/users/" + created.id());
return ResponseEntity.created(location).body(created);

// 204 No Content (for DELETE)
return ResponseEntity.noContent().build();

// 404 Not Found
return ResponseEntity.notFound().build();

// 400 Bad Request with body
return ResponseEntity.badRequest().body(new ErrorResponse("Invalid input"));

// Custom headers
return ResponseEntity
    .status(200)
    .header("X-Custom-Header", "value")
    .header("Cache-Control", "no-cache")
    .body(user);
```

---

## 🏗️ DTOs for Request and Response

```java
// Request DTO (what client sends)
public record CreateUserRequest(
    String name,
    String email,
    String password
) {}

// Response DTO (what we send back — NEVER expose entity directly!)
public record UserResponse(
    Long id,
    String name,
    String email
    // Note: NO password in response!
) {}

// Update DTO
public record UpdateUserRequest(
    String name,
    String email
) {}
```

**Why not return the Entity directly?**

```java
// ❌ DON'T DO THIS
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
    // Problems:
    // - Exposes password field!
    // - Triggers lazy loading issues
    // - Couples API to database schema
    // - JSON serialization circular references with relationships
}

// ✅ DO THIS
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).orElseThrow();
    return new UserResponse(user.getId(), user.getName(), user.getEmail());
}
```

---

## 🔍 Mapping Shorthand Annotations

All of these are shorthand for `@RequestMapping`:

```java
@GetMapping("/path")    // = @RequestMapping(path="/path", method=RequestMethod.GET)
@PostMapping("/path")   // = @RequestMapping(path="/path", method=RequestMethod.POST)
@PutMapping("/path")    // = @RequestMapping(path="/path", method=RequestMethod.PUT)
@PatchMapping("/path")  // = @RequestMapping(path="/path", method=RequestMethod.PATCH)
@DeleteMapping("/path") // = @RequestMapping(path="/path", method=RequestMethod.DELETE)
```

---

## 🚀 First Real Project: Task Management API

Let's start building. We'll add more features in each module.

### Project Setup

Go to `start.spring.io` and create:
- Group: `com.example`
- Artifact: `task-api`
- Java 21
- Dependencies: Spring Web, Spring Boot DevTools

### Task Controller (In-Memory Storage for Now)

```java
package com.example.taskapi.controller;

import com.example.taskapi.dto.CreateTaskRequest;
import com.example.taskapi.dto.TaskResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    // Temporary in-memory storage
    private final List<TaskResponse> tasks = new ArrayList<>();
    private final AtomicLong idCounter = new AtomicLong(1);
    
    @GetMapping
    public ResponseEntity<List<TaskResponse>> getAllTasks() {
        return ResponseEntity.ok(tasks);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<TaskResponse> getTaskById(@PathVariable Long id) {
        return tasks.stream()
            .filter(t -> t.id().equals(id))
            .findFirst()
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<TaskResponse> createTask(@RequestBody CreateTaskRequest request) {
        Long newId = idCounter.getAndIncrement();
        TaskResponse task = new TaskResponse(
            newId,
            request.title(),
            request.description(),
            "TODO",
            LocalDateTime.now()
        );
        tasks.add(task);
        
        URI location = URI.create("/api/tasks/" + newId);
        return ResponseEntity.created(location).body(task);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id) {
        boolean removed = tasks.removeIf(t -> t.id().equals(id));
        if (!removed) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.noContent().build();
    }
}
```

### DTOs

```java
package com.example.taskapi.dto;

import java.time.LocalDateTime;

// Request
public record CreateTaskRequest(
    String title,
    String description
) {}

// Response
public record TaskResponse(
    Long id,
    String title,
    String description,
    String status,
    LocalDateTime createdAt
) {}
```

### Test with curl

```bash
# Create a task
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Spring Boot", "description": "Become a Spring expert"}'

# Get all tasks
curl http://localhost:8080/api/tasks

# Get one task
curl http://localhost:8080/api/tasks/1

# Delete a task
curl -X DELETE http://localhost:8080/api/tasks/1
```

---

## 💥 Break It Exercise

The code below compiles but fails at runtime. Why?

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/profile/{username}")
    public ResponseEntity<UserResponse> getUserByUsername(@PathVariable String username) {
        // ...
    }
    
    @GetMapping("/{userId}/orders")
    public ResponseEntity<List<OrderResponse>> getUserOrders(@PathVariable String userId) {
        // ...
    }
}
```

Someone calls `GET /api/users/profile/alice` — what happens? Is `username` = "profile" or "alice"?

<details>
<summary>Answer</summary>

This is **ambiguous routing**. Spring MVC must decide between:
- `@GetMapping("/profile/{username}")` → matches if literal "profile" is at position 2
- `@GetMapping("/{userId}/orders")` → matches if "profile" is the userId and next segment is "orders"

For `GET /api/users/profile/alice`:
- `"/profile/{username}"` matches: username = "alice" ✅
- `"/{userId}/orders"` does NOT match: userId would be "profile", next must be "orders" but it's "alice" ✗

So Spring correctly routes to `getUserByUsername("alice")`.

BUT: `GET /api/users/123/orders` — both patterns could potentially match if Spring isn't careful. Spring uses **specificity rules**: `/profile/{username}` has a literal "profile" which is more specific than the wildcard `/{userId}`.

The actual problem to watch for: patterns like `/{id}` and `/{username}` at the SAME level will conflict because Spring can't tell which is which.

Fix: Use distinct patterns or add to path:
```java
@GetMapping("/by-username/{username}") // More specific
@GetMapping("/{userId}/orders")
```

</details>

---

## 🎯 Practice Exercise — Extend the Task API

Add to the task API:

1. `PATCH /api/tasks/{id}/status` — update only the status
   - Request body: `{"status": "IN_PROGRESS"}` or `{"status": "DONE"}`
   - Return the updated task

2. `GET /api/tasks?status=TODO` — filter tasks by status
   - Use `@RequestParam(required = false) String status`
   - If status is null, return all tasks

3. `GET /api/tasks/count` — return the count of tasks
   - Response: `{"count": 5}`

Test all endpoints with curl.

<details>
<summary>Solution Hints</summary>

```java
// Status update
@PatchMapping("/{id}/status")
public ResponseEntity<TaskResponse> updateStatus(
        @PathVariable Long id,
        @RequestBody UpdateStatusRequest request) {
    // Find task, update status, return updated
}

// Filter by status
@GetMapping
public ResponseEntity<List<TaskResponse>> getTasks(
        @RequestParam(required = false) String status) {
    List<TaskResponse> result = tasks;
    if (status != null) {
        result = tasks.stream()
            .filter(t -> t.status().equals(status))
            .collect(Collectors.toList());
    }
    return ResponseEntity.ok(result);
}

// Count
@GetMapping("/count")
public ResponseEntity<Map<String, Long>> getCount() {
    return ResponseEntity.ok(Map.of("count", (long) tasks.size()));
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you write a controller with GET, POST, PUT, DELETE methods?
- [ ] Can you use `@PathVariable`, `@RequestParam`, `@RequestBody`?
- [ ] Can you return `ResponseEntity` with different status codes?
- [ ] Can you create request and response DTOs as records?
- [ ] Did you build and test the Task API?
- [ ] Can you explain what `@RestController` vs `@Controller` does?

---

*Next: [05-01 — Layered Architecture](../05-architecture/01-layered-architecture.md)*

(We'll add the Service and Repository layers to our Task API next, then add PostgreSQL in Level 5)
