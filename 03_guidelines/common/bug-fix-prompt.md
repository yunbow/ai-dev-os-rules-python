# Bug Fix Prompt

A prompt template for systematically investigating and fixing bugs.

---

## Prompt

```
Please investigate and fix the following bug.

## Investigation Procedure

### 1. Identifying Reproduction Conditions
- Confirm the specific steps to reproduce the bug
- Track the related Request ID if available (common/logging.md)

### 2. Classifying the Cause (common/error-handling.md)

| Error Type | Investigation Direction |
|-----------|----------------------|
| System Error (500, uncaught exceptions) | Suspect infrastructure, DB connection, or external API failures |
| Application Error (domain rule violations) | Suspect business logic or validation issues |
| User Operation Error (input mistakes) | Suspect UI/UX problems or insufficient validation |

### 3. Identifying the Root Cause
- Narrow down the origin using error logs (requestId, action, service)
- Check recent changes with `git blame`
- Verify whether related tests are passing

## Checkpoints During Fix

### Security Verification (common/security.md)
- [ ] Does the fix not introduce new security holes

### Type Safety Verification (common/code.md)
- [ ] Is the fix not using bare `except:` or `# type: ignore`
- [ ] Are type hints complete and correct

### Error Handling Verification (common/error-handling.md)
- [ ] Is the root cause being fixed, not just masking symptoms
- [ ] Are error messages understandable to users
- [ ] Is internal information not being leaked

### Regression Risk Verification
- [ ] Have tests been added or updated for the fix location
- [ ] Do all existing tests pass
- [ ] Does the fix not affect other features

## Fix Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Silently swallowing errors with try-catch | Identify and fix the root cause |
| Using bare `except:` to hide errors | Catch specific exception types |
| Adding `# type: ignore` without error codes | Fix the type issue or add specific code |
| Avoiding only specific cases with conditional branches | Fix the fundamental logic error |
| Only changing the error message | Fix the cause of the error |

## Output Format

### Bug Investigation Report

| Item | Details |
|------|---------|
| Symptom | {Problem observed by the user} |
| Cause Classification | System / Application / User Operation |
| Root Cause | {Code-level cause} |
| Impact Scope | {Affected features and users} |
| Fix Details | {Changed files and changes made} |
| Regression Tests | {Tests added or updated} |
```
