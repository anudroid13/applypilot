# ApplyPilot — Product Requirements Document

**Version:** 1.1
**Status:** Draft — Ready for Development
**Author:** Anurag Nigam (updated)
**Date:** June 2026

---

## 1. Overview

### 1.1 Product name

**ApplyPilot** — an AI-powered, agentic job application assistant that automates the end-to-end job search and application workflow for a single user (personal tool, not SaaS). Controlled entirely through a Gradio web UI running locally.

### 1.2 Problem statement

Applying to jobs is repetitive, time-consuming, and easy to lose track of. A senior engineer targeting 20–30 roles simultaneously must manually search multiple platforms, assess each JD, customise applications, and chase status updates — often spending 5–10 hours per week on pure logistics rather than interview prep.

### 1.3 Goal

Reduce the manual overhead of a job search to: (1) uploading a resume once, (2) reviewing and approving a shortlist, and (3) reading notifications. Everything else — discovery, scoring, form-filling, status tracking, and alerting — is handled by the agent.

### 1.4 Success metrics

| Metric                                             | Target                           |
| -------------------------------------------------- | -------------------------------- |
| Resume parse accuracy                              | ≥ 95% fields correctly extracted |
| Relevance score correlation with manual assessment | ≥ 85% agreement                  |
| Successful form submissions (dry-run verified)     | ≥ 90% of approved jobs           |
| Time from "run" to notification                    | < 30 minutes end-to-end          |
| Zero accidental applications                       | 100% — human gate is mandatory   |

---

## 2. Users

Single user (the developer/owner). No authentication layer needed. All credentials and secrets should be stored securely — see Security (Section 5.3) for mandatory controls.

---

## 3. Architecture summary

```
Resume (PDF/DOCX)
       │
       ▼
┌─────────────────┐     ┌──────────────────┐
│  Stage 1        │────▶│  Stage 2         │
│  Resume parser  │     │  Profile builder │
└─────────────────┘     └────────┬─────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Stage 3                │
                    │  Job discovery          │
                    │  LinkedIn + Naukri      │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Stage 4                │
                    │  Relevance scoring      │
                    │  + Human approval gate  │
                    └────────────┬────────────┘
                                 │ (approved jobs only)
                    ┌────────────▼────────────┐
                    │  Stage 5                │
                    │  Auto-apply engine      │
                    │  (dry-run / live)       │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Stage 6                │
                    │  Status tracker         │
                    │  (poll + diff)          │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Stage 7                │
                    │  Notifications          │
                    │  Telegram / WhatsApp    │
                    └─────────────────────────┘
```

All stages are orchestrated via APScheduler (periodic) and can be triggered manually per-tab in the Gradio UI.

---

## 4. Functional requirements

(Sections condensed for brevity — implementation details retained in module specs.)

---

## 5. Non-functional requirements

### 5.1 Performance

- Resume parsing: < 10 seconds end-to-end.
- Relevance scoring: < 2 seconds per job (LLM call); batch of 20 < 60 seconds.
- Scraping: 50 jobs < 10 minutes (with anti-bot delays).
- Apply engine: 1 application < 3 minutes (including cover letter generation).

### 5.2 Reliability

- All scraping operations wrapped in tenacity retry (max 3 attempts, exponential backoff).
- DB writes use transactions — no partial state on crash.
- APScheduler persists job state across restarts using `SQLAlchemyJobStore`.
- Gradio UI must remain responsive during long-running background tasks (use threads/generators).

### 5.3 Security, Privacy & Compliance (UPDATED)

This section is mandatory. Follow these controls before running the agent with real credentials or personal data.

- Secrets management
  - **Never** commit secrets or API keys to version control. The `OPENAI_API_KEY`, platform credentials (LinkedIn, Naukri), Telegram, and Twilio keys must be provided via environment variables or a secrets manager.
  - Prefer using a system secrets manager where possible (macOS Keychain, AWS Secrets Manager, Azure Key Vault, HashiCorp Vault). If using a `.env` file for local dev, keep it out of source control and set restrictive permissions (`chmod 600 .env`).
  - Provide separate keys for development and production. Rotate keys periodically and after any suspected exposure.

- Environment variables
  - Required keys (example names): `OPENAI_API_KEY`, `LINKEDIN_EMAIL`, `LINKEDIN_PASSWORD`, `TELEGRAM_BOT_TOKEN`, etc. These must come from the runtime environment and not be hardcoded.
  - Include `RELEVANCE_THRESHOLD`, `DRY_RUN`, and `DATABASE_URL` as configurable env vars.

- Least privilege & API usage
  - Use scoped API keys where supported. Monitor usage and set quotas/alerts in your API provider dashboard.
  - Implement rate limiting and exponential backoff for LLM calls to avoid accidental overuse.

- LLM input/output hygiene
  - **Minimise PII**: redact or minimise personally identifiable information before sending to external LLMs when possible.
  - Use prompts that avoid exposing full sensitive documents when not required; truncate and anonymise fields where feasible.
  - Validate and sanitise LLM outputs before using them to drive actions (e.g., form field values, navigation commands). Do not treat model outputs as authoritative without checks.
  - Cache LLM outputs and avoid re-sending the same sensitive text repeatedly.

- Logging and observability
  - Do not log secrets or full PII. Logs should redact API keys, tokens, full resumes, or raw JD text when it contains sensitive data.
  - Store logs locally under `data/logs/` with limited retention by default (e.g., 90 days). Make retention configurable.
  - Ensure screenshots saved on errors are stored with restricted permissions and rotated/deleted per retention policy.

- Data storage
  - SQLite is fine for local dev; for production use Postgres with encryption at rest.
  - Protect the `data/` directory and DB files with restrictive FS permissions.
  - Use parameterised queries / ORM (SQLAlchemy) to avoid injection — avoid building SQL by string concatenation.

- Network security
  - All outbound API calls must use HTTPS.
  - Validate TLS certificates and avoid insecure HTTP connectors.

- Scraping legality and safety
  - Respect `robots.txt` where practical. Use a secondary development account for scraping and avoid aggressive scraping patterns.
  - Implement human-in-the-loop fallback for CAPTCHA and login failures; do not embed third-party CAPTCHA solving services without explicit user consent.

- Apply engine safety
  - `DRY_RUN=true` must be the default during development. The UI must make it explicit when live-submit is enabled.
  - The human approval gate (Stage 4) is mandatory — no automatic applications without user approval.

- Access control & audits
  - Run the agent only on trusted machines. Limit who has shell or file access to the machine running ApplyPilot.
  - Log state transitions with timestamps and job_id for auditability (avoid storing secrets in audit logs).

- Third-party dependencies
  - Pin dependency versions and periodically scan for vulnerabilities (e.g., `pip-audit`, `safety`).
  - Avoid running untrusted third-party code or dependencies that spawn arbitrary code execution.

- Privacy & retention
  - Default retention: parsed profiles, job metadata, logs, and screenshots should have configurable retention windows.
  - Provide an easy way to delete all user data (profile, jobs, logs, screenshots) from the local machine.

- Incident response
  - If a secret is exposed, revoke and rotate keys immediately and inspect logs for misuse.

**Security checklist (must be reviewed before running with real credentials)**

- [ ] `.env` or secrets manager configured and not committed to Git
- [ ] `DRY_RUN=true` set for initial runs
- [ ] Least-privilege API keys in use and rotate schedule defined
- [ ] Logs and screenshots retention/purging configured
- [ ] DB backups and DB file permission verified
- [ ] Dependency vulnerability scan run at least once per week (CI)

### 5.4 Observability

- Loguru structured logs to `data/logs/applypilot_{date}.log`.
- Log level configurable via `LOG_LEVEL` env var (default INFO).
- Every DB state transition logged with job_id, old status, new status, timestamp (without secrets).
- Gradio UI shows last run time + outcome for each stage.

### 5.5 Portability

- Runs on Linux, macOS, Windows (WSL2).
- No Docker required for local dev (plain venv).
- Database defaults to SQLite for zero-config local setup; Postgres URL accepted for prod.

---

## 6. Data model

(unchanged; ensure any stored PII fields are covered by retention/purging policies above)

---

## 7. Gradio UI spec

(unchanged; ensure UI highlights security state: DRY_RUN, last-run user, and whether secrets are loaded from env vs keychain)

---

## 8. Scheduler spec

(unchanged)

---

## 9. Build order & milestones

(unchanged)

---

## 10. Out of scope (v1)

- Multi-user support
- Job board beyond LinkedIn and Naukri (Indeed, Instahyre, etc.) — v2
- Interview scheduling automation
- Resume optimisation suggestions
- ATS keyword gap analysis (may add as Stage 1b in v2)
- Mobile app
- Cloud deployment / hosting

---

## 11. Risks & mitigations (UPDATED)

| Risk                            | Likelihood | Mitigation                                                                                                     |
| ------------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------- |
| LinkedIn blocks scraper account | High       | Use secondary account for dev; implement delays + undetected-playwright; cookie persistence to minimise logins |
| CAPTCHA on login                | Medium     | Manual solve fallback via `gr.Image` + input in UI; avoid 3rd-party captcha solving without consent            |
| Apply form structure changes    | Medium     | Stage 5 modular per-ATS; log failures, mark manual_required, notify                                            |
| OpenAI rate limits / key abuse  | Low        | Tenacity retry; batch scoring with sleep; cache LLM outputs; monitor usage and rotate keys quickly             |
| Accidental mass application     | Low        | `DRY_RUN` default-on; human gate mandatory; `max_applications_per_run` cap                                     |

---

_End of PRD v1.1_
