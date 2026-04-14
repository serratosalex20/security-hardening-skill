---
name: security-hardening
description: Comprehensive security hardening for production applications. Use this skill when the user asks to "secure", "harden", "audit security", "ship-safe", "security review", "check for vulnerabilities", "make production-ready", "apply security baseline", or before any deployment. Covers secrets management, input validation, authorization, dependency hygiene, observability, and verification. ALWAYS use this skill for any security-related code review or hardening task, even if the user only mentions one of these areas.
---

# Security Hardening

A systematic checklist for hardening applications before shipping to production. Work through each section in order, applying changes and using centralized reusable helpers wherever possible instead of duplicating logic.

## Process

Work through all 6 sections. For each issue found:
1. Fix it using a centralized helper (don't duplicate the same guard logic)
2. Note the fix in your running summary
3. Flag anything requiring manual follow-up (e.g., rotate a leaked key, set env var on hosting platform)

At the end, produce a **Security Hardening Summary** (see Section 6).

---

## 1. Secrets

**Goal:** No secrets exposed to the client; all sensitive config is server-only.

Steps:
- Search source files for hardcoded secrets: API keys, tokens, passwords, connection strings, JWTs
- Move any found to server-only environment variables — never `NEXT_PUBLIC_*` or any client-accessible var
- Audit `NEXT_PUBLIC_*` vars (and equivalents in other frameworks) — none should contain secrets
- Update `.env.example` with placeholder values only (e.g., `DATABASE_URL=your-database-url-here`)
- Scan log statements for secret exposure — replace with `[REDACTED]` or a log scrubber helper

**Centralized helper pattern:**
```typescript
// lib/env.server.ts — import ONLY in server-side code
import { z } from 'zod'
const schema = z.object({
  DATABASE_URL: z.string(),
  SECRET_KEY: z.string(),
  // add all required server vars
})
export const serverEnv = schema.parse(process.env)
// This throws at startup if any required var is missing, preventing silent failures
```

---

## 2. Input Handling

**Goal:** All untrusted input validated and sanitized at every trust boundary.

Trust boundaries to audit: API routes, server actions, webhooks, form submissions, URL/query params, request headers.

Steps:
- Add schema validation at every boundary (Zod, Joi, Yup, or equivalent)
- Sanitize any HTML/Markdown before rendering (DOMPurify client-side, sanitize-html server-side)
- Parameterize all database queries — never interpolate user input into SQL strings
- Escape shell arguments if any `exec`/`spawn` calls exist — better yet, avoid shell execution
- Validate redirect targets — reject or allowlist URLs to prevent open redirect attacks
- Validate Content-Type and key request headers where parsing depends on them

**Centralized helper pattern:**
```typescript
// lib/validate.ts
import { z } from 'zod'

export class ValidationError extends Error {
  constructor(public details: z.ZodError) {
    super('Validation failed')
  }
}

export function parseInput<T>(schema: z.ZodType<T>, data: unknown): T {
  const result = schema.safeParse(data)
  if (!result.success) throw new ValidationError(result.error)
  return result.data
}
```

---

## 3. Authorization

**Goal:** Every protected endpoint checks auth and ownership server-side — no frontend-only guards.

Steps:
- Audit every API route, server action, and mutation:
  - Does it verify the session/token is valid? (authentication)
  - Does it verify the user has permission? (authorization)
  - Does it verify the user owns the resource being accessed/mutated? (IDOR prevention)
- Remove auth logic that only lives in the frontend — it can be bypassed
- Use middleware or a centralized auth helper for consistent enforcement

**Centralized helper pattern:**
```typescript
// lib/auth.ts
export async function requireAuth(req: Request): Promise<Session> {
  const session = await getSession(req)
  if (!session) throw new UnauthorizedError('Not authenticated')
  return session
}

export function requireOwnership(currentUserId: string, resourceOwnerId: string): void {
  if (currentUserId !== resourceOwnerId) throw new ForbiddenError('Access denied')
}

// For multi-tenant apps:
export function requireTenantScope(userTenantId: string, resourceTenantId: string): void {
  if (userTenantId !== resourceTenantId) throw new ForbiddenError('Cross-tenant access denied')
}
```

---

## 4. Dependency Hygiene

**Goal:** No high-severity vulnerabilities; no unnecessary or risky dependencies.

Steps:
- Run dependency audit and fix high/critical issues:
  ```bash
  npm audit --audit-level=high
  npm audit fix
  # or: pnpm audit / yarn audit
  ```
- Review any packages flagged as unmaintained, deprecated, or with known supply chain issues
- Check HTTP clients (fetch, axios, got) — ensure timeouts, error handling, and no blind credential forwarding
- Check database clients — ensure connection pooling, prepared statements, and proper error handling
- Remove unused dependencies (`depcheck` or manual review of package.json)

---

## 5. Observability

**Goal:** Security-relevant events logged with context; secrets never appear in logs.

Events to log:
- **Auth failures**: timestamp, route, reason — NOT the token, password, or session ID
- **Validation failures**: route, field names — NOT the rejected values if they could be sensitive
- **Webhook failures**: source, event type, HTTP status, error message
- **Unhandled server errors**: stack trace (dev only), sanitized message (prod)

Steps:
- Audit existing log statements for secret leakage (`console.log(req.body)`, `console.log(token)`, etc.)
- Replace ad-hoc console.log with structured logger (pino, winston, or framework logger)
- Never log raw request bodies at info level in production — too likely to contain PII/secrets

**Centralized helper pattern:**
```typescript
// lib/logger.ts
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

---

## 6. Verification

**Goal:** All automated checks pass; a clear summary exists for human review.

Run in order:
```bash
npm run lint          # or eslint .
npm run typecheck     # or tsc --noEmit
npm test              # or vitest / jest
npm run build
npm audit --audit-level=moderate
```

If any check fails, fix it before delivering the summary.

### Security Hardening Summary (required output)

After completing all sections, always produce this summary:

```
## Security Hardening Summary

### Changes Made
- [file path]: [what was changed and why]
- ...

### Risks Fixed
- Secrets: [e.g., removed hardcoded API key in api/client.ts, moved to serverEnv]
- Input handling: [e.g., added Zod validation to POST /api/users]
- Authorization: [e.g., added requireAuth() to 3 unguarded routes]
- Dependencies: [e.g., npm audit fix resolved 2 high CVEs]
- Observability: [e.g., replaced console.log(token) with structured logger]

### Manual Follow-up Required
- [ ] [e.g., Rotate the API key that was previously hardcoded — key: STRIPE_SECRET_KEY]
- [ ] [e.g., Set DATABASE_URL on production hosting platform]
- [ ] [e.g., Review remaining moderate npm audit findings manually]

### Verification Results
- Lint:       PASS / FAIL (N errors)
- Typecheck:  PASS / FAIL (N errors)
- Tests:      PASS / FAIL (N failing)
- Build:      PASS / FAIL
- Audit:      N high, N moderate issues remaining
```
