
**One-line analogy for each piece:**
- `PasswordEncoder` = a one-way shredder that turns a password into something unreadable, but can *compare* a fresh password against the shredded one.
- `JwtUtil` = the machine that prints and checks wristbands.
- `AuthController` (/login) = the entry gate where you trade your ticket for a wristband.
- `JwtAuthFilter` = the guard standing at every door checking the wristband.
- `SecurityFilterChain` = the rulebook telling the guard which doors need a wristband check at all.

---

## 1. Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
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

---

## 2. Entity — add these two fields

```java
private String password;
private String role;   // e.g. "ROLE_USER" or "ROLE_ADMIN"
```

---

## 3 & 8. SecurityConfig.java (PasswordEncoder Bean + the Rulebook)

```java
package com.yourapp.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {

    @Autowired
    private JwtAuthFilter jwtAuthFilter;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/users").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

**Why `.csrf(disable)`:** CSRF protection is for browser-session apps. A stateless JWT REST API doesn't need it.
**Why `STATELESS`:** The server remembers nothing between requests — every request proves itself with its own token.
**Why `.addFilterBefore(...)`:** Forces our custom filter to run before Spring's own default filter.

---

## 4. Hash password on creation (inside your Service.java)

```java
@Autowired
private PasswordEncoder passwordEncoder;

public User createUser(User user) {
    user.setPassword(passwordEncoder.encode(user.getPassword()));
    user.setRole("ROLE_USER");
    return userRepository.save(user);
}
```

---

## 5. JwtUtil.java (the wristband machine)

```java
package com.yourapp.config;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;

@Component
public class JwtUtil {

    private final SecretKey secretKey = Keys.hmacShaKeyFor("ChangeThisToALongRandomSecretKeyString".getBytes());
    private final long EXPIRATION_TIME = 1000 * 60 * 60; // 1 hour

    public String generateToken(String email, String role) {
        return Jwts.builder()
                .subject(email)
                .claim("role", role)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
                .signWith(secretKey)
                .compact();
    }

    public String extractEmail(String token) {
        return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload()
                .getSubject();
    }

    public boolean isTokenValid(String token) {
        try {
            Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

---

## 6. AuthController.java (the entry gate)

```java
package com.yourapp.controller;

import com.yourapp.config.JwtUtil;
import com.yourapp.dto.LoginRequest;
import com.yourapp.dto.LoginResponse;
import com.yourapp.entity.User;
import com.yourapp.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

import java.util.Optional;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired private UserRepository userRepository;
    @Autowired private PasswordEncoder passwordEncoder;
    @Autowired private JwtUtil jwtUtil;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest loginRequest) {

        Optional<User> userOptional = userRepository.findAll().stream()
                .filter(u -> u.getEmail().equals(loginRequest.getEmail()))
                .findFirst();

        if (userOptional.isEmpty()) {
            return ResponseEntity.status(401).body("Invalid email or password");
        }

        User user = userOptional.get();

        if (!passwordEncoder.matches(loginRequest.getPassword(), user.getPassword())) {
            return ResponseEntity.status(401).body("Invalid email or password");
        }

        String token = jwtUtil.generateToken(user.getEmail(), user.getRole());
        return ResponseEntity.ok(new LoginResponse(token, user.getEmail()));
    }
}
```

DTOs needed:
```java
@Data
public class LoginRequest {
    private String email;
    private String password;
}

@Data @AllArgsConstructor
public class LoginResponse {
    private String token;
    private String email;
}
```

---

## 7. JwtAuthFilter.java (the guard at every door)

```java
package com.yourapp.config;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;

    public JwtAuthFilter(JwtUtil jwtUtil) {
        this.jwtUtil = jwtUtil;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7).trim();

            if (jwtUtil.isTokenValid(token)) {
                String email = jwtUtil.extractEmail(token);

                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(email, null, List.of());

                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 8 (bonus). Never return password in API responses

```java
@Data @AllArgsConstructor
public class UserResponseDTO {
    private Long id;
    private String name;
    private String email;
    private String phoneNumber;
    private String role;
}
```

---

## Testing Checklist (do this every time)

1. POST /api/users → check returned/stored password is a BCrypt hash, not plain text.
2. POST /api/auth/login with correct credentials → expect a JWT token back.
3. POST /api/auth/login with wrong password → expect 401 with generic message.
4. Call a protected endpoint without Authorization header → expect 401/403.
5. Call it with Authorization: Bearer <token> → expect success.
6. Tamper one character in the token and resend → expect rejection.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| /api/auth/login itself returns 401 | Spring Security secures everything by default once added | Explicitly permitAll() the login/signup paths |
| Token copy-paste sometimes fails | Extra whitespace in header value | Add .trim() after substring(7) |
| substring(7) confusion | Java indices are 0-based; "Bearer " is exactly 7 characters | Count from index 0, not 1 |
| Password visible in GET response | Returning Entity directly instead of a response DTO | Always map to a DTO without password field |
