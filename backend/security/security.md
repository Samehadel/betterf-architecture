# Security — Implementation Guide

> Detailed reference for Spring Security configuration, JWT authentication, and endpoint authorization.

---

## Package Layout

All security code lives in `com.{company}.security`. No domain module defines its own security rules.

```
com.{company}.security/
    SecurityConfig.java           ← filter chain, CORS, CSRF, public/protected paths
    JwtFilter.java                ← validates Bearer token on every request
    JwtService.java               ← token parsing, validation, generation
    SecurityUserDetails.java      ← adapts the authenticated identity to Spring's UserDetails
    SecurityUserDetailsService.java  ← loads user by username/phone for Spring Security
```

---

## `SecurityConfig`

Defines the security filter chain. This is the single place where endpoint authorization rules are configured.

```java
// com/{company}/security/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .cors(c -> c.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                // Public endpoints — no token required
                .requestMatchers(POST, "/api/v1/auth/login").permitAll()
                .requestMatchers(GET,  "/api/v1/customer/status/**").permitAll()
                .requestMatchers("/actuator/health", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                // Everything else requires a valid JWT
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.queueapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

**Rules:**
- No domain module adds its own `SecurityFilterChain` or `HttpSecurity` configuration
- Fine-grained endpoint rules (role checks) use `@PreAuthorize` on the controller or service — not more `requestMatchers` here
- The customer status endpoint is public — it's secured by the opaque token embedded in the URL, not by Bearer auth

---

## JWT Filter

Runs on every request. Extracts the JWT from the `access_token` **HttpOnly cookie**, validates it, and sets the authentication in the security context.

The token is read from a cookie — not from the `Authorization` header. This keeps the token out of JavaScript and prevents XSS-based token theft.

```java
// com/{company}/security/JwtFilter.java
@Component
@RequiredArgsConstructor
public class JwtFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final SecurityUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        String token = extractTokenFromCookies(request);
        if (token == null) {
            chain.doFilter(request, response);
            return;
        }

        try {
            String username = jwtService.extractUsername(token);
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities()
                        );
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException e) {
            // Invalid token — do not set authentication, request proceeds unauthenticated
            // The security config will reject it if the endpoint requires auth
        }

        chain.doFilter(request, response);
    }

    private String extractTokenFromCookies(HttpServletRequest request) {
        if (request.getCookies() == null) return null;
        return Arrays.stream(request.getCookies())
            .filter(c -> "access_token".equals(c.getName()))
            .map(Cookie::getValue)
            .findFirst()
            .orElse(null);
    }
}
```

**Rules:**
- The cookie name is `access_token` — this is the agreed contract with the frontend
- The cookie must be set as `HttpOnly` and `Secure` by the auth endpoint on login — never readable by JavaScript
- Do not fall back to the `Authorization` header — cookie is the only accepted token source

---

## JWT Service

Handles token generation and validation.

```java
// com/{company}/security/JwtService.java
@Service
public class JwtService {

    @Value("${app.jwt.secret}")
    private String secret;

    @Value("${app.jwt.expiration-ms:300000}")  // 5 minutes default
    private long expirationMs;

    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey())
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        return extractUsername(token).equals(userDetails.getUsername())
            && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return resolver.apply(claims);
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }
}
```

The JWT secret is never hardcoded — always read from `${app.jwt.secret}` in application properties or an environment variable.

---

## Role-Based Authorization

Use `@PreAuthorize` on controller methods for role-based access control.

```java
// Operator-only endpoint
@Override
@PreAuthorize("hasRole('OPERATOR')")
public ResponseEntity<ApiResponseQueueEntry> callNext(String businessId) {
    return ResponseEntity.ok(ApiResponse.success(queueService.callNext(businessId)));
}

// Business owner can only access their own business
@Override
@PreAuthorize("hasRole('OWNER') and @businessSecurityService.isOwner(#businessId, authentication)")
public ResponseEntity<ApiResponseBusiness> getBusiness(String businessId) {
    return ResponseEntity.ok(ApiResponse.success(businessService.getBusiness(businessId)));
}
```

For ownership checks, use a dedicated `@Service` bean in `com.{company}.security`:

```java
// com/{company}/security/BusinessSecurityService.java
@Service
@RequiredArgsConstructor
public class BusinessSecurityService {

    private final BusinessService businessService;   // from com.{company}.business.api

    public boolean isOwner(String businessId, Authentication auth) {
        BusinessView business = businessService.getBusiness(businessId);
        return business.ownerPhone().equals(auth.getName());
    }
}
```

---

Roles are stored in the `USER_ROLES` table and embedded in the JWT as a `roles` claim.

---

## Configuration

```properties
# application.properties
app.jwt.secret=${JWT_SECRET}        # Set via environment variable — never hardcoded
app.jwt.expiration-ms=300000        # 5 minutes — short-lived access token

# Refresh token
app.security.refresh-token.session-window-days=30       # session lifetime from login
app.security.refresh-token.rotation-threshold-hours=24  # rotate token after this idle period

# Disable Spring Security's default form login
spring.security.user.name=disabled
```

---

## Cookie Model — Access + Refresh Tokens

Authentication uses two HttpOnly cookies with different lifetimes and paths.

| Attribute | `access_token` | `refresh_token` |
|-----------|---------------|-----------------|
| Content | Signed JWT (HS256) | Opaque random value (Base64) |
| TTL | 5 minutes | 30 days (session window) |
| HttpOnly | Yes | Yes |
| Secure | Yes | Yes |
| SameSite | Strict | Strict |
| Path | `/` | `/api/v1/auth/refresh` |

**Rules:**
- Token values are never returned in response bodies — cookies are the sole delivery mechanism
- The `refresh_token` cookie is path-restricted to `/api/v1/auth/refresh` — the browser only sends it to that endpoint, not to regular API calls
- The raw refresh token is never stored in the database — only its `SHA-256` hash is persisted
- Session expiry is computed at runtime as `issued_at + session-window-days` — there is no `expires_at` column

### Refresh Token Rotation

Rotation uses a threshold-based (lazy) strategy. The token rotates only when `now - last_rotated_at` exceeds the configured threshold (default 24 hours), not on every use.

- **Within threshold:** new access token issued, refresh token unchanged
- **Past threshold:** new access token issued, new refresh token issued, `last_rotated_at` updated
- Rotation never resets the session window — a session issued on Day 1 always expires at Day 31

### Lifecycle

| Event | What happens |
|-------|-------------|
| Login / Register | New access token cookie + new refresh token cookie. Refresh token hash stored in DB. |
| Protected API call | `JwtFilter` validates access token from cookie. No refresh token involved. |
| Access token expired | Frontend receives 401 → calls `POST /auth/refresh` → new access token issued. Refresh token rotated if past threshold. |
| Logout | Refresh token revoked in DB. Both cookies cleared via `Max-Age=0`. |
| Session expired (30 days) | Refresh endpoint deletes the expired row and returns 401. Frontend shows session-expired overlay. |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Security rules inside a domain module | All security config belongs in `com.{company}.security` |
| Hardcoding the JWT secret | Use `${JWT_SECRET}` env variable |
| Not catching `JwtException` in the filter | Invalid tokens silently pass through — catch and log |
| Using `session.sessionCreationPolicy(ALWAYS)` | API is stateless — always use `STATELESS` |
| Adding a new public endpoint without updating `SecurityConfig` | All URL permit rules are in `SecurityConfig.filterChain()` |
| Querying the database in the JWT filter | `JwtFilter` extracts the username from the token; full user load happens in `UserDetailsService` only when needed |
| Reading the token from the `Authorization` header | Token is read exclusively from the `access_token` HttpOnly cookie — do not add a header fallback |
| Setting the cookie without `HttpOnly` and `Secure` flags | The cookie must be `HttpOnly` (no JS access) and `Secure` (HTTPS only) — set both on the login response |
