# Security Architecture

## Overview

This document outlines the comprehensive security architecture for the First Viscount e-commerce platform, covering authentication, authorization, data protection, and security best practices across all microservices.

## Security Principles

1. **Defense in Depth** - Multiple layers of security controls
2. **Zero Trust** - Never trust, always verify
3. **Least Privilege** - Minimal necessary permissions
4. **Secure by Default** - Security built into the design
5. **Audit Everything** - Comprehensive logging and monitoring

## Authentication Architecture

### OAuth2 + JWT Implementation

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Applications                      │
├─────────────────────────────────────────────────────────────┤
│                            │                                 │
│                            ▼                                 │
│                    ┌───────────────┐                        │
│                    │  API Gateway  │                        │
│                    │   (Auth)      │                        │
│                    └───────┬───────┘                        │
│                            │                                 │
│              ┌─────────────┴─────────────┐                 │
│              │                           │                  │
│              ▼                           ▼                  │
│      ┌───────────────┐          ┌───────────────┐         │
│      │ Auth Service  │          │  Resource     │         │
│      │ (OAuth2/JWT)  │          │  Services     │         │
│      └───────────────┘          └───────────────┘         │
│              │                           │                  │
│              └─────────┬─────────────────┘                 │
│                        │                                    │
│                        ▼                                    │
│                ┌───────────────┐                          │
│                │ User Database │                          │
│                └───────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

### JWT Token Structure

```java
@Component
public class JwtTokenProvider {
    
    @Value("${security.jwt.secret}")
    private String jwtSecret;
    
    @Value("${security.jwt.expiration}")
    private long jwtExpiration;
    
    public String generateToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        Date expiryDate = new Date(System.currentTimeMillis() + jwtExpiration);
        
        return Jwts.builder()
            .setSubject(userPrincipal.getId().toString())
            .claim("email", userPrincipal.getEmail())
            .claim("roles", userPrincipal.getAuthorities())
            .claim("tenantId", userPrincipal.getTenantId())
            .setIssuedAt(new Date())
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.error("Invalid JWT token", e);
        }
        return false;
    }
}
```

### API Gateway Security Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String token = extractToken(request);
        
        if (token != null && tokenProvider.validateToken(token)) {
            Claims claims = tokenProvider.getClaims(token);
            
            // Set authentication in context
            UsernamePasswordAuthenticationToken authentication = 
                new UsernamePasswordAuthenticationToken(
                    claims.getSubject(),
                    null,
                    extractAuthorities(claims)
                );
            
            SecurityContextHolder.getContext().setAuthentication(authentication);
            
            // Add user context to request headers for downstream services
            ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Email", claims.get("email", String.class))
                .header("X-User-Roles", String.join(",", extractRoles(claims)))
                .build();
        }
        
        filterChain.doFilter(request, response);
    }
}
```

## Authorization Architecture

### Role-Based Access Control (RBAC)

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors().and()
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests()
                // Public endpoints
                .requestMatchers("/api/v1/auth/**", "/api/v1/products/**").permitAll()
                // Admin endpoints
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                // Order management
                .requestMatchers(HttpMethod.POST, "/api/v1/orders").hasRole("CUSTOMER")
                .requestMatchers(HttpMethod.GET, "/api/v1/orders/**").hasAnyRole("CUSTOMER", "ADMIN")
                // All other endpoints require authentication
                .anyRequest().authenticated()
            .and()
            .exceptionHandling()
                .authenticationEntryPoint(unauthorizedHandler)
            .and()
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

### Method-Level Security

```java
@Service
public class OrderService {
    
    @PreAuthorize("hasRole('CUSTOMER') and #request.customerId == authentication.principal.id")
    public Order createOrder(CreateOrderRequest request) {
        // Only customers can create their own orders
        return orderRepository.save(buildOrder(request));
    }
    
    @PreAuthorize("hasRole('ADMIN') or (#customerId == authentication.principal.id)")
    public List<Order> getCustomerOrders(UUID customerId) {
        // Customers can view their own orders, admins can view all
        return orderRepository.findByCustomerId(customerId);
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    public void cancelOrder(UUID orderId) {
        // Only admins can cancel orders
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}
```

## Service-to-Service Security

### Mutual TLS (mTLS)

```yaml
# Service configuration
server:
  ssl:
    enabled: true
    key-store: classpath:keystore/service.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: service
    
    # Enable client certificate authentication
    client-auth: need
    trust-store: classpath:truststore/ca.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
```

### Service Authentication

```java
@Configuration
public class ServiceAuthenticationConfig {
    
    @Bean
    public RestTemplate serviceRestTemplate() throws Exception {
        // Load service certificate
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(new ClassPathResource("keystore/service.p12").getInputStream(), 
                     keystorePassword.toCharArray());
        
        SSLContext sslContext = SSLContextBuilder.create()
            .loadKeyMaterial(keyStore, keystorePassword.toCharArray())
            .loadTrustMaterial(new ClassPathResource("truststore/ca.p12").getFile(), 
                              truststorePassword.toCharArray())
            .build();
        
        HttpClient httpClient = HttpClients.custom()
            .setSSLContext(sslContext)
            .build();
        
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
    }
    
    @Bean
    public WebClient serviceWebClient() {
        SslContext sslContext = SslContextBuilder
            .forClient()
            .keyManager(clientCertificate, clientKey)
            .trustManager(trustedCertificates)
            .build();
            
        HttpClient httpClient = HttpClient.create()
            .secure(ssl -> ssl.sslContext(sslContext));
            
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader("X-Service-Name", serviceName)
            .defaultHeader("X-Service-Version", serviceVersion)
            .build();
    }
}
```

## Data Protection

### Encryption at Rest

```java
@Configuration
public class EncryptionConfig {
    
    @Value("${encryption.key}")
    private String masterKey;
    
    @Bean
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        
        config.setPassword(masterKey);
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        config.setStringOutputType("base64");
        
        encryptor.setConfig(config);
        return encryptor;
    }
}

@Entity
public class Customer {
    @Id
    private UUID id;
    
    @Convert(converter = EncryptedStringConverter.class)
    private String ssn;
    
    @Convert(converter = EncryptedStringConverter.class)
    private String creditCardNumber;
    
    private String email; // Not encrypted but hashed for lookups
}
```

### Sensitive Data Handling

```java
@Component
public class SensitiveDataHandler {
    
    @Autowired
    private StringEncryptor encryptor;
    
    // PII Masking
    public String maskPII(String value, PIIType type) {
        switch (type) {
            case SSN:
                return "***-**-" + value.substring(value.length() - 4);
            case CREDIT_CARD:
                return "**** **** **** " + value.substring(value.length() - 4);
            case EMAIL:
                int atIndex = value.indexOf('@');
                return value.substring(0, 2) + "****" + value.substring(atIndex);
            default:
                return "********";
        }
    }
    
    // Secure deletion
    public void secureDelete(SensitiveData data) {
        // Overwrite memory
        if (data.getValue() != null) {
            char[] chars = data.getValue().toCharArray();
            Arrays.fill(chars, '\0');
        }
        
        // Mark for deletion
        data.setDeleted(true);
        data.setDeletedAt(Instant.now());
        
        // Schedule permanent deletion
        deletionScheduler.schedule(() -> {
            repository.permanentlyDelete(data.getId());
        }, 30, TimeUnit.DAYS);
    }
}
```

## API Security

### Input Validation

```java
@RestController
@Validated
public class ProductController {
    
    @PostMapping("/api/v1/products")
    public ResponseEntity<Product> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        
        // Additional validation
        validateProductRequest(request);
        
        // Sanitize input
        request.setName(sanitizer.sanitize(request.getName()));
        request.setDescription(sanitizer.sanitize(request.getDescription()));
        
        return ResponseEntity.ok(productService.create(request));
    }
    
    private void validateProductRequest(CreateProductRequest request) {
        // Check for SQL injection patterns
        if (containsSqlInjectionPattern(request.getName())) {
            throw new InvalidInputException("Invalid product name");
        }
        
        // Check for XSS
        if (containsScriptTags(request.getDescription())) {
            throw new InvalidInputException("Invalid product description");
        }
    }
}
```

### Rate Limiting

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String clientId = getClientIdentifier(request);
        RateLimiter limiter = limiters.computeIfAbsent(clientId, 
            k -> RateLimiter.create(getPermitsPerSecond(k)));
        
        if (!limiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded");
            return;
        }
        
        filterChain.doFilter(request, response);
    }
    
    private double getPermitsPerSecond(String clientId) {
        // Different limits for different client types
        if (isAuthenticatedUser(clientId)) {
            return 100.0; // 100 requests per second for authenticated users
        }
        return 10.0; // 10 requests per second for anonymous users
    }
}
```

### CORS Configuration

```java
@Configuration
public class CorsConfig {
    
    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        
        configuration.setAllowedOrigins(allowedOrigins);
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        
        return source;
    }
}
```

## Security Monitoring

### Audit Logging

```java
@Aspect
@Component
public class SecurityAuditAspect {
    
    @Autowired
    private AuditService auditService;
    
    @Around("@annotation(Audited)")
    public Object auditMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        AuditEvent event = AuditEvent.builder()
            .timestamp(Instant.now())
            .userId(SecurityContextHolder.getContext().getAuthentication().getName())
            .action(joinPoint.getSignature().getName())
            .resource(joinPoint.getTarget().getClass().getSimpleName())
            .parameters(sanitizeParameters(joinPoint.getArgs()))
            .ipAddress(getClientIpAddress())
            .build();
        
        try {
            Object result = joinPoint.proceed();
            event.setStatus("SUCCESS");
            event.setResult(sanitizeResult(result));
            return result;
            
        } catch (Exception e) {
            event.setStatus("FAILURE");
            event.setErrorMessage(e.getMessage());
            throw e;
            
        } finally {
            auditService.log(event);
        }
    }
}
```

### Security Metrics

```java
@Component
public class SecurityMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        meterRegistry.counter("security.authentication.success",
            "type", event.getAuthentication().getClass().getSimpleName()
        ).increment();
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        meterRegistry.counter("security.authentication.failure",
            "type", event.getException().getClass().getSimpleName()
        ).increment();
    }
    
    @EventListener
    public void handleAuthorizationFailure(AuthorizationFailureEvent event) {
        meterRegistry.counter("security.authorization.failure",
            "resource", event.getSource().toString()
        ).increment();
    }
}
```

## Vulnerability Management

### Dependency Scanning

```xml
<!-- Maven configuration for OWASP dependency check -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>8.0.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Security Headers

```java
@Component
public class SecurityHeadersFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) throws ServletException, IOException {
        
        // Prevent clickjacking
        response.setHeader("X-Frame-Options", "DENY");
        
        // Prevent MIME type sniffing
        response.setHeader("X-Content-Type-Options", "nosniff");
        
        // Enable XSS protection
        response.setHeader("X-XSS-Protection", "1; mode=block");
        
        // Force HTTPS
        response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        
        // Content Security Policy
        response.setHeader("Content-Security-Policy", 
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'");
        
        // Referrer Policy
        response.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
        
        filterChain.doFilter(request, response);
    }
}
```

## Secret Management

### HashiCorp Vault Integration

```java
@Configuration
@EnableConfigurationProperties(VaultProperties.class)
public class VaultConfig {
    
    @Bean
    public VaultTemplate vaultTemplate(VaultProperties properties) {
        VaultEndpoint endpoint = VaultEndpoint.create(properties.getHost(), properties.getPort());
        endpoint.setScheme(properties.getScheme());
        
        ClientAuthentication authentication = new TokenAuthentication(properties.getToken());
        
        return new VaultTemplate(endpoint, authentication);
    }
    
    @Component
    public class SecretManager {
        
        @Autowired
        private VaultTemplate vaultTemplate;
        
        public String getDatabasePassword(String serviceName) {
            VaultResponse response = vaultTemplate.read("secret/database/" + serviceName);
            return response.getData().get("password").toString();
        }
        
        public void rotateApiKey(String serviceName) {
            String newKey = generateSecureApiKey();
            
            Map<String, Object> data = new HashMap<>();
            data.put("api_key", newKey);
            data.put("rotated_at", Instant.now().toString());
            
            vaultTemplate.write("secret/api-keys/" + serviceName, data);
        }
    }
}
```

## Compliance & Privacy

### GDPR Compliance

```java
@RestController
@RequestMapping("/api/v1/privacy")
public class PrivacyController {
    
    @PostMapping("/data-export/{customerId}")
    @PreAuthorize("#customerId == authentication.principal.id or hasRole('ADMIN')")
    public ResponseEntity<Resource> exportCustomerData(@PathVariable UUID customerId) {
        CustomerDataExport export = privacyService.exportAllCustomerData(customerId);
        
        ByteArrayResource resource = new ByteArrayResource(export.toJson().getBytes());
        
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=customer-data.json")
            .contentType(MediaType.APPLICATION_JSON)
            .body(resource);
    }
    
    @DeleteMapping("/data-deletion/{customerId}")
    @PreAuthorize("#customerId == authentication.principal.id")
    public ResponseEntity<Void> requestDataDeletion(@PathVariable UUID customerId) {
        privacyService.scheduleCustomerDataDeletion(customerId);
        
        return ResponseEntity.accepted().build();
    }
}
```

### PCI DSS Compliance

```java
@Component
public class PCIComplianceHandler {
    
    // Never store sensitive cardholder data
    public PaymentToken tokenizeCard(CardDetails card) {
        // Validate card
        if (!isValidCard(card)) {
            throw new InvalidCardException();
        }
        
        // Send to PCI-compliant tokenization service
        TokenizationRequest request = TokenizationRequest.builder()
            .cardNumber(card.getNumber())
            .expiryMonth(card.getExpiryMonth())
            .expiryYear(card.getExpiryYear())
            .cvv(card.getCvv())
            .build();
            
        TokenizationResponse response = tokenizationService.tokenize(request);
        
        // Only store the token
        return PaymentToken.builder()
            .token(response.getToken())
            .lastFourDigits(card.getNumber().substring(card.getNumber().length() - 4))
            .cardType(detectCardType(card.getNumber()))
            .build();
    }
}
```

## Security Testing

### Penetration Testing

```java
@Profile("security-test")
@Component
public class SecurityTestEndpoints {
    
    @GetMapping("/test/security/sql-injection")
    public void testSqlInjection(@RequestParam String input) {
        // Intentionally vulnerable endpoint for testing
        String query = "SELECT * FROM users WHERE name = '" + input + "'";
        jdbcTemplate.query(query, new UserRowMapper());
    }
    
    @GetMapping("/test/security/xss")
    public String testXss(@RequestParam String input) {
        // Intentionally vulnerable endpoint for testing
        return "<html><body>Hello " + input + "</body></html>";
    }
}
```

## Security Checklist

### Development Phase
- [ ] Input validation on all endpoints
- [ ] Output encoding for XSS prevention
- [ ] SQL injection prevention (parameterized queries)
- [ ] Authentication required for protected resources
- [ ] Authorization checks for sensitive operations
- [ ] Sensitive data encryption
- [ ] Secure communication (HTTPS/mTLS)
- [ ] Security headers configured
- [ ] Rate limiting implemented
- [ ] Audit logging enabled

### Deployment Phase
- [ ] Secrets in secure vault
- [ ] SSL certificates valid
- [ ] Security scanning completed
- [ ] Penetration testing performed
- [ ] Security monitoring active
- [ ] Incident response plan ready
- [ ] Backup encryption verified
- [ ] Access controls reviewed

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Security Documentation](https://spring.io/projects/spring-security)
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [PCI DSS Requirements](https://www.pcisecuritystandards.org/)