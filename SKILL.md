---
name: security-hardening
description: Comprehensive, AI-assisted security hardening for production web applications in any language (TypeScript/JavaScript, Python, Go, Ruby, PHP, Java, etc.). Use this skill when the user asks to "secure", "harden", "audit security", "scan for vulnerabilities", "threat model", "ship-safe", "security review", "check for vulnerabilities", "make production-ready", "apply security baseline", or before any deployment. It runs a threat-model-first scan (optionally fanning out parallel subagents), triages findings by severity and true-positive likelihood, then fixes secrets, input validation, authorization, dependencies, and observability using centralized reusable helpers, and verifies the result. ALWAYS use this skill for any security-related review or hardening task, even if the user mentions only one of these areas.
---

# Security Hardening

A systematic, threat-model-driven process for hardening web applications before shipping to production — in any language or framework.

The hard-won lesson from large-scale AI vulnerability discovery is that **finding candidate issues is cheap; the bottleneck is triage, prioritization, and fixing**. A scan that surfaces a thousand "maybe" findings is worse than useless if it buries the handful that matter. So this skill scans broadly against a known taxonomy, **triages ruthlessly** by severity and true-positive likelihood, and fixes the real issues — highest severity first — using centralized reusable helpers instead of duplicated guard logic.

## Process

1. **Threat model first** (Section 1) — map entry points, trust boundaries, assets, and likely attacker goals. This drives what you scan first and how you score severity.
2. **Scan systematically** (Section 1) — walk the vulnerability taxonomy. For anything larger than a small repo, fan out parallel read-only subagents per component or category.
3. **Triage every candidate** (Section 1) — confirm it's a true positive, assign a severity, and deduplicate by root cause. Do not report noise.
4. **Fix true positives, highest severity first** (Sections 2–6) — always via a centralized helper; never duplicate the same guard.
5. **Verify and summarize** (Section 7) — run the project's checks and produce the **Security Hardening Summary** with a findings table.

**Language note:** examples below appear in several languages. Use whichever matches the project's stack — the principles are identical across TypeScript/JavaScript, Python, Go, Ruby, PHP, Java, and others. When the stack isn't shown, apply the same pattern with the idiomatic library for that ecosystem.

---

## 1. Threat Model & AI-Assisted Scanning

**Goal:** a prioritized map of what to protect and where it's exposed, plus a systematic scan that doesn't skip whole categories of bug.

### 1a. Build the threat model first

Spend a few minutes mapping the application before scanning a single line:

- **Entry points / trust boundaries** — HTTP routes and handlers, server actions/RPC, GraphQL resolvers, webhooks and third-party callbacks, message-queue/event consumers, file uploads, auth callbacks, CLI arguments, and environment/config.
- **Assets** — user data/PII, credentials and secrets, payment and financial flows, admin capabilities, and internal network/service access.
- **Attacker goals** — account takeover, privilege escalation, data exfiltration, remote code execution, fraud/financial abuse, denial of service.
- **Prioritize** — rank surfaces by *exposure × asset sensitivity*. An unauthenticated route touching payments outranks an admin-only internal tool. Scan the highest-value, most-exposed surfaces first.

Write the threat model down, even as a short bulleted list. It determines scan order and anchors your severity scores.

### 1b. Scan systematically (OWASP-aligned taxonomy)

Walk the codebase against each category and find **concrete instances** (with `file:line`), not generic advice:

- **Broken access control** — missing authz, IDOR, horizontal/vertical privilege escalation (→ Section 4)
- **Injection** — SQL/NoSQL, OS command, LDAP, template (SSTI), header/CRLF (→ Section 3)
- **Cryptographic failures & secret exposure** — hardcoded secrets, weak/rolled-your-own crypto, secrets in logs (→ Section 2)
- **SSRF, open redirect, unsafe deserialization**
- **XSS** — stored, reflected, DOM-based; unsafe HTML/markdown rendering (→ Section 3)
- **CSRF** and state-changing `GET` requests
- **Authentication & session weaknesses** — weak password hashing, missing rate-limit/lockout, insecure cookies/tokens
- **Security misconfiguration** — missing security headers, permissive CORS, verbose error responses, exposed debug/admin endpoints
- **Vulnerable & outdated dependencies** (→ Section 5)
- **Insufficient logging & monitoring** (→ Section 6)

### 1c. Fan out for large codebases

For anything beyond a small repo, parallelize the scan with **read-only subagents** so the main context stays clean and the work goes faster:

- Launch one subagent **per top-level component** or **per vulnerability category**, each scoped to specific paths.
- Give each a tight brief: *which paths, which categories, and "report concrete findings with `file:line`, a severity estimate, and a one-line exploit sketch — do not fix anything."*
- Collect all findings centrally, then triage as a batch.

> In Claude Code: use the **Agent** tool with the **Explore** subagent for read-only scanning, and launch independent scans in parallel (multiple tool calls in one message). Keep fixes in the main agent after triage.

### 1d. Triage every candidate before acting

Acting on noise is the expensive failure mode. For each candidate finding:

1. **Confirm it's a true positive.** Trace the data/control flow: can untrusted input actually reach the dangerous sink under realistic conditions? Discard false positives explicitly — don't carry "maybes" forward.
2. **Assign a severity** from impact × exploitability:
   - **Critical** — unauthenticated RCE, authentication bypass, mass data exfiltration, secret leak granting broad access.
   - **High** — authenticated RCE, IDOR on sensitive data, stored XSS, SQL injection behind auth.
   - **Medium** — reflected XSS requiring interaction, CSRF on meaningful actions, SSRF to limited targets.
   - **Low** — info disclosure with minimal impact, missing defense-in-depth headers.
3. **Deduplicate by root cause.** Collapse the same flaw across many call sites into a single centralized fix.
4. **Record it** in the findings table (Section 7) with severity, location, and status.

**Fix in severity order: Critical and High first.** A handful of confirmed Critical fixes beats a long list of unverified Lows.

---

## 2. Secrets

**Goal:** No secrets exposed to the client; all sensitive config is server-only and validated at startup.

Steps:
- Search source for hardcoded secrets: API keys, tokens, passwords, connection strings, private keys, JWTs.
- Move any found to **server-only** environment variables. Never place a secret in a client-bundled variable — these prefixes ship to the browser:
  - Next.js `NEXT_PUBLIC_*` · Vite `VITE_*` · Create React App `REACT_APP_*` · Expo `EXPO_PUBLIC_*` · Nuxt `runtimeConfig.public`
- Audit those client-exposed prefixes — none should contain a secret.
- Update `.env.example` with placeholder values only (e.g., `DATABASE_URL=your-database-url-here`).
- Scan log statements for secret exposure — replace with `[REDACTED]` or a log scrubber (see Section 6).
- Validate required env vars at startup so a missing one fails loud and early, not silently at request time.

**Centralized helper pattern:**

```typescript
// TypeScript — lib/env.server.ts (import ONLY in server-side code)
import { z } from 'zod'
const schema = z.object({
  DATABASE_URL: z.string().url(),
  SECRET_KEY: z.string().min(16),
})
export const serverEnv = schema.parse(process.env) // throws at startup if invalid
```

```python
# Python — config.py
from pydantic_settings import BaseSettings
from pydantic import AnyUrl

class Settings(BaseSettings):
    database_url: AnyUrl
    secret_key: str

settings = Settings()  # raises at import time if a required var is missing
```

```go
// Go — config.go
type Config struct {
    DatabaseURL string `env:"DATABASE_URL,required"`
    SecretKey   string `env:"SECRET_KEY,required"`
}
// Parse with a library like caarlos0/env and fail fast on error at startup.
```

---

## 3. Input Handling

**Goal:** All untrusted input validated and sanitized at every trust boundary; no injection or XSS sinks reachable from user input.

Trust boundaries to audit: API routes, server actions/RPC, webhooks, GraphQL resolvers, form submissions, URL/query params, request headers, uploaded file contents and names.

Steps:
- **Validate with a schema at every boundary.** Reject unknown/extra fields; coerce types explicitly.
- **Sanitize HTML/Markdown** before rendering untrusted content (e.g., DOMPurify client-side, sanitize-html / bleach / bluemonday server-side).
- **Parameterize all database queries** — never interpolate user input into SQL/NoSQL strings.
- **Avoid shell execution.** If unavoidable, pass argument arrays (never a shell string) and never interpolate user input.
- **Validate redirect targets** — allowlist hosts/paths to prevent open redirects.
- **Validate outbound request targets** to prevent SSRF — block internal/link-local ranges and metadata endpoints.
- **Check `Content-Type`** and key headers when parsing depends on them.

**Validation libraries by stack:** TS/JS — Zod, Valibot, Joi · Python — Pydantic · Go — go-playground/validator, ozzo-validation · Ruby — dry-validation + strong params · PHP — respect/validation · Java — Bean Validation (Jakarta).

**Centralized helper pattern:**

```typescript
// TypeScript — lib/validate.ts
import { z } from 'zod'
export class ValidationError extends Error {
  constructor(public details: z.ZodError) { super('Validation failed') }
}
export function parseInput<T>(schema: z.ZodType<T>, data: unknown): T {
  const result = schema.safeParse(data)
  if (!result.success) throw new ValidationError(result.error)
  return result.data
}
```

**Parameterized queries (never string-build SQL):**

```python
# Python — placeholders, not f-strings
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```go
// Go — placeholders via database/sql
row := db.QueryRow("SELECT id FROM users WHERE email = $1", email)
```

```typescript
// TypeScript — parameterized
await db.query('SELECT id FROM users WHERE email = $1', [email])
```

---

## 4. Authorization

**Goal:** Every protected endpoint checks authentication and ownership **server-side** — no frontend-only guards.

Steps:
- Audit every API route, server action, RPC handler, and mutation:
  - Is the session/token verified? *(authentication)*
  - Does the user have permission for this action? *(authorization)*
  - Does the user own / have tenant scope over the resource? *(IDOR / cross-tenant prevention)*
- Remove auth logic that lives only in the frontend — it can be bypassed.
- Enforce through middleware or a centralized helper so every handler is covered consistently.
- Watch for IDOR: any handler taking a resource ID from the request must verify the caller may access *that specific* resource.

**Centralized helper pattern:**

```typescript
// TypeScript — lib/auth.ts
export async function requireAuth(req: Request): Promise<Session> {
  const session = await getSession(req)
  if (!session) throw new UnauthorizedError('Not authenticated')
  return session
}
export function requireOwnership(currentUserId: string, resourceOwnerId: string): void {
  if (currentUserId !== resourceOwnerId) throw new ForbiddenError('Access denied')
}
export function requireTenantScope(userTenantId: string, resourceTenantId: string): void {
  if (userTenantId !== resourceTenantId) throw new ForbiddenError('Cross-tenant access denied')
}
```

The same shape applies in any stack: a dependency/decorator in FastAPI, middleware in Express/Gin/Rails, or a filter in Spring. Centralize it once; call it everywhere.

---

## 5. Dependency Hygiene

**Goal:** No high-severity vulnerabilities; no unnecessary or risky dependencies.

Steps:
- Run the ecosystem's audit and fix high/critical issues:
  ```bash
  # JS/TS
  npm audit --audit-level=high && npm audit fix   # or pnpm audit / yarn audit
  # Python
  pip-audit            # or: safety check
  # Go
  govulncheck ./...
  # Ruby
  bundle audit
  # PHP
  composer audit
  # Rust
  cargo audit
  # Java
  mvn org.owasp:dependency-check-maven:check
  # Any ecosystem (universal scanner)
  osv-scanner scan .
  ```
- Review packages flagged as unmaintained, deprecated, or with known supply-chain issues.
- Check HTTP clients (fetch/axios/requests/net-http) for timeouts, error handling, and no blind credential forwarding to arbitrary hosts.
- Check database clients for connection pooling, prepared statements, and proper error handling.
- Remove unused dependencies (`depcheck`, `pip-autoremove`, `go mod tidy`, or manual review).
- Pin/lock versions and keep the lockfile committed.

---

## 6. Observability

**Goal:** Security-relevant events logged with context; secrets never appear in logs.

Events to log:
- **Auth failures** — timestamp, route, reason — NOT the token, password, or session ID.
- **Authorization denials** — user ID, attempted resource, action.
- **Validation failures** — route, field names — NOT the rejected values if they could be sensitive.
- **Webhook failures** — source, event type, HTTP status, error message.
- **Unhandled server errors** — stack trace (dev only), sanitized message (prod). Never leak stack traces or internal details to clients.

Steps:
- Audit existing log statements for secret/PII leakage (`console.log(req.body)`, `print(token)`, etc.).
- Replace ad-hoc prints with a **structured logger** that supports redaction.
- Never log raw request bodies at info level in production.

**Structured loggers by stack:** TS/JS — pino, winston · Python — structlog, stdlib `logging` + filters · Go — slog, zap · Ruby — lograge · Java — Logback/Log4j2 with masking.

**Centralized helper pattern:**

```typescript
// TypeScript — lib/logger.ts
import pino from 'pino'
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'req.body.password', 'req.body.token'],
})
export const logAuthFailure = (route: string, reason: string, meta?: object) =>
  logger.warn({ route, reason, ...meta }, 'auth_failure')
export const logValidationFailure = (route: string, fields: string[]) =>
  logger.info({ route, fields }, 'validation_failure')
export const logWebhookFailure = (source: string, event: string, error: string) =>
  logger.error({ source, event, error }, 'webhook_failure')
```

```python
# Python — structlog with redaction processor
import structlog
logger = structlog.get_logger()
def log_auth_failure(route: str, reason: str, **meta):
    logger.warning("auth_failure", route=route, reason=reason, **meta)
```

---

## 7. Verification

**Goal:** All automated checks pass; a clear, severity-ranked summary exists for human review.

Run the project's checks (use the stack's equivalents):

```bash
# JS/TS
npm run lint && npm run typecheck && npm test && npm run build && npm audit --audit-level=moderate
# Python
ruff check . && mypy . && pytest && pip-audit
# Go
go vet ./... && golangci-lint run && go test ./... && govulncheck ./...
```

If any check fails, fix it before delivering the summary.

### Security Hardening Summary (required output)

After completing all sections, always produce this summary, including the **findings table** so triage decisions are auditable:

```
## Security Hardening Summary

### Threat Model (top surfaces)
- [e.g., POST /api/transfer — unauthenticated-reachable, touches payments — HIGH priority]
- ...

### Findings (triaged)
| Severity | Location           | Issue                          | Status        |
|----------|--------------------|--------------------------------|---------------|
| Critical | api/auth.ts:42     | Auth bypass via empty token    | Fixed         |
| High     | api/users.ts:88    | IDOR — no ownership check       | Fixed         |
| Medium   | web/profile.tsx:30 | Reflected XSS in name field     | Fixed         |
| Low      | next.config.js     | Missing security headers        | Follow-up     |
| —        | api/legacy.ts:12   | Suspected SQLi                  | False positive |

### Changes Made
- [file path]: [what changed and why]
- ...

### Risks Fixed
- Secrets: [e.g., removed hardcoded API key in api/client.ts, moved to serverEnv]
- Input handling: [e.g., added schema validation to POST /api/users]
- Authorization: [e.g., added requireAuth() to 3 unguarded routes]
- Dependencies: [e.g., audit fix resolved 2 high CVEs]
- Observability: [e.g., replaced console.log(token) with redacted logger]

### Manual Follow-up Required
- [ ] [e.g., Rotate STRIPE_SECRET_KEY — was previously hardcoded in api/stripe.ts]
- [ ] [e.g., Set DATABASE_URL on production hosting platform]
- [ ] [e.g., Review remaining moderate audit findings manually]

### Verification Results
- Lint:       PASS / FAIL (N errors)
- Typecheck:  PASS / FAIL (N errors)
- Tests:      PASS / FAIL (N failing)
- Build:      PASS / FAIL
- Audit:      N high, N moderate issues remaining
```
