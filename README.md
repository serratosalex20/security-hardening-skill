# security-hardening

A Claude Code skill that applies a full security baseline to your application before shipping to production.

Covers secrets management, input validation, authorization, dependency hygiene, observability, and verification — with centralized reusable helpers and a final summary report.

---

## What it does

When triggered, the skill walks through 6 sections in order:

| Section | What it checks |
|---------|---------------|
| **1. Secrets** | Hardcoded API keys, tokens, passwords; client-exposed env vars; secrets in logs |
| **2. Input handling** | Validation at every trust boundary; SQL injection; XSS; open redirects; shell injection |
| **3. Authorization** | Every route/action checks auth + ownership server-side; no frontend-only guards |
| **4. Dependency hygiene** | `npm audit`; unmaintained packages; HTTP/DB client configuration |
| **5. Observability** | Structured logging for auth/validation/webhook failures; no secrets in logs |
| **6. Verification** | Lint, typecheck, tests, build, security scan; final written summary |

At the end, Claude produces a **Security Hardening Summary** listing every change made, every risk fixed, and any manual follow-up required (e.g. rotating a leaked key).

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
Check for security vulnerabilities.
Make this production-ready.
Run a security audit.
```

Claude will automatically invoke the skill and work through all 6 sections.

You can also invoke it explicitly:

```
/security-hardening
```

---

## What to expect

Claude will:

1. **Audit** your codebase across all 6 sections
2. **Fix** each issue using centralized helpers (no copy-pasted guards)
3. **Explain** every change it makes and why
4. **Flag** anything requiring manual action (e.g. rotate a leaked key, set an env var on your hosting platform)
5. **Run** lint, typecheck, tests, and build to verify nothing is broken
6. **Deliver** a written Security Hardening Summary

### Example summary output

```
## Security Hardening Summary

### Changes Made
- lib/env.server.ts: Created server-only env validation with Zod
- app/api/users/route.ts: Added requireAuth() + input schema validation
- convex/userLLMKeys.ts: Replaced Caesar cipher with AES-256-GCM
- lib/logger.ts: Added structured logger with secret redaction

### Risks Fixed
- Secrets: Moved 3 hardcoded API keys to server-only env vars
- Input handling: Added Zod validation to 5 API routes
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
