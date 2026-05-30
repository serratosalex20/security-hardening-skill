# Templates

Drop-in automation that backs the manual sections of the `security-hardening` skill.
The skill tells Claude *what* to check; these templates make those checks run
automatically on every commit and every pull request, so "verification PASS" is
grounded in something automated rather than a one-off local run.

Copy the files you need into your own web projects (they are **not** auto-applied —
this repo only ships the skill).

## What's here

| Path | Purpose |
|------|---------|
| `github-workflows/ci.yml` | PR gate: ruff (lint + format), mypy (types), pip-audit (dependency CVEs) |
| `github-workflows/security.yml` | PR + weekly gate: gitleaks (secrets), semgrep (SAST), trivy (deps + secrets + IaC misconfig) |
| `python/pyproject.toml` | Ruff + mypy config tuned for web projects (security `S` rules enabled) |
| `python/.pre-commit-config.yaml` | Local enforcement: ruff, mypy, gitleaks, hygiene hooks before each commit |

GitHub Actions live under `.github/workflows/` in your project, so copy
`github-workflows/ci.yml` → `.github/workflows/ci.yml`, etc.

## Recommended toolchain

A clean, non-overlapping stack — each tool owns one job, nothing is redundant:

| Job | Tool | Why this one |
|-----|------|--------------|
| Lint + format | **ruff** | All-in-one; replaces flake8, isort, black, pyupgrade, bandit, perflint |
| Type-check | **mypy** | Ruff doesn't type-check; complementary |
| Python deps | **pip-audit** | Known-CVE scan for installed packages |
| Secrets | **gitleaks** | One scanner — don't also run trufflehog/detect-secrets |
| SAST | **semgrep** | Best free multi-language static analysis (OWASP rulesets) |
| Deps + IaC + fs secrets | **trivy** | One binary covers dependencies, IaC misconfig, and filesystem secrets |

Deliberately omitted (redundant with the above): `black`, `isort`, `flake8`,
`pyupgrade`, `perflint`, `bandit`, `safety`, `detect-secrets`, `trufflehog`.

## How it maps to the skill

| Skill section | Automated by |
|---|---|
| 2. Secrets | gitleaks + trivy (secret scanner) |
| 3. Input handling | semgrep (injection / XSS rules) |
| 4. Authorization · 5. Data exposure | semgrep (OWASP ruleset) |
| 6. Headers / config / IaC | trivy misconfig |
| 7. Dependencies | trivy vuln + pip-audit |
| 9. Verification | the whole CI pipeline |

## Notes

- **No paid accounts or tokens** are required as configured (gitleaks personal/public,
  semgrep public rulesets, trivy, pip-audit are all free).
- `trivy` uses `ignore-unfixed: true` so the build only fails on CVEs that actually
  have a fix available — avoiding a permanently-red pipeline from un-patchable advisories.
- The weekly `cron` in `security.yml` re-scans even with no new commits, so freshly
  disclosed CVEs surface within a week.
- Pin versions (`rev:` / action tags) are current at time of writing — run
  `pre-commit autoupdate` and bump action tags periodically.
