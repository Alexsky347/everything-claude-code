---
paths:
  - "**/*.java"
---
# Java Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Java specific content.

## Naming Conventions

Use standard Java naming:

```java
// CORRECT: Classes and interfaces - PascalCase
public class DocumentService {}
public interface DocumentRepository {}
public record DocumentResponse(Long id, String name) {}

// CORRECT: Methods and variables - camelCase
private final DocumentRepository documentRepository;
public Document findByReferenceNumber(String referenceNumber) {}

// CORRECT: Constants - UPPER_SNAKE_CASE
private static final int MAX_PAGE_SIZE = 100;
private static final String DEFAULT_STATUS = "PENDING";

// CORRECT: Packages - lowercase
package com.example.invoice.domain.service;
```

## Immutability

Prefer immutable data structures:

```java
// WRONG: Mutable class
public class Document {
  public String name;
  public String status;
}

// CORRECT: Record or final fields
public record DocumentDto(Long id, String name, String status) {}

// OR with class
public class Document {
  private final Long id;
  private final String name;
  private String status; // Only if must be mutable
  
  // Constructor, getters only
}
```

## Error Handling

Use appropriate exceptions with meaningful messages:

```java
// CORRECT: Specific exceptions with context
try {
  Document document = documentRepository.findById(id)
      .orElseThrow(() -> new EntityNotFoundException(
          "Document not found with id: " + id));
  return document;
} catch (DataAccessException e) {
  log.error("Database error while fetching document {}", id, e);
  throw new ServiceException("Failed to retrieve document", e);
}

// WRONG: Generic exceptions or swallowing errors
try {
  // ...
} catch (Exception e) {
  // Silent failure - NEVER DO THIS
}
```

## Null Safety

Use Optional for potentially absent values:

```java
// CORRECT: Optional for return values
public Optional<Document> findById(Long id) {
  return documentRepository.findByIdOptional(id);
}

// CORRECT: Check and handle
documentService.findById(id)
    .ifPresent(doc -> log.info("Found: {}", doc))
    .orElseThrow(() -> new EntityNotFoundException("Not found"));

// WRONG: Returning null
public Document findById(Long id) {
  return null; // NEVER DO THIS
}
```

## Collections and Streams

Prefer streams for collection operations:

```java
// CORRECT: Stream API
List<String> activeNames = documents.stream()
    .filter(doc -> doc.getStatus() == DocumentStatus.ACTIVE)
    .map(Document::getName)
    .sorted()
    .toList();

// CORRECT: Collect to immutable collection
Set<Long> ids = documents.stream()
    .map(Document::getId)
    .collect(Collectors.toUnmodifiableSet());
```

## Lombok Usage

Use Lombok annotations for boilerplate reduction:

```java
// CORRECT: Service with Lombok
@Slf4j
@ApplicationScoped
@RequiredArgsConstructor
public class DocumentService {
  private final DocumentRepository documentRepository;
  private final EventService eventService;
  
  // No need for constructor or logger declaration
}

// CORRECT: Data classes
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DocumentEntity {
  private Long id;
  private String name;
  private DocumentStatus status;
}
```

## Annotations Placement

Place annotations in consistent order:

```java
// CORRECT: Annotation order
@ApplicationScoped     // CDI scope
@RequiredArgsConstructor  // Lombok
@Slf4j                 // Logging
public class MyService {}

// CORRECT: Method annotations
@Transactional
@CacheResult(cacheName = "documents")
public Document save(@Valid DocumentRequest request) {}
```

## Package Structure

Follow standard package layout:

```
com.example.invoice/
├── domain/
│   ├── entity/         # JPA entities
│   ├── repository/     # Panache repositories
│   └── service/        # Business logic
├── api/
│   ├── resource/       # JAX-RS resources
│   ├── dto/            # Request/Response DTOs
│   └── mapper/         # DTO mappers
├── infrastructure/
│   ├── config/         # Configuration
│   ├── messaging/      # Camel routes, publishers
│   └── storage/        # File storage, S3
└── exception/          # Custom exceptions
```

## Comments

Use Javadoc for public APIs:

```java
/**
 * Processes an invoice file through validation and storage.
 * 
 * @param filePath the path to the invoice file
 * @throws ValidationException if the invoice fails validation
 * @throws StorageException if file storage fails
 */
public void processFile(Path filePath) throws ValidationException, StorageException {
  // Implementation
}

// Use inline comments for complex logic
// Bypass schematron validation for CHORUS_FLOW messages
ValidationFlowConfig config = isChorusFlow 
    ? ValidationFlowConfig.xsdOnly()  // Only XSD validation
    : ValidationFlowConfig.allValidations();
```
