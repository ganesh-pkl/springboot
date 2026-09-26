# Lesson 13-02 — Unit Testing Services with Mockito

## 🎯 Learning Objective

Write meaningful unit tests for Spring Boot service classes. Test business logic in complete isolation from the database and external dependencies.

---

## 🤔 Think Before Reading

You've built a `TaskService.createTask()` method. To test it:
- Do you need a running database?
- Do you need to start the Spring context?
- How do you test the "duplicate title" business rule?

**Question**: In Express/Jest, you'd mock the repository. How is this different in Java?

---

## 🔍 Unit Test vs Integration Test

| | Unit Test | Integration Test |
|--|-----------|-----------------|
| What's tested | Single class in isolation | Multiple components together |
| Spring context | NO — no Spring | YES — full Spring context |
| Database | Mocked | Real (H2 or Testcontainers) |
| Speed | Fast (milliseconds) | Slow (seconds) |
| When to use | Business logic | Full request flow |

For service layer: **unit tests with mocks**. Fast, isolated, precise.

---

## 💻 Mockito — Mocking Framework

Mockito creates fake implementations of interfaces/classes:

```java
// Create a mock
UserRepository mockRepo = Mockito.mock(UserRepository.class);

// Tell the mock what to return
when(mockRepo.findById(1L)).thenReturn(Optional.of(new User("Alice", "alice@example.com", "hash")));
when(mockRepo.findByEmail("alice@example.com")).thenReturn(Optional.empty());

// The mock only returns what you configure — no real database calls
```

---

## 💻 Complete Service Test

```java
package com.example.taskapi.service;

import com.example.taskapi.dto.CreateTaskRequest;
import com.example.taskapi.dto.TaskResponse;
import com.example.taskapi.entity.Task;
import com.example.taskapi.entity.User;
import com.example.taskapi.exception.DuplicateTitleException;
import com.example.taskapi.exception.TaskNotFoundException;
import com.example.taskapi.repository.TaskRepository;
import com.example.taskapi.repository.UserRepository;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class) // Enables @Mock, @InjectMocks
class TaskServiceTest {
    
    @Mock
    private TaskRepository taskRepository;
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks  // Creates TaskService and injects the mocks above
    private TaskService taskService;
    
    private User testUser;
    private Task testTask;
    
    @BeforeEach
    void setUp() {
        testUser = new User("Alice", "alice@example.com", "hashedPassword");
        // Use reflection to set ID (since no public setter)
        ReflectionTestUtils.setField(testUser, "id", 1L);
        
        testTask = new Task("Learn Spring Boot", "Become expert");
        ReflectionTestUtils.setField(testTask, "id", 1L);
    }
    
    // ─── findById tests ────────────────────────────────────────────
    
    @Test
    @DisplayName("findById should return task when task exists")
    void findById_shouldReturnTask_whenExists() {
        // Given
        when(taskRepository.findById(1L)).thenReturn(Optional.of(testTask));
        
        // When
        TaskResponse result = taskService.findById(1L);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.id()).isEqualTo(1L);
        assertThat(result.title()).isEqualTo("Learn Spring Boot");
        
        // Verify the repository was called exactly once with the right argument
        verify(taskRepository, times(1)).findById(1L);
    }
    
    @Test
    @DisplayName("findById should throw TaskNotFoundException when task not found")
    void findById_shouldThrow_whenNotFound() {
        // Given
        when(taskRepository.findById(99L)).thenReturn(Optional.empty());
        
        // When / Then
        assertThatThrownBy(() -> taskService.findById(99L))
            .isInstanceOf(TaskNotFoundException.class)
            .hasMessageContaining("99");
        
        verify(taskRepository).findById(99L);
    }
    
    // ─── createTask tests ──────────────────────────────────────────
    
    @Test
    @DisplayName("createTask should create and return task when title is unique")
    void createTask_shouldCreateTask_whenTitleIsUnique() {
        // Given
        CreateTaskRequest request = new CreateTaskRequest(
            "New Task", "Description", 1L
        );
        
        when(taskRepository.existsByTitleAndUserId("New Task", 1L)).thenReturn(false);
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));
        
        // Capture the Task object saved to repository
        when(taskRepository.save(any(Task.class))).thenAnswer(invocation -> {
            Task task = invocation.getArgument(0);
            ReflectionTestUtils.setField(task, "id", 100L);
            return task;
        });
        
        // When
        TaskResponse result = taskService.createTask(request);
        
        // Then
        assertThat(result.id()).isEqualTo(100L);
        assertThat(result.title()).isEqualTo("New Task");
        assertThat(result.status()).isEqualTo("TODO");
        
        // Verify save was called
        verify(taskRepository).save(any(Task.class));
    }
    
    @Test
    @DisplayName("createTask should throw DuplicateTitleException when title already exists")
    void createTask_shouldThrow_whenTitleExists() {
        // Given
        CreateTaskRequest request = new CreateTaskRequest("Existing Task", "Desc", 1L);
        when(taskRepository.existsByTitleAndUserId("Existing Task", 1L)).thenReturn(true);
        
        // When / Then
        assertThatThrownBy(() -> taskService.createTask(request))
            .isInstanceOf(DuplicateTitleException.class);
        
        // Verify save was NEVER called
        verify(taskRepository, never()).save(any());
    }
    
    // ─── updateStatus tests ────────────────────────────────────────
    
    @Test
    @DisplayName("updateStatus should change status to IN_PROGRESS")
    void updateStatus_shouldUpdate_toInProgress() {
        // Given
        when(taskRepository.findById(1L)).thenReturn(Optional.of(testTask));
        when(taskRepository.save(any(Task.class))).thenReturn(testTask);
        
        UpdateStatusRequest request = new UpdateStatusRequest("IN_PROGRESS");
        
        // When
        TaskResponse result = taskService.updateStatus(1L, request);
        
        // Then
        assertThat(result.status()).isEqualTo("IN_PROGRESS");
    }
    
    @Test
    @DisplayName("updateStatus should throw when status is invalid")
    void updateStatus_shouldThrow_whenInvalidStatus() {
        when(taskRepository.findById(1L)).thenReturn(Optional.of(testTask));
        
        UpdateStatusRequest request = new UpdateStatusRequest("INVALID_STATUS");
        
        assertThatThrownBy(() -> taskService.updateStatus(1L, request))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("INVALID_STATUS");
    }
    
    // ─── deleteTask tests ──────────────────────────────────────────
    
    @Test
    @DisplayName("deleteTask should delete when task exists")
    void deleteTask_shouldDelete_whenExists() {
        // Given
        when(taskRepository.findById(1L)).thenReturn(Optional.of(testTask));
        doNothing().when(taskRepository).delete(testTask);
        
        // When (no return value, just check no exception)
        assertThatCode(() -> taskService.deleteTask(1L))
            .doesNotThrowAnyException();
        
        // Verify delete was called
        verify(taskRepository).delete(testTask);
    }
    
    @Test
    @DisplayName("deleteTask should throw when task not found")
    void deleteTask_shouldThrow_whenNotFound() {
        when(taskRepository.findById(99L)).thenReturn(Optional.empty());
        
        assertThatThrownBy(() -> taskService.deleteTask(99L))
            .isInstanceOf(TaskNotFoundException.class);
        
        verify(taskRepository, never()).delete(any());
    }
}
```

---

## 🔍 Key Mockito Methods

```java
// Stubbing — what to return
when(mock.method(args)).thenReturn(value);
when(mock.method(args)).thenThrow(new RuntimeException());
when(mock.method(args)).thenAnswer(invocation -> {
    Object arg = invocation.getArgument(0);
    return processArg(arg);
});

// Argument matchers
when(mock.method(any())).thenReturn(value);        // Any argument
when(mock.method(anyLong())).thenReturn(value);    // Any Long
when(mock.method(eq(42L))).thenReturn(value);      // Specifically 42L
when(mock.method(isNull())).thenReturn(value);     // null argument

// Verification
verify(mock).method(args);              // Called exactly once
verify(mock, times(2)).method(args);   // Called twice
verify(mock, never()).method(args);    // Never called
verify(mock, atLeast(1)).method(args); // Called at least once

// Capture arguments
ArgumentCaptor<Task> captor = ArgumentCaptor.forClass(Task.class);
verify(taskRepository).save(captor.capture());
Task savedTask = captor.getValue();
assertThat(savedTask.getTitle()).isEqualTo("Expected Title");
```

---

## 🔍 AssertJ — Fluent Assertions

```java
// Simple assertions
assertThat(result).isNotNull();
assertThat(result.id()).isEqualTo(1L);
assertThat(result.title()).isEqualTo("Learn Spring Boot");
assertThat(result.status()).isEqualTo("TODO");

// Collections
assertThat(result).hasSize(3);
assertThat(result).contains(task1, task2);
assertThat(result).extracting("title").contains("Task A", "Task B");

// Exceptions
assertThatThrownBy(() -> service.method())
    .isInstanceOf(TaskNotFoundException.class)
    .hasMessageContaining("99")
    .hasNoCause();

// No exception
assertThatCode(() -> service.deleteTask(1L))
    .doesNotThrowAnyException();
```

---

## 🎯 Practice Exercise

Write tests for this method:

```java
@Service
public class UserService {
    
    public UserResponse updateEmail(Long userId, String newEmail) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        if (userRepository.existsByEmail(newEmail)) {
            throw new DuplicateEmailException(newEmail);
        }
        
        if (!newEmail.contains("@")) {
            throw new ValidationException("Invalid email format");
        }
        
        user.setEmail(newEmail);
        User saved = userRepository.save(user);
        return new UserResponse(saved.getId(), saved.getName(), saved.getEmail());
    }
}
```

Write tests covering:
1. Successful email update
2. User not found
3. Email already taken
4. Invalid email format

<details>
<summary>Solution</summary>

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock private UserRepository userRepository;
    @InjectMocks private UserService userService;
    
    private User testUser;
    
    @BeforeEach
    void setUp() {
        testUser = new User("Alice", "old@example.com", "hash");
        ReflectionTestUtils.setField(testUser, "id", 1L);
    }
    
    @Test
    void updateEmail_shouldUpdate_whenValid() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));
        when(userRepository.existsByEmail("new@example.com")).thenReturn(false);
        when(userRepository.save(any())).thenReturn(testUser);
        
        UserResponse result = userService.updateEmail(1L, "new@example.com");
        
        assertThat(result).isNotNull();
        verify(userRepository).save(testUser);
    }
    
    @Test
    void updateEmail_shouldThrow_whenUserNotFound() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());
        
        assertThatThrownBy(() -> userService.updateEmail(99L, "new@example.com"))
            .isInstanceOf(UserNotFoundException.class);
        
        verify(userRepository, never()).save(any());
    }
    
    @Test
    void updateEmail_shouldThrow_whenEmailTaken() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));
        when(userRepository.existsByEmail("taken@example.com")).thenReturn(true);
        
        assertThatThrownBy(() -> userService.updateEmail(1L, "taken@example.com"))
            .isInstanceOf(DuplicateEmailException.class);
        
        verify(userRepository, never()).save(any());
    }
    
    @Test
    void updateEmail_shouldThrow_whenInvalidFormat() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));
        when(userRepository.existsByEmail("notanemail")).thenReturn(false);
        
        assertThatThrownBy(() -> userService.updateEmail(1L, "notanemail"))
            .isInstanceOf(ValidationException.class)
            .hasMessageContaining("Invalid email format");
        
        verify(userRepository, never()).save(any());
    }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you use `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`?
- [ ] Can you stub methods with `when(...).thenReturn(...)` and `thenThrow()`?
- [ ] Can you verify calls with `verify(mock, times(n)).method()`?
- [ ] Can you test exception throwing with `assertThatThrownBy()`?
- [ ] Can you use `ArgumentCaptor` to inspect saved objects?
- [ ] Did you complete the practice exercise?

---

*Next: [16-02 — AOP Concepts](../16-aop-proxies/02-aop-concepts.md)*
