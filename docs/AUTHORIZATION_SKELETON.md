# Role-Based Authorization — Reusable Skeleton & Mental Model

> Companion to the JWT Authentication skeleton. Authentication answers "who are you"; this answers "what are you allowed to do". Use this AFTER JWT login/token verification is already working.

---

## The Mental Model (memorize this order)

Entity already has a role field (added during Authentication setup)

JwtUtil: add extractRole(token) — role was already embedded at login time

JwtAuthFilter: extract the role, wrap it in SimpleGrantedAuthority,
pass it as the 3rd argument to UsernamePasswordAuthenticationToken
(this is the ONE line that upgrades "authenticated" into "authenticated with a role")

SecurityConfig: add specific requestMatchers rules ABOVE the generic .anyRequest() rule,
using .hasRole("X") or .hasAnyRole("X","Y")

(Optional, method-level) @PreAuthorize on specific Controller/Service methods
for finer-grained checks than URL-pattern matching allows






**One-line analogy:** Authentication gives you an employee badge. Authorization is the rule painted on each door — "Server Room: IT staff only." The badge doesn't change; which doors open depends on what's printed on the badge.

---

## Where exactly does each piece live?

| What | File | What changes |
|---|---|---|
| Role field on the user | entity/User.java | Already there from Authentication setup — nothing new |
| Pull role out of the token | config/JwtUtil.java | Add extractRole(token) method |
| Attach the role to the request's identity | config/JwtAuthFilter.java | Change 3rd constructor argument from List.of() to List.of(new SimpleGrantedAuthority(role)) |
| Decide which roles can hit which URLs | config/SecurityConfig.java | Add .requestMatchers(...).hasRole("X") rules, ordered most-specific-first |
| Decide inside a specific method (rare, advanced) | Controller or Service method | @PreAuthorize("hasRole('ADMIN')") above the method |

---

## Step-by-step code

### 1. JwtUtil.java — extract the role from the token

```java
public String extractRole(String token) {
    return Jwts.parser()
            .verifyWith(secretKey)
            .build()
            .parseSignedClaims(token)
            .getPayload()
            .get("role", String.class);
}
```

### 2. JwtAuthFilter.java — turn the role into a Spring "authority"

```java
import org.springframework.security.core.authority.SimpleGrantedAuthority;

if (jwtUtil.isTokenValid(token)) {
    String email = jwtUtil.extractEmail(token);
    String role = jwtUtil.extractRole(token);

    UsernamePasswordAuthenticationToken authToken =
            new UsernamePasswordAuthenticationToken(
                    email,
                    null,
                    List.of(new SimpleGrantedAuthority(role))
            );

    SecurityContextHolder.getContext().setAuthentication(authToken);
}
```

**Why SimpleGrantedAuthority:** Spring Security doesn't understand a plain String role — it needs an object implementing GrantedAuthority. SimpleGrantedAuthority is Spring's ready-made wrapper for exactly this.

### 3. SecurityConfig.java — the actual door rules

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**", "/api/users", "/h2-console/**").permitAll()
    .requestMatchers(HttpMethod.DELETE, "/api/users/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.PUT, "/api/products/**").hasAnyRole("ADMIN", "MANAGER")
    .anyRequest().authenticated()
)
```

**Ordering rule — most specific rule first, most general rule last.** Spring checks top to bottom and stops at the first match.

**Naming convention:** stored role is "ROLE_ADMIN", but .hasRole("ADMIN") — Spring automatically prepends ROLE_. Don't write .hasRole("ROLE_ADMIN") (double prefix = never matches).

### 4. (Optional) Method-level checks with @PreAuthorize

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public ResponseEntity<String> deleteUser(@PathVariable Long id) { ... }
```

Requires @EnableMethodSecurity on SecurityConfig. Use URL-pattern rules for simple cases; @PreAuthorize when the rule needs data from inside the method.

---

## Testing Checklist

1. Promote one test user's role to ROLE_ADMIN directly in the DB (H2 console or SQL).
2. Log in as that admin → get token → call the restricted endpoint → expect success.
3. Create a second, normal ROLE_USER → log in → call the same restricted endpoint → expect 403 Forbidden.
4. Call the same endpoint with no token at all → expect 401 Unauthorized.
5. Confirm a permitAll() path still works for anyone, regardless of role.

---

## 401 vs 403 — don't mix these up

| Code | Means | When it fires here |
|---|---|---|
| 401 Unauthorized | "I don't know who you are" | No token, or invalid/expired/tampered token |
| 403 Forbidden | "I know who you are, but you're not allowed" | Valid token, but role doesn't match .hasRole(...) |

If you're expecting a 403 and get a 401 instead, the token itself is failing validation — check that first.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| ADMIN-restricted rule never applies | .anyRequest().authenticated() placed before the specific rule | Move specific rules above .anyRequest() |
| .hasRole("ROLE_ADMIN") never matches | Double ROLE_ prefix | Use .hasRole("ADMIN") |
| Role always null in filter | Forgot to update 3rd constructor argument | Pass List.of(new SimpleGrantedAuthority(role)) |
| Test admin loses role after restart | H2 in-memory data wiped on restart | Re-create user and re-run UPDATE SQL after every restart |
| H2 console returns 403 with Security added | /h2-console/** not in permitAll(), frame options block UI | Add to permitAll() AND disable frame options (dev-only) |
