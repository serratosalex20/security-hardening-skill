# Changelog

## 2.0.0 — 2026-10-23

Makes the skill safe to activate in any repo or live site, and fixes templates that broke
when copied into real projects.

### Skill (`SKILL.md`)
- **New Section 0: ground rules.**
  - Audit vs Fix mode. Audits and pre-deploy checks are read-only.
  - Build/test baseline before any edit, with a regression check after.
  - Scope limits; target-repo conventions and existing helpers/libraries win.
  - Minimal, revertible diffs and git safety; secret hygiene for the agent itself.
- **Change-risk tiers.** Behavior-preserving fixes are applied. Caller-dependent fixes are
  applied with evidence. Fixes that break sites by default or need deploy-time config are
  proposed for approval.
- **Headers rollout.**
  - CSP ships as `Report-Only` first, after a third-party inventory. Next.js 16 `proxy.ts`
    nonces are covered.
  - HSTS is staged, with no `includeSubDomains`/`preload` without sign-off.
  - `X-Frame-Options` and `Permissions-Policy` only after checking embeds and feature use.
  - Merge into existing header config; never send two CSPs.
  - helmet's strict defaults are called out.
- **Secrets.**
  - Lists which "public" keys are fine (Supabase anon, Stripe `pk_`, analytics IDs).
  - Env validation is lazy, so `next build` and CI don't fail without production secrets.
  - Rotation without history rewrites.
- **Input handling.** Strip unknown fields instead of rejecting them on existing endpoints
  (webhooks, CRMs, UTM params). Also adds public-by-design lead forms, webhook signatures
  over the raw body, and AI/LLM prompt/output handling.
- **Authorization.**
  - Public-by-design endpoints don't get login walls.
  - New Supabase RLS/BaaS subsection: enabling RLS without policies locks the app out, so
    write the policies first.
- **Dependencies.**
  - Use the lockfile's package manager and production-first audits.
  - No `audit fix --force`; report unused dependencies instead of removing them.
  - Updated commands (`osv-scanner scan source -r`, Yarn 1 vs 2+; dropped `safety`).
- **Verification.** Before/after baseline table, smoke tests, and new summary sections for
  mode, risk tier, Tier 3 proposals, and deploy-time actions.
- Frontmatter description trimmed to under the 1024-character skill limit; it no longer
  says "ALWAYS use", and pre-deploy triggers run in audit mode.

### Templates
- **Fixed:** `.pre-commit-config.yaml` failed `pre-commit` validation (`additional_dependencies` was empty/null), so no hooks ran.
- **Fixed:** `security.yml` referenced `aquasecurity/trivy-action@0.28.0`. That tag was hijacked in the March 2026 Trivy supply-chain attack and no longer exists, so the job could not start. Now pinned to the v0.36.0 commit SHA.
- **Fixed:** `ci.yml` ran `pip install -e .` on any `pyproject.toml`. That failed for apps whose `pyproject.toml` only holds tool config (including this repo's own template). It now installs only when a `[project]` table exists.
- **Fixed:** `ci.yml` used `pip-audit --strict`, which always fails on the project's own unpublished package. Vulnerabilities still fail the job.
- **Changed:** all actions are pinned to full commit SHAs, and the semgrep image is pinned by digest. Bumped to Node 24 action majors (Node 20 was removed from runners in September 2026). `persist-credentials: false` everywhere. Workflow audit: zizmor reported 18 findings on the old templates (6 medium, 12 high) and none on the new ones.
- **Changed:** semgrep fails PRs only on findings they introduce (`--baseline-commit`); push/weekly runs still scan everything. Dropped the duplicate `p/secrets` ruleset.
- **Changed:** pre-commit revs are frozen to SHAs; ruff 0.17.0, mypy 2.4.0, gitleaks 8.30.1, pre-commit-hooks 6.0.0. The ruff rule `TCH` is renamed to `TC`.
- **Added:** `ci-node.yml` for JS/TS. It detects npm/pnpm/yarn from the lockfile and runs only the scripts that exist.
- **Added:** "Adopting on an existing repo" guide in `templates/README.md`.

### Docs
- README: new "Safe to run in any repo" section and the Audit/Fix modes. Replaced the non-existent `claude skills list` command.

## 1.1.0

- AI-assisted scanning (threat model first, parallel read-only subagents, triage), polyglot examples, automation templates (#1).

## 1.0.0

- Initial skill.
