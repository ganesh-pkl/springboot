# 🗺️ Course Map — Spring Boot Mastery

> This is your GPS for the entire course. Bookmark this file. Return to it after every module.

---

## Overview

```
LEVEL 0  →  Java for Spring Boot
LEVEL 1  →  Spring Framework (IoC, DI, Beans)
LEVEL 2  →  Spring Boot (Auto-config, Starters)
LEVEL 3  →  REST APIs
LEVEL 4  →  Layered Architecture
LEVEL 5  →  JPA + Hibernate + PostgreSQL
LEVEL 6  →  Transactions
LEVEL 7  →  Validation + Exception Handling
LEVEL 8  →  Spring Security + JWT
LEVEL 9  →  Testing
LEVEL 10 →  Spring Internals (AOP, Proxies, Filters)
LEVEL 11 →  Production (Actuator, Docker, Caching, Async)
LEVEL 12 →  Interview Preparation + System Design
```

---

## Difficulty Progression

```
★☆☆☆☆  Beginner        → Levels 0–2
★★☆☆☆  Comfortable     → Levels 3–4
★★★☆☆  Intermediate    → Levels 5–7
★★★★☆  Advanced        → Levels 8–10
★★★★★  Production      → Levels 11–12
```

---

## Module Map

### LEVEL 0 — Java for Spring Boot

> Only the Java you need. No fluff.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 01-01 | Classes and Objects | Java vs JS objects, constructors, `this` | |
| 01-02 | Interfaces and Abstract Classes | Why Spring loves interfaces | |
| 01-03 | Generics and Collections | `List<T>`, `Optional<T>`, why generics matter | |
| 01-04 | Exceptions | Checked vs unchecked, why Spring uses unchecked | |
| 01-05 | Lambdas and Streams | Functional programming in Java | |
| 01-06 | Optional | Null safety without NPE | |
| 01-07 | Records | Modern Java DTOs | |
| 01-08 | Annotations | What annotations ARE, custom annotations | |

**Checkpoint**: Can you write a Java class, implement an interface, use generics, and write a lambda?

---

### LEVEL 1 — Spring Framework

> This is the hardest conceptual jump. Take your time here.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 02-01 | Why Spring Exists | The problem before Spring | |
| 02-02 | IoC and Dependency Injection | The most important concept | |
| 02-03 | ApplicationContext | The Spring container | |
| 02-04 | Beans and Lifecycle | How Spring manages objects | |
| 02-05 | Component Scanning | How Spring finds your classes | |
| 02-06 | Configuration | @Configuration and @Bean | |

**Checkpoint**: Can you explain IoC without looking at notes? Can you draw the DI flow?

---

### LEVEL 2 — Spring Boot

> Spring Boot is not a framework. It's an opinionated configuration tool on top of Spring.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 03-01 | Why Spring Boot Exists | Spring without Boot = 500 XML lines | |
| 03-02 | Auto-Configuration | The magic explained | |
| 03-03 | Starters | Curated dependency bundles | |
| 03-04 | Embedded Server | Tomcat is inside the JAR | |
| 03-05 | Application Properties | Your app's config system | |

**Checkpoint**: Can you create a Spring Boot app from scratch and explain what @SpringBootApplication does?

---

### LEVEL 3 — REST API Development

> Build your first real endpoints. Map your Express knowledge.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 04-01 | Controllers | @RestController vs Express Router | |
| 04-02 | Request Mapping | GET, POST, PUT, PATCH, DELETE | |
| 04-03 | Request/Response | @RequestBody, @PathVariable, @RequestParam | |
| 04-04 | DTOs and Records | Request/response models | |
| 04-05 | ResponseEntity | Control HTTP status codes | |

**Project Start**: Task Management API (in-memory, no DB yet)

---

### LEVEL 4 — Layered Architecture

> Where business logic belongs. Why layers exist.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 05-01 | Layered Architecture | Controller → Service → Repository | |
| 05-02 | Service Layer | Business logic, orchestration | |
| 05-03 | Repository Layer | Data access abstraction | |
| 05-04 | Dependency Inversion | Programming to interfaces | |

---

### LEVEL 5 — JPA + Hibernate + PostgreSQL

> The most important and most misunderstood part. Don't rush this.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 08-01 | JPA Overview | What JPA IS vs what Hibernate IS | |
| 08-02 | Entities and Mapping | @Entity, @Id, @Column, ID generation | |
| 08-03 | Relationships | @OneToMany, @ManyToOne, @ManyToMany | |
| 08-04 | Lazy vs Eager | Loading strategies and their tradeoffs | |
| 08-05 | Persistence Context | First-level cache, dirty checking | |
| 08-06 | N+1 Problem | The most common JPA bug | |
| 08-07 | Pagination and Sorting | Real-world data retrieval | |
| 09-01 | Spring Data JPA | JpaRepository, CrudRepository | |
| 09-02 | Repository Methods | Query derivation, @Query | |
| 09-03 | Custom Queries | JPQL, native SQL | |

**Project**: Add PostgreSQL to Task Management API

---

### LEVEL 6 — Transactions

> The most dangerous area if misunderstood.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 10-01 | ACID and Transactions | What a transaction actually is | |
| 10-02 | @Transactional | How Spring wraps methods in transactions | |
| 10-03 | Propagation | REQUIRED, REQUIRES_NEW, etc. | |
| 10-04 | Isolation | Read committed, repeatable read, serializable | |

---

### LEVEL 7 — Validation + Exception Handling

> Building professional APIs.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 06-01 | Bean Validation | @Valid, @NotNull, @Size, etc. | |
| 06-02 | Custom Validators | @Constraint annotation | |
| 07-01 | Global Exception Handler | @ControllerAdvice | |
| 07-02 | Custom Exceptions | ResourceNotFoundException, etc. | |
| 07-03 | Error Response Format | RFC 7807 problem details | |

---

### LEVEL 8 — Spring Security + JWT

> The most complex module. Build it step by step.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 11-01 | Auth vs Authorization | Conceptual foundation | |
| 11-02 | Security Filter Chain | How every request is intercepted | |
| 11-03 | UserDetailsService | Custom user loading | |
| 11-04 | Roles and Authorities | RBAC in Spring | |
| 12-01 | JWT Fundamentals | Header, payload, signature | |
| 12-02 | JWT Implementation | Full auth flow | |
| 12-03 | Refresh Tokens | Long-lived sessions | |

**Project**: Add auth to E-commerce API

---

### LEVEL 9 — Testing

> Untested code is legacy code.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 13-01 | JUnit + Mockito | The testing foundation | |
| 13-02 | Unit Testing Services | Test business logic in isolation | |
| 13-03 | Controller Testing | MockMvc | |
| 13-04 | Integration Testing | Real DB, real HTTP | |

---

### LEVEL 10 — Spring Internals

> Go from "it works" to "I know WHY it works."

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 15-01 | Filters | Servlet-level interception | |
| 15-02 | Interceptors | Spring MVC-level interception | |
| 16-01 | Proxy Pattern | The foundation of Spring's power | |
| 16-02 | AOP Concepts | Cross-cutting concerns | |
| 16-03 | AOP in Spring | @Aspect, @Around, @Before | |

---

### LEVEL 11 — Production

> Real apps aren't just APIs. They're systems.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 14-01 | Profiles | dev, staging, prod configurations | |
| 14-02 | Secrets and Env Vars | Never hardcode credentials | |
| 17-01 | Actuator and Health | Monitoring your application | |
| 18-01 | Caching with Redis | @Cacheable and cache management | |
| 19-01 | Async and Scheduling | @Async, @Scheduled | |
| 20-01 | Dockerizing Spring Boot | Dockerfile, docker-compose | |
| 21-01 | Performance and Scalability | Connection pools, optimizations | |

---

### LEVEL 12 — Interview Preparation

> Pass interviews. Lead design conversations.

| # | Lesson | Key Concept | Status |
|---|--------|-------------|--------|
| 22-01 | Spring Core Interview | IoC, DI, Bean lifecycle | |
| 22-02 | JPA Interview | Persistence context, N+1, transactions | |
| 22-03 | Security Interview | Filter chain, JWT, auth flow | |
| 22-04 | Scenario Questions | Architecture and debugging scenarios | |

---

## Projects Timeline

```
Level 3-4  →  Project 1: Task Management API (in-memory then PostgreSQL)
Level 5-8  →  Project 2: E-commerce Backend (users, products, orders, auth)
Level 9-11 →  Project 3: Subscription Platform (webhooks, idempotency, bg jobs)
Level 12   →  Project 4: Production-Grade Backend (full stack — capstone)
```

---

## Key Mental Models to Master

These mental models appear again and again. Master them:

### 1. The IoC Container Model
```
Your Classes
     ↓
Spring reads annotations (component scan)
     ↓
Creates Bean Definitions
     ↓
Instantiates beans in dependency order
     ↓
Injects dependencies
     ↓
Runs BeanPostProcessors (proxies, AOP, etc.)
     ↓
Application is ready
```

### 2. The Request Processing Model
```
HTTP Request
     ↓
Servlet Container (Tomcat)
     ↓
Filter Chain (Security lives here)
     ↓
DispatcherServlet
     ↓
HandlerMapping (find the right controller method)
     ↓
Interceptors (preHandle)
     ↓
Controller Method
     ↓
Interceptors (postHandle)
     ↓
HTTP Response
```

### 3. The Transaction Model
```
HTTP Request
     ↓
Transaction Proxy (created by Spring)
     ↓
BEGIN TRANSACTION
     ↓
Your Service Method
     ↓
Repository → Database
     ↓
COMMIT (or ROLLBACK on exception)
     ↓
Response
```

### 4. The Security Model
```
HTTP Request
     ↓
UsernamePasswordAuthenticationFilter (or JwtAuthFilter)
     ↓
AuthenticationManager
     ↓
UserDetailsService.loadUserByUsername()
     ↓
SecurityContextHolder.setAuthentication()
     ↓
AuthorizationFilter (checks roles/authorities)
     ↓
Controller Method (if authorized)
```

---

## Track Your Progress

Use the checkboxes as you complete each module:

```
[ ] Level 0 — Java for Spring
[ ] Level 1 — Spring Fundamentals  ← Most important
[ ] Level 2 — Spring Boot
[ ] Level 3 — REST APIs
[ ] Level 4 — Architecture
[ ] Level 5 — JPA + Hibernate  ← Most complex
[ ] Level 6 — Transactions
[ ] Level 7 — Validation + Errors
[ ] Level 8 — Security + JWT  ← Most intimidating
[ ] Level 9 — Testing
[ ] Level 10 — Internals
[ ] Level 11 — Production
[ ] Level 12 — Interview Prep

[ ] Project 1 complete
[ ] Project 2 complete
[ ] Project 3 complete
[ ] Project 4 (Capstone) complete
```

---

*Next: [Level 0 → Java for Spring Boot: Classes and Objects](./01-java-for-spring/01-classes-and-objects.md)*
