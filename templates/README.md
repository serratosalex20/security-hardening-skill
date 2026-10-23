# Templates

Drop-in automation that backs the manual sections of the `security-hardening` skill.
The skill tells Claude *what* to check; these templates make those checks run
automatically on every commit and every pull request, so "verification PASS" is
grounded in something automated rather than a one-off local run.

Copy the files you need into your own projects. They are **not** auto-applied, and the
skill only offers them; it doesn't install them unless you ask.

## What's here

| Path | Stack | Purpose |
|------|-------|---------|
| `github-workflows/security.yml` | Any | PR + weekly gate: gitleaks (secrets), semgrep (SAST), trivy (deps + secrets + IaC misconfig) |
| `github-workflows/ci.yml` | Python | PR gate: ruff (lint + format), mypy (types), pip-audit (dependency CVEs) |
| `github-workflows/ci-node.yml` | JS/TS | PR gate: installs with the lockfile's package manager (npm/pnpm/yarn), then runs `lint`, `typecheck`, `test`, `build`, but only the scripts your `package.json` defines |
| `python/pyproject.toml` | Python | Ruff + mypy config tuned for web projects (security `S` rules on) |
| `python/.pre-commit-config.yaml` | Python | Local enforcement: ruff, mypy, gitleaks, hygiene hooks before each commit |

GitHub Actions live under `.github/workflows/` in your project. Copy
`github-workflows/security.yml` → `.github/workflows/security.yml`, and **one** of
`ci.yml` (Python) or `ci-node.yml` (JS/TS) → `.github/workflows/ci.yml`.

## Adopting on an existing repo (read this first)

These templates are strict by design. Dropped into a codebase that has never run them,
the first run will usually be red. That's expected, not breakage. Adopt them in this order
so they never block unrelated work:

1. **Add the workflows without making them required checks.** Let them run on a branch.
2. **`security.yml`.** Pull requests fail only on what they introduce: semgrep compares
   against the PR's base commit, and gitleaks scans the PR's commits. The push and weekly
   runs scan everything, so they show the existing backlog:
   - **Secrets found in history:** rotate the secret first, *then* add its fingerprint
     to `.gitleaksignore`. Removing it from the code doesn't un-leak it.
   - **semgrep findings that are false positives:** add `# nosemgrep: <rule-id>` with
     a reason on that line.
   - **trivy:** upgrade what has a fix. `ignore-unfixed: true` already skips CVEs with no fix.
3. **`ci.yml` (Python).**
   - Run `ruff format .` once in its own PR, so the reformat is reviewable on its own.
   - Then start ruff with the bug + security families
     (`select = ["E4", "E7", "E9", "F", "B", "S"]`) and widen one family per PR.
   - Keep mypy's `disallow_untyped_defs = false` until the code is annotated.
4. **`ci-node.yml` (JS/TS).** It runs only the scripts that exist. If `build` needs env vars,
   add them as repository secrets/variables in the workflow, or validate them lazily at
   runtime instead of at import (SKILL.md Section 2).
5. **Pre-commit.** `pre-commit run --all-files` rewrites files on first run (ruff fixes and
   formatting, whitespace). Commit that sweep on its own.
6. **Once green, make the checks required** in branch protection.

Also: the gitleaks action needs a free license key (`GITLEAKS_LICENSE` secret) for repos
owned by a GitHub **organization**; personal repos don't.

## Recommended toolchain

A clean stack: each tool owns one job.

| Job | Tool | Why this one |
|-----|------|--------------|
| Lint + format | **ruff** | All-in-one; replaces flake8, isort, black, pyupgrade, bandit, perflint |
| Type-check | **mypy** | Ruff doesn't type-check; complementary |
| Python deps | **pip-audit** | Known-CVE scan for installed packages |
| Secrets | **gitleaks** | One scanner. Don't also run trufflehog/detect-secrets (trivy's secret scanner is a free second pass) |
| SAST | **semgrep** | Best free multi-language static analysis (OWASP rulesets) |
| Deps + IaC + fs secrets | **trivy** | One binary covers dependencies (all ecosystems), IaC misconfig, and filesystem secrets |

Deliberately omitted (redundant with the above): `black`, `isort`, `flake8`,
`pyupgrade`, `perflint`, `bandit`, `safety`, `detect-secrets`, `trufflehog`, and semgrep's
`p/secrets` ruleset.

## How it maps to the skill

| Skill section | Automated by |
|---|---|
| 2. Secrets | gitleaks + trivy (secret scanner) |
| 3. Input handling | semgrep (injection / XSS rules) |
| 4. Authorization · 5. Data exposure | semgrep (OWASP ruleset) |
| 6. Headers / config / IaC | trivy misconfig |
| 7. Dependencies | trivy vuln (all ecosystems) + pip-audit |
| 9. Verification | the whole CI pipeline (`ci.yml` or `ci-node.yml`) |

## Supply-chain notes

- **Every action is pinned to a full commit SHA**, with the version in a comment, and the
  semgrep container is pinned by digest. Tags can be moved: in March 2026 attackers
  force-pushed malicious code onto existing `aquasecurity/trivy-action` tags, stealing CI
  secrets from workflows that ran by tag during the exposure window. Afterwards upstream moved to
  `v`-prefixed tags and deleted the old ones, so the earlier `@0.28.0` pin no longer resolves.
- **Pre-commit hooks are frozen to SHAs** the same way (`# frozen: vX.Y.Z`).
- **Updating:** use Dependabot (`package-ecosystem: github-actions`) or `pinact` for
  workflows, and `pre-commit autoupdate --freeze` for hooks. Prefer releases that are at
  least a week or two old over same-day releases.
- **No paid accounts or tokens** are required for personal repos: gitleaks (personal
  repos), semgrep public rulesets, trivy, and pip-audit are all free.
- `trivy` uses `ignore-unfixed: true`, so the build fails only on CVEs that have a fix.
- The weekly `cron` in `security.yml` re-scans even with no new commits, so newly
  disclosed CVEs surface within a week.
- Checkouts use `persist-credentials: false`, so the job token isn't left in `.git/config`
  for later steps.
