---
description: Enforce TDD workflow for Java/Quarkus. Write @Nested/@DisplayName tests with AAA pattern FIRST, then implement. Verify 80%+ coverage with JaCoCo.
---

# Java TDD Command

This command enforces test-driven development methodology for Java/Quarkus code using your specific test patterns.

## What This Command Does

1. **Define Interfaces/DTOs**: Scaffold method signatures and records first
2. **Write @Nested Tests**: Create comprehensive test cases with @DisplayName (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ line, 70%+ branch coverage with JaCoCo

## When to Use

Use `/java-test` when:
- Implementing new Java services or features
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in Java/Quarkus

## TDD Cycle

```
RED     → Write failing @Nested test with @DisplayName
GREEN   → Implement minimal code to pass
REFACTOR→ Improve code, keep tests green
VERIFY  → Check JaCoCo coverage 80%/70%
```

## Required Test Structure

Your tests MUST follow these patterns:

### @Nested Classes with @DisplayName

```java
@QuarkusTest
@RequiredArgsConstructor
class DocumentServiceTest {
  private final DocumentService documentService;
  
  @InjectMock
  DocumentRepository documentRepository;
  
  @InjectMock
  EventService eventService;
  
  @Nested
  @DisplayName("Create document")
  class CreateDocument {
    
    @Test
    @DisplayName("givenValidRequest_whenCreate_thenReturnDocument")
    void givenValidRequest_whenCreate_thenReturnDocument() {
      // Arrange - Setup test data and mocks
      CreateDocumentRequest request = new CreateDocumentRequest(...);
      
      // Act - Execute the method under test
      Document result = assertDoesNotThrow(() -> documentService.create(request));
      
      // Assert - Verify outcomes
      assertThat(result).isNotNull();
      assertThat(result.getReferenceNumber()).isEqualTo("REF-001");
      verify(documentRepository).persist(any(Document.class));
    }
  }
  
  @Nested
  @DisplayName("Find document")
  class FindDocument {
    // More tests...
  }
}
```

### Required Elements

1. **@QuarkusTest** - Integration test annotation
2. **@Nested classes** - Group related tests
3. **@DisplayName** - Descriptive names on classes and methods
4. **givenX_whenY_thenZ** - Test method naming convention
5. **AAA Pattern** - Explicit Arrange-Act-Assert comments
6. **AssertJ** - Use `assertThat()` NOT `assertEquals()`
7. **@InjectMock** - Mock dependencies in Quarkus tests
8. **assertDoesNotThrow** - For operations that should succeed

## TDD Workflow Steps

### Step 1: Define Interface (Type-First)

```java
public interface DocumentService {
  Document create(CreateDocumentRequest request);
  Optional<Document> findById(Long id);
  List<Document> findByStatus(DocumentStatus status);
}
```

### Step 2: Write Failing Tests (RED)

```java
@Test
@DisplayName("givenValidRequest_whenCreate_thenReturnDocument")
void givenValidRequest_whenCreate_thenReturnDocument() {
  // Arrange
  CreateDocumentRequest request = new CreateDocumentRequest(
      "REF-001", "Test", Instant.now(), List.of("invoice"));
  
  // Act
  Document result = documentService.create(request);
  
  // Assert
  assertThat(result).isNotNull();
  assertThat(result.getReferenceNumber()).isEqualTo("REF-001");
}
```

Run: `mvn test` → Should FAIL (implementation doesn't exist)

### Step 3: Implement Minimal Code (GREEN)

```java
@ApplicationScoped
@RequiredArgsConstructor
@Slf4j
public class DocumentService {
  private final DocumentRepository documentRepository;
  private final EventService eventService;
  
  @Transactional
  public Document create(CreateDocumentRequest request) {
    Document document = Document.from(request);
    documentRepository.persist(document);
    eventService.createSuccessEvent(document, "DOCUMENT_CREATED");
    return document;
  }
}
```

Run: `mvn test` → Should PASS

### Step 4: Add Edge Cases

```java
@Nested
@DisplayName("Error handling")
class ErrorHandling {
  
  @Test
  @DisplayName("givenDuplicateReference_whenCreate_thenThrowException")
  void givenDuplicateReference_whenCreate_thenThrowException() {
    // Arrange
    when(documentRepository.findByReferenceNumber("REF-001"))
        .thenReturn(Optional.of(new Document()));
    
    // Act & Assert
    assertThatThrownBy(() -> documentService.create(request))
        .isInstanceOf(DuplicateException.class)
        .hasMessageContaining("REF-001");
  }
}
```

### Step 5: Refactor with Tests Green

- Extract methods
- Improve naming
- Remove duplication
- Optimize logic

Run `mvn test` after each change → Keep tests passing

### Step 6: Verify Coverage

```powershell
mvn clean verify
```

Check: `target/site/jacoco/index.html`
- ✅ Line coverage ≥ 80%
- ✅ Branch coverage ≥ 70%

## Test Types to Write

### Unit Tests (Service Layer)
- Happy path scenarios
- Error conditions
- Edge cases (null, empty, boundary values)
- Business rule validation

### Integration Tests (Repository)
- Database queries
- Transactional behavior
- Complex joins

### Camel Route Tests
```java
@Test
@DisplayName("givenPayload_whenPublish_thenSendToQueue")
void givenPayload_whenPublish_thenSendToQueue() throws Exception {
  // Arrange
  AdviceWith.adviceWith(camelContext, "my-route", route -> {
    route.weaveByToString(".*rabbitmq.*").replace().to("mock:output");
  });
  MockEndpoint mock = camelContext.getEndpoint("mock:output", MockEndpoint.class);
  mock.expectedMessageCount(1);
  
  // Act
  producerTemplate.sendBody("direct:my-route", payload);
  
  // Assert
  mock.assertIsSatisfied();
}
```

### Async Tests (CompletableFuture)
```java
@Test
@DisplayName("givenFile_whenUploadAsync_thenReturnInfo")
void givenFile_whenUploadAsync_thenReturnInfo() {
  // Act
  CompletableFuture<StoredInfo> future = storageService.upload(file, context);
  
  // Assert
  assertThat(future).succeedsWithin(Duration.ofSeconds(5));
}
```

## Coverage Requirements

JaCoCo plugin configuration in pom.xml:

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.13</version>
  <executions>
    <execution>
      <id>check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
              <limit>
                <counter>BRANCH</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.70</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

## Principles

- **Tests first**: Always write test before implementation
- **Red-Green-Refactor**: Follow the cycle strictly
- **Minimal code**: Only write enough to pass the test
- **One test at a time**: Don't write all tests upfront
- **Comprehensive coverage**: Happy path + errors + edge cases
- **AssertJ only**: Never use JUnit assertions
- **Explicit AAA**: Always include Arrange-Act-Assert comments

## Related Commands

- `/java-build` - Fix build errors first
- `/java-review` - Review code quality and patterns
- `/tdd` - Generic TDD workflow (language-agnostic)
- `/test-coverage` - Check coverage metrics

## Related Skills

- `quarkus-tdd` - Complete TDD patterns and examples
- `quarkus-patterns` - Service layer patterns to implement
- `java-testing` - Testing standards and AssertJ usage

## Agent Details

Agent: `tdd-guide` (generic TDD agent)
Applies patterns from: `quarkus-tdd/SKILL.md` and `rules/java/testing.md`
