---
paths:
  - "**/*.java"
---
# Java Security Patterns

> This file extends [common/security.md](../common/security.md) with Java/Quarkus specific patterns.

## JWT Authentication

Use SmallRye JWT for token-based auth:

```java
@Path("/api/documents")
@Produces(MediaType.APPLICATION_JSON)
public class DocumentResource {
  @Inject
  JsonWebToken jwt;
  
  @Inject
  SecurityIdentity identity;
  
  @GET
  @RolesAllowed("user")
  public List<DocumentResponse> list() {
    String userId = jwt.getName();  // Get user from token
    log.info("User {} listing documents", userId);
    return documentService.findByUser(userId);
  }
}
```

Configure JWT in application.yaml:

```yaml
mp:
  jwt:
    verify:
      publickey:
        location: ${JWT_PUBLIC_KEY}
      issuer: ${JWT_ISSUER}
```

## OIDC Integration

Use Quarkus OIDC for SSO:

```yaml
quarkus:
  oidc:
    auth-server-url: ${OIDC_SERVER_URL}
    client-id: ${OIDC_CLIENT_ID}
    credentials:
      secret: ${OIDC_CLIENT_SECRET}
    tls:
      verification: required
```

```java
@Path("/api/secure")
public class SecureResource {
  @Inject
  SecurityIdentity identity;
  
  @GET
  @Authenticated
  public String getData() {
    String username = identity.getPrincipal().getName();
    Set<String> roles = identity.getRoles();
    return "Hello " + username;
  }
}
```

## Role-Based Access Control

Use @RolesAllowed for authorization:

```java
@Path("/api/admin")
public class AdminResource {
  
  @GET
  @RolesAllowed("admin")
  public List<UserResponse> listUsers() {
    return adminService.getAllUsers();
  }
  
  @POST
  @Path("/users")
  @RolesAllowed({"admin", "user-manager"})
  public Response createUser(@Valid CreateUserRequest request) {
    return Response.ok(userService.create(request)).build();
  }
}
```

Programmatic security checks:

```java
@ApplicationScoped
@RequiredArgsConstructor
public class DocumentService {
  private final SecurityIdentity identity;
  
  public void delete(Long documentId) {
    Document document = findById(documentId);
    
    // Check ownership or admin role
    if (!identity.hasRole("admin") && 
        !document.getOwnerId().equals(identity.getPrincipal().getName())) {
      throw new UnauthorizedException("Cannot delete document");
    }
    
    documentRepository.delete(document);
  }
}
```

## Input Validation

Use Bean Validation extensively:

```java
public record CreateDocumentRequest(
    @NotBlank(message = "Reference number is required")
    @Size(max = 200, message = "Reference number too long")
    @Pattern(regexp = "^[A-Z0-9-]+$", message = "Invalid reference format")
    String referenceNumber,
    
    @NotNull(message = "Valid until date required")
    @FutureOrPresent(message = "Date must be in present or future")
    Instant validUntil,
    
    @NotEmpty(message = "At least one category required")
    @Size(max = 10, message = "Maximum 10 categories")
    List<@NotBlank(message = "Category cannot be blank") String> categories,
    
    @Email(message = "Invalid email format")
    String notificationEmail) {}
```

Custom validators:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DocumentReferenceValidator.class)
public @interface ValidDocumentReference {
  String message() default "Invalid document reference";
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};
}

public class DocumentReferenceValidator 
    implements ConstraintValidator<ValidDocumentReference, String> {
  
  @Override
  public boolean isValid(String value, ConstraintValidatorContext context) {
    if (value == null) return false;
    return value.matches("^DOC-[0-9]{8}-[A-Z]{3}$");
  }
}
```

## Secrets Management

Use HashiCorp Vault for secrets:

```yaml
quarkus:
  vault:
    url: ${VAULT_URL}
    authentication:
      app-role:
        role-id: ${VAULT_ROLE_ID}
        secret-id: ${VAULT_SECRET_ID}
```

```java
@ConfigProperty(name = "database.password")
String databasePassword;  // Loaded from Vault

@ConfigProperty(name = "api.key")
String apiKey;  // Loaded from Vault
```

## Password Hashing

Use BCrypt for password storage:

```java
@ApplicationScoped
public class PasswordService {
  private static final BCryptPasswordEncoder encoder = 
      new BCryptPasswordEncoder(12);
  
  public String hashPassword(String plainPassword) {
    return encoder.encode(plainPassword);
  }
  
  public boolean verifyPassword(String plainPassword, String hashedPassword) {
    return encoder.matches(plainPassword, hashedPassword);
  }
}

// Usage
@ApplicationScoped
@RequiredArgsConstructor
public class UserService {
  private final PasswordService passwordService;
  
  @Transactional
  public User createUser(CreateUserRequest request) {
    String hashedPassword = passwordService.hashPassword(request.getPassword());
    User user = new User(request.getUsername(), hashedPassword);
    userRepository.persist(user);
    return user;
  }
}
```

## CORS Configuration

Configure CORS properly:

```yaml
quarkus:
  http:
    cors:
      ~: true  # Enable CORS
      origins: ${ALLOWED_ORIGINS}  # Specific origins only
      methods: GET,POST,PUT,DELETE
      headers: Authorization,Content-Type
      exposed-headers: Location
      access-control-max-age: 24H
```

For programmatic control:

```java
@Provider
public class CorsFilter implements ContainerResponseFilter {
  @Override
  public void filter(ContainerRequestContext requestContext,
                     ContainerResponseContext responseContext) {
    responseContext.getHeaders().add(
        "Access-Control-Allow-Origin", getAllowedOrigin(requestContext));
    responseContext.getHeaders().add(
        "Access-Control-Allow-Credentials", "true");
  }
}
```

## SQL Injection Prevention

Use parameterized queries with Panache:

```java
// CORRECT: Parameterized query
public List<Document> findByStatus(String status) {
  return list("status = ?1", status);
}

// CORRECT: Named parameters
public Optional<Document> findByReference(String reference) {
  return find("referenceNumber = :ref", Map.of("ref", reference))
      .firstResultOptional();
}

// WRONG: String concatenation - NEVER DO THIS
public List<Document> findByStatus(String status) {
  return list("status = '" + status + "'");  // SQL INJECTION RISK!
}
```

## Audit Logging

Log security-relevant events:

```java
@ApplicationScoped
@RequiredArgsConstructor
public class AuditService {
  private final AuditLogRepository auditLogRepository;
  
  public void logAccess(String userId, String resource, String action) {
    AuditLog log = new AuditLog();
    log.setUserId(userId);
    log.setResource(resource);
    log.setAction(action);
    log.setTimestamp(Instant.now());
    log.setIpAddress(getCurrentIpAddress());
    auditLogRepository.persist(log);
  }
}

@Path("/api/documents")
public class DocumentResource {
  @Inject
  AuditService auditService;
  
  @GET
  @RolesAllowed("user")
  public List<DocumentResponse> list() {
    auditService.logAccess(identity.getPrincipal().getName(), 
                           "documents", 
                           "list");
    return documentService.list();
  }
}
```

## Rate Limiting

Implement rate limiting:

```java
@ApplicationScoped
public class RateLimiter {
  private final Map<String, RateLimitBucket> buckets = new ConcurrentHashMap<>();
  
  public boolean isAllowed(String userId, int maxRequests, Duration window) {
    RateLimitBucket bucket = buckets.computeIfAbsent(
        userId, k -> new RateLimitBucket(maxRequests, window));
    return bucket.tryConsume();
  }
}

@Provider
public class RateLimitFilter implements ContainerRequestFilter {
  @Inject
  RateLimiter rateLimiter;
  
  @Override
  public void filter(ContainerRequestContext requestContext) {
    String userId = extractUserId(requestContext);
    if (!rateLimiter.isAllowed(userId, 100, Duration.ofMinutes(1))) {
      requestContext.abortWith(
          Response.status(429).entity("Rate limit exceeded").build());
    }
  }
}
```

## Sensitive Data Protection

Never log sensitive data:

```java
// CORRECT: Mask sensitive data
log.info("Processing payment for card ending {}", 
    maskCardNumber(card.getNumber()));

// WRONG: Logging sensitive data
log.info("Processing payment for card {}", card.getNumber());  // NEVER!

private String maskCardNumber(String cardNumber) {
  if (cardNumber.length() < 4) return "****";
  return "****" + cardNumber.substring(cardNumber.length() - 4);
}
```
