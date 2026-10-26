# REVIEW.md — AI Output Review Log

Complete **one review entry per slice** before moving to the next.  


---

## Slice 1 — Signup (S1)

### A. Bucket Check
List every class and method the AI produced. Tag each one:  
`Controller` / `Service` / `Repository` / `DTO` / `Security` / `Config` / `Exception`

| File / Method | Tag | In Allowed List? |
|---|---|---|
| `SignupRequest` | DTO | Yes |
| `UserResponse` | DTO | Yes |
| `AuthService.signup()` | Service | Yes |
| `AuthService.toResponse()` | Service | Yes |
| `AuthController.signup()` | Controller | Yes |

> **Important:** Anything **not** in the allowed list → **reject, re-prompt**. Don't keep it because it works.

### B. Contract Check
Compare against API contract row A1 line by line.

| Check | Pass / Fail |
|---|---|
| Path is `POST /api/auth/signup` | Pass |
| Returns **201** | Pass |
| Response shape: `{ id, email, fullName, role, createdAt }` | Pass |
| No `passwordHash` field in response | Pass |
| Role in response is always `STUDENT` | Pass |

### C. Validation & Errors

| Check | Pass / Fail |
|---|---|
| `@Valid` on controller parameter | Pass |
| `@Email`, `@NotBlank`, `@Size` on DTO fields | Pass |
| Duplicate email throws `DuplicateResourceException` | Pass |
| No new error/response wrapper class invented | Pass |
| No stack trace or exception message leaked to client | Pass |

### D. Security Specifics

| Check | Pass / Fail |
|---|---|
| Password hashed via injected `PasswordEncoder` (not `new BCryptPasswordEncoder()`) | Pass |
| `SignupRequest` has no `role` field, or it is ignored | Pass |
| Duplicate check uses normalised (lowercased/trimmed) email | Pass |
| AI did **not** touch `SecurityConfig`, add JWT class, or add `UserDetailsService` | Pass |

### E. Understanding Questions
*Answer in your own words (1–2 lines each)*

1. Where exactly is the password hashed, and why is the same password stored as a different string for two users?

   > `AuthService.signup()` hashes the password with the injected `PasswordEncoder` before saving. Encoders use a random salt, so the same plaintext password produces different stored hashes.

2. What would happen if `SignupRequest` accepted a `role` field and the AI used it?

   > If the service trusted a request-supplied role, a user could register with elevated privileges such as ADMIN. The request has no role field, and the service explicitly assigns STUDENT.

3. Which class turns `DuplicateResourceException` into a 409 response?

   > `GlobalExceptionHandler` catches `DuplicateResourceException` and returns HTTP 409.

### F. Decision

| File | Decision | Reason (if modified/rejected) |
|---|---|---|
| `SignupRequest.java` | Accepted | Added the required validation constraints. |
| `UserResponse.java` | Accepted | Existing response contains only the specified public fields. |
| `AuthService.java` | Accepted | Normalizes email, detects duplicates, hashes the password, and assigns STUDENT. |
| `AuthController.java` | Accepted | Signup endpoint returns 201 with `UserResponse`. |
| `SecurityConfig.java` | Accepted | Not modified. |

---

## Slice 2 — Login & JWT (S2)

### A. Bucket Check

| File / Method | Tag | In Allowed List? |
|---|---|---|
| `LoginRequest` | DTO | Yes |
| `LoginResponse` | DTO | Yes |
| `JwtService.generateToken()` | Security | Yes |
| `AuthService.login()` | Service | Yes |
| `AuthController.login()` | Controller | Yes |

### B. Contract Check

| Check | Pass / Fail |
|---|---|
| Path is `POST /api/auth/login` | Pass |
| Returns **200** | Pass |
| Response: `{ accessToken, tokenType: "Bearer", expiresIn }` | Pass |
| `expiresIn` matches configured value (not hardcoded) | Pass |

### C. Validation & Errors

| Check | Pass / Fail |
|---|---|
| Same 401 message for unknown email and wrong password | Pass |
| No `try/catch` swallowing exceptions silently | Pass |

### D. Security Specifics

| Check | Pass / Fail |
|---|---|
| JWT secret comes from `@Value`, no string literal secret | Pass |
| Uses jjwt 0.12 API: `.subject()`, `.claim()`, `.id()`, `.issuedAt()`, `.expiration()`, `.signWith(key)` | Pass |
| Rejects deprecated: `setSubject`, `setExpiration`, `signWith(SignatureAlgorithm.HS256, secret)`, `Jwts.parserBuilder()` | Pass |
| `jti` is a random UUID (not the user id) | Pass |
| No password or hash in token claims | Pass |
| Parse/validate method **not** written yet (that is S3) | Pass |

### E. Understanding Questions

1. Why must the error message be identical for "no such email" and "wrong password"?

   > Using the same error avoids revealing whether an email is registered, which makes account discovery harder.

2. Who can read the claims inside your JWT? Who can change them?

   > Anyone who gets the token can decode and read its claims. Only someone with the signing key can create a valid modified token.

3. What does `jti` give us that `sub` doesn't? (Think about S5.)

   > `sub` identifies the user, while `jti` uniquely identifies this particular token so it can be revoked or tracked independently.

### F. Decision

| File | Decision | Reason |
|---|---|---|
| `LoginRequest.java` | Accepted | Validates email and requires a password. |
| `LoginResponse.java` | Accepted | Includes token, Bearer type, and configured lifetime. |
| `JwtService.java` | Accepted | Issues signed JWTs with required claims using configured secret and JJWT 0.12 APIs. |
| `AuthService.java` | Accepted | Normalizes email, checks credentials uniformly, and issues a login response. |
| `AuthController.java` | Accepted | Exposes login and returns 200. |

---

## Slice 3 — JWT Filter & Protected API (S3)

### A. Bucket Check

| File / Method | Tag | In Allowed List? |
|---|---|---|
| `JwtService.parseToken()` | Security | Yes |
| `JwtAuthenticationFilter.doFilterInternal()` | Security | Yes |
| `SecurityConfig.filterChain()` | Config | Yes |
| `UserController.getCurrentUser()` | Controller | Yes |

### B. Contract Check

| Check | Pass / Fail |
|---|---|
| `GET /api/courses` works **without** token | Pass (implementation verified; runtime pending) |
| `GET /api/users/me` returns **401** with no token | Pass (implementation verified; runtime pending) |
| `GET /api/users/me` returns **200** with valid token | Pass (implementation verified; runtime pending) |
| 401 body is an `ApiError` JSON object | Pass (existing entry point wired; runtime pending) |

### C. Validation & Errors

| Check | Pass / Fail |
|---|---|
| Filter does not write a response on error — continues chain | Pass |
| Filter clears SecurityContext on bad token | Pass |

### D. Security Specifics

| Check | Pass / Fail |
|---|---|
| Spring Security 6 style: lambda DSL, `requestMatchers`, `authorizeHttpRequests` | Pass |
| Rejected: `WebSecurityConfigurerAdapter`, `antMatchers`, `authorizeRequests`, `.and()` | Pass |
| `permitAll` covers only the three specific public endpoints | Pass |
| `GET /api/courses` is public but `POST /api/courses` is NOT (watch for broad matcher) | Pass |
| Signature + expiry verified with `Jwts.parser().verifyWith(key).build().parseSignedClaims(token)` | Pass |
| Authorities use `ROLE_` prefix | Pass |
| Entry point and access-denied handler wired via `exceptionHandling(...)` | Pass |
| No `UserDetailsService`, `AuthenticationManager`, sessions or CORS added | Pass |
| `GET /api/users/me` returns `UserResponse`, not the entity | Pass |

### E. Understanding Questions

1. Why is CSRF protection disabled here, and when would disabling it be wrong?

   > CSRF protection is disabled because this API authenticates with explicitly supplied bearer tokens and uses stateless sessions. Disabling it would be unsafe for browser apps that rely on cookies automatically sent with requests.

2. Walk through what happens to a request with an expired token, step by step, until the client sees 401.

   > The JWT parser rejects the expired token, the filter clears the security context and continues the chain, then Spring Security sees the request as unauthenticated and the entry point returns a JSON 401.

3. Why don't 401/403 errors from the filter chain reach `GlobalExceptionHandler`?

   > The filter chain handles authentication and authorization failures before MVC controller execution, so Spring Security's entry point or access-denied handler handles those responses instead.

### F. Decision

| File | Decision | Reason |
|---|---|---|
| `JwtService.java` | Accepted | Parses signed claims with signature and expiry verification. |
| `JwtAuthenticationFilter.java` | Accepted | Extracts Bearer tokens, sets `ROLE_` authentication, and clears context on invalid tokens. |
| `SecurityConfig.java` | Accepted | Uses stateless Spring Security 6 configuration and exact public matchers. |
| `UserController.java` | Accepted | Returns the authenticated user's public `UserResponse`. |

---

## Slice 4 — Role Restriction (S4)

### A. Bucket Check

| File / Method | Tag | In Allowed List? |
|---|---|---|
| | | |

### B. Contract Check

| Check | Pass / Fail |
|---|---|
| `POST /api/courses` with no token → **401** | |
| `POST /api/courses` with STUDENT token → **403** | |
| `POST /api/courses` with ADMIN token → **201** | |
| 403 body is an `ApiError` JSON object | |

### D. Security Specifics

| Check | Pass / Fail |
|---|---|
| Role naming consistent: `hasRole("ADMIN")` ↔ authority `ROLE_ADMIN` | |
| If `@PreAuthorize` used: `@EnableMethodSecurity` is present | |
| Rule placed **before** `anyRequest().authenticated()` if in SecurityConfig | |
| Diff is small — AI did not rewrite the whole config | |

### E. Understanding Questions

1. A STUDENT calls `POST /api/courses`. Why does the server return 403 and not 401?

   > *Your answer:*

2. Where does the role in the JWT come from, and can the client change it?

   > *Your answer:*

### F. Decision

| File | Decision | Reason |
|---|---|---|
| | | |

---

## Slice 5 — Logout (S5) *(stretch)*

### A. Bucket Check

| File / Method | Tag | In Allowed List? |
|---|---|---|
| | | |

### B. Contract Check

| Check | Pass / Fail |
|---|---|
| `POST /api/auth/logout` returns **204** | |
| Same token returns **401** after logout | |
| Fresh login after logout returns a new working token | |

### D. Security Specifics

| Check | Pass / Fail |
|---|---|
| Logout adds entry to server-side denylist (not just tells client to delete token) | |
| Denylist keyed by **jti**, not email | |
| Map is `ConcurrentHashMap` (thread-safe) | |
| Logout endpoint is NOT under `permitAll` | |
| Filter checks the denylist and treats denylisted jti as invalid | |
| (Bonus) Expired entries are removed from the map | |

### E. Understanding Questions

1. Why does logout need server-side state if JWT is "stateless"?

   > *Your answer:*

2. What happens to the denylist when the server restarts? What if there are two server instances?

   > *Your answer:*

3. Why store the expiry time with each jti?

   > *Your answer:*

### F. Decision

| File | Decision | Reason |
|---|---|---|
| | | |

---

## No-AI Change Tasks



**Files changed:**

| File | What changed and why |
|---|---|
| | |

**Explanation (your own words):**
> *Describe what you changed and why it works.*
