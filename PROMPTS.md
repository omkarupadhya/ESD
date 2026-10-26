# PROMPTS.md — Structured Prompt Drafts

The drafts below are reconstructed from the implemented slices to document the
engineering intent; they are not verbatim historical prompts. S1 was completed
manually and is intentionally not recorded as an AI prompt. No follow-up prompts
are included because none were provided for reconstruction.

> Each draft follows the lab's required prompt structure. If using these for a
> future run, paste the relevant existing classes into the context before
> submitting the prompt.

---

## Slice 2 — Login & JWT (S2)

**Prompt 1 (reconstructed draft):**

```
CONTEXT
This project uses Java 17, Spring Boot 3.2, Spring Security 6, and JJWT 0.12.6.
Do not add dependencies. S1 signup is already implemented. Reuse the existing
UserRepository, PasswordEncoder bean, InvalidCredentialsException, AuthService,
AuthController, and User entity.

TASK
Implement only the login slice:
- Add LoginRequest with a required, valid email and required password.
- Add LoginResponse with accessToken, tokenType ("Bearer"), and expiresIn in
  seconds.
- Implement JwtService.generateToken(User) using the configured JWT secret and
  expiration. Include sub=email, role, a random UUID jti, iat, and exp, and sign
  with HS256 using the JJWT 0.12 API.
- Implement AuthService.login(LoginRequest). Normalize email by trimming and
  lowercasing with Locale.ROOT, verify credentials using PasswordEncoder.matches,
  and return the token response.
- Expose POST /api/auth/login with request validation and HTTP 200.

The unknown-email and incorrect-password cases must both return HTTP 401 with
the exact message "Invalid email or password". Do not disclose password data
in the response or token. Keep token parsing and validation for the next slice.

ALLOWED TO CREATE OR CHANGE
dto/LoginRequest.java
dto/LoginResponse.java
security/JwtService.java (token generation only)
service/AuthService.java (login method only)
controller/AuthController.java (login endpoint only)

RULES
- Do not modify any file outside the allowed list.
- Use the existing PasswordEncoder and exception classes.
- Read JWT secret and expiry from existing configuration; do not hardcode either.
- Use JJWT 0.12 APIs such as subject(), claim(), id(), issuedAt(),
  expiration(), and signWith(key). Do not use deprecated APIs.
- Keep the response DTO free of user entities and password fields.
- If the current configuration or an existing API is ambiguous, report the
  assumption rather than inventing a new dependency or configuration source.

OUTPUT
Provide each changed file in full with its path, followed by concise assumptions
and a checklist of the login contract points implemented.
```

**Follow-up:** No follow-up prompt documented.

---

## Slice 3 — JWT Filter & Protected API (S3)

**Prompt 1 (reconstructed draft):**

```
CONTEXT
This project uses Java 17, Spring Boot 3.2, Spring Security 6, and JJWT 0.12.6.
The login slice issues signed JWTs containing sub=email, role, jti, iat, and exp.
Reuse the existing JwtService, JSON authentication entry point, JSON access
denied handler, UserRepository, UserResponse, and ResourceNotFoundException.

TASK
Implement the protected-request slice:
- Add JwtService.parseToken(String) using JJWT 0.12 to verify the signature and
  expiration before returning claims.
- Implement a OncePerRequestFilter that reads a Bearer token, validates it,
  extracts the email and role claims, and sets an authenticated principal with
  a ROLE_ authority. For a missing or invalid token, clear the security context
  and continue the filter chain so Spring Security can produce the appropriate
  response for the endpoint.
- Configure a stateless SecurityFilterChain: disable CSRF; permit only POST
  /api/auth/signup, POST /api/auth/login, and GET /api/courses; require
  authentication for all other requests; wire the existing JSON 401 and 403
  handlers; and place the JWT filter before UsernamePasswordAuthenticationFilter.
- Add GET /api/users/me. Look up the principal by authenticated email and return
  UserResponse, never the User entity.

Preserve GET /api/courses as public without making POST /api/courses public.

ALLOWED TO CREATE OR CHANGE
security/JwtService.java (token parsing only)
security/JwtAuthenticationFilter.java
config/SecurityConfig.java
controller/UserController.java

RULES
- Do not modify files outside the allowed list.
- Use Spring Security 6 lambda DSL, requestMatchers, and authorizeHttpRequests.
  Do not use WebSecurityConfigurerAdapter, antMatchers, authorizeRequests, or
  deprecated JJWT parser APIs.
- Do not add a UserDetailsService, AuthenticationManager, session-based
  authentication, CORS configuration, or new dependencies.
- Do not write an error response directly from the JWT filter; leave 401/403
  response formatting to the configured security handlers.
- Require a verified signed JWT before trusting any claim, including role.

OUTPUT
Provide each changed file in full with its path, then list assumptions and a
request-level checklist for public access, protected access, invalid tokens,
and the current-user response.
```

**Follow-up:** No follow-up prompt documented.

---

## Slice 4 — Role Restriction (S4)

**Prompt 1 (reconstructed draft):**

```
CONTEXT
This project uses Java 17, Spring Boot 3.2, and Spring Security 6. The existing
JWT filter creates authorities with the ROLE_ prefix. SecurityConfig already
permits GET /api/courses and requires authentication for protected routes.
CourseController and CourseService already implement course creation and its
201 response. The JSON access-denied handler is already available.

TASK
Restrict POST /api/courses to users with the ADMIN role. Add the method-specific
authorization rule to the existing SecurityFilterChain before the catch-all
authenticated rule. Keep GET /api/courses public and leave course creation,
validation, response mapping, and existing error behavior unchanged.

ALLOWED TO CREATE OR CHANGE
config/SecurityConfig.java (authorization matcher only)

RULES
- Do not modify any file outside the allowed list.
- Use requestMatchers(HttpMethod.POST, "/api/courses").hasRole("ADMIN").
- Keep the authorization rule before anyRequest().authenticated().
- Do not broaden public matchers or rewrite the security configuration.
- Preserve the existing ROLE_ADMIN authority convention and 403 JSON handler.

OUTPUT
Provide the changed file in full with its path, explain why a STUDENT receives
403 while an unauthenticated request receives 401, and list the expected
ADMIN/STUDENT/anonymous outcomes for POST /api/courses.
```

**Follow-up:** No follow-up prompt documented.

---

## Slice 5 — Logout (S5)

**Prompt 1 (reconstructed draft):**

```
CONTEXT
This project uses Java 17, Spring Boot 3.2, Spring Security 6, and JJWT 0.12.6.
JWTs contain a random jti and exp claim. The JWT filter authenticates requests
and the security configuration requires authentication for routes not explicitly
public. Reuse JwtService, AuthController, the existing JSON security handlers,
and the current exception conventions.

TASK
Implement server-side logout with an in-memory token denylist:
- Create a Spring component backed by ConcurrentHashMap<String, Instant>, keyed
  by JWT jti. Provide revoke(jti, expiry), isRevoked(jti), and expired-entry
  cleanup so revocations do not outlive their tokens.
- Add POST /api/auth/logout. It must require a valid authenticated Bearer token,
  obtain the verified jti and expiration through JwtService, revoke that token,
  and return 204 No Content. Missing or invalid credentials must not produce a
  success response.
- Integrate the denylist into the JWT filter. A revoked token must not establish
  authentication; protected requests using it must be rejected with 401.
- Do not make logout public. A fresh login must issue a distinct usable token.

ALLOWED TO CREATE OR CHANGE
security/TokenDenylist.java
controller/AuthController.java (logout endpoint only)
security/JwtAuthenticationFilter.java (denylist check only)

RULES
- Do not modify files outside the allowed list.
- Use jti, not email or the raw token, as the denylist key.
- Keep the map thread-safe and only retain entries until their token expiry.
- Validate the Bearer scheme and claims before revocation; do not trust
  unverified client-supplied claims.
- Preserve the existing security error response conventions and avoid silently
  treating malformed credentials as a successful logout.
- Do not add persistence, dependencies, or unrelated authentication features.

OUTPUT
Provide each changed file in full with its path, explain the in-memory
denylist's restart and multi-instance limitations, and include expected
responses for successful logout and subsequent use of the revoked token.
```

**Follow-up:** No follow-up prompt documented.
