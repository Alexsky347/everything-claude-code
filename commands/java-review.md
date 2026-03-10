---
description: Comprehensive Java/Quarkus code review for CDI patterns, Camel messaging, security, LogContext usage, and idiomatic Java. Invokes the java-reviewer agent.
---

# Java Code Review

This command invokes the **java-reviewer** agent for comprehensive Java/Quarkus-specific code review.

## What This Command Does

1. **Identify Java Changes**: Find modified `.java` files via `git diff`
2. **Run Static Analysis**: Execute `mvn compile`, check for errors
3. **Security Scan**: Check for SQL injection, hardcoded secrets, missing validation
4. **Pattern Review**: Verify CDI, Camel, LogContext, EventService patterns
5. **Quality Check**: Analyze code structure, naming, error handling
6. **Generate Report**: Categorize issues by severity (CRITICAL/HIGH/MEDIUM)

## When to Use

Use `/java-review` when:
- After writing or modifying Java code
- Before committing Java/Quarkus changes
- Reviewing pull requests with Java code
- Onboarding to a Java/Quarkus codebase
- Learning Quarkus patterns and idioms

## Review Categories

### CRITICAL (Must Fix)
- **SQL injection**: String concatenation in queries
- **Command injection**: Unvalidated input in ProcessBuilder
- **Hardcoded secrets**: API keys, passwords in code
- **Missing validation**: No @Valid on DTOs
- **Logging sensitive data**: Passwords, tokens in logs
- **Swallowed exceptions**: Empty catch blocks
- **Null returns**: Returning null instead of Optional

### HIGH (Should Fix)
- **Missing scopes**: Services without @ApplicationScoped
- **Wrong injection**: Not using @RequiredArgsConstructor
- **Missing @Transactional**: Data modifications without transactions
- **Poor error handling**: Generic Exception catches
- **Camel issues**: Routes without routeId or error handling
- **Large methods**: > 50 lines
- **Deep nesting**: > 4 levels

### MEDIUM (Consider Fixing)
- **Missing tests**: New code without tests
- **JUnit assertions**: Using assertEquals instead of AssertJ
- **No @DisplayName**: Tests without descriptive names
- **Optional misuse**: Optional as parameter or field
- **Missing LogContext**: Operations without tracing
- **No EventService**: Operations not tracked
- **Missing final**: Fields that could be final
- **POM version management**: Dependencies not using property variables
- **Unsorted properties**: Library versions not alphabetically sorted
- **Internal modules misplaced**: Internal module versions not at top of properties

## Review Checklist

For each modified Java file, the agent verifies:

### Code Structure
- [ ] Follows naming conventions (PascalCase classes, camelCase methods)
- [ ] Appropriate package structure (domain/api/infrastructure)
- [ ] Methods under 50 lines
- [ ] Classes under 800 lines
- [ ] Nesting depth ≤ 4 levels

### Dependency Injection
- [ ] Uses @RequiredArgsConstructor for constructor injection
- [ ] Has appropriate scope (@ApplicationScoped, @RequestScoped)
- [ ] Final fields for injected dependencies
- [ ] No field injection (@Inject on fields)

### Error Handling
- [ ] Specific exception types (not generic Exception)
- [ ] No empty catch blocks
- [ ] Exceptions logged before throwing
- [ ] Returns Optional for nullable values
- [ ] Try-with-resources for streams/connections

### Logging
- [ ] Uses @Slf4j (not System.out.println)
- [ ] No sensitive data in logs (passwords, tokens, PII)
- [ ] Uses LogContext for operation tracking
- [ ] Structured logging with parameters

### Validation & Security
- [ ] @Valid on all DTO parameters
- [ ] Bean Validation annotations (@NotNull, @Size, etc.)
- [ ] Parameterized queries (no string concatenation)
- [ ] No hardcoded secrets
- [ ] @RolesAllowed on secured endpoints

### Quarkus Patterns
- [ ] Services have @ApplicationScoped
- [ ] Repositories extend PanacheRepository
- [ ] Uses @Transactional for data modifications
- [ ] Config via @ConfigProperty with defaults
- [ ] Resources use JAX-RS annotations

### Camel Patterns
- [ ] Routes have .routeId()
- [ ] Uses asyncSendBody for async messaging
- [ ] Error handling with .onException()
- [ ] Queue names externalized to config
- [ ] Proper type conversions

### Event Tracking
- [ ] Uses EventService for operation tracking
- [ ] Calls createSuccessEvent on success
- [ ] Calls createErrorEvent in catch blocks
- [ ] LogContext passed to async operations

### Testing
- [ ] Tests exist for new/modified code
- [ ] Uses @Nested classes
- [ ] Has @DisplayName on tests
- [ ] Follows givenX_whenY_thenZ naming
- [ ] Uses AAA pattern with comments
- [ ] Uses AssertJ (not JUnit assertions)
- [ ] Coverage ≥ 80% lines, 70% branches

### Maven POM Structure
- [ ] All dependency versions use property variables (no magic numbers)
- [ ] Properties are sorted alphabetically
- [ ] Internal module versions at top (internal*)
- [ ] External library versions follow internal modules, alphabetically sorted
- [ ] Property names follow kebab-case.version pattern

## Quarkus-Specific Patterns Checked

### Service Layer
```java
// ✅ CORRECT
@ApplicationScoped
@RequiredArgsConstructor
@Slf4j
public class DocumentService {
  private final DocumentRepository documentRepository;
  private final EventService eventService;
  
  @Transactional
  public Document create(CreateDocumentRequest request) {
    // Implementation
  }
}

// ❌ WRONG - Missing scope, injection pattern
public class DocumentService {
  @Inject DocumentRepository repo;
  public Document create(CreateDocumentRequest req) { }
}
```

### Repository
```java
// ✅ CORRECT
@ApplicationScoped
public class DocumentRepository implements PanacheRepository<Document> {
  public Optional<Document> findByRef(String ref) {
    return find("referenceNumber = ?1", ref).firstResultOptional();
  }
}

// ❌ WRONG - SQL concatenation, not using Optional
public Document findByRef(String ref) {
  return find("referenceNumber = '" + ref + "'").firstResult();
}
```

### Camel Routes
```java
// ✅ CORRECT
@ApplicationScoped
public class MyRoute extends RouteBuilder {
  @ConfigProperty(name = "rabbitmq.queue")
  String queueName;
  
  @Override
  public void configure() {
    from("direct:my-route")
      .routeId("my-route")
      .log("Processing: ${body}")
      .marshal().json()
      .toF("spring-rabbitmq:%s", queueName);
  }
}

// ❌ WRONG - Hardcoded queue, no routeId
from("direct:my-route")
  .to("spring-rabbitmq:hardcoded-queue");
```

### LogContext Usage
```java
// ✅ CORRECT
public void process(Path file) {
  LogContext context = CustomLog.getCurrentContext();
  try (SafeAutoCloseable ignored = CustomLog.startScope(context)) {
    context.put("fileId", file.toString());
    log.info("Processing file");
    // All logs include context
  }
}

// ❌ WRONG - No LogContext
public void process(Path file) {
  log.info("Processing file: {}", file);
}
```

### EventService Tracking
```java
// ✅ CORRECT
try {
  Document doc = processDocument(input);
  eventService.createSuccessEvent(doc, "DOCUMENT_PROCESSED");
  return doc;
} catch (ProcessingException e) {
  eventService.createErrorEvent(input, "DOCUMENT_PROCESSED", e.getMessage());
  throw e;
}

// ❌ WRONG - No event tracking
try {
  return processDocument(input);
} catch (ProcessingException e) {
  log.error("Failed", e);
  throw e;
}
```

### Maven POM Versioning
```xml
<!-- ✅ CORRECT - Internal modules first, then alphabetically sorted external libs -->
<properties>
  <!-- Internal Modules -->
  <internal.common.version>ND-1729-LIFECYCLE-SNAPSHOT</internal.common.version>
  
  <!-- External Libraries (alphabetically sorted) -->
  <assertj-core.version>3.24.2</assertj-core.version>
  <camel-quarkus-junit5.version>4.14.0</camel-quarkus-junit5.version>
  <compiler-plugin.version>3.14.0</compiler-plugin.version>
  <httpclient.version>5.5.1</httpclient.version>
  <lombok.version>1.18.42</lombok.version>
  <quarkus.platform.version>3.27.0</quarkus.platform.version>
</properties>

<dependencies>
  <dependency>
    <groupId>com.example</groupId>
    <artifactId>internal-common</artifactId>
    <version>${internal.common.version}</version>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>${lombok.version}</version>
  </dependency>
</dependencies>

<!-- ❌ WRONG - Magic numbers, no organization -->
<properties>
  <httpclient.version>5.5.1</httpclient.version>
  <internal.common.version>ND-1729-LIFECYCLE-SNAPSHOT</internal.common.version>
  <assertj-core.version>3.24.2</assertj-core.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.42</version> <!-- Magic number! -->
  </dependency>
</dependencies>
```

## Diagnostic Commands

The agent runs:

```powershell
# Get modified files
git diff --name-only HEAD -- '*.java'

# Check compilation
mvn compile

# Run tests
mvn test

# Static analysis (if available)
mvn spotbugs:check
mvn pmd:check
```

## Approval Criteria

- **✅ APPROVED**: No CRITICAL or HIGH issues
- **⚠️ WARNING**: MEDIUM issues only, can merge with ticket to fix
- **❌ BLOCKED**: CRITICAL or HIGH issues found, must fix before merge

## Report Format

```markdown
# Java Code Review Report

## Summary
- Files reviewed: 5
- CRITICAL issues: 0
- HIGH issues: 2
- MEDIUM issues: 3

## CRITICAL Issues
(none)

## HIGH Issues

### [DocumentService.java:45] Missing @Transactional
**Severity**: HIGH
**Category**: Quarkus Patterns
**Current Code**: 
```java
public Document create(CreateDocumentRequest req) {
  documentRepository.persist(doc);
}
```
**Recommendation**: Add @Transactional for data modifications

## Approval Status
⚠️ WARNING - Fix HIGH issues before merge
```

## Related Commands

- `/java-build` - Fix build errors first
- `/java-test` - Write tests with TDD
- `/code-review` - Generic code review
- `/security-review` - Deep security analysis

## Related Skills

- `quarkus-patterns` - Service, repository, Camel patterns
- `quarkus-security` - JWT, OIDC, validation
- `quarkus-tdd` - Testing patterns
- `java-coding-standards` - Naming, style, conventions

## Agent Details

Agent: `java-reviewer`
Tools: Read, Grep, Glob, PowerShell
Model: Sonnet
Standards: rules/java/* and skills/quarkus-*
