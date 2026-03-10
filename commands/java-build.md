---
description: Fix Java/Quarkus build errors, Maven compilation failures, and dependency conflicts incrementally. Invokes the java-build-resolver agent for minimal, surgical fixes.
---

# Java Build and Fix

This command invokes the **java-build-resolver** agent to incrementally fix Java/Quarkus build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `mvn clean compile`, `mvn dependency:tree`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/java-build` when:
- `mvn clean compile` fails with errors
- Dependency conflicts prevent compilation
- Lombok annotation processing fails
- Quarkus dev mode won't start
- After pulling changes that break the build
- CDI deployment errors occur

## Diagnostic Commands Run

```powershell
# Primary build check
mvn clean compile

# Dependency analysis
mvn dependency:tree
mvn dependency:analyze

# Effective POM (resolved versions)
mvn help:effective-pom

# Quarkus validation
quarkus dev --dry-run
```

## Common Issues Fixed

### Compilation Errors
- Missing imports or wrong package paths
- Type mismatches and casting issues
- Undefined methods or fields
- Access modifier violations
- Generic type issues

### Dependency Issues
- Version conflicts (e.g., multiple Jackson versions)
- Missing dependencies in pom.xml
- Wrong scope (compile vs provided vs test)
- Exclusion needed for transitive dependencies
- Quarkus BOM version mismatches

### Lombok Issues
- Missing annotation processor configuration
- Wrong Lombok version for Java version
- Conflicting annotation processors
- Missing @RequiredArgsConstructor fields

### Quarkus/CDI Issues
- Missing bean scopes (@ApplicationScoped)
- Ambiguous injectable beans
- Missing Quarkus extensions
- Configuration property errors
- Panache entity issues

### Build Configuration
- Java version mismatch (source/target)
- Missing maven-compiler-plugin configuration
- Wrong encoding settings
- Plugin version conflicts

## How It Works

The java-build-resolver agent will:

1. **Run `mvn clean compile`** and capture all errors
2. **Categorize errors** by type (compilation, dependency, configuration)
3. **Fix highest priority error** (compilation before configuration)
4. **Verify fix** with `mvn compile`
5. **Continue** until build succeeds or stop condition met
6. **Run tests** to ensure nothing broke

## Fix Strategy

```text
Priority  →  Issue Type           →  Action
────────────────────────────────────────────────
1         →  Dependency conflict  →  Add exclusion or pin version
2         →  Missing dependency   →  Add to pom.xml
3         →  Compilation error    →  Fix syntax/imports
4         →  Lombok issue         →  Configure annotation processor
5         →  CDI issue            →  Add scope or qualifier
6         →  Configuration        →  Fix application.yml
```

## Principles

- **Minimal change**: Only fix what's broken
- **Root cause**: Don't suppress, fix the underlying issue
- **Verify each step**: Build after each fix
- **No refactoring**: Scope is build fix only
- **Preserve style**: Keep existing patterns

## Stop Conditions

Agent will stop and report if:
- Build succeeds (success!)
- Same error persists after 3 attempts (needs manual review)
- Dependency conflict requires architecture decision
- Missing Quarkus extension requires pom.xml update decision

## Example Usage

### Scenario 1: Dependency Conflict
```
Error: Cannot find symbol: class ObjectMapper
→ Agent adds jackson-databind to pom.xml
→ Build succeeds
```

### Scenario 2: Lombok Not Processing
```
Error: Cannot resolve symbol 'RequiredArgsConstructor'
→ Agent adds Lombok to annotationProcessorPaths
→ Build succeeds
```

### Scenario 3: CDI Ambiguous Bean
```
Error: Ambiguous dependencies for DocumentService
→ Agent adds @Named qualifier or specific @Inject
→ Build succeeds
```

## Related Commands

- `/java-test` - Run TDD workflow with proper test structure
- `/java-review` - Comprehensive code review
- `/code-review` - Generic security and quality review
- `/build-fix` - Generic build error resolution

## Related Skills

- `quarkus-patterns` - Service layer, Camel, LogContext patterns
- `quarkus-security` - JWT, OIDC, validation patterns
- `java-coding-standards` - Naming, immutability, error handling

## Agent Details

Agent: `java-build-resolver`
Tools: Read, Write, Edit, PowerShell, Grep, Glob
Model: Sonnet
