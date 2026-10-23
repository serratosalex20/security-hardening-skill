---
name: security-hardening
description: Threat-model-driven security review and hardening for web apps, APIs, and sites in any stack (TypeScript/JavaScript, Next.js, Python, Go, Ruby, PHP, Java; Supabase/Firebase; static sites on Vercel/Netlify). Use when the user asks to secure, harden, audit security, scan for vulnerabilities, threat model, do a security review, ship-safe, make production-ready, or apply a security baseline, or for a pre-deploy security check. Also use for a single area (secrets, auth/IDOR, input validation, headers/CSP/CORS, dependencies, Supabase RLS) and run just that section. Safe in any repo. Audit-only unless asked to fix, records a build/test baseline before editing, follows the target repo's own conventions and libraries, applies only behavior-preserving fixes directly, and proposes anything that can break a site or needs deploy-time config (CSP enforcement, HSTS, CORS, cookies, RLS, new env vars, major upgrades) for approval. Ends with a Security Hardening Summary and a triaged findings table.
---

# Security Hardening

A systematic, threat-model-driven process for hardening web applications before shipping to production, in any language or framework.

Two lessons shape this skill:

1. **Finding candidate issues is cheap; triage and fixing are the bottleneck.** A scan that surfaces a thousand "maybe" findings buries the handful that matter. Scan broadly, **triage ruthlessly**, and fix the real issues, highest severity first.
2. **A security fix that takes the site down is an outage, not a fix.** This skill runs inside other people's repos and live client sites. It must leave every legitimate user, integration, build, and deploy working. Section 0 governs every other section.

## Process

0. **Ground rules** (Section 0): pick the mode, record a baseline, and apply the change-risk tiers. Do this every run.
1. **Threat model and scan** (Section 1): map entry points and assets, then walk the vulnerability taxonomy, fanning out read-only subagents on larger repos.
2. **Triage** (Section 1d): confirm true positives, assign severity, deduplicate by root cause.
3. **Fix true positives, highest severity first** (Sections 2–8): Fix mode only, within the risk tiers, reusing the project's own helpers and libraries.
4. **Verify against the baseline and summarize** (Section 9).

For a request that covers one area ("check our headers", "is this endpoint safe?"), run Section 0, then only the relevant section, and still produce the summary.

**Language note:** examples are illustrations in several languages, not code to paste. Match the target project's stack, libraries, and style. The principles are identical across ecosystems.

---

## 0. Ground Rules: Don't Break the Target Project

### 0a. Pick the mode

- **Audit mode (read-only).** Edit no files. Use it when the user says review, audit, scan, check, threat model, or "is this safe", and whenever the skill fires as a **pre-deploy check**. Don't start rewriting code in the middle of someone's deploy. Deliver the summary with the fixes you recommend, then ask which to apply.
- **Fix mode.** Use it when the user says harden, secure, fix, apply the baseline, or make production-ready, or approves fixes from an audit. Apply Tier 1 and Tier 2 changes (Section 0d). List Tier 3 changes as proposals.
- **An explicit fix request beats the pre-deploy default.** A workflow's dedicated hardening phase that says "apply the baseline" before the first deploy runs in Fix mode. A deploy that merely triggers the skill runs in Audit mode.
- **Unclear?** Run an audit, then offer the fixes. Asking costs one message; guessing wrong can cost a broken site.

This skill never deploys, merges, rotates keys, or changes hosting, DNS, or dashboard settings itself. Those go in **Manual follow-up**.

### 0b. Scope and conventions

- Work only in the repo, or the monorepo package, the user pointed at. Leave vendored code, generated files, build output, and already-applied migrations alone.
- **The target repo's rules win.** Read its `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, and lint config first and follow them, including branch, commit, and test conventions.
- **Reuse before you create.** Search for an existing auth guard, validator, logger, env loader, or headers config before writing one. Extend it; don't add a parallel `lib/auth.ts` beside the existing one.
- **No new dependencies when an installed one does the job.** If the project uses Yup, don't add Zod; if it uses winston, don't add pino. Adding a dependency is a Tier 2 change and needs a reason in the summary.
- **Minimal diffs.** No repo-wide reformatting, renames, or drive-by refactors. Commit one logical change at a time, grouped by section, so any one can be reverted on its own.
- **Git safety.** Use a branch if the repo's workflow does. Don't mix your edits into the user's uncommitted work. Never force-push, rewrite history, or delete branches.

### 0c. Record a baseline before touching anything

1. **Detect the stack.** Language, framework and version, package manager (from the lockfile: `package-lock.json` npm, `pnpm-lock.yaml` pnpm, `yarn.lock` yarn, `bun.lock`/`bun.lockb` bun, `poetry.lock`/`uv.lock`/`requirements*.txt` Python), and deploy target (`vercel.json`, `netlify.toml`, `wrangler.toml`, `Dockerfile`, `fly.toml`, CI workflows).
2. **Find the real check commands.** Look in `package.json` scripts, `Makefile`/`justfile`, `pyproject.toml`/`tox.ini`/`noxfile.py`, and CI config. Don't invent commands; if a project has no tests, say so.
3. **Run them before any edit** and record pass/fail per check. Pre-existing failures are not yours: note them, don't fix them unless asked, and don't count them against your changes.
   - If dependencies aren't installed, install them from the lockfile without rewriting it (`npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable` / `--frozen-lockfile`).
   - If a check needs secrets, a database, or services you don't have, record it as "not run". Don't wire up credentials to make it pass.
4. After each batch of fixes, re-run. A check that passed in the baseline and now fails is a regression you caused: fix it or revert that change before moving on.

### 0d. Change-risk tiers

Every fix belongs to a tier. **When in doubt, use the higher tier.**

**Tier 1: apply in Fix mode.** No legitimate user, client, or integration can tell the difference.
- Parameterizing a query, encoding output, or sanitizing untrusted HTML before render
- Redacting secrets and PII from logs; deleting debug prints of tokens or request bodies
- `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`
- Adding timeouts to outbound HTTP calls, set longer than the slowest legitimate call (AI and file-processing calls can take minutes)

**Tier 2: apply after checking the callers; record the evidence in the summary.** Breaks things only if your assumption about callers is wrong.
- **Auth or ownership checks:** first confirm the endpoint is not public by design (Section 1a).
- **Validation schemas on existing endpoints:** model them on what callers send today, and strip unknown fields rather than rejecting them (Section 3).
- **Allowlist serializers:** keep every field a consumer actually reads; grep the frontend and clients first (Section 5).
- **Clickjacking protection** (`frame-ancestors` / `X-Frame-Options`): only after confirming nothing embeds the site (Section 6).
- **Dependency patch and minor upgrades:** use the project's own package manager, then re-run the baseline checks (Section 7).
- **Moving a secret off a client-exposed variable:** only once no client code reads it. Otherwise add a server route first (Section 2).
- **Generic error messages:** keep status codes and machine-readable error codes stable, because clients may branch on them.

**Tier 3: propose only; apply with explicit approval.** Breaks sites by default, needs deploy-time config, or is hard to undo.
- **Enforcing a CSP.** Ship `Content-Security-Policy-Report-Only` first (Section 6).
- **HSTS changes.** Never add `includeSubDomains` or `preload` without the domain owner's sign-off.
- **CORS, cookie, and session changes:** CORS allowlist changes; cookie name, attribute, or domain changes; session lifetime changes.
- **A `Permissions-Policy` that disables a browser feature**
- **Anything that needs a value set in the host first:** new *required* env vars, webhook-signature enforcement, secret rotation
- **Enabling RLS or changing database policies, rules, or migrations**
- **Risky dependency work:** major-version upgrades, `npm audit fix --force`, removing dependencies
- **Rate limits, CAPTCHA, and changes to login, signup, or password-reset flows**
- **New CI workflows or pre-commit hooks** that will fail on the existing code (Section 10)
- **Deletions:** files, routes, endpoints, or git history

**A Critical finding whose fix is Tier 3** (say, a Supabase table readable with the public key): raise it immediately with the proposed fix. Don't hold it for the end-of-run summary.

### 0e. Secret hygiene for you

- **Never print a secret value**, whether in chat, commits, PR text, the summary, or test fixtures. Report `file:line` and the key type ("Stripe live secret key"), with at most a 4-character prefix.
- **Don't open real env files** (`.env`, `.env.local`, credential JSON) to read values. To see which variables exist, list names only (for example `sed 's/=.*//' .env`). If a deny rule blocks env files, don't work around it: use `.env.example` and the code's env references instead.
- **Removing a committed secret from code does not un-leak it.** Always add "rotate it" to Manual follow-up. Never rewrite git history yourself: it breaks every existing clone and needs the owner's coordination.

---

## 1. Threat Model & AI-Assisted Scanning

**Goal:** a prioritized map of what to protect and where it's exposed, plus a systematic scan that doesn't skip whole categories of bug.

### 1a. Build the threat model first

Spend a few minutes mapping the application before scanning a single line:

- **Shape of the app.** It may be a static site (no server code), a server-rendered framework, an API, or a SPA backed by a BaaS (Supabase/Firebase). Mark sections that don't apply as **N/A** in the summary instead of inventing code for them. A static marketing site mostly needs Sections 2 (client-bundle secrets), 6 (headers via host config), and 7.
- **Entry points and trust boundaries:** HTTP routes and handlers, server actions/RPC, GraphQL resolvers, webhooks and third-party callbacks, queue/event consumers, file uploads, auth callbacks, AI/LLM endpoints, CLI arguments, and env/config.
- **Public by design.** Lead and contact forms, webhooks, OAuth callbacks, health checks, and public content APIs are *meant* to be reachable without login. Their controls are validation, signature verification, rate limiting, and bot protection. **Adding a login requirement to them breaks the business.**
- **Assets:** user data/PII, credentials and secrets, payment and financial flows, admin capabilities, paid-API budgets (AI, SMS, email), and internal network access.
- **Attacker goals:** account takeover, privilege escalation, data exfiltration, remote code execution, fraud/financial abuse, spam/abuse of paid APIs ("denial of wallet"), denial of service.
- **Prioritize** by *exposure × asset sensitivity*. An unauthenticated route touching payments outranks an admin-only internal tool. Scan the most exposed, highest-value surfaces first.

Write the threat model down, even as a short bulleted list. It determines scan order and anchors your severity scores.

### 1b. Scan systematically (OWASP-aligned taxonomy)

Walk the codebase against each category and find **concrete instances** (with `file:line`), not generic advice:

- **Broken access control:** missing authz, IDOR, horizontal/vertical privilege escalation (→ Section 4)
- **Database/BaaS access rules:** Supabase tables without RLS or with permissive policies, service-role/secret keys in client code, public storage buckets, open Firebase rules (→ Section 4)
- **Injection:** SQL/NoSQL, OS command, LDAP, template (SSTI), header/CRLF (→ Section 3)
- **Cryptographic failures and secret exposure:** hardcoded secrets, weak or home-made crypto, secrets in logs or client bundles (→ Section 2)
- **SSRF, open redirect, unsafe deserialization** (→ Section 3)
- **XSS:** stored, reflected, DOM-based; unsafe HTML/markdown rendering (→ Section 3)
- **Webhooks and public endpoints:** missing signature verification, no rate limiting or bot protection on forms (→ Section 3)
- **AI/LLM features:** prompt injection reaching tools or data, model output passed to HTML/SQL/shell, AI provider keys in client code, unauthenticated endpoints that spend paid AI credits (→ Sections 2–4)
- **Sensitive data exposure:** over-broad responses, internal fields/PII in output, verbose errors (→ Section 5)
- **Security misconfiguration:** missing security headers, permissive CORS, insecure cookies, exposed debug/admin endpoints (→ Section 6)
- **CSRF** and state-changing `GET` requests (→ Section 6)
- **Authentication and session weaknesses:** weak password hashing, missing rate limit/lockout, insecure cookies/tokens
- **Vulnerable and outdated dependencies** (→ Section 7)
- **Insufficient logging and monitoring** (→ Section 8)

Useful inputs alongside reading the code: the platform's own advisors (Supabase Security Advisor, GitHub Dependabot/code scanning alerts) and the response headers of the live site (`curl -sI https://<site>`).

### 1c. Fan out for large codebases

For anything beyond a small repo, parallelize the scan with **read-only subagents** so the main context stays clean and the work goes faster:

- Launch one subagent **per top-level component** or **per vulnerability category**, each scoped to specific paths.
- Give each a tight brief: *"Paths: …; categories: …. Report concrete findings with `file:line`, a severity estimate, and a one-line exploit sketch. Do not edit files and do not print secret values."*
- Collect all findings centrally, then triage as a batch.

> In Claude Code: use the **Agent** tool with the **Explore** subagent for read-only scanning, and launch independent scans in parallel (multiple tool calls in one message). Keep fixes in the main agent after triage.

### 1d. Triage every candidate before acting

Acting on noise is the expensive failure mode. For each candidate finding:

1. **Confirm it's a true positive.** Trace the data and control flow: can untrusted input reach the dangerous sink under realistic conditions? Discard false positives explicitly; don't carry "maybes" forward.
2. **Assign a severity** from impact × exploitability:
   - **Critical:** unauthenticated RCE, authentication bypass, mass data exfiltration (including a BaaS table readable with the public key), or a secret leak granting broad access.
   - **High:** authenticated RCE, IDOR on sensitive data, stored XSS, SQL injection behind auth, or an unauthenticated endpoint that spends paid credits without limits.
   - **Medium:** reflected XSS requiring interaction, CSRF on meaningful actions, SSRF to limited targets.
   - **Low:** info disclosure with minimal impact, missing defense-in-depth headers.
3. **Deduplicate by root cause.** Collapse the same flaw across many call sites into a single centralized fix.
4. **Assign a risk tier** (Section 0d) to the fix you'd make.
5. **Record it** in the findings table (Section 9) with severity, location, tier, and status.

**Fix in severity order: Critical and High first.** A handful of confirmed Critical fixes beats a long list of unverified Lows.

---

## 2. Secrets

**Goal:** no secrets in source or client bundles; all sensitive config is server-only and validated where it's used.

Steps:
- Search source, config, and committed files for hardcoded secrets: API keys, tokens, passwords, connection strings, private keys, JWTs.
- Audit client-exposed variables. These prefixes ship to the browser: Next.js `NEXT_PUBLIC_*`, Vite `VITE_*`, Create React App `REACT_APP_*`, Expo `EXPO_PUBLIC_*`, Nuxt `runtimeConfig.public`.
- **Know what is *not* a secret, so you don't "fix" it and break the app.** These are designed to be public:
  - Supabase anon/publishable key (safe *only* with RLS on every exposed table; see Section 4)
  - Stripe publishable key (`pk_…`) and Firebase web config
  - Analytics and tag IDs (GA4, GTM, Meta Pixel)
  - reCAPTCHA/Turnstile site keys and Sentry DSNs
  - Browser Maps keys (these should be HTTP-referrer-restricted in the provider console; that's a follow-up, not a code change)
- **These are always secrets:**
  - Supabase `service_role` / `sb_secret_…`, Stripe `sk_…`/`rk_…`, and webhook signing secrets
  - OpenAI/Anthropic/other AI provider keys and CRM tokens (for example GoHighLevel private integration tokens)
  - SMTP/email API keys, database URLs, and private keys
- **To move a secret server-side** (Tier 2/3):
  - If client code reads it, first add a server route or proxy that uses it.
  - The variable must exist in *every* deploy environment (for example Vercel Production, Preview, and Development) before merge. Put that in Manual follow-up.
- **Rotation.** Any secret that was ever committed or shipped to a browser is compromised: list it for rotation in Manual follow-up.
- Make sure `.gitignore` covers `.env*` (except `.env.example`). Keep `.env.example` to placeholder values (e.g., `DATABASE_URL=your-database-url-here`).
- Scan log statements for secret exposure (see Section 8).
- **Validate env vars where they're used, without breaking builds:**
  - Validate only variables the code already requires. Never make an optional variable required, and keep existing defaults.
  - Don't impose length or format rules that existing valid values might fail.
  - Don't validate at module import in code the build or the client imports. `next build` and CI often run without production secrets, so an import-time throw turns a deploy into a build failure.

**Centralized helper pattern (illustrative; use the project's own validator):**

```typescript
// TypeScript — lib/env.server.ts (validated lazily on first use, not at build)
import 'server-only' // Next.js: blocks client imports. Other stacks: drop this line.
import { z } from 'zod'
const schema = z.object({
  DATABASE_URL: z.string().min(1),
  STRIPE_SECRET_KEY: z.string().min(1),
})
let cached: z.infer<typeof schema> | undefined
export function serverEnv() {
  return (cached ??= schema.parse(process.env)) // clear error at first use if a var is missing
}
```

```python
# Python — config.py
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str

@lru_cache
def get_settings() -> Settings:
    return Settings()  # raises a clear error on first use if a required var is missing
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

**Goal:** all untrusted input validated at every trust boundary; no injection or XSS sinks reachable from user input, without rejecting what legitimate callers send today.

Trust boundaries to audit: API routes, server actions/RPC, webhooks, GraphQL resolvers, form submissions, URL/query params, request headers, uploaded file contents and names, and LLM prompts and outputs.

Steps:
- **Validate with a schema at every boundary**; coerce types explicitly.
- **On existing endpoints, strip unknown fields; don't reject them** (Tier 2). Webhook senders, form builders, CRMs, and marketing tools (UTM params, tracking IDs) add fields freely. A `.strict()` / `extra="forbid"` schema rejects them and silently drops leads or events. Model the schema on what callers actually send (frontend forms, provider payload docs) and reserve strict rejection for new endpoints. (Zod's `z.object()` strips by default; `.strict()` rejects.)
- **Sanitize HTML/Markdown** before rendering untrusted content (DOMPurify client-side; sanitize-html, bleach/nh3, or bluemonday server-side).
- **Parameterize all database queries.** Never interpolate user input into SQL/NoSQL strings.
- **Avoid shell execution.** If unavoidable, pass argument arrays (never a shell string) and never interpolate user input.
- **Validate redirect targets** against an allowlist of hosts and paths to prevent open redirects.
- **Validate outbound request targets** to prevent SSRF: block internal and link-local ranges and cloud metadata endpoints.
- **Check `Content-Type`** and key headers when parsing depends on them.
- **Public-by-design endpoints** (lead and contact forms): validation, body-size limits, a honeypot or Turnstile/reCAPTCHA, and rate limiting. Don't echo input back. New CAPTCHA or rate limits are Tier 3, because they can block real leads if tuned wrong. On serverless, in-memory counters don't work across instances; use the platform's firewall or a shared store.
- **Webhooks:** verify the provider's signature over the **raw** request body (read the raw text before JSON parsing, or the signature won't match), reject stale timestamps, and make handlers idempotent. Enforcing a signature before the signing secret is set in every environment rejects every real webhook. That makes it Tier 3, with deploy order "set secret → deploy verification".
- **AI/LLM features:**
  - Treat both the prompt and the model's output as untrusted. Apply the same sink rules to output (no raw HTML, SQL, shell, or `eval`).
  - Authorize tool calls as the end user, never as an all-powerful service account.
  - Cap tokens and rate-limit endpoints that spend paid credits.

**Validation libraries by stack:** TS/JS: Zod, Valibot, Joi, Yup · Python: Pydantic · Go: go-playground/validator, ozzo-validation · Ruby: dry-validation and strong params · PHP: respect/validation · Java: Bean Validation (Jakarta).

**Centralized helper pattern:**

```typescript
// TypeScript — lib/validate.ts
import { z } from 'zod'
export class ValidationError extends Error {
  constructor(public details: z.ZodError) { super('Validation failed') }
}
export function parseInput<T>(schema: z.ZodType<T>, data: unknown): T {
  const result = schema.safeParse(data) // z.object() strips unknown keys; add .strict() only on new endpoints
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

**Goal:** every protected endpoint checks authentication and ownership **server-side**. No frontend-only guards, and no public-by-design endpoint accidentally locked.

Steps:
- **Decide each endpoint's intended audience first** from routes, frontend callers, docs, and naming. Public by design (Section 1a) gets abuse controls (Section 3), not login. When the intent is unclear, ask; don't guess.
- For every endpoint meant to be protected (API route, server action, RPC handler, mutation):
  - Is the session/token verified? *(authentication)*
  - Does the user have permission for this action? *(authorization)*
  - Does the user own the resource, or have tenant scope over it? *(IDOR / cross-tenant prevention)*
- Remove auth logic that lives only in the frontend; it can be bypassed. Keep the frontend check for UX, and add the server check.
- Enforce through the project's existing middleware or a centralized helper so every handler is covered consistently.
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

### 4a. Supabase / Postgres RLS and other BaaS rules

When the client talks to the database directly (Supabase, Firebase), the **database rules are the authorization layer**.

- **Supabase:** every table in an exposed schema (usually `public`) needs RLS enabled *and* policies. With RLS off, anyone holding the public anon/publishable key can read and write that table through the REST API. That's Critical.
- **Enabling RLS without policies blocks all access, so the app breaks instantly** (Tier 3):
  - Write the policies first, then enable RLS and add the policies in the same migration.
  - Test as an anonymous user and as a signed-in user before deploying.
- Policies should scope rows with `auth.uid()` (or a tenant claim). Avoid `using (true)` on anything but genuinely public data.
- The `service_role` / secret key bypasses RLS: server-only, never in client code or `NEXT_PUBLIC_*`.
- `security definer` functions run with the owner's privileges. Pin `search_path` and check the caller inside the function.
- Storage buckets: confirm which are public. Private files need bucket policies, not just unguessable paths.
- **Firebase:** no `allow read, write: if true` outside local emulators. Scope rules to `request.auth.uid`.

---

## 5. Data Exposure & Output Minimization

**Goal:** responses, errors, and metadata expose only what the client actually needs: never internal fields, other users' data, secrets, or PII.

This is the most common *quiet* leak: an endpoint that serializes a whole database row or ORM object, shipping password hashes, internal flags, or other users' fields without anyone noticing.

Steps:
- **Serialize with an allowlist, never a denylist.** Return explicitly named fields via a DTO/serializer instead of `return user` / `JSON(row)`, so new columns can't auto-leak. **Before trimming, grep every consumer** (frontend, mobile, integrations) so you keep each field they read (Tier 2).
- **Strip sensitive fields:** password hashes, tokens, API keys, internal/admin flags, soft-delete markers, other users' identifiers.
- **Scope queries to the caller.** List and search endpoints must filter to the caller's own data (ties to Section 4 / IDOR).
- **Don't leak structure via errors.** No stack traces, framework banners, or raw DB errors to clients: return a generic message and log the detail (Section 8). Keep HTTP status codes and machine-readable error codes unchanged.
- **Keep PII out of URLs and query strings** (they land in access logs, analytics, and third-party referrers) and out of third-party calls that don't need it.
- **Set `Cache-Control: private, no-store`** on authenticated, per-user responses so shared caches and CDNs don't serve one user's data to another.
- **Disable introspection and debug in prod:** GraphQL introspection off, no debug endpoints, and no public source maps that embed secrets.

**Centralized helper pattern: explicit output serializer.**

```typescript
// TypeScript — allowlist the fields that leave the server
export function toPublicUser(u: UserRow) {
  return { id: u.id, name: u.name, avatarUrl: u.avatarUrl }
  // u.passwordHash, u.isAdmin, u.email, ... simply never appear
}
```

```python
# Python (Pydantic) — response model declares what's allowed out;
# anything not listed is dropped from the response automatically.
class PublicUser(BaseModel):
    id: int
    name: str
    avatar_url: str
```

---

## 6. HTTP Security Headers & Configuration

**Goal:** every response carries hardening headers, and cross-origin, cookie, and transport config is locked down, rolled out so no page, embed, or integration breaks.

**Before adding anything:**
- See what's already sent: run `curl -sI https://<site>`, and inspect `next.config.*`, `proxy.ts`/`middleware.ts`, `vercel.json`, `netlify.toml`, `_headers`, server middleware, and `<meta http-equiv>` tags.
- **Merge into the existing config; never add a second source for the same header.** Two CSPs from different layers are *both* enforced, so the effective policy is stricter than either and things break.

**Rollout by header:**

| Header | Tier | Safe starting value and notes |
|---|---|---|
| `X-Content-Type-Options` | 1 | `nosniff`. First confirm the host serves scripts and styles with correct types (`curl -sI` on a `.js`/`.css`/`.mjs`/`.wasm` URL). Some S3/nginx setups send `application/octet-stream`, and `nosniff` then blocks those files. |
| `Referrer-Policy` | 1 | `strict-origin-when-cross-origin` (already the browser default). If the site already sets a different value on purpose (for example for partner or affiliate attribution), leave it. |
| `X-Frame-Options` / CSP `frame-ancestors` | 2 | `DENY` / `'none'`, but only if nothing embeds the site: widgets, booking or chat embeds served from it, client portals, site-builder previews. Use `SAMEORIGIN` or an allowlist otherwise. |
| `Content-Security-Policy` | 3 | Ship as `Content-Security-Policy-Report-Only` first. See the CSP rollout below. |
| `Strict-Transport-Security` | 3 | Many hosts already send it, so check first. If adding: `max-age=86400`, then raise to `31536000` once verified. `includeSubDomains` only when *every* subdomain serves valid HTTPS (mail, legacy, CNAMEs to third-party tools). `preload` only with the owner's sign-off, because removal from browser preload lists takes months. |
| `Permissions-Policy` | 3 | Disable only features nothing uses. Check the code and third-party embeds first:<br>• `microphone`: voice/AI receptionists<br>• `camera`<br>• `geolocation`: store locators<br>• `payment`: Apple/Google Pay<br>• `fullscreen`: video embeds<br>Also check iframe `allow=` attributes. |

**CSP rollout (the header most likely to break a site):**
1. **Inventory everything the pages load.**
   - Scripts and tags: analytics and tag managers (GA4, GTM, Meta Pixel, Vercel Analytics/Speed Insights)
   - Assets: fonts and maps
   - Embeds: chat/voice widgets, booking embeds (Calendly, CRM forms), payments (Stripe), video
   - Site-builder assets, plus inline `<script>` and `style=` usage
   - A tag manager can inject new third parties at runtime, which a code search won't show.
2. **Ship `Content-Security-Policy-Report-Only`** with that allowlist. Load the key pages (home, forms, checkout, login) and read the violations in the browser console, or add a `report-uri` endpoint.
3. **Enforce only after the violations are clean, and with approval.**
   - Next.js inlines framework scripts. A strict `script-src` needs per-request nonces set in `proxy.ts` (`middleware.ts` before Next.js 16), and nonces force dynamic rendering. Raise that cost/performance trade-off with the owner.
   - The alternative is `'unsafe-inline'` for scripts as an interim step. Say plainly that it's weak against XSS.

**Cross-origin, cookies, transport (all Tier 3):**
- **CORS:** list today's legitimate cross-origin callers first (other frontends, mobile apps, partner sites).
  - Allowlist explicit origins; never reflect an arbitrary `Origin`, and never combine `*` with credentials.
  - Server-to-server calls such as webhooks don't use CORS; don't "fix" them with it.
- **Cookies:** the target for session cookies is `HttpOnly; Secure; SameSite=Lax`. Each of these breaks something:
  - `SameSite=Strict` drops the cookie on cross-site navigation, so users arriving from email links, OAuth, or payment returns land logged out.
  - Renaming a cookie or changing `Domain`/`Path` logs everyone out.
  - A cookie that client JS reads can't be `HttpOnly`. Check before flipping it.
- **Transport:** redirect HTTP→HTTPS at the host or edge.
- **CSRF:** protect state-changing requests with a token or `SameSite` cookies, and make sure no `GET` changes state.

**Config examples (merge into what exists):**

```javascript
// next.config.js — extend an existing headers() instead of adding a second one
// (next.config.mjs / next.config.ts: same shape with `export default`).
// With output: 'export' (static export) headers() is ignored; set headers in the host config instead.
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },                         // Tier 1
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },        // Tier 1
  // Tier 2 — only if nothing embeds this site:
  // { key: 'X-Frame-Options', value: 'DENY' },
  // Tier 3 — observe first; enforce only after violations are clean and the owner approves:
  { key: 'Content-Security-Policy-Report-Only',
    value: "default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'self'" },
]
module.exports = {
  async headers() {
    return [{ source: '/:path*', headers: securityHeaders }]
  },
}
```

For a static site on Vercel, put the same headers in `vercel.json` (Netlify and Cloudflare Pages use a `_headers` file). JSON allows no comments, so merge into the existing file rather than pasting over it:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ]
}
```

```javascript
// Express — helmet()'s defaults are strict on purpose: enforced CSP (script-src 'self'),
// Cross-Origin-Resource-Policy: same-origin, Cross-Origin-Opener-Policy: same-origin,
// and HSTS with includeSubDomains. On an existing app, opt in deliberately instead of bare helmet():
app.use(helmet({
  contentSecurityPolicy: { reportOnly: true },                          // Tier 3 to enforce
  strictTransportSecurity: false,                                       // Tier 3 (see HSTS above)
  crossOriginResourcePolicy: { policy: 'same-site' },                   // same-origin breaks assets your other origins load
  crossOriginOpenerPolicy: { policy: 'same-origin-allow-popups' },      // same-origin breaks popup OAuth/payment flows
}))
```

Other stacks: Django `SECURE_*` settings plus `django-csp` (report-only mode first), Rails `content_security_policy_report_only`, FastAPI/Starlette `secure`, Go `secweb`.

---

## 7. Dependency Hygiene

**Goal:** no exploitable high/critical vulnerabilities in what ships, without a dependency change breaking the build.

Steps:
- **Use the project's own package manager**, chosen from the lockfile (Section 0c). Running npm in a pnpm, yarn, or bun repo creates a second lockfile and can change what the host installs.
- Audit, production dependencies first:
  ```bash
  # JS/TS (match the lockfile)
  npm audit --omit=dev
  pnpm audit --prod
  yarn npm audit --environment production   # Yarn 2+
  yarn audit --groups dependencies          # Yarn 1
  # Python
  pip-audit                                 # or: pip-audit -r requirements.txt
  # Go (reports only vulnerable code you actually call)
  govulncheck ./...
  # Ruby
  bundle audit
  # PHP
  composer audit
  # Rust
  cargo audit
  # Java
  mvn org.owasp:dependency-check-maven:check
  # Any ecosystem
  osv-scanner scan source -r .
  ```
- **Judge exploitability before churning the lockfile.** A dev-only build tool advisory rarely ships to users. Note it and fix it lower in the order.
- **Fix in the least disruptive way (Tier 2), then re-run the baseline checks:**
  - Upgrade the vulnerable package to the minimum fixed version.
  - For transitive dependencies, use `overrides` (npm), `pnpm.overrides`, or `resolutions` (yarn).
  - `npm audit fix` *without* `--force` stays within semver ranges, but can still break things, so re-run the baseline checks.
- **Tier 3:** major-version upgrades and `npm audit fix --force`.
- **Never hand-edit lockfiles.**
- **Report unused dependencies; don't remove them.** depcheck/knip-style tools flag packages used only by config files, CLIs, PostCSS/Tailwind plugins, peer deps, and types.
- Check HTTP clients (fetch/axios/requests/net-http) for timeouts, error handling, and no blind credential forwarding to arbitrary hosts.
- Check database clients for connection pooling, prepared statements, and proper error handling.
- Keep the lockfile committed.

---

## 8. Observability

**Goal:** security-relevant events logged with context; secrets and PII never appear in logs.

Events to log:
- **Auth failures:** timestamp, route, and reason. Never the token, password, or session ID.
- **Authorization denials:** user ID, attempted resource, action.
- **Validation failures:** route and field names. Not the rejected values if they could be sensitive.
- **Webhook failures:** source, event type, HTTP status, error message.
- **Unhandled server errors:** full detail in server logs, a generic message to the client. Never leak stack traces or internal details to clients.

Steps:
- Audit existing log statements for secret and PII leakage (`console.log(req.body)`, `print(token)`, etc.).
- Use the project's existing logger and add redaction to it. Introduce a structured logger only if there is none, and note it as a new dependency.
- Never log raw request bodies at info level in production.
- Don't change the format of log lines that existing alerts, dashboards, or log drains parse without saying so in the summary.

**Structured loggers by stack:** TS/JS: pino, winston · Python: structlog, stdlib `logging` with filters · Go: slog, zap · Ruby: lograge · Java: Logback/Log4j2 with masking.

**Centralized helper pattern:**

```typescript
// TypeScript — lib/logger.ts
import pino from 'pino'
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'req.headers.cookie', 'req.body.password', 'req.body.token'],
})
export const logAuthFailure = (route: string, reason: string, meta?: object) =>
  logger.warn({ route, reason, ...meta }, 'auth_failure')
export const logValidationFailure = (route: string, fields: string[]) =>
  logger.info({ route, fields }, 'validation_failure')
export const logWebhookFailure = (source: string, event: string, error: string) =>
  logger.error({ source, event, error }, 'webhook_failure')
```

```python
# Python — structlog
import structlog
logger = structlog.get_logger()
def log_auth_failure(route: str, reason: str, **meta):
    logger.warning("auth_failure", route=route, reason=reason, **meta)
```

---

## 9. Verification & Summary

**Goal:** no regressions against the baseline, and a clear, severity-ranked summary for human review.

1. **Re-run the exact baseline commands from Section 0c.** Don't substitute commands the project doesn't have. Typical shapes:
   ```bash
   # JS/TS — only scripts that exist in package.json, via the project's package manager
   npm run lint && npm run typecheck && npm test && npm run build
   # Python
   ruff check . && mypy . && pytest
   # Go
   go vet ./... && go test ./...
   ```
2. **Compare with the baseline.** Every check that passed before must still pass. Fix or revert any regression before delivering the summary.
3. **Smoke-test what you touched** when a dev server or preview URL is available (use Playwright or a browser tool if present):
   - Load the key pages and check the console for CSP violations and broken embeds.
   - Submit a form you added validation to.
   - Hit an endpoint you added a guard to, both as an allowed user and a denied one.
   - If no environment is available, list these as Manual follow-up checks.
4. **Write the summary.**

### Security Hardening Summary (required output)

Always produce this summary, including the **findings table**, so triage decisions are auditable. Never include secret values in it.

```
## Security Hardening Summary

**Mode:** Audit / Fix   **Scope:** [repo or package]   **Stack:** [e.g., Next.js 16 on Vercel, Supabase]

### Threat Model (top surfaces)
- [e.g., POST /api/transfer — unauthenticated-reachable, touches payments — HIGH priority]
- [e.g., POST /api/lead — public by design — needs validation + bot protection, not auth]

### Findings (triaged)
| Severity | Location           | Issue                          | Tier | Status          |
|----------|--------------------|--------------------------------|------|-----------------|
| Critical | supabase: leads    | RLS disabled; anon key can read | 3    | Proposed        |
| High     | api/users.ts:88    | IDOR — no ownership check       | 2    | Fixed           |
| Medium   | web/profile.tsx:30 | Reflected XSS in name field     | 1    | Fixed           |
| Low      | next.config.js     | No CSP                          | 3    | Report-Only shipped; enforce pending |
| —        | api/legacy.ts:12   | Suspected SQLi                  | —    | False positive  |

### Changes Made
- [file path]: [what changed and why] — [evidence for Tier 2, e.g., "no frontend reads user.email"]

### Proposed — needs your approval (Tier 3)
- [change, why, what could break, how to roll it out]

### Deploy-time actions (do these BEFORE merging)
- [ ] [e.g., Set STRIPE_WEBHOOK_SECRET in Vercel Production + Preview]

### Manual Follow-up
- [ ] [e.g., Rotate the Stripe secret key previously committed in api/stripe.ts]
- [ ] [e.g., Review remaining moderate audit findings]

### Verification (baseline → after)
| Check     | Before | After |
|-----------|--------|-------|
| Lint      | PASS   | PASS  |
| Typecheck | FAIL (3, pre-existing) | FAIL (3, unchanged) |
| Tests     | PASS (47/47) | PASS (49/49) |
| Build     | PASS   | PASS  |
| Audit     | 2 high | 0 high, 1 moderate |
| Smoke     | —      | [pages/flows checked, or "not run — no preview"] |
```

---

## 10. Automation Templates (offer, don't impose)

The skill's repository ships CI and pre-commit templates under `templates/` (https://github.com/serratosalex20/security-hardening-skill/tree/master/templates):
- `security.yml`: gitleaks, semgrep, and trivy; language-agnostic
- `ci.yml`: Python only
- `ci-node.yml`: JS/TS
- Python `pyproject.toml` and `.pre-commit-config.yaml`

- **Offer them in the summary; add them only when asked.** A new workflow or hook is Tier 3.
- **Adapt to the stack.** Don't add a Python workflow to a Node repo or vice versa.
- **On an existing codebase, expect the first run to be red.**
  - Fix or baseline the existing findings first: `.gitleaksignore` for already-rotated secrets, `nosemgrep` with a reason, and gradual ruff/mypy adoption as described in the templates README.
  - Then make the checks required in branch protection.
- **Keep the version pins.** Actions are pinned to commit SHAs because tags can be moved, as happened in the March 2026 trivy-action compromise.
