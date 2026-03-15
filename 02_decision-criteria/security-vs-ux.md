$NOTE
# Balancing Security and UX

Criteria for judging the appropriate balance by considering the impact of security measures on user experience.

---

## Fundamental Principle

**"Security is enabled by default. When relaxing it for UX reasons, make the risks explicit before deciding."**

---

## 1. Security Measure Priority

### Non-Negotiable (Required even at the expense of UX)

| Measure | Reason | Impact on UX |
|------|------|-----------|
| IDOR check (ownership verification) | Prevent access to other users' data | None (transparent) |
| CSRF protection | Prevent cross-site attacks | None (handled by framework middleware) |
| SQL injection protection | ORM-only access | None (transparent) |

### Required but Adjustable (Balance with UX)

| Measure | Strict Setting | UX-Conscious Adjustment | Decision Criteria |
|------|----------|---------------|---------|
| Rate limiting | 5req/min starting point | Adjust per endpoint type | See rate limiting table below. 5req/min is the starting point for destructive/sensitive operations because it's low enough to block automated attacks while still allowing a human to retry after a genuine mistake. Scale up from there based on endpoint type. |
| Session timeout | 15 minutes | Sliding extension during activity | Avoid interrupting user work |
| 2FA | Required for all operations | Only for sensitive operations (payment, settings changes) | Based on risk level |
| Password requirements | 16+ characters + special characters required | 12+ characters + strength meter. 12 characters is the current NIST SP 800-63B recommendation as the minimum that resists offline brute-force attacks with modern hardware. Below 12, GPU-based cracking becomes feasible within days. A strength meter encourages longer passwords without forcing frustrating complexity rules. | Balance with drop-off rate |
| Suspicious login detection | Block immediately | Notify + confirm (non-blocking) | Prevent lockout of legitimate users |

### Recommended but Optional (Decide based on risk assessment)

| Measure | Adoption Condition | When It Can Be Skipped |
|------|---------|---------------|
| CSP headers | Recommended for production | Early development, stage with many inline scripts |
| Audit logs | When handling sensitive data | Personal projects, MVP stage |
| IP blocking | When attacks are observed | Not needed under normal circumstances |

---

## 2. Rate Limiting Decisions

| Endpoint Type | Limit | Reason |
|----------------|-------|------|
| Authentication (login, password reset) | 10 / min | Brute force prevention |
| AI generation (high-cost operations) | 30 / hour | Resource protection, usage-based billing control |
| General API | 100 / min | Sufficient for normal usage |
| Strict (account deletion, etc.) | 5 / min | Protection of irreversible operations |

### UX When Rate Limit Is Exceeded

```
Limit Exceeded
  |
  +-- Return Retry-After header
  +-- Include remaining time in response body ("You can retry in 30 seconds")
  +-- Returning only a generic error message is unacceptable (user can't understand the cause)
```

---

## 3. Authentication UX Decisions

### Session Management

| Decision Point | Choice | Reason |
|------------|------|------|
| Concurrent sessions | Max 5 | Allow multi-device usage while detecting anomalies |
| Session extension | Sliding method | Prevent timeouts during active work |
| On logout | Option to log out from all devices | Security incident response |

### Suspicious Login Detection Flow

```
Login Attempt
  |
  +-- First-time login? -> No alert (accept)
  |
  +-- Access from a country not used in the past 30 days?
  |    +-- Yes -> Async notification to user (don't block)
  |    30 days is the window because: shorter (7 days) creates false positives for
  |    monthly business travelers; longer (90 days) delays detection of compromised
  |    accounts. 30 days balances signal quality with detection speed.
  |
  +-- Error in detection process? -> Fail-safe (pass through as "not suspicious")
```

**Fail-safe principle**: Security detection failures must never lock out legitimate users. When in doubt, allow access and alert asynchronously.

---

## 4. Validation UX Decisions

### Error Message Granularity

| Situation | Message Granularity | Reason |
|------|-------------|------|
| Login failure | "Email address or password is incorrect" | Prevent account existence guessing |
| Form input error | Specific message per field | User can identify what to fix |
| Permission error | "You do not have access permissions" | Don't reveal resource existence (treat same as 404) |
| Server error | "A temporary error occurred" | Don't leak internal information |

---

## 5. Encryption and Data Protection Decisions

### Deciding Which Fields to Encrypt

| Data Type | Encryption | Reason |
|----------|--------|------|
| API keys, tokens | Required | Severe damage on leakage |
| Personal information (PII) | Recommended | Legal requirements (GDPR, etc.) |
| User settings, preferences | Not needed | Low sensitivity |
| Public profile information | Not needed | Public data by nature |

### PII in Logs Decisions

| Information | Log Output | Method |
|------|---------|------|
| User ID | Output | Needed for tracing |
| Email address | Mask | `***@example.com` |
| Password | Never output | Structlog / logging filter processors |
| API key | Never output | Structlog / logging filter processors |
| Request ID | Output | Needed for tracing |
| IP address | Depends on context | Security logs only |

---

## 6. Decision Tree When in Doubt

```
Want to relax a security measure
  |
  +-- Can the user not complete their operation without relaxation?
  |    +-- No -> Don't relax
  |    +-- Yes |
  |
  +-- What is the worst case if relaxed?
  |    +-- Data leakage / unauthorized access -> Don't relax. Find a UX workaround instead
  |    +-- Minor user inconvenience -> Don't relax
  |    +-- Significant user drop-off -> Next |
  |
  +-- Is there an alternative mitigation?
  |    +-- Yes -> Adopt the alternative (e.g., block -> notify + confirm)
  |    +-- No -> Relax after documenting the risk
  |
  +-- If relaxed, can we detect and monitor?
       +-- Yes -> Relax + add monitoring
       +-- No -> Don't relax
```
