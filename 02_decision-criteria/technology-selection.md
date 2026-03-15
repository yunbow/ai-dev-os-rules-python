$NOTE
# Decision Criteria for Technology Selection

A decision framework for introducing new technologies, libraries, and tools.

---

## Library Evaluation Criteria

| Evaluation Axis | Weight | Decision Criteria |
|--------|------|---------|
| **Type stub availability** | High | Does the library ship inline types (py.typed), or require a separate `types-*` stub package? Untyped libraries break static analysis |
| **Maintenance status** | High | Has there been a release in the last 6 months? This threshold catches abandoned projects (no release in 6+ months means likely unmaintained) while allowing for stable libraries with slower release cycles. Also check issue response speed. |
| **Dependency footprint** | High | How many transitive dependencies does it pull in? Fewer is better for security and reproducibility |
| **Breaking change frequency** | Medium | Frequency of major version upgrades and migration cost |
| **Learning cost** | Medium | Can the entire team understand it? |
| **Community size** | Low | Activity on Stack Overflow, GitHub, and PyPI download counts |
| **License** | Required | MIT / Apache 2.0 preferred. GPL requires consideration |

Weight definitions: **High** = can veto adoption on its own. **Medium** = weigh against other factors. **Low** = tiebreaker between otherwise equal options. **Required** = must pass, no trade-offs.

### De Facto Standard Identification

A library qualifies as "de facto standard" when it meets 2+ of:
- Referenced in the framework's official documentation (e.g., FastAPI docs mention it)
- 10x+ GitHub stars compared to the next alternative
- Used by the framework's own tutorials or starter templates
- Dominant in PyPI download counts for its category (>70% market share)

### Dependency Health Guidelines

| Concern | Acceptable | Countermeasure |
|---------|----------|------|
| Transitive dependencies | < 10 for non-framework libs | Audit with `pip-audit`, `pipdeptree` |
| Install size | Reasonable for the functionality | Consider lighter alternatives |
| Native extensions (C/Rust) | Only when performance requires it | Ensure wheels are available for all target platforms |
| Python version support | Supports current and previous stable Python | Avoid libraries that only support bleeding-edge Python |

---

## "Build vs Use" Decision

| Condition | Decision |
|------|------|
| Can be implemented in 50 lines or less | **Build**. Adding a dependency costs more |
| Project-specific domain logic | **Build**. Don't distort the domain to fit an external library |
| An area with 3+ competing libraries | **Choose carefully**. Prefer the de facto standard (see identification criteria above) |

---

## Web Framework Decision

| Approach | Verdict | Reason |
|------|------|------|
| FastAPI | Recommended | Async-first, Pydantic-native, auto-generated OpenAPI docs |
| Django + DRF | Acceptable | Full-featured, batteries-included, mature ORM |
| Flask | Acceptable | Simple, well-understood, but less opinionated about types |
| Litestar | Acceptable | Performance-focused, good type support |

---

## Database Selection Decision

| Requirement | Option | Decision Criteria |
|------|--------|---------|
| Single server, medium scale | SQLite | Prioritize simplicity. Easy to operate with file-based storage |
| High concurrent writes | PostgreSQL | When SQLite's WAL mode reaches its limits |
| Global distribution | CockroachDB / PlanetScale | When fast reads across regions are needed |
| Schemaless | MongoDB | Not recommended. Poor compatibility with type safety |

---

## ORM / Database Toolkit Decision

| Tool | Verdict | Reason |
|------|------|------|
| SQLAlchemy 2.0 | Recommended | Industry standard, excellent type support with 2.0 style |
| Alembic | Recommended | Pairs with SQLAlchemy for migrations |
| SQLModel | Acceptable | SQLAlchemy + Pydantic hybrid, good for FastAPI projects |
| Tortoise ORM | Acceptable | Async-native, Django-inspired |
| Raw SQL | Prohibited (except performance-critical paths) | SQL injection risk, no type safety |

---

## Authentication Method Decision

| Factor | JWT (Default) | Database Session |
|------|-----------------|-----------------|
| Scalability | Excellent | Shared store required |
| Distributed environments | Excellent | Session sharing required |
| Immediate invalidation | Handle with version field | Can be invalidated immediately |
| Recommended case | **Default** | When multi-session management is essential |
