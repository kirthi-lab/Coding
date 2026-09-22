# 7. Security with Spring Security & JWT

[← Back](06-validation-exception-handling.md) | [Index](README.md) | Next → [8. Advanced Features](08-advanced-features.md)

Now we protect the API. You'll learn how Spring Security works, then build stateless **JWT** authentication with role-based authorization.

---

## 7.1 Authentication vs authorization

- **Authentication** — *who are you?* Verifying identity (username + password).
- **Authorization** — *what are you allowed to do?* Checking permissions/roles.

---

## 7.2 How Spring Security works

Spring Security inserts a **filter chain** in front of your controllers. Every request passes through filters that authenticate and authorize before reaching your code.

```
  Request
     │
     ▼
 ┌───────────────────────────────────────────────┐
 │            Security Filter Chain                │
 │  ┌──────────────┐   ┌───────────────────────┐  │
 │  │ JWT Filter    │─► │ Authentication check   │  │
 │  │ (reads token) │   │ + Authorization check  │  │
 │  └──────────────┘   └───────────────────────┘  │
 └───────────────────────────────────────────────┘
     │  (only if allowed)
     ▼
  Controller
```

Add the starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Just adding this secures **every** endpoint with HTTP Basic auth and a generated password (printed at startup). We'll replace that with JWT.

---

## 7.3 Why JWT for REST APIs

Traditional session auth stores state on the server. REST APIs prefer **stateless** auth: the client sends a signed token with each request, and the server verifies it without storing session state. That token is a **JWT** (JSON Web Token).

A JWT has three base64url parts separated by dots: `header.payload.signature`.

```
   eyJhbGciOiJIUzI1NiJ9  .  eyJzdWIiOiJhZGEiLCJyb2xlIjoiVVNFUiJ9  .  <signature>
        header                       payload (claims)                  signature
```

- **Header** — algorithm (e.g., HS256).
- **Payload** — claims: subject (username), roles, expiry.
- **Signature** — the server signs header+payload with a secret key so it can detect tampering.

### The login + request flow

```
  1. POST /auth/login {username, password}
        │
        ▼
     Server verifies credentials ──► issues signed JWT
        │
        ▼
  2. Client stores the JWT
        │
        ▼
  3. Every request: Authorization: Bearer <JWT>
        │
        ▼
     JWT filter validates signature + expiry ──► sets authentication
        │
        ▼
     Controller runs (if authorized)
```

---

## 7.4 Dependencies for JWT

Add the JJWT library:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

Config in `application.properties` (never hard-code secrets — use environment variables in production):

```properties
app.jwt.secret=${JWT_SECRET:a-very-long-dev-only-secret-key-at-least-256-bits-long!!}
app.jwt.expiration-ms=3600000
```

---

## 7.5 The User entity and roles

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;   // BCrypt-hashed, never plain text

    @Enumerated(EnumType.STRING)
    private Role role = Role.USER;

    // getters/setters, constructors
}

public enum Role { USER, ADMIN }
```

Load users via `UserDetailsService`:

```java
@Service
public class AppUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;
    public AppUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())
            .roles(user.getRole().name())   // ROLE_USER / ROLE_ADMIN
            .build();
    }
}
```

---

## 7.6 The JWT utility

Generates and validates tokens.

```java
package com.example.demo.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

@Component
public class JwtService {
    private final SecretKey key;
    private final long expirationMs;

    public JwtService(@Value("${app.jwt.secret}") String secret,
                      @Value("${app.jwt.expiration-ms}") long expirationMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expirationMs = expirationMs;
    }

    public String generateToken(String username, String role) {
        Date now = new Date();
        return Jwts.builder()
            .subject(username)
            .claim("role", role)
            .issuedAt(now)
            .expiration(new Date(now.getTime() + expirationMs))
            .signWith(key)
            .compact();
    }

    public String extractUsername(String token) {
        return parse(token).getPayload().getSubject();
    }

    public boolean isValid(String token) {
        try {
            parse(token);   // throws if signature/expiry invalid
            return true;
        } catch (JwtException e) {
            return false;
        }
    }

    private Jws<Claims> parse(String token) {
        return Jwts.parser().verifyWith(key).build().parseSignedClaims(token);
    }
}
```

---

## 7.7 The JWT authentication filter

Runs once per request; reads the token and sets the authenticated user.

```java
package com.example.demo.security;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;

@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            if (jwtService.isValid(token)) {
                String username = jwtService.extractUsername(token);
                UserDetails user = userDetailsService.loadUserByUsername(username);
                var auth = new UsernamePasswordAuthenticationToken(
                    user, null, user.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(request, response);
    }
}
```

---

## 7.8 The security configuration

Spring Boot 3.x uses a `SecurityFilterChain` bean (no more `WebSecurityConfigurerAdapter`).

```java
package com.example.demo.security;

import org.springframework.context.annotation.*;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableMethodSecurity   // enables @PreAuthorize
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;
    public SecurityConfig(JwtAuthFilter jwtAuthFilter) { this.jwtAuthFilter = jwtAuthFilter; }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())   // stateless API: CSRF not needed
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/h2-console/**").permitAll()  // public
                .requestMatchers("/api/admin/**").hasRole("ADMIN")          // admin only
                .anyRequest().authenticated())                             // everything else needs login
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();   // hash passwords, never store plain text
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
        return cfg.getAuthenticationManager();
    }
}
```

---

## 7.9 The auth controller (register + login)

```java
package com.example.demo.auth;

import com.example.demo.security.JwtService;
import org.springframework.security.authentication.*;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

record RegisterRequest(String username, String password) {}
record LoginRequest(String username, String password) {}
record AuthResponse(String token) {}

@RestController
@RequestMapping("/auth")
public class AuthController {
    private final UserRepository userRepository;
    private final PasswordEncoder encoder;
    private final AuthenticationManager authManager;
    private final JwtService jwtService;

    public AuthController(UserRepository userRepository, PasswordEncoder encoder,
                          AuthenticationManager authManager, JwtService jwtService) {
        this.userRepository = userRepository;
        this.encoder = encoder;
        this.authManager = authManager;
        this.jwtService = jwtService;
    }

    @PostMapping("/register")
    public AuthResponse register(@RequestBody RegisterRequest req) {
        User user = new User();
        user.setUsername(req.username());
        user.setPassword(encoder.encode(req.password()));   // hash it
        user.setRole(Role.USER);
        userRepository.save(user);
        return new AuthResponse(jwtService.generateToken(user.getUsername(), user.getRole().name()));
    }

    @PostMapping("/login")
    public AuthResponse login(@RequestBody LoginRequest req) {
        // throws if credentials are wrong
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.username(), req.password()));
        User user = userRepository.findByUsername(req.username()).orElseThrow();
        return new AuthResponse(jwtService.generateToken(user.getUsername(), user.getRole().name()));
    }
}
```

---

## 7.10 Method-level authorization

With `@EnableMethodSecurity`, protect individual methods:

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    @GetMapping("/stats")
    @PreAuthorize("hasRole('ADMIN')")
    public Map<String, Object> stats() { ... }
}
```

Access the current user in any handler:

```java
@GetMapping("/me")
public String me(java.security.Principal principal) {
    return "Logged in as " + principal.getName();
}
```

---

## 7.11 Test the flow

```bash
# Register (returns a token)
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"ada","password":"secret123"}'
# → {"token":"eyJhbGciOiJIUzI1NiJ9..."}

# Call a protected endpoint WITHOUT a token → 401 Unauthorized
curl http://localhost:8080/api/tasks

# Call WITH the token → 200 OK
curl http://localhost:8080/api/tasks \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."
```

---

## Mini-project: secure the Task API

1. Add the `User` entity, `UserRepository`, and `AppUserDetailsService`.
2. Wire up `JwtService`, `JwtAuthFilter`, and `SecurityConfig`.
3. Make `/auth/**` public and everything under `/api/**` require authentication.
4. Add an admin-only endpoint `GET /api/admin/all-tasks` protected with `@PreAuthorize("hasRole('ADMIN')")`.
5. Associate each created task with the logged-in user (via `Principal`).
6. Verify: unauthenticated requests get 401, a USER can't hit admin routes (403), an ADMIN can.

## Key takeaways

- Spring Security guards requests via a filter chain before they reach controllers.
- Use stateless JWT auth for REST APIs: log in once, send `Authorization: Bearer <token>` thereafter.
- Always hash passwords with BCrypt; never store or log plain text.
- Configure security with a `SecurityFilterChain` bean (Spring Boot 3.x style).
- Use `@PreAuthorize` for fine-grained, method-level authorization.
- Keep the JWT secret out of source code — inject it from the environment.

Next → [8. Advanced Features](08-advanced-features.md)
