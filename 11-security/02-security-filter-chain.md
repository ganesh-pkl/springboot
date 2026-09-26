# Lesson 11-02 — Spring Security Filter Chain

## 🎯 Learning Objective

Understand how Spring Security intercepts every HTTP request before it reaches your controller. Understand the filter chain — this is the foundation for implementing JWT authentication.

---

## 🤔 Think Before Reading

In Express, you add authentication middleware:

```javascript
app.use('/api', authenticateToken); // All /api routes need auth
app.use('/api/admin', requireAdmin); // Admin-only routes
```

**Question**: In Spring Boot, where do you put this? Is it in the controller? In an interceptor? In a filter? What's the execution order?

---

## 🔍 The Security Filter Chain

Spring Security is implemented as a **chain of servlet filters** that run BEFORE your application code (controllers).

```
HTTP Request
     ↓
[Tomcat]
     ↓
Filter 1: CorsFilter               ← CORS headers
Filter 2: CsrfFilter               ← CSRF protection
Filter 3: UsernamePasswordAuthFilter ← Login form
Filter 4: BasicAuthFilter           ← HTTP Basic Auth
Filter 5: BearerTokenAuthFilter     ← JWT tokens (your custom filter)
Filter 6: ExceptionTranslationFilter ← Convert auth exceptions to HTTP responses
Filter 7: AuthorizationFilter       ← Check permissions
     ↓
[DispatcherServlet]
     ↓
Your Controller
```

Every request passes through this chain. Each filter can:
- Let the request continue to the next filter
- Stop the chain and return a response (e.g., 401 Unauthorized)
- Modify the request/response

---

## 💻 SecurityFilterChain Configuration

In Spring Security 6+ (used in Spring Boot 3), you configure security with a `SecurityFilterChain` bean:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF for stateless REST APIs (JWT-based)
            .csrf(csrf -> csrf.disable())
            
            // Configure session management — STATELESS for JWT
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            // Configure authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()       // No auth needed
                .requestMatchers("/api/public/**").permitAll()     // No auth needed
                .requestMatchers("/actuator/health").permitAll()   // Health check
                .requestMatchers("/api/admin/**").hasRole("ADMIN") // Admin only
                .anyRequest().authenticated()                       // Everything else needs auth
            );
        
        return http.build();
    }
}
```

---

## 🔍 AuthorizationHttpRequests Rules

Rules are evaluated **in order** — first match wins:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()    // CORS preflight
    .requestMatchers("/api/auth/login").permitAll()
    .requestMatchers("/api/auth/register").permitAll()
    .requestMatchers("/api/docs/**").permitAll()               // Swagger docs
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/users/{id}/**").hasAuthority("USER_READ")
    .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
    .anyRequest().authenticated()                               // Must come LAST
)
```

---

## 🔍 Request Matchers

```java
// Exact path
.requestMatchers("/api/login")

// Wildcards: * = single path segment, ** = multiple
.requestMatchers("/api/public/**")       // /api/public/anything/here
.requestMatchers("/api/users/*/orders")  // /api/users/42/orders

// HTTP method + path
.requestMatchers(HttpMethod.GET, "/api/products/**")
.requestMatchers(HttpMethod.POST, "/api/users")

// Pattern matching
.requestMatchers(new AntPathRequestMatcher("/api/admin/**", "DELETE"))
```

---

## 🔍 hasRole vs hasAuthority

```java
// hasRole automatically prepends "ROLE_"
.hasRole("ADMIN") // checks for authority "ROLE_ADMIN"

// hasAuthority checks exact string
.hasAuthority("ROLE_ADMIN") // same as hasRole("ADMIN")
.hasAuthority("USER_READ")  // custom permission
.hasAuthority("TASK_CREATE")

// Multiple roles
.hasAnyRole("ADMIN", "MANAGER")
.hasAnyAuthority("ROLE_ADMIN", "ROLE_MANAGER")

// Custom expression (SpEL)
.access("hasRole('ADMIN') or hasRole('MANAGER')")
```

---

## 💻 UserDetails and UserDetailsService

Spring Security needs to know what constitutes a "user" in your system.

### UserDetails — Your User, Security's Way

```java
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

public class AppUserDetails implements UserDetails {
    
    private final User user; // Your domain entity
    
    public AppUserDetails(User user) {
        this.user = user;
    }
    
    // Expose your user if needed
    public User getUser() { return user; }
    public Long getUserId() { return user.getId(); }
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // Convert your roles to Spring authorities
        return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole()));
    }
    
    @Override
    public String getPassword() {
        return user.getPasswordHash(); // BCrypt hash
    }
    
    @Override
    public String getUsername() {
        return user.getEmail(); // Username is email in our system
    }
    
    @Override
    public boolean isAccountNonExpired() { return true; }
    
    @Override
    public boolean isAccountNonLocked() { return true; }
    
    @Override
    public boolean isCredentialsNonExpired() { return true; }
    
    @Override
    public boolean isEnabled() { return user.isActive(); }
}
```

### UserDetailsService — Load User by Username

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    public UserDetailsServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
        
        return new AppUserDetails(user);
    }
}
```

---

## 💻 Password Hashing

NEVER store plain-text passwords. Spring Security uses `BCryptPasswordEncoder`:

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Cost factor 12 (2^12 iterations)
    }
    
    // Also configure AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager();
    }
}
```

Using it:

```java
@Service
public class AuthService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public UserResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException(request.email());
        }
        
        User user = new User(
            request.name(),
            request.email(),
            passwordEncoder.encode(request.password()) // ← BCrypt hash
        );
        
        return mapToResponse(userRepository.save(user));
    }
}
```

Verifying:
```java
// BCryptPasswordEncoder handles verification
boolean matches = passwordEncoder.matches(rawPassword, storedHash);
// Spring's AuthenticationManager does this automatically
```

---

## 🔍 SecurityContextHolder — Accessing Current User

After authentication succeeds, Spring stores user info in `SecurityContextHolder`:

```java
@Service
public class TaskService {
    
    public TaskResponse createTask(CreateTaskRequest request) {
        // Get currently authenticated user
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        AppUserDetails userDetails = (AppUserDetails) auth.getPrincipal();
        Long currentUserId = userDetails.getUserId();
        
        Task task = new Task(request.title(), request.description(), currentUserId);
        return mapToResponse(taskRepository.save(task));
    }
}
```

Or, more elegantly in controller methods:

```java
@GetMapping("/my-tasks")
public ResponseEntity<List<TaskResponse>> getMyTasks(
        @AuthenticationPrincipal AppUserDetails userDetails) {
    // Spring injects the current user via @AuthenticationPrincipal
    return ResponseEntity.ok(taskService.findByUserId(userDetails.getUserId()));
}
```

---

## 🔍 Method-Level Security

Instead of (or in addition to) URL-level rules, you can secure individual methods:

```java
@Configuration
@EnableMethodSecurity  // Enable method security
public class SecurityConfig { ... }

@Service
public class TaskService {
    
    @PreAuthorize("hasRole('ADMIN')")  // Only admins
    public void deleteAllTasks() { ... }
    
    @PreAuthorize("hasRole('USER') and #userId == authentication.principal.userId")
    public List<TaskResponse> getTasksByUser(Long userId) { ... }
    // ^ User can only get their OWN tasks
    
    @PostAuthorize("returnObject.userId == authentication.principal.userId or hasRole('ADMIN')")
    public TaskResponse getTask(Long id) { ... }
    // ^ After loading, verify the task belongs to current user
}
```

---

## 💥 Break It Exercise

What's wrong with this security configuration?

```java
@Bean
public SecurityFilterChain chain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .anyRequest().authenticated()    // ← Position 1
        .requestMatchers("/api/auth/**").permitAll()  // ← Position 2
    );
    return http.build();
}
```

<details>
<summary>Answer</summary>

Rules are matched in ORDER. `anyRequest().authenticated()` is a catch-all that matches EVERYTHING.

Since it comes FIRST, every request (including `/api/auth/login`) requires authentication. The `.permitAll()` for `/api/auth/**` comes AFTER and is NEVER reached.

**Fix**: Always put specific rules BEFORE catch-all rules:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**").permitAll()  // First: specific
    .anyRequest().authenticated()                 // Last: catch-all
)
```

This is one of the most common Spring Security mistakes.

</details>

---

## 🎯 Practice Exercise — Configure Security

For the Task API, write the security configuration that:

1. Permits `POST /api/auth/login` and `POST /api/auth/register`
2. Permits `GET /actuator/health`
3. Requires `ADMIN` role for `DELETE /api/tasks/**`
4. Requires authentication for all other `/api/**` requests
5. Is stateless (JWT-based)
6. Disables CSRF
7. Creates a `BCryptPasswordEncoder` bean

<details>
<summary>Solution</summary>

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.DELETE, "/api/tasks/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().permitAll()
            );
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you draw the Spring Security filter chain?
- [ ] Can you write a `SecurityFilterChain` bean with URL authorization rules?
- [ ] Can you implement `UserDetails` and `UserDetailsService`?
- [ ] Can you explain why `anyRequest()` must be last?
- [ ] Can you use `@AuthenticationPrincipal` in a controller?
- [ ] Can you use `@PreAuthorize` for method-level security?

---

*Next: [12-01 — JWT Fundamentals](../12-jwt/01-jwt-fundamentals.md)*
