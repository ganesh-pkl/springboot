# Project 1 — Task Management API

## 🎯 Project Goal

Build a complete, production-quality REST API for managing tasks. This project evolves across modules — adding more features as you learn new concepts.

---

## 📋 Features

### Phase 1 — In-Memory (After Level 3-4)
- [ ] Create, read, update, delete tasks
- [ ] Layered architecture (Controller → Service → Repository)
- [ ] DTOs and Records
- [ ] Basic exception handling

### Phase 2 — PostgreSQL (After Level 5)
- [ ] JPA entities
- [ ] Spring Data JPA repository
- [ ] PostgreSQL integration
- [ ] Pagination and sorting

### Phase 3 — Full Features (After Level 7)
- [ ] Bean validation
- [ ] Global exception handler with RFC 7807 format
- [ ] Custom exception hierarchy

### Phase 4 — Security (After Level 8)
- [ ] User registration and login
- [ ] JWT authentication
- [ ] Users can only access their own tasks
- [ ] Admin role can see all tasks

### Phase 5 — Production (After Level 11)
- [ ] Actuator health endpoint
- [ ] Logging
- [ ] Docker + docker-compose
- [ ] Tests for all layers

---

## 🏗️ Project Structure

```
task-api/
├── src/main/java/com/example/taskapi/
│   ├── TaskApiApplication.java
│   ├── config/
│   │   └── SecurityConfig.java
│   ├── controller/
│   │   ├── AuthController.java
│   │   └── TaskController.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateTaskRequest.java
│   │   │   ├── UpdateTaskRequest.java
│   │   │   ├── UpdateStatusRequest.java
│   │   │   ├── LoginRequest.java
│   │   │   └── RegisterRequest.java
│   │   └── response/
│   │       ├── TaskResponse.java
│   │       ├── UserResponse.java
│   │       ├── AuthResponse.java
│   │       └── ErrorResponse.java
│   ├── entity/
│   │   ├── Task.java
│   │   └── User.java
│   ├── exception/
│   │   ├── AppException.java
│   │   ├── TaskNotFoundException.java
│   │   ├── UserNotFoundException.java
│   │   └── DuplicateTitleException.java
│   ├── handler/
│   │   └── GlobalExceptionHandler.java
│   ├── repository/
│   │   ├── TaskRepository.java
│   │   └── UserRepository.java
│   ├── security/
│   │   ├── JwtService.java
│   │   ├── JwtAuthenticationFilter.java
│   │   ├── AppUserDetails.java
│   │   └── UserDetailsServiceImpl.java
│   └── service/
│       ├── AuthService.java
│       └── TaskService.java
├── src/main/resources/
│   ├── application.yml
│   └── application-dev.yml
├── src/test/java/com/example/taskapi/
│   ├── service/
│   │   ├── TaskServiceTest.java
│   │   └── AuthServiceTest.java
│   └── controller/
│       └── TaskControllerTest.java
├── docker-compose.yml
├── Dockerfile
└── pom.xml
```

---

## 📦 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>task-api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>task-api</name>
    
    <properties>
        <java.version>21</java.version>
        <jjwt.version>0.12.3</jjwt.version>
    </properties>
    
    <dependencies>
        <!-- Core -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Data -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        
        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Dev tools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## ⚙️ application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/taskdb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  security:
    open-in-view: false  # Disable open session in view

server:
  port: 8080

app:
  jwt:
    # Generate with: openssl rand -base64 64
    secret: ${JWT_SECRET:dGhpcy1pcy1hLXZlcnktbG9uZy1zZWNyZXQta2V5LWZvci1IUzI1Ni1zaWduaW5n}
    expiration: 86400000        # 24 hours
    refresh-expiration: 604800000  # 7 days

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    com.example.taskapi: INFO
    org.hibernate.SQL: DEBUG
```

---

## 🐳 Docker Setup

### Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

WORKDIR /app

# Create non-root user for security
RUN addgroup -S spring && adduser -S spring -G spring

COPY target/*.jar app.jar

USER spring

EXPOSE 8080

ENTRYPOINT ["java", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", \
  "/app/app.jar"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_USERNAME=taskuser
      - DB_PASSWORD=taskpass
      - JWT_SECRET=your-secret-key-change-in-production
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/taskdb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - task-network

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=taskdb
      - POSTGRES_USER=taskuser
      - POSTGRES_PASSWORD=taskpass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskuser -d taskdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - task-network

volumes:
  postgres_data:

networks:
  task-network:
```

---

## 🔧 Global Exception Handler

```java
package com.example.taskapi.handler;

import com.example.taskapi.exception.AppException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ProblemDetail> handleAppException(AppException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.valueOf(ex.getStatusCode()),
            ex.getMessage()
        );
        problem.setTitle(ex.getClass().getSimpleName());
        problem.setProperty("timestamp", Instant.now());
        return ResponseEntity.status(ex.getStatusCode()).body(problem);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String message = error.getDefaultMessage();
            errors.put(fieldName, message);
        });
        
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setDetail("One or more fields have errors");
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());
        
        return ResponseEntity.badRequest().body(problem);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleGenericException(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setTitle("Internal Server Error");
        problem.setDetail("An unexpected error occurred");
        problem.setProperty("timestamp", Instant.now());
        // Log the actual exception for debugging, but don't expose it to client
        return ResponseEntity.internalServerError().body(problem);
    }
}
```

---

## 🧪 Running the Project

```bash
# Start database
docker run --name taskdb \
  -e POSTGRES_DB=taskdb \
  -e POSTGRES_USER=taskuser \
  -e POSTGRES_PASSWORD=taskpass \
  -p 5432:5432 \
  -d postgres:16

# Run the application
./mvnw spring-boot:run

# Or with Docker Compose (full stack)
docker-compose up --build

# Run tests
./mvnw test

# Package (skip tests for speed)
./mvnw package -DskipTests

# API Documentation (if Swagger is added)
# http://localhost:8080/swagger-ui.html
```

---

## 📝 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/api/auth/register` | Register new user | Public |
| POST | `/api/auth/login` | Login, get JWT | Public |
| POST | `/api/auth/refresh` | Refresh JWT | Public |
| GET | `/api/tasks` | Get my tasks (paginated) | Required |
| GET | `/api/tasks/{id}` | Get specific task | Required |
| POST | `/api/tasks` | Create task | Required |
| PUT | `/api/tasks/{id}` | Update task | Required (owner) |
| PATCH | `/api/tasks/{id}/status` | Update status | Required (owner) |
| DELETE | `/api/tasks/{id}` | Delete task | Required (owner) |
| GET | `/api/admin/tasks` | Get ALL tasks | Admin only |
| GET | `/actuator/health` | Health check | Public |

---

## 🏁 Completion Criteria

The project is complete when:

- [ ] All endpoints work and return correct status codes
- [ ] JWT auth works (login, protected endpoints, 401 without token)
- [ ] Users can only access their own tasks
- [ ] Validation errors return 400 with field-level messages
- [ ] Not-found errors return 404
- [ ] All service methods have unit tests
- [ ] At least one controller integration test
- [ ] Application runs with `docker-compose up`
- [ ] Health endpoint returns 200
- [ ] All SQL queries logged in dev mode

---

*Next Project: [E-commerce API](../02-ecommerce-api/README.md)*
