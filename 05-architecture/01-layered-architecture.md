# Lesson 05-01 — Layered Architecture

## 🎯 Learning Objective

Understand WHY Spring Boot applications use a layered architecture, where each type of code belongs, and how this maps to your Express/Node.js experience.

---

## 🤔 Think Before Reading

In your Task API, everything is currently in the controller:

```java
@PostMapping
public ResponseEntity<TaskResponse> createTask(@RequestBody CreateTaskRequest request) {
    Long newId = idCounter.getAndIncrement();
    TaskResponse task = new TaskResponse(newId, request.title(), ...);
    tasks.add(task);
    return ResponseEntity.created(location).body(task);
}
```

**Question**: What happens when you need to:
- Add validation (title must not be duplicate)
- Add authorization (only certain users can create tasks)
- Add email notifications after creation
- Write unit tests (how do you test without HTTP?)
- Reuse the "create task" logic from a scheduled job that doesn't use HTTP?

What would you restructure?

---

## 🏗️ The Layered Architecture

```
┌──────────────────────────────────────────┐
│           HTTP / Client                  │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           Controller Layer               │
│  Receives HTTP, validates input format,  │
│  delegates to service, returns response  │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           Service Layer                  │
│  Business logic, orchestration,          │
│  transactions, domain rules              │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           Repository Layer               │
│  Data access, database queries,          │
│  no business logic                       │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           Database                       │
└──────────────────────────────────────────┘
```

---

## 🔄 Node.js vs Spring Boot Layers

```javascript
// Node.js — you build this structure manually
routes/tasks.js          // Router (like Controller)
controllers/taskController.js  // Request handling
services/taskService.js   // Business logic
repositories/taskRepo.js  // DB access
```

```java
// Spring Boot — same structure, Spring-managed
@RestController → UserController    // HTTP handling
@Service → UserService              // Business logic
@Repository → UserRepository        // Data access
```

The **concept** is identical. Spring Boot provides annotations and dependency injection to make the wiring automatic.

---

## 🔍 What Belongs Where

### Controller Layer — ONLY these responsibilities:

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    private final TaskService taskService;
    
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }
    
    @PostMapping
    public ResponseEntity<TaskResponse> createTask(@RequestBody @Valid CreateTaskRequest request) {
        // ✅ Parse HTTP request
        // ✅ Basic input format validation (@Valid)
        // ✅ Call service
        // ✅ Map to HTTP response
        TaskResponse task = taskService.createTask(request);
        return ResponseEntity.status(201).body(task);
    }
}
```

**Controller should NOT contain:**
- Business rules
- Database queries
- Transaction management
- Domain logic ("only one task per title")

### Service Layer — Business Logic Lives Here:

```java
@Service
public class TaskService {
    
    private final TaskRepository taskRepository;
    private final UserRepository userRepository;
    private final NotificationService notificationService;
    
    public TaskService(TaskRepository taskRepository,
                       UserRepository userRepository,
                       NotificationService notificationService) {
        this.taskRepository = taskRepository;
        this.userRepository = userRepository;
        this.notificationService = notificationService;
    }
    
    @Transactional
    public TaskResponse createTask(CreateTaskRequest request) {
        // ✅ Business validation
        if (taskRepository.existsByTitle(request.title())) {
            throw new DuplicateTitleException(request.title());
        }
        
        // ✅ Domain logic
        Task task = new Task(request.title(), request.description(), "TODO");
        
        // ✅ Orchestration
        Task saved = taskRepository.save(task);
        notificationService.sendTaskCreatedNotification(saved);
        
        // ✅ Mapping to response DTO
        return mapToResponse(saved);
    }
    
    private TaskResponse mapToResponse(Task task) {
        return new TaskResponse(task.getId(), task.getTitle(), task.getDescription(), task.getStatus());
    }
}
```

**Service should NOT contain:**
- HTTP-specific code (`HttpServletRequest`, status codes)
- Jackson annotations
- View rendering

### Repository Layer — Data Access Only:

```java
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    boolean existsByTitle(String title);
    List<Task> findByStatus(String status);
    List<Task> findByUserId(Long userId);
}
```

**Repository should NOT contain:**
- Business rules
- Transaction management (let the service decide)
- HTTP concerns

---

## 📊 Dependency Flow — One Direction Only

```
Controller  →  Service  →  Repository  →  Database
    ↓              ↓
  DTOs        Domain Objects
```

**Rule**: Dependencies only flow DOWNWARD.

```java
// ✅ Allowed
class Controller { uses Service }
class Service { uses Repository }
class Service { uses other Services }

// ❌ Never do this
class Repository { uses Service }  // ← Repository should not know about services
class Service { uses Controller }  // ← Service should not know about HTTP
```

---

## 💻 Refactoring the Task API

Let's refactor the Task API to use proper layers:

### Task Entity (Domain Object)

```java
package com.example.taskapi.entity;

import java.time.LocalDateTime;

public class Task {
    private Long id;
    private String title;
    private String description;
    private String status;
    private LocalDateTime createdAt;
    
    public Task(String title, String description) {
        this.title = title;
        this.description = description;
        this.status = "TODO";
        this.createdAt = LocalDateTime.now();
    }
    
    // Getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

### Task Repository (In-Memory for Now)

```java
package com.example.taskapi.repository;

import com.example.taskapi.entity.Task;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class InMemoryTaskRepository {
    
    private final List<Task> tasks = new ArrayList<>();
    private final AtomicLong idCounter = new AtomicLong(1);
    
    public Task save(Task task) {
        if (task.getId() == null) {
            task.setId(idCounter.getAndIncrement());
        }
        tasks.removeIf(t -> t.getId().equals(task.getId()));
        tasks.add(task);
        return task;
    }
    
    public Optional<Task> findById(Long id) {
        return tasks.stream().filter(t -> t.getId().equals(id)).findFirst();
    }
    
    public List<Task> findAll() {
        return new ArrayList<>(tasks);
    }
    
    public boolean deleteById(Long id) {
        return tasks.removeIf(t -> t.getId().equals(id));
    }
    
    public boolean existsByTitle(String title) {
        return tasks.stream().anyMatch(t -> t.getTitle().equals(title));
    }
}
```

### Task Service (Business Logic)

```java
package com.example.taskapi.service;

import com.example.taskapi.dto.CreateTaskRequest;
import com.example.taskapi.dto.TaskResponse;
import com.example.taskapi.dto.UpdateStatusRequest;
import com.example.taskapi.entity.Task;
import com.example.taskapi.exception.DuplicateTitleException;
import com.example.taskapi.exception.TaskNotFoundException;
import com.example.taskapi.repository.InMemoryTaskRepository;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class TaskService {
    
    private final InMemoryTaskRepository taskRepository;
    
    public TaskService(InMemoryTaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }
    
    public List<TaskResponse> findAll() {
        return taskRepository.findAll()
            .stream()
            .map(this::mapToResponse)
            .collect(Collectors.toList());
    }
    
    public TaskResponse findById(Long id) {
        return taskRepository.findById(id)
            .map(this::mapToResponse)
            .orElseThrow(() -> new TaskNotFoundException(id));
    }
    
    public TaskResponse createTask(CreateTaskRequest request) {
        // Business rule: no duplicate titles
        if (taskRepository.existsByTitle(request.title())) {
            throw new DuplicateTitleException(request.title());
        }
        
        Task task = new Task(request.title(), request.description());
        Task saved = taskRepository.save(task);
        return mapToResponse(saved);
    }
    
    public TaskResponse updateStatus(Long id, UpdateStatusRequest request) {
        Task task = taskRepository.findById(id)
            .orElseThrow(() -> new TaskNotFoundException(id));
        
        // Business rule: can only move to valid statuses
        validateStatusTransition(task.getStatus(), request.status());
        
        task.setStatus(request.status());
        Task saved = taskRepository.save(task);
        return mapToResponse(saved);
    }
    
    public void deleteTask(Long id) {
        boolean deleted = taskRepository.deleteById(id);
        if (!deleted) {
            throw new TaskNotFoundException(id);
        }
    }
    
    private void validateStatusTransition(String from, String to) {
        List<String> validStatuses = List.of("TODO", "IN_PROGRESS", "DONE");
        if (!validStatuses.contains(to)) {
            throw new IllegalArgumentException("Invalid status: " + to);
        }
    }
    
    private TaskResponse mapToResponse(Task task) {
        return new TaskResponse(
            task.getId(),
            task.getTitle(),
            task.getDescription(),
            task.getStatus(),
            task.getCreatedAt()
        );
    }
}
```

### Refactored Controller (Thin!)

```java
package com.example.taskapi.controller;

import com.example.taskapi.dto.*;
import com.example.taskapi.service.TaskService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    private final TaskService taskService;
    
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }
    
    @GetMapping
    public ResponseEntity<List<TaskResponse>> getAllTasks() {
        return ResponseEntity.ok(taskService.findAll());
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<TaskResponse> getTask(@PathVariable Long id) {
        return ResponseEntity.ok(taskService.findById(id));
    }
    
    @PostMapping
    public ResponseEntity<TaskResponse> createTask(@RequestBody CreateTaskRequest request) {
        TaskResponse task = taskService.createTask(request);
        URI location = URI.create("/api/tasks/" + task.id());
        return ResponseEntity.created(location).body(task);
    }
    
    @PatchMapping("/{id}/status")
    public ResponseEntity<TaskResponse> updateStatus(
            @PathVariable Long id,
            @RequestBody UpdateStatusRequest request) {
        return ResponseEntity.ok(taskService.updateStatus(id, request));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id) {
        taskService.deleteTask(id);
        return ResponseEntity.noContent().build();
    }
}
```

Notice: The controller has **no business logic**. It's a translation layer.

---

## 💥 Break It Exercise

What's wrong with this service method?

```java
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    
    @PostMapping("/orders")  // ← ???
    public ResponseEntity<OrderResponse> createOrder(OrderRequest request) {
        // ...
    }
}
```

<details>
<summary>Answer</summary>

`@PostMapping` is a Spring MVC annotation meant for **controllers only**. Putting it on a service method does nothing — Spring MVC only scans methods in `@Controller` / `@RestController` classes for route mappings.

The service method just won't be reachable via HTTP. And the `ResponseEntity` return type is an HTTP concern — services should return domain objects or DTOs.

Fix:
```java
@Service
public class OrderService {
    
    // Service method — returns domain DTO
    public OrderResponse createOrder(CreateOrderRequest request) {
        // business logic
    }
}

@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        OrderResponse response = orderService.createOrder(request); // delegate
        return ResponseEntity.status(201).body(response);
    }
}
```

</details>

---

## 🎯 Build Challenge

Add a `UserService` and `UserController` to the Task API:

- `POST /api/users` — create user (name, email)
- `GET /api/users/{id}` — get user
- `GET /api/users` — get all users

Requirements:
1. Use proper layering (Controller → Service → Repository)
2. No business logic in Controller
3. No HTTP code in Service
4. UserRepository is in-memory (same pattern as TaskRepository)
5. `UserNotFoundException` when user not found

Don't look at solutions. Build it from scratch.

---

## ✅ Completion Checklist

- [ ] Can you explain what belongs in each layer?
- [ ] Can you explain why the dependency arrow goes downward only?
- [ ] Did you refactor the Task API to use 3 layers?
- [ ] Can you explain why Controllers should be "thin"?
- [ ] Can you explain what happens if you put business logic in a controller?

---

*Next: [08-01 — JPA Overview](../08-jpa-hibernate/01-jpa-overview.md)*

(We'll add PostgreSQL + JPA to the Task API — replacing the in-memory repository)
