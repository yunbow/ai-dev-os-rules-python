$NOTE

# Design Principles

This document defines the fundamental principles that underpin all projects.
It articulates the "why" behind our guidelines and serves as the ultimate reference point when making difficult decisions.

---

## Three Pillars: Correctness, Observability, Pragmatism

All design decisions are made in this order of priority.

```text
1. Correctness   — Guarantee correct behavior through types and validation
2. Observability  — Ensure the running state can be observed and traced
3. Pragmatism     — Achieve goals with the minimum necessary complexity
```

### 1. Correctness

> *"Eliminate runtime errors by catching them during static analysis (mypy/pyright)."*

- Type hints catch errors via static analysis (mypy/pyright) before code reaches production
- Security is built into the architecture, not bolted on after the fact

> The "cost" of type hints is negligible compared to the cost of runtime errors.

### 2. Observability

> *"Understand what is happening in production without redeploying."*

- Assign a Trace ID to every request and propagate it across layers
- Automatically mask PII. Never expose secrets in logs
- Continuously measure metrics (error rates, latency, SLO/SLA)

> Always maintain a state where "looking at the logs tells you everything."

### 3. Pragmatism

> *"Don't build it until you need it. When you need it, build it the simplest way possible."*

- Copy-paste is not evil. Don't abstract until duplication appears in 3 places. The number 3 is deliberate: with 1 occurrence you have no pattern; with 2 you see similarity but may be misled by coincidence; with 3 you have enough evidence to extract a correct, stable abstraction. Abstracting at 2 often produces the *wrong* abstraction because you lack sufficient examples to identify the true commonality
- Convention over Configuration
- Prefer "constrained systems" over "flexible systems." Constraints force design decisions early, reduce the surface area for bugs, and make code predictable. A constrained system (e.g., a fixed set of allowed status values as a `Literal` type or `Enum`) prevents invalid states by construction, whereas a flexible system (e.g., accepting arbitrary strings) defers validation to runtime and scatters guard logic throughout the codebase
- Exceptions are allowed, but make the reasons visible (comments are required on `# noqa` or `type: ignore`)

> Over-generalization produces code that is generically useless.

---

## Core Policies

### API First

- Business logic lives in the service layer, not in route handlers or CLI commands
- All data mutations go through service functions. Direct database queries from route handlers are prohibited
- Data flows unidirectionally: Route Handlers / CLI Commands -> Service Layer -> Repository Layer -> Database

### Separation of Concerns

- Each layer (API -> Service -> Repository -> Database) has clearly defined responsibilities
- Features are self-contained per domain (routes, services, schemas, models)
- Shared logic is consolidated in a `lib/` or `core/` package. Do not create vague `utils.py` files

---

## Security Philosophy: Zero Trust

> *"Trust no input. Every user is a potential attacker."*

1. **API Layer**: Pydantic validation, CORS, rate limiting
2. **Auth Layer**: Session/token validation, IDOR checks (always verify ownership for resource access)
3. **DB Layer**: ORM-only access (raw SQL is prohibited), user_id filtering
4. **Data Layer**: Encrypt sensitive fields

Apply the principle of least privilege to everything (DB users, API scopes, feature flags).

---

## Approach to Errors

> *"Don't try to prevent every failure — detect, contain, and communicate them."*

- Errors fall into 3 categories: System (unexpected), Application (expected), User (input mistakes)
- Do not leak internal error details to the client (return sanitized messages)
- Non-critical feature failures must not crash the entire app (exception handlers, fallback responses, circuit breakers)
- Always ensure traceability via Request ID

---

## API Design Philosophy

> *"Constraints create consistency, and consistency creates speed."*

- Consistent response schemas using Pydantic models
- Pydantic models serve as the Single Source of Truth for data shapes
- Layers are separated: Pydantic schemas -> Service layer -> Domain models -> ORM models
