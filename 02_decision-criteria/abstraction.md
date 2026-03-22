$NOTE

# Decision Criteria for Generalization and Abstraction

Criteria for deciding whether to "extract" or "leave as-is" when you find code duplication.

---

## Fundamental Principle

> *"Premature abstraction leads to wrong abstraction."*

Abstraction increases the cost of understanding. It only has value when "enough reuse" exists — meaning 3+ call sites with genuinely identical logic, not just superficially similar code. Two similar-looking functions that evolve independently don't justify abstraction.

---

## Extraction Thresholds

| Occurrences | Decision | Extraction Target |
|---------|------|--------|
| 1 location | **Write it inline** | — |
| 2 locations | Extract to a helper function if the logic is identical | `lib/` or `_helpers.py` in the same package |
| 3+ locations | Extract to a shared module if the pattern is identical | `lib/` or `core/` |
| 5+ locations | Consider creating a base class or generic utility | `core/` or `common/` |

### Patterns That Should Be Consolidated

| Pattern | Consolidation Target | Reason |
|---------|--------|------|
| Authentication/authorization checks | `auth/dependencies.py` | Consistency of security |
| Error handling | `core/exceptions.py` | Unified error handling |
| Validation schemas | `schemas/` directory | Schema reuse |

---

## Signs You Should NOT Extract

### Bloated Configuration Dicts

```python
# BAD: More than 5 options -> the abstraction direction is wrong
action = create_action(
    schema=schema,
    handler=handler,
    middleware=middleware,
    error_handler=error_handler,
    retry_policy=retry_policy,
    cache_policy=cache_policy,
    rate_limit_policy=rate_limit_policy,
)

# GOOD: Extract only the common parts; implement special cases individually
base_action = create_base_action(common_options)

async def special_action(input_data: InputModel) -> OutputModel:
    transformed = transform(input_data)
    return await base_action(transformed)
```

### "We might use it in the future"

- Apply YAGNI (You Aren't Gonna Need It)
- Address future requirements when they actually emerge
- Don't abstract for a second occurrence that doesn't exist yet

### Similar but Not Identical

```python
# BAD: Forced generalization resulting in excessive conditional branching
def process_item(item: Item, item_type: str) -> Result:
    if item_type == "A":
        # Logic specific to A
        ...
    elif item_type == "B":
        # Logic specific to B
        ...
    # Only 10% is shared

# GOOD: If the shared portion is small, write them separately
def process_item_a(item: ItemA) -> ResultA: ...
def process_item_b(item: ItemB) -> ResultB: ...
```

---

## Class / Module Generalization Decisions

| Criteria | Generalize | Keep Separate |
|---------|----------|--------------|
| Logic is identical, only data types differ | Use generics: `Repository[T]` | |
| 70%+ of logic is shared | Base class + template method pattern | |
| Less than 30% of logic is shared | | Separate implementations |
| Generalization would result in 10+ constructor params | | Abstraction is inappropriate — 10+ params signals the class is trying to serve too many use cases. At that point, conditional logic inside the class typically exceeds the code you'd save by reusing it, and the parameter combinations become difficult to test and document. Split into focused variants instead. |

---

## Service / Repository Generalization Decisions

| Situation | Decision |
|------|------|
| Same CRUD pattern in 3+ repositories | Extract to a generic `BaseRepository[T]` |
| Same fetch -> validate -> transform pattern in 3+ services | Extract to a shared service mixin or base class |
| 3+ branches inside a single service method | Split into separate methods or strategy pattern |
| 5+ parameters for a service method | Too much responsibility. Consider splitting |
