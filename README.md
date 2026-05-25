# security-hardening

A Claude Code skill that runs an AI-assisted, threat-model-driven security review of your web app and applies a full hardening baseline before shipping to production. Works across stacks — TypeScript/JavaScript, Python, Go, Ruby, PHP, Java, and more.

It scans broadly against an OWASP-aligned taxonomy, **triages findings by severity and true-positive likelihood**, then fixes the real issues (highest severity first) with centralized reusable helpers — and produces a final summary report with an auditable findings table.

---

## What it does

When triggered, the skill walks through 7 sections in order:

| Section | What it does |
|---------|---------------|
| **1. Threat model & AI-assisted scanning** | Maps entry points, assets, and attacker goals; scans the OWASP-aligned taxonomy (optionally fanning out parallel read-only subagents); triages every candidate by severity and confirms true positives before acting |
| **2. Secrets** | Hardcoded API keys, tokens, passwords; client-bundled env vars (`NEXT_PUBLIC_*`, `VITE_*`, …); secrets in logs; startup env validation |
| **3. Input handling** | Validation at every trust boundary; SQL/NoSQL/command injection; XSS; SSRF; open redirects |
| **4. Authorization** | Every route/action checks auth + ownership server-side; IDOR / cross-tenant prevention; no frontend-only guards |
| **5. Dependency hygiene** | `npm`/`pip`/`go`/`bundler`/`composer`/`cargo` audits + `osv-scanner`; unmaintained packages; HTTP/DB client config |
| **6. Observability** | Structured logging for auth/authz/validation/webhook failures; no secrets in logs |
| **7. Verification** | Lint, typecheck, tests, build, security scan; final written summary with findings table |

The core philosophy mirrors lessons from large-scale AI vulnerability discovery: **finding candidate issues is cheap; the bottleneck is triage, prioritization, and fixing.** So the skill confirms true positives, ranks by severity, and fixes the issues that matter first.

At the end, Claude produces a **Security Hardening Summary** with a triaged findings table, every change made, every risk fixed, and any manual follow-up required (e.g. rotating a leaked key).

---

## Installation

### Claude Code (recommended)

**Option A — via `npx skills add` (installs across all supported agents)**

```bash
npx skills add https://github.com/serratosalex20/security-hardening-skill --skill security-hardening
```

This installs the skill for Claude Code, Cursor, Windsurf, Copilot, and 40+ other agents simultaneously.

**Option B — manual install (Claude Code only)**

```bash
# Global install (available in all projects)
mkdir -p ~/.claude/skills/security-hardening
curl -o ~/.claude/skills/security-hardening/SKILL.md \
  https://raw.githubusercontent.com/serratosalex20/security-hardening-skill/master/SKILL.md
```

```bash
# Project-scoped install (available in current project only)
mkdir -p .claude/skills/security-hardening
curl -o .claude/skills/security-hardening/SKILL.md \
  https://raw.githubusercontent.com/serratosalex20/security-hardening-skill/master/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/skills/security-hardening
curl -o .cursor/skills/security-hardening/SKILL.md \
  https://raw.githubusercontent.com/serratosalex20/security-hardening-skill/master/SKILL.md
```

### Windsurf / other agents

```bash
mkdir -p .windsurf/skills/security-hardening
curl -o .windsurf/skills/security-hardening/SKILL.md \
  https://raw.githubusercontent.com/serratosalex20/security-hardening-skill/master/SKILL.md
```

---

## Verification

After installing, confirm the skill is available:

```bash
# Claude Code
claude skills list | grep security-hardening

# Or open a new Claude Code session and run:
/skills
```

You should see `security-hardening` in the list.

---

## Usage

Once installed, trigger the skill by saying any of these naturally in conversation:

```
Apply a full security baseline to this repo.
Do a security review before we ship.
Harden this codebase.
Scan this project for vulnerabilities.
Threat model this app and fix what matters.
Make this production-ready.
Run a security audit.
```

Claude will automatically invoke the skill and work through all 7 sections.

You can also invoke it explicitly:

```
/security-hardening
```

---

## What to expect

Claude will:

1. **Threat model** your app — map entry points, assets, and attacker goals to prioritize the scan
2. **Scan** the codebase across the OWASP-aligned taxonomy (fanning out parallel subagents for large repos)
3. **Triage** every candidate — confirm true positives, assign severity, deduplicate; discard noise
4. **Fix** real issues highest-severity-first, using centralized helpers (no copy-pasted guards)
5. **Explain** every change and **flag** anything requiring manual action (e.g. rotate a leaked key)
6. **Run** lint, typecheck, tests, and build to verify nothing is broken
7. **Deliver** a written Security Hardening Summary with a triaged findings table

### Example summary output

```
## Security Hardening Summary

### Findings (triaged)
| Severity | Location           | Issue                        | Status         |
|----------|--------------------|------------------------------|----------------|
| Critical | api/auth.ts:42     | Auth bypass via empty token  | Fixed          |
| High     | api/users.ts:88    | IDOR — no ownership check     | Fixed          |
| Medium   | web/profile.tsx:30 | Reflected XSS in name field   | Fixed          |
| Low      | next.config.js     | Missing security headers      | Follow-up      |
| —        | api/legacy.ts:12   | Suspected SQLi                | False positive |

### Changes Made
- lib/env.server.ts: Created server-only env validation
- app/api/users/route.ts: Added requireAuth() + input schema validation
- lib/logger.ts: Added structured logger with secret redaction

### Risks Fixed
- Secrets: Moved 3 hardcoded API keys to server-only env vars
- Input handling: Added schema validation to 5 API routes
- Authorization: Added requireAuth() to 4 unguarded routes
- Observability: Replaced console.log(token) with redacted logger

### Manual Follow-up Required
- [ ] Rotate STRIPE_SECRET_KEY — was previously hardcoded in api/stripe.ts
- [ ] Set DATABASE_URL on production hosting platform

### Verification Results
- Lint:       PASS
- Typecheck:  PASS
- Tests:      PASS (47/47)
- Build:      PASS
- Audit:      0 high, 1 moderate issue remaining
```

---

## Requirements

- **Claude Code** (CLI, desktop app, or IDE extension) — or any agent that supports the skills format
- The skill itself has no runtime dependencies — it instructs Claude to use whatever tools/libraries are already in your project (Zod, pino, bcrypt, etc.)
- For the `npx skills add` install method: Node.js 18+

---

## Updating

```bash
# Re-run the same install command to get the latest version
npx skills add https://github.com/serratosalex20/security-hardening-skill --skill security-hardening

# Or manually overwrite:
curl -o ~/.claude/skills/security-hardening/SKILL.md \
  https://raw.githubusercontent.com/serratosalex20/security-hardening-skill/master/SKILL.md
```

---

## Uninstalling

```bash
# Global
rm -rf ~/.claude/skills/security-hardening

# Project-scoped
rm -rf .claude/skills/security-hardening
```

---

## License

MIT
