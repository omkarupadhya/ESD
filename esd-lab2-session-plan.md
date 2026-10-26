# ESD Lab 2: Spring Boot Auth, Built with Restricted AI

**Audience:** TAs running the lab
**Duration:** 90 minutes
**Prerequisites:** Lab 1 (basic Spring Boot REST APIs + DB) and the Auth lecture (hashing, JWT)

---

## 0. Lab goals

By the end of the session, every student should have:

1. A Spring Boot app with **signup, login, logout**, a **JWT filter**, one **public** API, one **protected** API and one **role-restricted** API.
2. Built it **one slice at a time with AI**, using a fixed prompt template that limits what the AI may produce.
3. **Judged** every AI output with a review sheet before accepting it, and logged the decision.
4. Made one change **without AI** to show they understand the code.

The point of the lab is the process, not just working code. A student who pastes one giant prompt and gets green tests has missed the lab.

---

## 1. Problem statement

You are adding authentication to a small course catalogue service. Users sign up and log in with email and password. The server issues its own JWT. Protected APIs need a valid token, one API needs the ADMIN role, and logging out must make the token unusable.

### 1.1 Data

**`users` table** (entity provided in starter code)

| Column | Type | Notes |
|---|---|---|
| id | BIGINT, PK, auto | |
| email | VARCHAR(255), unique, not null | Login identifier |
| password_hash | VARCHAR(255), not null | BCrypt hash, never plaintext |
| full_name | VARCHAR(100), not null | |
| role | VARCHAR(20), not null | `STUDENT` or `ADMIN` |
| created_at | TIMESTAMP, not null | |

**`courses` table** (entity, repository and `GET` already provided)

| Column | Type | Notes |
|---|---|---|
| id | BIGINT, PK, auto | |
| code | VARCHAR(20), unique, not null | e.g. `CS501` |
| title | VARCHAR(150), not null | |
| description | VARCHAR(1000) | |

A seeded ADMIN user is created at startup (`admin@esd.local`, password from config).

### 1.2 API contract

All error responses use the provided `ApiError` shape (section 3.3).

| # | Method & path | Access | Request body | Success | Errors |
|---|---|---|---|---|---|
| A1 | `POST /api/auth/signup` | Public | `{ email, password, fullName }` | **201** `UserResponse` | 400 validation, 409 email exists |
| A2 | `POST /api/auth/login` | Public | `{ email, password }` | **200** `LoginResponse` | 400 validation, 401 bad credentials |
| A3 | `POST /api/auth/logout` | Any logged-in user | none | **204** | 401 no/invalid token |
| P1 | `GET /api/courses` | **Public** | none | **200** list of courses | |
| P2 | `GET /api/users/me` | **Any logged-in user** | none | **200** `UserResponse` | 401 no/invalid/expired/logged-out token |
| P3 | `POST /api/courses` | **ADMIN only** | `{ code, title, description }` | **201** course | 400, 401, **403** for STUDENT, 409 code exists |

**DTOs**

```text
SignupRequest  { email: valid email, required
                 password: required, 8–64 chars
                 fullName: required, not blank, max 100 }

UserResponse   { id, email, fullName, role, createdAt }      // never contains the password hash

LoginRequest   { email: required, password: required }

LoginResponse  { accessToken, tokenType: "Bearer", expiresIn: <seconds> }
```

**Rules that are part of the contract**

- Signup **always** creates a `STUDENT`. A `role` field in the request must be ignored, so nobody can sign up as ADMIN.
- Email is stored in lowercase and trimmed.
- Login failure returns the **same** 401 message whether the email is unknown or the password is wrong: `"Invalid email or password"`.
- The JWT contains: `sub` = email, `role` claim, `jti` (unique ID), `iat`, `exp`. It is signed with HS256 using a secret from configuration.
- Token is sent as `Authorization: Bearer <token>`.
- After logout, the same token gets 401 on every protected API, even though it has not expired.

---

## 2. Slices

Each slice is one prompt (or a short follow-up), one review, one test run.

| Slice | Delivers | APIs | Priority | Time |
|---|---|---|---|---|
| S1 Signup | Registration with BCrypt | A1 | Core | 10 min |
| S2 Login | Password check + JWT issuing | A2 | Core | 12 min |
| S3 Protect | JWT filter, security rules, current user | P2 (P1 stays public) | Core | 15 min |
| S4 Role | ADMIN-only course creation | P3 | Expected | 5 min |
| S5 Logout | Token denylist | A3 | Stretch | 8 min |

Students do not move to the next slice until the current slice's tests pass and its review entry is written.

---

## 3. Starter code (provided before any prompting)

The starter repo removes boilerplate so student time goes into security logic and review. TAs: make sure every student has cloned it and `./mvnw spring-boot:run` starts cleanly **before** the lab.

### 3.1 Project layout

```text
com.esd.lab2
├── config/       SecurityConfig.java      (temporary: permits everything, see 3.4)
│                 AdminSeeder.java         (creates the ADMIN user at startup)
├── controller/   CourseController.java    (GET and POST /api/courses, both unprotected for now)
├── service/      CourseService.java
├── dto/          CourseRequest.java, CourseResponse.java
├── entity/       User.java, Role.java (enum), Course.java
├── repository/   UserRepository.java      (findByEmail, existsByEmail)
│                 CourseRepository.java
├── exception/    ApiError.java
│                 GlobalExceptionHandler.java
│                 DuplicateResourceException.java      -> 409
│                 InvalidCredentialsException.java     -> 401
│                 ResourceNotFoundException.java       -> 404
└── security/     JsonAuthenticationEntryPoint.java    (writes ApiError for 401)
                  JsonAccessDeniedHandler.java         (writes ApiError for 403)
```

The `security/` package has the two handlers but **no** JWT service and **no** filter. Those are AI-generated in S2 and S3.

### 3.2 Dependencies (`pom.xml`)

- `spring-boot-starter-web`, `-data-jpa`, `-validation`, `-security`
- PostgreSQL driver (as in Lab 1); H2 for the test profile
- `io.jsonwebtoken:jjwt-api`, `jjwt-impl`, `jjwt-jackson` (0.12.x)
- `spring-boot-starter-test`, `spring-security-test`

**No other dependency may be added during the lab.**

### 3.3 Error handling

```java
public record ApiError(
    Instant timestamp,
    int status,
    String error,        // e.g. "Conflict"
    String message,      // human-readable
    String path,
    Map<String, String> fieldErrors   // only for 400 validation errors, else null
) {}
```

`GlobalExceptionHandler` already maps: `MethodArgumentNotValidException` → 400 with `fieldErrors`, the three custom exceptions → 409 / 401 / 404, and anything else → 500 with a generic message.

> Why the two security handlers exist: 401 and 403 raised inside the Spring Security filter chain never reach `@RestControllerAdvice`. Without these handlers, students get empty 403 responses and wonder why. The AI must **wire** them in S3, not write new ones.

### 3.4 Temporary `SecurityConfig`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // TEMPORARY: replaced in Slice 3
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
        return http.build();
    }
}
```

### 3.5 Configuration (`application.yml`)

```yaml
jwt:
  secret: ${JWT_SECRET}            # Base64, at least 32 bytes; set in env / run config
  expiration-seconds: 3600
admin:
  password: ${ADMIN_PASSWORD}
```

### 3.6 Instructor test suite (`src/test`)

Black-box MockMvc tests, written only against the contract, so they pass for any correct implementation. One class per slice:

```bash
./mvnw test -Dtest='S1*'      # signup only
./mvnw test -Dtest='S2*'      # login only
./mvnw test                   # everything
```

| Class | Checks |
|---|---|
| `S1SignupTest` | 201 + body shape; no password/hash field in response; DB value ≠ plaintext and starts with `$2`; 409 duplicate (case-insensitive); 400 + `fieldErrors` for bad email / short password / blank name; `role: "ADMIN"` in request still creates STUDENT |
| `S2LoginTest` | 200 + `tokenType`, `expiresIn`; token has `sub`, `role`, `jti`, `exp`; same 401 message for unknown email and wrong password |
| `S3ProtectTest` | P1 200 without token; P2 401 without token, with garbage, with tampered signature, with expired token; P2 200 with valid token; 401 body is `ApiError` |
| `S4RoleTest` | P3 401 no token; 403 STUDENT token (body is `ApiError`); 201 ADMIN token |
| `S5LogoutTest` | login → P2 200 → logout 204 → P2 401 with the same token; a fresh login works again |

### 3.7 Logging files (in repo root)

- `PROMPTS.md`: every prompt sent to the AI, in order, labelled by slice.
- `REVIEW.md`: one filled review sheet (section 5) per slice.

### 3.8 AI tool

Any free chat assistant is fine. **Chat mode is preferred**: the student pastes context in and copies code out, which keeps the restriction visible. Students using an IDE agent must still give it one slice at a time and review the full diff before accepting.

---

## 4. The prompt template

Every prompt follows this template. Students copy it from `PROMPTS.md` and fill in the blanks. Free-form prompts like "add JWT auth to my app" are not allowed.

```text
CONTEXT
Spring Boot 3.x, Spring Security 6, Java 17+, jjwt 0.12.x. No other libraries.
Package layout: <paste the tree from section 3.1>
Existing code you must use (do not rewrite it):
<paste the relevant existing classes>

TASK
Implement only: <one API or one component>
Contract: <paste the row(s) and DTOs from the contract>

ALLOWED TO CREATE OR CHANGE
<exact list of files/methods>

RULES
- Do not create or modify any file not listed above.
- Use the existing exceptions in com.esd.lab2.exception. Do not create new error or response-wrapper classes.
- Use DTOs for request and response. Never return an entity from a controller.
- Use jakarta.validation annotations for input validation.
- Use only APIs that exist in Spring Security 6 and jjwt 0.12. No deprecated methods.
- If something is unclear, add a TODO comment instead of guessing.

OUTPUT
Each file in full, with its path. Then a list of anything you assumed.
```

---

## 5. Review sheet (filled per slice in `REVIEW.md`)

**A. Bucket check.** List every class and method the AI produced and tag each one: Controller / Service / Repository / DTO / Security / Config / Exception. Anything not in the slice's "allowed" list: **reject it, then re-prompt**. Don't keep it just because it works.

**B. Contract check.** Paths, HTTP methods, request and response fields, status codes. Compare line by line against section 1.2.

**C. Validation and errors.**
- `@Valid` on the controller parameter, and `jakarta.validation` annotations on the DTO.
- Errors are thrown as the provided exceptions and reach `GlobalExceptionHandler`.
- No `try/catch` that swallows an exception or returns `null`. No stack traces or exception messages leaked to the client.
- No new `ErrorResponse` / `ApiResponse<T>` wrapper invented by the AI.

**D. Security specifics** (see each slice for details).

**E. Understanding.** Answer the slice's questions in your own words, 1–2 lines each. TAs spot-check these.

**F. Decision.** For each file: **Accepted / Modified (what and why) / Rejected (why)**.

---

## 6. Slice details

### S1: Signup (A1)

**Give the AI:** the template; `User` entity, `Role` enum, `UserRepository`; contract row A1; `SignupRequest` and `UserResponse` definitions; the signup rules (always STUDENT, email lowercase).

**Allowed:** `dto/SignupRequest`, `dto/UserResponse`, `service/AuthService` (method `signup`), `controller/AuthController` (method `signup`).

**Example prompt (TASK and ALLOWED parts)**

```text
TASK
Implement only POST /api/auth/signup.
Request: SignupRequest { email (valid email, required), password (required, 8-64 chars),
fullName (required, not blank, max 100) }.
Response: 201 with UserResponse { id, email, fullName, role, createdAt }.
The new user's role is always STUDENT, even if the request contains a role field.
Store email trimmed and lowercased. Hash the password with the existing PasswordEncoder bean.
If the email already exists, throw DuplicateResourceException.

ALLOWED TO CREATE OR CHANGE
dto/SignupRequest.java, dto/UserResponse.java,
service/AuthService.java (signup method only),
controller/AuthController.java (signup endpoint only)
```

**Judge: what to look for**
- Password is hashed through the **injected** `PasswordEncoder`, not `new BCryptPasswordEncoder()` inside the service, and never hand-rolled (MD5, SHA-256).
- `UserResponse` is built explicitly. The entity is not returned. No `passwordHash` field in the DTO.
- `SignupRequest` has **no `role` field**, or if it does, it is ignored. (AI often adds `role` "for flexibility". That is a privilege-escalation bug.)
- Duplicate check uses `existsByEmail` on the **normalised** email.
- Returns **201**, not 200.
- AI did **not** touch `SecurityConfig`, add a JWT class, or add a `UserDetailsService`.

**Understanding questions**
1. Where exactly is the password hashed, and why is the same password stored as a different string for two users?
2. What would happen if `SignupRequest` accepted `role`?
3. Which class turns `DuplicateResourceException` into a 409?

**Run:** `./mvnw test -Dtest='S1*'`

---

### S2: Login and JWT issuing (A2)

**Give the AI:** the template; `AuthService` and `AuthController` from S1; contract row A2; `LoginRequest`, `LoginResponse`; the JWT claim list; `jwt.secret` and `jwt.expiration-seconds` property names.

**Allowed:** `dto/LoginRequest`, `dto/LoginResponse`, `security/JwtService` (method `generateToken` only), `AuthService.login`, `AuthController.login`.

**Prompt guide (key lines for TASK)**
- "Look up the user by lowercased email. Check the password with `passwordEncoder.matches`. If either fails, throw `InvalidCredentialsException("Invalid email or password")`."
- "`JwtService.generateToken(User)` sets `subject` = email, claim `role`, a random UUID as `id` (jti), `issuedAt`, `expiration`. Sign with HS256 using a key from `Keys.hmacShaKeyFor` on the Base64-decoded `jwt.secret`. Read both properties with `@Value`."
- "Do not use AuthenticationManager or UserDetailsService."

**Judge: what to look for**
- **Same** exception and message for unknown email and wrong password. AI frequently throws "User not found" separately, which leaks which emails are registered.
- Secret comes from config. No string literal secret anywhere, including as a `@Value` default.
- Current jjwt 0.12 API: `Jwts.builder().subject(...).claim(...).id(...).issuedAt(...).expiration(...).signWith(key).compact()`.
  **Reject** `setSubject`, `setExpiration`, `signWith(SignatureAlgorithm.HS256, secret)`, `Jwts.parserBuilder()`. These are deprecated or removed.
- `jti` is present and random (UUID), not the user id.
- `expiresIn` in the response matches the configured expiry, not a hardcoded number.
- No password or hash in the token claims.
- The parsing / validation method is **not** written yet (that is S3). If the AI wrote it anyway, note it as out of scope.

**Understanding questions**
1. Why must the error message be identical for "no such email" and "wrong password"?
2. Who can read the claims inside your JWT? Who can change them?
3. What does `jti` give us that `sub` doesn't? (Needed in S5.)

**Run:** `./mvnw test -Dtest='S2*'`

---

### S3: JWT filter and protected API (P2, P1 stays public)

This is the hardest slice. Budget the most TA attention here.

**Give the AI:** the template; `JwtService` from S2; the temporary `SecurityConfig`; `JsonAuthenticationEntryPoint` and `JsonAccessDeniedHandler`; contract rows P1 and P2; list of public paths.

**Allowed:** `JwtService` (add a parse/validate method), `security/JwtAuthenticationFilter` (new), `config/SecurityConfig` (replace the `filterChain` bean only), `controller/UserController` with `GET /api/users/me`.

**Prompt guide (key lines for TASK)**
- "Write `JwtAuthenticationFilter extends OncePerRequestFilter`. If there is no `Bearer` header, continue the chain unchanged. If the token is valid, set an `Authentication` in the `SecurityContext` with the email as principal and authority `ROLE_<role>`. If parsing fails for any reason (bad signature, expired, malformed), clear the context and continue. Do not write a response from the filter."
- "In `SecurityConfig`: stateless sessions, CSRF disabled, `POST /api/auth/signup`, `POST /api/auth/login` and `GET /api/courses` are public; everything else requires authentication. Add the filter before `UsernamePasswordAuthenticationFilter`. Use the existing entry point and access-denied handler."
- "`GET /api/users/me` reads the email from the current `Authentication`, loads the user and returns `UserResponse`."

**Judge: what to look for**
- **Spring Security 6 style only:** a `SecurityFilterChain` bean, lambda DSL, `requestMatchers`, `authorizeHttpRequests`.
  **Reject** `WebSecurityConfigurerAdapter`, `antMatchers`, `authorizeRequests`, `.and()` chaining.
- `permitAll` covers **only** the three public endpoints, not `/api/**` or `/api/auth/**` broadly (logout must be protected in S5).
- `GET` vs `POST` on `/api/courses`: only **GET** is public. Watch for `requestMatchers("/api/courses").permitAll()`, which opens POST too.
- The filter verifies the **signature and expiry** with `Jwts.parser().verifyWith(key).build().parseSignedClaims(token)`. Reject any code that decodes the payload with Base64 and trusts it.
- Exceptions in the filter are caught and the chain continues. The filter does not swallow them silently and then authenticate anyway.
- The entry point and access-denied handler are wired via `exceptionHandling(...)`, not rewritten.
- Authorities use the `ROLE_` prefix, because `hasRole("ADMIN")` in S4 depends on it.
- If the filter is annotated `@Component`, Spring Boot also registers it as a normal servlet filter and it runs twice. Acceptable if students can explain it; cleaner to create it as a `@Bean` or construct it in `SecurityConfig`.
- AI did not add `UserDetailsService`, `AuthenticationManager`, sessions or CORS config.
- `/api/users/me` returns `UserResponse`, not the entity.

**Understanding questions**
1. Why is CSRF protection disabled here, and when would disabling it be wrong?
2. Walk through what happens to a request with an expired token, step by step, until the client sees 401.
3. Why don't 401/403 errors from the filter chain reach `GlobalExceptionHandler`?

**Run:** `./mvnw test -Dtest='S3*'`, then re-run `S1*` and `S2*` to confirm signup and login are still public.

---

### S4: Role-restricted API (P3)

**Give the AI:** the template; the current `SecurityConfig`; `CourseController`; contract row P3.

**Allowed:** `SecurityConfig` only, **or** `@PreAuthorize` on `CourseController.create` plus `@EnableMethodSecurity`. Students state which approach they asked for.

**Prompt guide:** "Restrict `POST /api/courses` to users with role ADMIN. Change nothing else."

**Judge: what to look for**
- Correct role naming: `hasRole("ADMIN")` checks for authority `ROLE_ADMIN`; `hasAuthority("ADMIN")` checks `ADMIN`. A mismatch with S3 makes every admin call fail with 403. This is the most common bug in this slice.
- If using `@PreAuthorize`, `@EnableMethodSecurity` is present. Otherwise the annotation does nothing and the test for 403 fails.
- The rule is placed **before** the general `anyRequest().authenticated()` rule. Order matters.
- The diff is tiny. If the AI rewrote the whole config, reject and re-prompt.

**Understanding questions**
1. A STUDENT calls P3. Why 403 and not 401?
2. Where does the role in the request come from, and can the client change it?

**Run:** `./mvnw test -Dtest='S4*'`

---

### S5: Logout (A3), stretch

**Give the AI:** the template; `JwtService`, `JwtAuthenticationFilter`, `AuthController`; contract row A3; the requirement that the token stops working immediately.

**Allowed:** `security/TokenDenylist` (new), `JwtAuthenticationFilter` (add the denylist check), `AuthController.logout`.

**Prompt guide (key lines for TASK)**
- "`TokenDenylist` is an in-memory `@Component` holding a `ConcurrentHashMap<String, Instant>` of jti → expiry time."
- "`POST /api/auth/logout` requires authentication, reads the jti and expiry from the current token and adds them to the denylist. Returns 204."
- "The filter treats a denylisted jti as invalid."

**Judge: what to look for**
- The logout actually changes server state. **Reject** a logout that only returns 200 or says "the client should delete the token". This is the classic AI answer and the point of this slice.
- Denylist is keyed by **jti**, not by email (keying by email would log the user out on all devices and block re-login).
- Map is thread-safe (`ConcurrentHashMap`, not `HashMap`).
- Logout endpoint is not under a `permitAll`.
- Bonus: expired entries are removed (e.g. on each add, or with `@Scheduled`), so the map doesn't grow forever.

**Understanding questions**
1. Why does logout need server-side state if JWT is "stateless"?
2. What happens to the denylist when the server restarts? What if there are two server instances?
3. Why store the expiry time with each jti?

**Run:** `./mvnw test -Dtest='S5*'`, then `./mvnw test` for the full suite.

---

## 7. Session timeline

| Time | Activity | TA role |
|---|---|---|
| 0–10 | **Brief.** Goals, contract (section 1.2), prompt template, review sheet. Show one bad prompt ("add JWT auth to my app") and its output vs. one templated prompt. | Lead TA presents; others check every student's starter app runs |
| 10–25 | **Judge first.** Students review a pre-generated, flawed `SecurityConfig` + `JwtService` (section 8) with the review sheet. 10 min individual, 5 min debrief. | Walk the room; debrief by calling on students |
| 25–75 | **Build S1 → S5** with the template, review sheet and tests. | Circulate; at each slice check that `PROMPTS.md` and `REVIEW.md` are being filled, not back-filled |
| 75–85 | **No-AI change.** TA assigns one change (section 9). AI tools closed. | Assign different changes to neighbours |
| 85–90 | **Submit** repo with `PROMPTS.md` and `REVIEW.md`. | Confirm submissions |

If a student is far behind at minute 60, they skip S5 and go to the no-AI change at minute 75 anyway. The no-AI change is not optional.

---

## 8. "Judge first" exercise: flawed AI output (TA answer key)

The handout shows a `SecurityConfig` and `JwtService` presented as "AI-generated". It compiles in a Spring Boot 2 project and contains these planted flaws:

| # | Flaw | Why it matters |
|---|---|---|
| 1 | `extends WebSecurityConfigurerAdapter` | Removed in Spring Security 6; won't compile on our stack |
| 2 | `.antMatchers("/api/**").permitAll()` | Opens every endpoint |
| 3 | `private static final String SECRET = "mysecret";` | Hardcoded and far too short for HS256 |
| 4 | `Jwts.parser().setSigningKey(SECRET).parseClaimsJwt(token)` | Deprecated API; `parseClaimsJwt` accepts **unsigned** tokens |
| 5 | Filter catches `Exception` and calls `chain.doFilter` after setting authentication anyway | Invalid tokens still authenticate |
| 6 | `UserController` returns the `User` entity | Password hash leaks in the JSON |
| 7 | Login throws `UserNotFoundException("No user with email ...")` | Reveals which emails are registered |
| 8 | Adds `UserDetailsService`, a new `ErrorResponse` class and a Lombok dependency | Out of scope; violates the restriction rules |

Debrief question: "Which of these would the tests catch, and which only a human reviewer would?" (Flaws 3, 7 and 8 are mostly reviewer-only.)

---

## 9. No-AI change (pick one per student)

1. Change token expiry to 15 minutes and make `expiresIn` reflect it.
2. Add `GET /api/courses/{id}` as a public endpoint (404 via `ResourceNotFoundException`).
3. Add a `PUT /api/users/me` that updates `fullName` only, for any logged-in user.
4. Make signup reject passwords without at least one digit, returning a 400 with a field error.
5. Make `GET /api/courses` protected instead of public, and explain which test now fails and why.

Students note in `REVIEW.md` which files they changed and why.

---

## 10. Common problems (TA quick reference)

| Symptom | Likely cause |
|---|---|
| App won't start: `WeakKeyException` | `JWT_SECRET` not set, or under 32 bytes after Base64 decoding |
| Every protected call gives **403** with empty body instead of 401 | Entry point not wired in `exceptionHandling` |
| Admin gets 403 on P3 | `ROLE_` prefix mismatch between filter and `hasRole` |
| `@PreAuthorize` has no effect | Missing `@EnableMethodSecurity` |
| Signup/login return 401 after S3 | Public paths missing from `requestMatchers`, or wrong HTTP method in the matcher |
| SQL error on table `user` | `user` is a reserved word in Postgres; the entity maps to `users` |
| Filter runs twice | Filter annotated `@Component` and also added to the chain |
| `cannot find symbol: WebSecurityConfigurerAdapter` | AI generated Spring Boot 2 code; reject and re-prompt with the stack line |
| Logout "works" but token still accepted | Denylist populated but filter never checks it |

---

## 11. What is graded

| Item | How | Weight (suggested) |
|---|---|---|
| Instructor tests S1–S4 (S5 bonus) | Automated | 50% |
| `PROMPTS.md`: one slice per prompt, template used | TA spot-check | 15% |
| `REVIEW.md`: buckets, decisions, understanding answers | TA spot-check | 20% |
| No-AI change works and is explained | TA check in lab | 15% |
