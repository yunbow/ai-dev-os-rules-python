# AI-Assisted Coding Guidelines

This guideline defines the **standards for working with AI coding assistants** to maximize quality and minimize rework.

# 1. Key Principles

* **Rules over instructions**
  Do not rely on natural language instructions alone. Codify expectations as explicit rules in guidelines.
* **Verify, don't trust**
  AI-generated code must be reviewed with the same rigor as human-written code. AI output confidence does not equal correctness.
* **Structure for AI readability**
  Write code in patterns that AI can consistently reproduce. Predictable structures reduce hallucination.
* **Less is more**
  Too many rules degrade AI performance. A poorly written or overly verbose context file can produce worse results than no context file at all. Only include rules that address real, observed problems.

# 2. Prompting and Context

## 2.1 Context Management
* MUST reference AI Dev OS guidelines from CLAUDE.md / .cursorrules / AGENTS.md — do not inline rules in prompts
* MUST keep the context file (CLAUDE.md, etc.) under 500 lines — split into referenced files if longer
* MUST NOT include sensitive information (API keys, credentials, PII) in prompts or context files
* SHOULD NOT list all guideline files in the context file — only include the 10-15 most impactful guidelines (or approximately 300-500 lines total)
* SHOULD prioritize guidelines for areas where AI frequently makes mistakes (security, error handling, naming)
* MUST NOT include vague rules ("write clean code", "handle errors appropriately") — these override the model's good defaults with ambiguity

### Why fewer rules can be better
Research shows that AI models perform worse when given too many context rules. The model's attention is diluted across all rules, causing it to miss important ones. A focused set of 10-15 high-impact rules (approximately 300-500 lines) consistently outperforms an exhaustive set of 30+ rules. Add rules only when you observe a recurring problem that the AI fails to solve on its own.

### Two-tier context strategy: Static context + Dynamic checks

| Tier | What | How | Purpose |
|------|------|-----|---------|
| **Static context** | CLAUDE.md / .cursorrules / AGENTS.md | Always loaded by AI | Guide real-time code generation with 10-15 high-impact rules (~300-500 lines) |
| **Dynamic checks** | `ai-dev-os-check`, `ai-dev-os-scan`, `ai-dev-os-review` | Invoked on demand | Verify against ALL guidelines (30+) at review time |

**Static context (CLAUDE.md) should include:**
- Framework-specific overview (architecture, patterns)
- Security rules (always critical, even if rarely violated)
- Error handling patterns (most common AI mistake area)
- Naming conventions (consistency across codebase)
- Project structure (AI needs this for correct file placement)

**Static context should NOT include:**
- Every guideline file (causes attention dilution)
- Rules the AI already follows by default (e.g., "use type hints" — modern models know this)
- Rarely relevant rules (rate-limiting, CORS, CI/CD — check these via `ai-dev-os-scan` instead)

**Why not auto-remove low-frequency rules from CLAUDE.md?**
Low violation frequency does not mean low importance. Security rules with zero violations are *working* — removing them would cause immediate regression. Instead, curate the static context manually at setup time, and rely on dynamic checks for comprehensive coverage.

### Token cost awareness
The static context (8-10 guideline files referenced in CLAUDE.md) typically adds approximately 3,000-5,000 tokens to each request. This is a small fraction of modern model context windows (128K-1M tokens) but can accumulate across many requests. To minimize overhead:
* Keep CLAUDE.md references to 10-15 files (the two-tier strategy above)
* Use `ai-dev-os-report` to measure your actual token footprint
* Dynamic checks (`ai-dev-os-check`, `ai-dev-os-scan`) do not add to per-request cost — they run on demand

## 2.2 Task Scoping
* MUST break large tasks into small, focused requests (one file or one function per request)
* MUST specify the exact file path and function name when requesting changes to existing code
* SHOULD provide a Before/After example when explaining a pattern you want the AI to follow

## 2.3 Effective Prompting Patterns
* **Do**: "Add input validation using Pydantic BaseModel as defined in validation.md"
* **Don't**: "Add validation" (too vague — AI will guess the implementation pattern)
* **Do**: "Refactor this function following the early-return pattern in code.md section 2.3"
* **Don't**: "Clean up this code" (no specific standard referenced)

# 3. AI-Generated Code Review

## 3.1 Mandatory Review Checklist
When reviewing AI-generated code, MUST check:

* [ ] **Security**: No hardcoded secrets, proper input validation, no SQL injection vectors
* [ ] **Error handling**: Follows the project's error handling pattern (custom exceptions, not bare except)
* [ ] **Naming**: Follows naming.md conventions (not AI's default naming preferences)
* [ ] **Imports**: No phantom imports (libraries that don't exist or aren't installed)
* [ ] **Types**: Full type hints, no `Any` type introduced
* [ ] **Consistency**: Matches existing code patterns in the same file/module
* [ ] **Tests**: If the change is testable, tests are included or updated

## 3.2 Common AI Pitfalls
AI coding assistants frequently make these mistakes. Be especially vigilant:

| Pitfall | What to Check | Guideline Reference |
|---------|--------------|---------------------|
| **Phantom imports** | Library exists in pyproject.toml / requirements.txt | code.md |
| **Outdated API usage** | API matches the installed version | code.md |
| **Bare except clauses** | Uses specific exception types | error-handling.md |
| **Hardcoded values** | Uses environment variables or pydantic-settings | env.md, security.md |
| **Missing auth checks** | Endpoints use Depends() for auth | security.md |
| **Inconsistent naming** | Follows project naming conventions | naming.md |
| **Unnecessary abstraction** | Justified by actual reuse, not hypothetical | code.md |
| **Missing edge cases** | None checks, empty collections, boundary values | validation.md |
| **shell=True in subprocess** | Never with user input | security.md |

## 3.3 Review Depth by Risk Level

| Risk Level | Scope | Review Depth |
|-----------|-------|-------------|
| **High** | Auth, payment, data deletion, security | Line-by-line review, test coverage required |
| **Medium** | Business logic, API endpoints, data models | Functional review, spot-check implementation |
| **Low** | Utility functions, formatting, documentation | Quick scan for obvious issues |

# 4. Code Structure for AI Consistency

## 4.1 Predictable File Structure
* MUST follow the project structure defined in project-structure.md — AI generates more consistent code when file locations are predictable
* MUST use `__init__.py` for package exports — AI can import correctly without guessing paths
* MUST NOT create deeply nested packages (max 4 levels) — AI struggles with deeply nested paths

## 4.2 Function Design for AI
* MUST keep functions ≤ 30 lines — AI generates better code when functions are focused
* MUST use early returns — AI is more likely to handle edge cases correctly with early-return patterns
* MUST add type hints for all function parameters and return values — AI uses them as contracts
* SHOULD use docstrings for complex business logic — AI reads docstrings as additional context

```python
# GOOD: AI can reproduce this pattern consistently
async def process_refund(
    refund_request: RefundRequest,
    current_user: User = Depends(get_current_user),
) -> RefundResult:
    """Validates and processes a payment refund."""
    validation = validate_refund(refund_request)
    if not validation.is_valid:
        raise ValidationError(validation.errors)

    # ... business logic
    return RefundResult(success=True, refund=refund)


# BAD: AI will generate inconsistent patterns
def process_refund(req):
    try:
        # ... everything in one big try-except
    except:
        print("error")
```

## 4.3 Pattern Consistency
* MUST use the same pattern for similar operations across the codebase (e.g., all CRUD operations follow the same structure)
* MUST NOT mix patterns in the same module (e.g., sync and async, different error handling approaches)
* SHOULD create a reference implementation for each pattern, then tell AI "follow the pattern in [file]"

# 5. AI Dev OS Integration

## 5.1 Rule Harvesting Workflow
After every code review where AI-generated code was corrected:
1. Identify what the AI got wrong
2. Ask: "Is this a one-off mistake or a recurring pattern?"
3. If recurring, extract it as a rule using `ai-dev-os-extract` (or equivalent)
4. Add the rule to the appropriate L3 guideline

## 5.2 Guideline Effectiveness Tracking
* SHOULD track which guidelines are most frequently violated by AI — these may need clearer wording or examples
* SHOULD track which guidelines AI follows perfectly — these confirm the rule format is effective
* Use `ai-dev-os-report` periodically to identify patterns

## 5.3 When AI Cannot Follow a Rule
If AI consistently fails to follow a specific guideline:
1. Add a concrete code example (Before/After) to the guideline
2. Add the rule to the project's checklist template
3. If still failing, consider adding a pre-commit hook or linter rule for automated enforcement

## 5.4 Rule Suppression (Escape Hatch)
When a specific guideline rule is not applicable to a particular module or file, suppress it explicitly rather than ignoring it silently:

* **Project-level suppression**: Add a `project-specific/` guideline that overrides the rule for specific directories or modules. The Specificity Cascade ensures project-specific rules take precedence over common rules.
* **File-level suppression**: Add a comment at the top of the file explaining why a rule does not apply:
  ```
  # ai-dev-os-ignore: rule-name — reason for suppression
  ```
* **Documentation**: All suppressions MUST include a reason. Undocumented suppressions should be flagged by `ai-dev-os-scan`.

This is analogous to `// eslint-disable-next-line` or `# noqa` — every project has legitimate exceptions.

## 5.5 Rule Maturity Labels

Tag each L3 guideline rule with a maturity label to indicate confidence level:

| Label | Meaning | Action |
|-------|---------|--------|
| `[draft]` | Newly extracted rule, not yet validated across projects | Review after 2-4 weeks of use |
| `[proven]` | Validated through multiple code reviews, AI consistently follows it | Promote to static context (CLAUDE.md) if high-impact |
| `[deprecated]` | Rule is outdated, superseded, or no longer applicable | Remove in next guideline update cycle |

Use these labels in guideline files as inline markers:

```markdown
* MUST: Always validate user input at API boundaries `[proven]`
* MUST: Use structured logging with request trace IDs `[draft]`
* ~~MUST: Use moment.js for date formatting~~ `[deprecated]` — replaced by native Intl API
```

The `ai-dev-os-evolve` command can propose label changes based on violation patterns.
