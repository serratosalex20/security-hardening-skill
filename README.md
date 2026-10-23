# security-hardening

A Claude Code skill that runs an AI-assisted, threat-model-driven security review of your web app and applies a hardening baseline before shipping to production. Works across stacks: TypeScript/JavaScript, Next.js, Python, Go, Ruby, PHP, Java, Supabase/Firebase backends, and static sites on Vercel/Netlify.

It scans broadly against an OWASP-aligned taxonomy, **triages findings by severity and true-positive likelihood**, then fixes the real issues (highest severity first) with centralized reusable helpers. It finishes with a summary report and an auditable findings table.

It is built to run safely inside **any** repo or live site: it audits before it edits, records a build/test baseline, follows the target repo's own conventions, and never applies a change that could take a site down without asking first.

---

## Safe to run in any repo

Security fixes are a classic way to break production. A strict CSP blocks your analytics and booking widget, `X-Frame-Options: DENY` breaks an embed, or HSTS `preload` locks a subdomain out for months. A new required env var fails the next deploy, or enabling RLS with no policies blocks every query. The skill has explicit guardrails against all of these (Section 0 of `SKILL.md`):

| Guardrail | What it means |
|---|---|
| **Audit vs Fix mode** | "Review", "audit", "scan", and pre-deploy checks are read-only. Edits happen only when you ask to harden/fix, or approve fixes from an audit. |
| **Baseline first** | Detects the stack and package manager, runs the project's *own* lint/typecheck/test/build before touching anything, and re-runs them after. Pre-existing failures are reported, not blamed on the fix. Any new failure is fixed or reverted. |
| **Repo conventions win** | Follows the target repo's `CLAUDE.md`/`AGENTS.md`, reuses its existing auth guards, validators, and loggers, and doesn't add a new library when one already installed does the job. |
| **Change-risk tiers** | **Tier 1**: behavior-preserving fixes (parameterized queries, log redaction, `nosniff`) are applied directly. **Tier 2**: applied after checking real callers (auth guards, validation schemas, serializers). **Tier 3**: proposed for your approval (CSP enforcement, HSTS, CORS, cookies, RLS, new env vars, major upgrades, new CI gates). |
| **Public by design** | Lead forms, webhooks, and OAuth callbacks get validation, signatures, and abuse controls, not a login wall that would break the business. |
| **Secret hygiene** | Never prints secret values, doesn't read real `.env` values, never rewrites git history, and always lists leaked keys for rotation. |
| **Minimal, revertible diffs** | No repo-wide reformatting or drive-by refactors; one logical change per commit. |

---

## What it does

When triggered, the skill applies the ground rules, then walks through 9 sections:

| Section | What it does |
|---------|---------------|
| **0. Ground rules** | Picks Audit or Fix mode, records the build/test baseline, scopes to the target repo, and assigns every fix a risk tier |
| **1. Threat model & AI-assisted scanning** | Maps entry points, assets, public-by-design endpoints, and attacker goals; scans the OWASP-aligned taxonomy (fanning out parallel read-only subagents on large repos); triages every candidate by severity and confirms true positives before acting |
| **2. Secrets** | Hardcoded keys and tokens; client-bundled env vars (`NEXT_PUBLIC_*`, `VITE_*`, …), including which "public" keys are fine; rotation; build-safe env validation |
| **3. Input handling** | Validation at every trust boundary without rejecting legitimate callers; SQL/NoSQL/command injection; XSS; SSRF; open redirects; webhook signatures; lead-form abuse; LLM prompt/output handling |
| **4. Authorization** | Server-side auth + ownership checks; IDOR / cross-tenant prevention; Supabase RLS and Firebase rules rolled out without locking the app out |
| **5. Data exposure & output minimization** | Allowlist serializers that keep every field consumers read; no PII in URLs; generic errors with stable status codes |
| **6. HTTP security headers & config** | Staged rollout: `nosniff`/`Referrer-Policy` → frame protection → CSP in Report-Only → enforcement; careful HSTS, `Permissions-Policy`, CORS, cookies, CSRF |
| **7. Dependency hygiene** | Audits with the project's own package manager (npm/pnpm/yarn/pip/go/bundler/composer/cargo + `osv-scanner`); least-disruptive upgrades; no `--force` |
| **8. Observability** | Security-event logging via the project's existing logger; no secrets or PII in logs |
| **9. Verification** | Re-runs the baseline, smoke-tests touched pages and flows, and writes the summary with findings, proposals, deploy-time actions, and before/after results |

The core philosophy mirrors lessons from large-scale AI vulnerability discovery: **finding candidate issues is cheap; the bottleneck is triage, prioritization, and fixing.** So the skill confirms true positives, ranks by severity, and fixes the issues that matter first.

---

## Installation

### Claude Code (recommended)

**Option A: via `npx skills add` (installs across all supported agents)**

```bash
npx skills add https://github.com/serratosalex20/security-hardening-skill --skill security-hardening
```

This installs the skill for Claude Code, Cursor, Windsurf, Copilot, and 40+ other agents simultaneously.

**Option B: manual install (Claude Code only)**

`SKILL.md` is self-contained, so the single file is all an agent needs.

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

After installing, confirm the skill file is in place and Claude Code sees it:

```bash
# Global install
ls ~/.claude/skills/security-hardening/SKILL.md
# Project-scoped install
ls .claude/skills/security-hardening/SKILL.md
```

Then open a new Claude Code session and run `/skills`. You should see `security-hardening` in the list.

---

## Usage

Trigger the skill by saying any of these naturally in conversation:

```
Do a security review before we ship.          → Audit mode (read-only report)
Scan this project for vulnerabilities.        → Audit mode
Threat model this app.                        → Audit mode
Harden this codebase.                         → Fix mode
Apply a full security baseline to this repo.  → Fix mode
Make this production-ready.                   → Fix mode
Check our security headers.                   → Only Section 6 (plus ground rules)
```

You can also invoke it explicitly:

```
/security-hardening
```

To apply fixes after an audit, approve them in your reply ("apply the Tier 1 and Tier 2 fixes", "go ahead with the RLS migration").

---

## What to expect

Claude will:

1. **Set ground rules:** pick Audit or Fix mode, detect the stack and package manager, and record a baseline of the project's own checks.
2. **Threat model** your app: map entry points, assets, public-by-design endpoints, and attacker goals to prioritize the scan.
3. **Scan** the codebase across the OWASP-aligned taxonomy, fanning out parallel subagents on large repos.
4. **Triage** every candidate: confirm true positives, assign severity and a risk tier, deduplicate, and discard noise.
5. **Fix** (Fix mode only) real issues highest-severity-first within the risk tiers, using the project's own helpers and libraries.
6. **Propose** anything risky (Tier 3) with what could break and how to roll it out, and **flag** deploy-time actions (env vars to set before merging, keys to rotate).
7. **Verify** against the baseline (no new failures) and smoke-test the pages and flows it touched.
8. **Deliver** a written Security Hardening Summary with a triaged findings table.

### Example summary output

```
## Security Hardening Summary

**Mode:** Fix   **Scope:** whole repo   **Stack:** Next.js 16 on Vercel, Supabase

### Findings (triaged)
| Severity | Location           | Issue                           | Tier | Status          |
|----------|--------------------|---------------------------------|------|-----------------|
| Critical | supabase: leads    | RLS disabled; anon key can read  | 3    | Proposed        |
| High     | api/users.ts:88    | IDOR — no ownership check        | 2    | Fixed           |
| Medium   | web/profile.tsx:30 | Reflected XSS in name field      | 1    | Fixed           |
| Low      | next.config.js     | No CSP                           | 3    | Report-Only shipped; enforce pending |
| —        | api/legacy.ts:12   | Suspected SQLi                   | —    | False positive  |

### Changes Made
- app/api/users/[id]/route.ts: added requireOwnership() — no other caller reads other users' records
- lib/logger.ts: added redaction for authorization headers and tokens

### Proposed — needs your approval (Tier 3)
- Enable RLS on `leads` with an insert-only policy for anon and read for the owner (migration drafted, not applied)

### Deploy-time actions (do these BEFORE merging)
- [ ] Set STRIPE_WEBHOOK_SECRET in Vercel Production + Preview

### Manual Follow-up
- [ ] Rotate the Stripe secret key previously committed in api/stripe.ts

### Verification (baseline → after)
| Check     | Before       | After        |
|-----------|--------------|--------------|
| Lint      | PASS         | PASS         |
| Typecheck | PASS         | PASS         |
| Tests     | PASS (47/47) | PASS (49/49) |
| Build     | PASS         | PASS         |
| Audit     | 2 high       | 0 high       |
```

---

## Automation templates

The skill drives a manual review pass. To make those checks run automatically on
every commit and PR, this repo ships drop-in templates under [`templates/`](templates/):

| Template | What it gives you |
|----------|-------------------|
| `templates/github-workflows/security.yml` | Any stack. PR + weekly gate: gitleaks (secrets), semgrep (SAST, PRs fail only on new findings), trivy (deps + secrets + IaC) |
| `templates/github-workflows/ci.yml` | Python. PR gate: ruff (lint + format), mypy (types), pip-audit (dependency CVEs) |
| `templates/github-workflows/ci-node.yml` | JS/TS. PR gate: installs with your lockfile's package manager and runs only the lint/typecheck/test/build scripts you have |
| `templates/python/pyproject.toml` | Ruff + mypy config tuned for web projects (security `S` rules on) |
| `templates/python/.pre-commit-config.yaml` | Local pre-commit enforcement (ruff, mypy, gitleaks, hygiene) |

These are **not** auto-applied. Copy what you need into your own projects, and read
["Adopting on an existing repo"](templates/README.md#adopting-on-an-existing-repo-read-this-first)
first so strict checks don't block unrelated work. Every action is pinned to a commit SHA;
see [`templates/README.md`](templates/README.md) for the rationale and how each tool maps to a skill section.

---

## Requirements

- **Claude Code** (CLI, desktop app, or IDE extension), or any agent that supports the skills format
- The skill itself has no runtime dependencies. It instructs Claude to use whatever tools and libraries are already in your project.
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

If you've copied `SKILL.md` into other repos or skill bundles, re-copy it there too.
Copies don't update themselves. See [`CHANGELOG.md`](CHANGELOG.md) for what changed.

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
