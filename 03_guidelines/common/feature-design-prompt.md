# Feature Design Prompt

A prompt template for making guideline-based design decisions during the design phase of new features.

---

## Prompt

```text
Please design the following feature.

## Design Process

### 1. Organizing Requirements
- What problem does this feature solve
- Who is affected and what is the expected outcome

### 2. Architecture Decisions (02_decision-criteria/architecture.md)

#### Data Flow Design
- Data sources (DB / External API / User input)
- Data flow (Router → Service → Repository)
- State management (DB session, caching)

### 3. File Structure Design

Design the file structure following project-structure.md:
- List files to create or modify
- Assign each file to the correct layer (router/service/repository/schema/model)
- Verify no cross-feature imports are introduced

### 4. Security Design (common/security.md)
- [ ] Is authentication required → Depends(get_current_user)
- [ ] Is resource ownership check required
- [ ] Input validation schema definition (Pydantic BaseModel)
- [ ] Is rate limiting required
- [ ] Handling of sensitive data (encryption/masking)

### 5. Error Handling Design (02_decision-criteria/error-strategy.md)
- List and classify possible errors (System / Application / User)
- Map to custom exception classes
- Identify retryable errors

### 6. Test Strategy (common/testing.md)
- Unit test targets (business logic, Pydantic schemas)
- Integration test targets (API endpoints, DB operations)
- E2E test targets (user journeys)

## Design Checklist

### Required
- [ ] Authentication via Depends() in protected endpoints
- [ ] Server-side input validation with Pydantic
- [ ] Custom exception hierarchy for error handling
- [ ] Structured log output (structlog)
- [ ] File and variable names following naming conventions (common/naming.md)

### Recommended
- [ ] Internationalization support
- [ ] API documentation (OpenAPI schema)
- [ ] Performance optimization (async, query optimization)
- [ ] Edge case testing

### Requires Decision
- [ ] Can existing common patterns be reused (02_decision-criteria/abstraction.md)
- [ ] Is introducing new technology/libraries needed (02_decision-criteria/technology-selection.md)

## Output Format

### Design Document

#### Overview
{Explain the purpose of the feature in 1-2 sentences}

#### File Structure
{List of files to create or modify}

#### Data Flow
{Flow from request → router → service → repository → database}

#### Security
{Authentication, authorization, and validation design}

#### Error Handling
{Possible errors and how to handle them}

#### Test Plan
{Test targets and types}

#### Open Items
{Points that cannot be decided at the design stage}
```
