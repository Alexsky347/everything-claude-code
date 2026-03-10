# Java Patterns

## Service Layer Pattern

Use CDI with constructor injection:

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

## Repository Pattern (Panache)

Use Panache for data access:

```java
@ApplicationScoped
public class DocumentRepository implements PanacheRepository<Document> {
  
  public Optional<Document> findByReferenceNumber(String referenceNumber) {
    return find("referenceNumber", referenceNumber).firstResultOptional();
  }
  
  public List<Document> findByStatus(DocumentStatus status, int page, int size) {
    return find("status = ?1 order by createdAt desc", status)
        .page(page, size)
        .list();
  }
}
```

## Event Service Pattern

Track operations with event service:

```java
@ApplicationScoped
@RequiredArgsConstructor
public class EventService {
  private final EventRepository eventRepository;
  
  public void createSuccessEvent(Object payload, String eventType) {
    Event event = new Event();
    event.setType(eventType);
    event.setStatus(EventStatus.SUCCESS);
    event.setPayload(serializePayload(payload));
    event.setTimestamp(Instant.now());
    eventRepository.persist(event);
  }
  
  public void createErrorEvent(Object payload, String eventType, String errorMessage) {
    Event event = new Event();
    event.setType(eventType);
    event.setStatus(EventStatus.ERROR);
    event.setErrorMessage(errorMessage);
    event.setPayload(serializePayload(payload));
    eventRepository.persist(event);
  }
}
```

## Camel Messaging Pattern

Use Camel for message routing:

```java
@ApplicationScoped
@RequiredArgsConstructor
public class BusinessRulesPublisher {
  private final ProducerTemplate producerTemplate;
  
  public void publishAsync(BusinessRulesPayload payload) {
    producerTemplate.asyncSendBody("direct:business-rules-publisher", payload);
  }
}

@ApplicationScoped
public class BusinessRulesRoute extends RouteBuilder {
  @Override
  public void configure() {
    from("direct:business-rules-publisher")
        .routeId("business-rules-publisher")
        .log("Publishing message: ${body}")
        .marshal().json(JsonLibrary.Jackson)
        .toF("spring-rabbitmq:%s?hostname=%s", queueName, rabbitHost);
  }
}
```

## LogContext Pattern

Use scoped logging context for tracing:

```java
public void processDocument(Path filePath) {
  LogContext logContext = CustomLog.getCurrentContext();
  try (SafeAutoCloseable ignored = CustomLog.startScope(logContext)) {
    logContext.put("documentId", document.getId().toString());
    logContext.put("traceId", generateTraceId());
    
    log.info("Processing document");
    // All logs in this scope include context
    
  } catch (Exception e) {
    log.error("Processing failed", e);
    throw e;
  }
}
```

## Async Operations with CompletableFuture

Use CompletableFuture for async operations:

```java
public CompletableFuture<StoredDocumentInfo> uploadFile(
    InputStream inputStream, 
    long size,
    LogContext logContext) {
  
  return CompletableFuture.supplyAsync(() -> {
    try (SafeAutoCloseable ignored = CustomLog.startScope(logContext)) {
      String path = generateStoragePath();
      s3Client.putObject(request, body);
      return new StoredDocumentInfo(path, size, Instant.now());
    } catch (Exception e) {
      log.error("Upload failed", e);
      throw new StorageException("Upload failed", e);
    }
  }, executorService);
}

// Usage
CompletableFuture<StoredDocumentInfo> future = uploadFile(stream, size, context);
StoredDocumentInfo result = future.join(); // Wait for completion
```

## Conditional Flow Pattern

Implement conditional logic based on runtime parameters:

```java
boolean isChorusFlow = Boolean.parseBoolean(logContext.get(CHORUS_FLOW));

// Different validation based on flow type
ValidationFlowConfig config = isChorusFlow
    ? ValidationFlowConfig.xsdOnly()
    : ValidationFlowConfig.allValidations();

InvoiceValidationResult result = validator.validateFlowWithConfig(
    filePath, config, format, logContext);

FlowProfile profile = isChorusFlow
    ? FlowProfile.EXTENDED_CTC_FR  // Default for CHORUS
    : computeFlowProfile(result);
```

## DTO Validation

Use Bean Validation on DTOs:

```java
public record CreateDocumentRequest(
    @NotBlank @Size(max = 200) String referenceNumber,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant validUntil,
    @NotEmpty List<@NotBlank String> categories) {}

@Path("/api/documents")
public class DocumentResource {
  @POST
  public Response create(@Valid CreateDocumentRequest request) {
    // Validation happens automatically
  }
}
```

## Exception Mapping

Map exceptions to HTTP responses:

```java
@Provider
public class ValidationExceptionMapper 
    implements ExceptionMapper<ConstraintViolationException> {
  
  @Override
  public Response toResponse(ConstraintViolationException exception) {
    String message = exception.getConstraintViolations().stream()
        .map(cv -> cv.getPropertyPath() + ": " + cv.getMessage())
        .collect(Collectors.joining(", "));
    
    return Response.status(Response.Status.BAD_REQUEST)
        .entity(Map.of("error", "validation_error", "message", message))
        .build();
  }
}
```

## Configuration Pattern

Use YAML configuration with profiles:

```yaml
"%dev":
  quarkus:
    datasource:
      jdbc:
        url: jdbc:postgresql://localhost:5432/dev_db
  rabbitmq:
    host: localhost
    
"%prod":
  quarkus:
    datasource:
      jdbc:
        url: ${DATABASE_URL}
  rabbitmq:
    host: ${RABBITMQ_HOST}
```

Inject configuration:

```java
@ConfigProperty(name = "rabbitmq.host")
String rabbitHost;

@ConfigProperty(name = "file.max-size", defaultValue = "10485760")
long maxFileSize;
```
