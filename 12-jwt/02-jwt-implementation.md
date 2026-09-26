# Lesson 12-02 — JWT Implementation in Spring Boot

## 🎯 Learning Objective

Build a complete, production-quality JWT authentication system from scratch. Understand every component and how they connect.

---

## 🔍 The Complete JWT Flow

```
REGISTRATION:
Client → POST /api/auth/register → Save user with hashed password
Client ← UserResponse (no token — must login)

LOGIN:
Client → POST /api/auth/login (email + password)
Server: Verify password against hash
Server: Generate JWT token
Client ← { token, refreshToken, expiresIn }

AUTHENTICATED REQUESTS:
Client → GET /api/tasks (Authorization: Bearer <token>)
Server: JwtAuthFilter extracts token
Server: Validate token (signature + expiry)
Server: Load user from token's subject (email)
Server: Set authentication in SecurityContextHolder
Server: Request proceeds to controller
Client ← task data

TOKEN EXPIRY:
Client → POST /api/auth/refresh (refreshToken)
Server: Validate refresh token
Server: Generate new access token
Client ← { newToken, expiresIn }
```

---

## 📦 Dependencies

```xml
<!-- JJWT — JWT library for Java -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

---

## ⚙️ Configuration

```yaml
# application.yml
app:
  jwt:
    secret: your-very-long-secret-key-must-be-at-least-256-bits-for-HS256
    expiration: 86400000    # 24 hours in milliseconds
    refresh-expiration: 604800000  # 7 days
```

```java
@ConfigurationProperties(prefix = "app.jwt")
@Configuration
public class JwtProperties {
    private String secret;
    private long expiration;
    private long refreshExpiration;
    
    // Getters and setters
    public String getSecret() { return secret; }
    public void setSecret(String secret) { this.secret = secret; }
    public long getExpiration() { return expiration; }
    public void setExpiration(long expiration) { this.expiration = expiration; }
    public long getRefreshExpiration() { return refreshExpiration; }
    public void setRefreshExpiration(long refreshExpiration) { this.refreshExpiration = refreshExpiration; }
}
```

---

## 🔧 JwtService

```java
package com.example.taskapi.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {
    
    private final JwtProperties jwtProperties;
    
    public JwtService(JwtProperties jwtProperties) {
        this.jwtProperties = jwtProperties;
    }
    
    // ─── Token Generation ──────────────────────────────────────────
    
    public String generateAccessToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, jwtProperties.getExpiration());
    }
    
    public String generateAccessToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return generateToken(extraClaims, userDetails, jwtProperties.getExpiration());
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, jwtProperties.getRefreshExpiration());
    }
    
    private String generateToken(Map<String, Object> extraClaims, UserDetails userDetails, long expiration) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername()) // Email as subject
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSignKey())
            .compact();
    }
    
    // ─── Token Validation ──────────────────────────────────────────
    
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }
    
    public boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    // ─── Claims Extraction ─────────────────────────────────────────
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSignKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
    
    private SecretKey getSignKey() {
        byte[] keyBytes = java.util.Base64.getDecoder().decode(jwtProperties.getSecret());
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

## 🔧 JWT Authentication Filter

```java
package com.example.taskapi.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }
    
    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {
        
        // Step 1: Extract Authorization header
        final String authHeader = request.getHeader("Authorization");
        
        // Step 2: Skip if no Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // Continue chain — no token
            return;
        }
        
        // Step 3: Extract token (remove "Bearer " prefix)
        final String jwt = authHeader.substring(7);
        
        try {
            // Step 4: Extract username from token
            final String userEmail = jwtService.extractUsername(jwt);
            
            // Step 5: If we have an email and not already authenticated
            if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                
                // Step 6: Load user from database
                UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);
                
                // Step 7: Validate token
                if (jwtService.isTokenValid(jwt, userDetails)) {
                    
                    // Step 8: Create authentication object
                    UsernamePasswordAuthenticationToken authToken = 
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    
                    // Step 9: Set authentication in SecurityContext
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            // Invalid token — don't set authentication, request will fail at authorization
            // Optionally set error response:
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Invalid or expired token\"}");
            return;
        }
        
        // Step 10: Continue filter chain
        filterChain.doFilter(request, response);
    }
}
```

---

## 🔧 Register the Filter

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;
    
    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter, 
                          UserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            // Register JWT filter BEFORE Spring Security's default auth filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
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

---

## 🔧 Auth Controller and Service

```java
// DTOs
public record LoginRequest(String email, String password) {}
public record RegisterRequest(String name, String email, String password) {}
public record AuthResponse(String token, String refreshToken, long expiresIn) {}

// Controller
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    private final AuthService authService;
    
    public AuthController(AuthService authService) {
        this.authService = authService;
    }
    
    @PostMapping("/register")
    public ResponseEntity<UserResponse> register(@RequestBody @Valid RegisterRequest request) {
        return ResponseEntity.status(201).body(authService.register(request));
    }
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
        return ResponseEntity.ok(authService.refresh(request.refreshToken()));
    }
}

// Service
@Service
public class AuthService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authManager;
    
    public AuthService(UserRepository userRepository, PasswordEncoder passwordEncoder,
                       JwtService jwtService, AuthenticationManager authManager) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
        this.authManager = authManager;
    }
    
    @Transactional
    public UserResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException(request.email());
        }
        
        User user = new User(
            request.name(),
            request.email(),
            passwordEncoder.encode(request.password())
        );
        
        return mapToResponse(userRepository.save(user));
    }
    
    public AuthResponse login(LoginRequest request) {
        // This validates credentials and throws if wrong
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );
        
        // If we get here, credentials were valid
        User user = userRepository.findByEmail(request.email()).orElseThrow();
        AppUserDetails userDetails = new AppUserDetails(user);
        
        // Add custom claims (optional)
        Map<String, Object> claims = Map.of(
            "userId", user.getId(),
            "role", user.getRole()
        );
        
        String accessToken = jwtService.generateAccessToken(claims, userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);
        
        return new AuthResponse(accessToken, refreshToken, 86400000L);
    }
    
    public AuthResponse refresh(String refreshToken) {
        String email = jwtService.extractUsername(refreshToken);
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UserNotFoundException(email));
        
        AppUserDetails userDetails = new AppUserDetails(user);
        
        if (!jwtService.isTokenValid(refreshToken, userDetails)) {
            throw new InvalidTokenException("Refresh token is invalid or expired");
        }
        
        String newAccessToken = jwtService.generateAccessToken(userDetails);
        return new AuthResponse(newAccessToken, refreshToken, 86400000L);
    }
    
    private UserResponse mapToResponse(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
    }
}
```

---

## 🧪 Testing the Auth Flow

```bash
# Register
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com", "password": "SecurePass123"}'

# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@example.com", "password": "SecurePass123"}'
# Response: {"token": "eyJhbGci...", "refreshToken": "eyJhbGci...", "expiresIn": 86400000}

# Authenticated request
curl http://localhost:8080/api/tasks \
  -H "Authorization: Bearer eyJhbGci..."
```

---

## 💥 Break It Exercise

This JWT filter has a security vulnerability:

```java
if (jwtService.isTokenValid(jwt, userDetails)) {
    UsernamePasswordAuthenticationToken authToken = 
        new UsernamePasswordAuthenticationToken(
            userDetails,
            jwt, // ← Stores raw token as credentials
            userDetails.getAuthorities()
        );
    SecurityContextHolder.getContext().setAuthentication(authToken);
}
```

What's the issue with storing `jwt` as credentials?

<details>
<summary>Answer</summary>

While not catastrophically wrong, storing the raw JWT as the `credentials` field of `UsernamePasswordAuthenticationToken` is not a good practice:

1. The credentials field is sometimes logged or exposed in debug output — this could leak the token
2. Spring Security may call `eraseCredentials()` on the authentication object after authentication, potentially nulling it out
3. The convention is to pass `null` as credentials for token-based auth (the token itself is the credential, not something to store separately)

Fix:
```java
new UsernamePasswordAuthenticationToken(
    userDetails,
    null,                        // credentials = null for JWT auth
    userDetails.getAuthorities()
);
```

</details>

---

## ✅ Completion Checklist

- [ ] Can you explain the JWT auth flow (register → login → request → refresh)?
- [ ] Can you implement `JwtService` with generate/validate/extract methods?
- [ ] Can you implement `JwtAuthenticationFilter` (extends `OncePerRequestFilter`)?
- [ ] Can you register the filter in `SecurityFilterChain`?
- [ ] Can you implement the full Auth controller and service?
- [ ] Did you test the complete flow with curl?

---

*Next: [13-01 — JUnit and Mockito](../13-testing/01-junit-and-mockito.md)*
