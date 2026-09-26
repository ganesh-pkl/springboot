# 🚀 Spring Boot Mastery — From Node.js to Production Java

> A deeply practical, project-driven course for developers who already know backend engineering and want to become **genuinely strong** in Spring Boot — not just "can write CRUD" strong, but **architect and debug production systems** strong.

---

## 👤 Who This Course Is For

You are a **backend developer** who already knows:

- JavaScript / TypeScript
- Node.js + Express
- REST APIs (routes, controllers, services, repositories)
- SQL + PostgreSQL
- Basic Docker
- Software design principles (SOLID, patterns)
- Git

You want to learn Spring Boot because the ecosystem demands it — and you want to understand it **deeply**, not just enough to pass a coding test.

---

## ⚠️ Who This Is NOT For

- Absolute beginners who have never built a REST API
- Developers who just want copy-paste Spring Boot snippets
- Developers who are okay not understanding what's happening under the hood

---

## 🎯 What You Will Be Able to Do After This Course

After completing this course, you will be able to:

```
✅ Build production-quality Spring Boot APIs from scratch
✅ Understand the Spring IoC container and DI mechanism
✅ Design clean layered architectures
✅ Work with JPA/Hibernate deeply (not just annotations)
✅ Handle transactions correctly
✅ Implement JWT-based authentication and authorization
✅ Write unit and integration tests
✅ Configure Spring Boot for different environments
✅ Use Spring Actuator and observability tools
✅ Dockerize Spring Boot applications
✅ Understand AOP, proxies, and interceptors
✅ Understand how annotations actually work
✅ Debug Spring Boot applications independently
✅ Confidently answer Spring Boot interview questions
✅ Read and understand real-world Spring Boot codebases
```

---

## 🛠️ Setup Requirements

### Java Version

**Java 21 (LTS)** — This course uses Java 21 features where appropriate (Records, Pattern Matching, etc.)

```bash
# Install via SDKMAN (recommended)
curl -s "https://get.sdkman.io" | bash
sdk install java 21.0.3-tem

# Verify
java -version
# openjdk version "21.0.x"
```

### IDE

**IntelliJ IDEA Community Edition** (Free) or Ultimate

Download: https://www.jetbrains.com/idea/download/

> IntelliJ understands Spring deeply. It provides auto-completion, dependency resolution, and error detection that VS Code cannot match for Spring projects.

### Build Tool

**Maven** (used throughout this course)

Maven comes bundled with IntelliJ. No separate install needed.

### Spring Boot Version

**Spring Boot 3.3.x** (Jakarta EE 10 namespace — `jakarta.*` not `javax.*`)

### Database

**PostgreSQL 16+**

```bash
# macOS
brew install postgresql@16
brew services start postgresql@16

# Docker (recommended)
docker run --name spring-course-pg \
  -e POSTGRES_USER=springuser \
  -e POSTGRES_PASSWORD=springpass \
  -e POSTGRES_DB=springcourse \
  -p 5432:5432 \
  -d postgres:16
```

### Project Generator

Use **Spring Initializr**: https://start.spring.io

Or use IntelliJ's built-in Spring Initializr support.

---

## 📁 Course Structure

```
springboot-course/
│
├── README.md                          ← You are here
├── 00-course-map.md                   ← Full course overview
│
├── 01-java-for-spring/                ← Level 0: Java essentials
│   ├── 01-classes-and-objects.md
│   ├── 02-interfaces-and-abstract.md
│   ├── 03-generics-and-collections.md
│   ├── 04-exceptions.md
│   ├── 05-lambdas-and-streams.md
│   ├── 06-optional.md
│   ├── 07-records.md
│   └── 08-annotations.md
│
├── 02-spring-fundamentals/            ← Level 1: Spring Framework
│   ├── 01-why-spring-exists.md
│   ├── 02-ioc-and-di.md
│   ├── 03-application-context.md
│   ├── 04-beans-and-lifecycle.md
│   ├── 05-component-scanning.md
│   └── 06-configuration.md
│
├── 03-spring-boot-fundamentals/       ← Level 2: Spring Boot
│   ├── 01-why-spring-boot.md
│   ├── 02-auto-configuration.md
│   ├── 03-starters.md
│   ├── 04-embedded-server.md
│   └── 05-application-properties.md
│
├── 04-rest-api/                       ← Level 3: REST APIs
│   ├── 01-controllers.md
│   ├── 02-request-mapping.md
│   ├── 03-request-response.md
│   ├── 04-dtos-and-records.md
│   └── 05-response-entity.md
│
├── 05-architecture/                   ← Level 4: Layered Architecture
│   ├── 01-layered-architecture.md
│   ├── 02-service-layer.md
│   ├── 03-repository-layer.md
│   └── 04-dependency-inversion.md
│
├── 06-validation/                     ← Validation
│   ├── 01-bean-validation.md
│   ├── 02-custom-validators.md
│   └── 03-validation-groups.md
│
├── 07-exception-handling/             ← Error handling
│   ├── 01-global-exception-handler.md
│   ├── 02-custom-exceptions.md
│   └── 03-error-response-format.md
│
├── 08-jpa-hibernate/                  ← Level 5: JPA + Hibernate
│   ├── 01-jpa-overview.md
│   ├── 02-entities-and-mapping.md
│   ├── 03-relationships.md
│   ├── 04-lazy-vs-eager.md
│   ├── 05-persistence-context.md
│   ├── 06-n-plus-one-problem.md
│   └── 07-pagination-and-sorting.md
│
├── 09-postgresql/                     ← PostgreSQL + Spring Data
│   ├── 01-spring-data-jpa.md
│   ├── 02-repositories.md
│   └── 03-custom-queries.md
│
├── 10-transactions/                   ← Level 6: Transactions
│   ├── 01-acid-and-transactions.md
│   ├── 02-transactional-annotation.md
│   ├── 03-propagation.md
│   └── 04-isolation.md
│
├── 11-security/                       ← Level 8: Spring Security
│   ├── 01-authentication-vs-authorization.md
│   ├── 02-security-filter-chain.md
│   ├── 03-user-details-service.md
│   └── 04-roles-and-authorities.md
│
├── 12-jwt/                            ← JWT
│   ├── 01-jwt-fundamentals.md
│   ├── 02-jwt-implementation.md
│   └── 03-refresh-tokens.md
│
├── 13-testing/                        ← Level 9: Testing
│   ├── 01-junit-and-mockito.md
│   ├── 02-unit-testing-services.md
│   ├── 03-controller-testing.md
│   └── 04-integration-testing.md
│
├── 14-configuration/                  ← Config
│   ├── 01-profiles.md
│   └── 02-secrets-and-env.md
│
├── 15-filters-interceptors/           ← Internals
│   ├── 01-filters.md
│   └── 02-interceptors.md
│
├── 16-aop-proxies/                    ← AOP
│   ├── 01-proxy-pattern.md
│   ├── 02-aop-concepts.md
│   └── 03-aop-in-spring.md
│
├── 17-actuator/
│   └── 01-actuator-and-health.md
│
├── 18-caching/
│   └── 01-caching-with-redis.md
│
├── 19-async-processing/
│   └── 01-async-and-scheduling.md
│
├── 20-docker/
│   └── 01-dockerizing-spring-boot.md
│
├── 21-performance/
│   └── 01-performance-and-scalability.md
│
├── 22-interview-preparation/
│   ├── 01-spring-core-interview.md
│   ├── 02-jpa-interview.md
│   ├── 03-security-interview.md
│   └── 04-scenario-questions.md
│
└── projects/
    ├── 01-task-management-api/        ← Project 1
    ├── 02-ecommerce-api/              ← Project 2
    ├── 03-subscription-platform/     ← Project 3
    └── 04-production-grade-backend/  ← Capstone
```

---

## 📅 Recommended Daily Learning Routine

**DO NOT** do 3-hour passive reading sessions. That's how you forget everything.

Instead, use this loop:

```
⏱️  15 min  →  Read the lesson concept
⏱️  30 min  →  Write the code yourself (no copy-paste)
⏱️  15 min  →  Break it deliberately, debug it
⏱️  15 min  →  Do the challenge
⏱️  10 min  →  Recall: explain it back in your own words
─────────────────────────────────────────────────────
Total: ~85 min per lesson
```

> **Recall** is the most underrated step. Close the notes. Explain the concept aloud as if you're teaching it. This forces your brain to find gaps.

---

## 🗺️ Learning Strategy

### If you're pressed for time (intensive mode)

Focus on this critical path first:

```
01-java-for-spring (just the unfamiliar bits)
    ↓
02-spring-fundamentals (ALL of it — this is the core)
    ↓
03-spring-boot-fundamentals
    ↓
04-rest-api
    ↓
05-architecture
    ↓
06-validation + 07-exception-handling
    ↓
08-jpa-hibernate (the hardest and most important part)
    ↓
10-transactions
    ↓
11-security + 12-jwt
    ↓
13-testing
    ↓
Project 1 + Project 2
```

### If you have time (deep mode)

Follow the full course sequentially. Don't skip modules.

---

## 🏗️ How to Run Projects

Each project folder contains its own README with setup instructions.

General pattern:

```bash
# Navigate to project
cd projects/01-task-management-api

# Run with Maven Wrapper
./mvnw spring-boot:run

# Or import into IntelliJ and hit the green Run button
```

Database is always configured in:

```
src/main/resources/application.properties
# or
src/main/resources/application.yml
```

---

## 📋 Node.js → Spring Boot Mental Map

Keep this handy. We'll expand on each row throughout the course.

| Node.js / Express | Spring Boot | Key Difference |
|---|---|---|
| `express()` app | `@SpringBootApplication` | Spring manages an IoC container |
| Route handler | Controller method | Spring manages the whole request lifecycle |
| `app.use(middleware)` | `Filter` / `Interceptor` | Spring has two distinct interception layers |
| Service class | `@Service` bean | Spring creates and injects it |
| Repository class | `@Repository` + JPA | Spring generates most SQL for you |
| `require()` / `import` | `@Autowired` / constructor injection | Spring resolves dependencies at startup |
| `.env` | `application.properties` / `application.yml` | Spring has profiles for environments |
| Error middleware | `@ControllerAdvice` | Centralized, class-based |
| `try/catch` in async | `@Transactional` | Spring wraps methods in DB transactions |
| `passport.js` | Spring Security | Deep, filter-chain based security |
| `jest` | JUnit + Mockito | JVM-native, annotation-driven |
| `nodemon` | Spring DevTools | Auto-restart on code changes |

---

## 🧠 The Most Important Mindset Shift

In Node.js/Express, **you** create objects and wire them:

```javascript
const userRepository = new UserRepository(db);
const userService = new UserService(userRepository);
const userController = new UserController(userService);
app.use('/users', userController.router);
```

In Spring Boot, **you declare intent** and the framework does the wiring:

```java
@RestController
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {  // Spring injects this
        this.userService = userService;
    }
}
```

**This is not magic.** The course will explain exactly what Spring does to make this work.

---

## 💬 How to Use This Course

1. **Read** the lesson objectives first
2. **Answer** the "Think Before Reading" questions before scrolling
3. **Type** all code yourself — no copy-paste
4. **Break** working code intentionally
5. **Debug** it
6. **Complete** the challenge before moving on
7. **Recall** the concept before the next lesson

> The learner who moves slowly and understands deeply will beat the learner who rushes and memorizes.

---

## 🆘 When You Get Stuck

1. Re-read the "How Spring Works Internally" section
2. Check the Node.js comparison
3. Add a breakpoint in IntelliJ and trace the execution
4. Google: `"spring boot [exact error message] site:stackoverflow.com"`
5. Read the Spring documentation (it's actually good once you know the concepts)

---

*Let's build something real. Start with [00-course-map.md](./00-course-map.md).*
