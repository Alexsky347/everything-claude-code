---
paths:
  - "**/*Test.java"
  - "**/*IT.java"
---
# Java Testing Patterns

> This file extends [common/testing.md](../common/testing.md) with Java/Quarkus specific patterns.

## Test Structure with @Nested Classes

Group related tests using @Nested classes with @DisplayName:

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
      // Arrange
      CreateDocumentRequest request = new CreateDocumentRequest(
          "REF-001", "Test document", Instant.now(), List.of("invoice"));
      
      // Act
      Document result = assertDoesNotThrow(() -> documentService.create(request));
      
      // Assert
      assertThat(result).isNotNull();
      assertThat(result.getReferenceNumber()).isEqualTo("REF-001");
      verify(documentRepository).persist(any(Document.class));
      verify(eventService).createSuccessEvent(any(), eq("DOCUMENT_CREATED"));
    }
    
    @Test
    @DisplayName("givenDuplicateReference_whenCreate_thenThrowException")
    void givenDuplicateReference_whenCreate_thenThrowException() {
      // Arrange
      CreateDocumentRequest request = new CreateDocumentRequest(
          "REF-001", "Test", Instant.now(), List.of("invoice"));
      when(documentRepository.findByReferenceNumber("REF-001"))
          .thenReturn(Optional.of(new Document()));
      
      // Act & Assert
      assertThatThrownBy(() -> documentService.create(request))
          .isInstanceOf(DuplicateException.class)
          .hasMessageContaining("REF-001");
    }
  }
  
  @Nested
  @DisplayName("Find document")
  class FindDocument {
    
    @Test
    @DisplayName("givenExistingId_whenFindById_thenReturnDocument")
    void givenExistingId_whenFindById_thenReturnDocument() {
      // Arrange
      Long documentId = 1L;
      Document expected = createTestDocument(documentId);
      when(documentRepository.findByIdOptional(documentId))
          .thenReturn(Optional.of(expected));
      
      // Act
      Optional<Document> result = documentService.findById(documentId);
      
      // Assert
      assertThat(result).isPresent();
      assertThat(result.get().getId()).isEqualTo(documentId);
    }
  }
}
```

## AAA Pattern with Explicit Comments

Always use Arrange-Act-Assert pattern with comments:

```java
@Test
@DisplayName("givenValidFile_whenProcessFile_thenStoreDocument")
void givenValidFile_whenProcessFile_thenStoreDocument() throws Exception {
  // Arrange - Setup test data, mocks, and expectations
  Path filePath = Paths.get("test-invoice.xml");
  LogContext logContext = new LogContext();
  InvoiceValidationResult validationResult = InvoiceValidationResult.valid();
  when(validator.validateFlowWithConfig(any(), any(), any(), any()))
      .thenReturn(validationResult);
  
  // Act - Execute the method under test
  DocumentProcessingResult result = assertDoesNotThrow(
      () -> processingService.processFile(filePath, logContext));
  
  // Assert - Verify outcomes
  assertThat(result).isNotNull();
  assertThat(result.getStatus()).isEqualTo(ProcessingStatus.SUCCESS);
  verify(fileStorage).store(any(InputStream.class), anyLong(), any());
  verify(eventService).createSuccessEvent(any(), eq("FILE_PROCESSED"));
}
```

## Mocking with Mockito

Use @InjectMock for Quarkus CDI beans:

```java
@QuarkusTest
class MyServiceTest {
  @InjectMock
  DependencyService dependencyService;
  
  @Inject
  MyService myService;  // Service under test with mocked dependencies
  
  @BeforeEach
  void setup() {
    // Reset mocks if needed
    Mockito.reset(dependencyService);
  }
  
  @Test
  void testWithMock() {
    // Given
    when(dependencyService.getData()).thenReturn("test-data");
    
    // When
    String result = myService.process();
    
    // Then
    assertThat(result).contains("test-data");
    verify(dependencyService).getData();
  }
}
```

## AssertJ Assertions

Use AssertJ for all assertions:

```java
// CORRECT: AssertJ assertions
assertThat(document).isNotNull();
assertThat(document.getName()).isEqualTo("Test");
assertThat(documents).hasSize(3)
    .extracting(Document::getStatus)
    .containsOnly(DocumentStatus.ACTIVE);

// Collection assertions
assertThat(list)
    .isNotEmpty()
    .hasSize(2)
    .contains(item1, item2)
    .doesNotContain(item3);

// Exception assertions
assertThatThrownBy(() -> service.methodThatThrows())
    .isInstanceOf(ValidationException.class)
    .hasMessageContaining("invalid")
    .hasNoCause();

// WRONG: JUnit assertions
assertEquals("Test", document.getName());  // Don't use this
assertTrue(list.size() == 2);  // Don't use this
```

## Camel Route Testing

Test Camel routes with AdviceWith and MockEndpoint:

```java
@QuarkusTest
class BusinessRulesRouteTest {
  @Inject
  CamelContext camelContext;
  
  @Inject
  ProducerTemplate producerTemplate;
  
  @Test
  @DisplayName("givenPayload_whenPublish_thenSendToRabbitMQ")
  void givenPayload_whenPublish_thenSendToRabbitMQ() throws Exception {
    // Arrange - Mock RabbitMQ endpoint
    AdviceWith.adviceWith(camelContext, "business-rules-publisher", route -> {
      route.weaveByToString(".*spring-rabbitmq.*")
          .replace()
          .to("mock:rabbitmq-output");
    });
    
    MockEndpoint mockEndpoint = camelContext.getEndpoint(
        "mock:rabbitmq-output", MockEndpoint.class);
    mockEndpoint.expectedMessageCount(1);
    
    BusinessRulesPayload payload = new BusinessRulesPayload("test-data");
    
    // Act - Send message
    producerTemplate.sendBody("direct:business-rules-publisher", payload);
    
    // Assert - Verify message sent
    mockEndpoint.assertIsSatisfied();
    String messageBody = mockEndpoint.getReceivedExchanges().get(0)
        .getIn().getBody(String.class);
    assertThat(messageBody).contains("test-data");
  }
}
```

## Testing Async Operations

Test CompletableFuture operations:

```java
@Test
@DisplayName("givenInputStream_whenUploadAsync_thenReturnStoredInfo")
void givenInputStream_whenUploadAsync_thenReturnStoredInfo() {
  // Arrange
  InputStream inputStream = new ByteArrayInputStream("test".getBytes());
  long size = 4L;
  LogContext logContext = new LogContext();
  
  // Act
  CompletableFuture<StoredDocumentInfo> future = 
      storageService.uploadFile(inputStream, size, logContext);
  
  // Assert - Wait and verify
  assertThat(future)
      .succeedsWithin(Duration.ofSeconds(5))
      .satisfies(info -> {
        assertThat(info.getPath()).isNotBlank();
        assertThat(info.getSize()).isEqualTo(size);
        assertThat(info.getUploadedAt()).isBeforeOrEqualTo(Instant.now());
      });
}

@Test
@DisplayName("givenUploadError_whenUploadAsync_thenCompletesExceptionally")
void givenUploadError_whenUploadAsync_thenCompletesExceptionally() {
  // Arrange
  InputStream inputStream = new ByteArrayInputStream("test".getBytes());
  when(s3Client.putObject(any(), any())).thenThrow(S3Exception.class);
  
  // Act
  CompletableFuture<StoredDocumentInfo> future = 
      storageService.uploadFile(inputStream, 4L, new LogContext());
  
  // Assert
  assertThatThrownBy(future::join)
      .hasCauseInstanceOf(StorageException.class);
}
```

## Testing with Test Containers

Use TestContainers for integration tests:

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
class DocumentRepositoryIT {
  @Inject
  DocumentRepository documentRepository;
  
  @Test
  @DisplayName("givenDocument_whenPersist_thenCanRetrieve")
  void givenDocument_whenPersist_thenCanRetrieve() {
    // Arrange
    Document document = new Document();
    document.setReferenceNumber("REF-001");
    document.setStatus(DocumentStatus.ACTIVE);
    
    // Act
    documentRepository.persist(document);
    Optional<Document> retrieved = 
        documentRepository.findByReferenceNumber("REF-001");
    
    // Assert
    assertThat(retrieved).isPresent();
    assertThat(retrieved.get().getStatus()).isEqualTo(DocumentStatus.ACTIVE);
  }
}

public class PostgresTestResource implements QuarkusTestResourceLifecycleManager {
  private PostgreSQLContainer<?> postgres;
  
  @Override
  public Map<String, String> start() {
    postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    postgres.start();
    return Map.of(
        "quarkus.datasource.jdbc.url", postgres.getJdbcUrl(),
        "quarkus.datasource.username", postgres.getUsername(),
        "quarkus.datasource.password", postgres.getPassword());
  }
  
  @Override
  public void stop() {
    postgres.stop();
  }
}
```

## JaCoCo Coverage Configuration

Configure JaCoCo in pom.xml:

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.13</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>
    <execution>
      <id>check</id>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
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

## Comprehensive Test Coverage

Cover all scenarios:

```java
@Nested
@DisplayName("Error handling")
class ErrorHandling {
  @Test
  @DisplayName("givenDatabaseError_whenSave_thenThrowServiceException")
  void givenDatabaseError_whenSave_thenThrowServiceException() {
    when(repository.persist(any())).thenThrow(PersistenceException.class);
    assertThatThrownBy(() -> service.save(document))
        .isInstanceOf(ServiceException.class);
  }
  
  @Test
  @DisplayName("givenInvalidData_whenValidate_thenThrowValidationException")
  void givenInvalidData_whenValidate_thenThrowValidationException() {
    assertThatThrownBy(() -> service.validate(invalidDocument))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining("invalid");
  }
}

@Nested
@DisplayName("Edge cases")
class EdgeCases {
  @Test
  @DisplayName("givenEmptyList_whenProcess_thenReturnEmptyResult")
  void givenEmptyList_whenProcess_thenReturnEmptyResult() {
    assertThat(service.processAll(List.of())).isEmpty();
  }
  
  @Test
  @DisplayName("givenNullOptional_whenFindById_thenReturnEmpty")
  void givenNullOptional_whenFindById_thenReturnEmpty() {
    when(repository.findByIdOptional(999L)).thenReturn(Optional.empty());
    assertThat(service.findById(999L)).isEmpty();
  }
}
```
