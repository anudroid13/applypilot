# ApplyPilot Maintenance & Required Skills

This document lists the operational skills, runbook steps, and commands required to maintain ApplyPilot securely and reliably. Reference the Security checklist in `PRD.md` before performing any maintenance.

## Required Skills

- Secrets management: experience with macOS Keychain, HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault.
- Python packaging and dependency management: pip, virtualenv/venv, `pip-audit`, `pip-tools`.
- CI/CD: GitHub Actions workflow authoring and debugging.
- Observability: log management (Loguru), rotating logs, and secure storage.
- Web automation: Playwright and browser automation best practices, anti-detection strategies, and CAPTCHAs handling.
- LLM safety: prompt engineering, PII minimisation, caching LLM outputs, and output validation.
- Databases: SQLAlchemy, migrations with Alembic, backups and restore for SQLite/Postgres.
- Incident response: secret rotation, forensic log review, and remediation steps.
- Legal & compliance: basic awareness of scraping policies and platform ToS.

## Operational Runbook

1. Secrets
   - Store all API keys in a secrets manager. For local dev use `.env` with `chmod 600` and never commit it.
   - To rotate a key: revoke old key in provider, generate new key, update secret store, and restart the service.

2. Dependency vulnerability scanning
   - Run `pip-audit` locally in a fresh venv or via CI. Example:

```bash
python -m venv .venv_audit
source .venv_audit/bin/activate
python -m pip install --upgrade pip
python -m pip install pip-audit
python -m pip_audit --format=json
```

3. Secret scanning
   - Run `gitleaks` locally:

```bash
curl -sSL https://github.com/zricethezav/gitleaks/releases/download/v8.18.0/gitleaks_8.18.0_macOS_x86_64.tar.gz | tar -xz
./gitleaks detect --source . --report-path gitleaks-report.json
```

4. LLM hygiene
   - Before sending resumes or JDs to LLMs, trim to necessary fields and redact sensitive identifiers (SSNs, private emails, tokens).
   - Cache LLM outputs keyed by prompt hash.

5. Playwright & scraping
   - Use a secondary LinkedIn account for dev.
   - Persist cookies and re-use to avoid frequent logins.
   - On CAPTCHA detection, surface an image in the UI and wait for manual input.

6. Backups & retention
   - Backup Postgres DB nightly in prod; for local SQLite, copy DB to secure backup folder with restricted permissions.
   - Enforce log and screenshot retention (default 90 days). Provide `data/cleanup.py` utility if needed.

7. Incident response
   - If a secret is leaked: rotate it immediately, search logs for suspicious activity, and invalidate sessions if possible.

## Useful Commands

- Run tests:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest -q
```

- Run security checks (local):

```bash
python -m venv .venv_audit
source .venv_audit/bin/activate
pip install pip-audit
python -m pip_audit --format=json > pip_audit_report.json
# gitleaks
# Download binary and run: ./gitleaks detect --source . --report-path gitleaks-report.json
```

- Code formatting & linting (local):

```bash
python -m venv .venv_fmt
source .venv_fmt/bin/activate
python -m pip install --upgrade pip
python -m pip install black isort pylint
# Check formatting and lint (fails on issues)
black --check .
isort --check-only .
pylint $(git ls-files '*.py')
```

## Onboarding new maintainers

- Read `PRD.md` Security section and complete the Security checklist.
- Get access to secrets manager and follow least-privilege practice.
- Run CI locally and ensure `pip-audit` and `gitleaks` pass.

## Contact & Escalation

- Owner: Anurag Nigam
- For security incidents: rotate keys, collect logs, and escalate to owner.

---

Keep this document updated as the codebase or operational requirements evolve.
