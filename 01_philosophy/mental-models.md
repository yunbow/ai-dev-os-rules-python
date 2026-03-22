$NOTE

# Mental Models

This defines the thinking patterns behind design decisions.
These are not concrete rules, but the "way of thinking" from which rules emerge.

---

## 1. Constraints Breed Creativity

> *"Fewer choices lead to faster and better decisions."*

"Faster" here means reduced decision fatigue and shorter code review cycles. When the framework constrains choices (e.g., "use Literal types over plain strings," "no bare `except`"), developers spend zero time debating the approach and reviewers can focus on logic rather than style. This compounds across a team — eliminating 10 micro-decisions per PR across 50 PRs/week saves meaningful cognitive load.

- `strict` mode in mypy/pyright doesn't take away freedom — it takes away room for bugs
- Ruff's opinionated defaults don't restrict expression — they guarantee consistency
- "No `Any`," "No bare `except`," "No `print()` in production code" are constraints that raise the quality floor

> Unlimited freedom produces unlimited inconsistency.

---

## 2. Code is Read More Than Written

> *"Write code that the future reader (including yourself 3 months later) can understand as quickly as possible."*

- Write code where intent is clear, rather than optimizing for brevity or shortness
- Consistency in naming conventions directly impacts searchability and refactoring ease
- Don't use comments to explain logic. Only document intent, side effects, and edge cases

> The code should convey not "what it does" but "why it does it this way."

---

## 3. Validate at Boundaries

> *"Trust the internals, but validate strictly at the points of contact with the outside."*

```text
External Input ──[Pydantic]──> Route Handler ──[Type Hints]──> Service Layer ──[ORM]──> DB
               ^ Guard here                    ^ Protected by types here
```

- User input, API requests, webhooks, environment variables — all are "external"
- Once data passes validation, trust the types internally
- Don't over-apply defensive programming inside internal code (types provide the guarantee)

---

## 4. Design for Failure

> *"Don't write code that assumes things work. Design with the assumption that things break."*

- All external communication can fail (timeouts, rate limits, service outages)
- Use types and explicit result patterns to prevent forgetting error handling (e.g., `Result[T, E]` or custom exception hierarchies)
- Design so that non-critical feature failures don't take down critical features. To distinguish: **critical features** are those whose failure prevents the user from completing the core task they came to do (e.g., authentication, checkout, data saving). **Non-critical features** are enhancements that improve the experience but whose absence doesn't block the core workflow (e.g., analytics, notifications, recommendations, avatar display). When in doubt, ask: "Can the user still accomplish their primary goal without this feature?"
- Distinguish between retryable and non-retryable errors

> Testing only the happy path is like selling umbrellas only on sunny days.

---

## 5. Layered Architecture

> *"If the data flow is consistent, most bugs disappear."*

```text
Route Handlers / CLI Commands (entry points)
       |
    Service Layer (business logic / orchestration)
       |
  Repository Layer (data access)
       |
    Database (persistence)
```

- Data flows down through layers; each layer only calls the layer directly below it
- Do not use global mutable state. Use dependency injection to share services
- The service layer is the single source of truth for business rules. Route handlers are thin coordination layers

> Skipping layers is convenient in small apps but creates chaos in large ones.

---

## 6. Rule of Three (Timing of Abstraction)

> *"Premature abstraction leads to wrong abstraction."*

| Occurrences | Action |
|-------------|--------|
| 1 place | Write it inline — you have no evidence of a pattern yet |
| 2 places | Extract to a helper function if the logic is identical — but remain skeptical; two instances may be coincidentally similar |
| 3+ places | Extract to a shared module if the pattern is identical — three occurrences give enough evidence to identify the true common pattern and design a stable interface |
| 5+ places | Consider creating a base class or generic utility — at this scale the maintenance cost of duplication exceeds the comprehension cost of abstraction |

- An abstraction used in only one place only adds comprehension cost
- If the configuration dict/dataclass keeps growing, it's a sign the abstraction is heading in the wrong direction

---

## 7. Make the Implicit Explicit

> *"Implicit agreements become bugs the moment the team changes."*

- Make side effects clear through naming (`fetch_user()` = has side effects, `calculate_price()` = pure)
- Always include a reason with `# noqa` or `type: ignore`
- Make technical debt visible with `FIXME` comments — don't leave it untracked
- Convey intent through naming convention suffixes and prefixes (`*_service.py`, `*_repository.py`, `create_*_schema`)
