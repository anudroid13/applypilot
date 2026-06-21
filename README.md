# ✈️ ApplyPilot

> Your personal AI co-pilot for job hunting — parses your resume, discovers relevant roles on LinkedIn and Naukri, auto-applies, and pushes real-time updates to Telegram or WhatsApp.

---

## What it does

1. **Parses your resume** (PDF or DOCX) and builds a structured candidate profile using an LLM.
2. **Discovers jobs** by logging into LinkedIn and Naukri with your credentials and scraping matching listings.
3. **Scores each job** for relevance against your profile (0–100) and presents a ranked shortlist for your approval.
4. **Auto-applies** with a tailored cover letter, using a dry-run mode so you can verify every action before going live.
5. **Tracks application status** (applied → viewed → shortlisted → rejected) by polling platforms for changes.
6. **Notifies you** via Telegram Bot or WhatsApp (Twilio) with daily digests and instant status alerts.

All controlled through a **Gradio SPA** — one tab per stage, fully testable in isolation.

---

## Quickstart

### 1. Clone and install

```bash
git clone https://github.com/yourhandle/applypilot.git
cd applypilot
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
playwright install chromium
```

### 2. Configure secrets (IMPORTANT — SECURITY)

Create a `.env` from the provided `.env.example` and populate the secrets. DO NOT commit `.env` to version control.

Preferred options (in order):

- Use your OS or cloud secrets manager (macOS Keychain, AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) and inject keys into the environment at runtime.
- For local development only: create a `.env` file in the project root and set restrictive permissions:

```bash
cp .env.example .env
# Edit .env — add your OpenAI key, LinkedIn/Naukri credentials, Telegram token
chmod 600 .env
```

Environment variable notes:

- `OPENAI_API_KEY` must be provided via environment — do not hardcode.
- Prefer separate keys for development vs production.
- Rotate keys regularly and revoke immediately if exposed.

Sensitive files and directories should be excluded from git using `.gitignore` (the repo includes `.env` and `data/` by default). Verify `.env` is not present in commits before pushing.

### 3. Run (safe default)

Default settings are safe for development (`DRY_RUN=true`). Verify and then run:

```bash
python app.py
# Opens at http://localhost:7860
```

Notes:

- Keep `DRY_RUN=true` until you have validated the apply flows.
- The UI shows whether live submit is enabled; the human-approval gate is mandatory.

---

## Environment variables (short)

| Variable             | Required | Description                                                 |
| -------------------- | -------- | ----------------------------------------------------------- |
| `OPENAI_API_KEY`     | ✅       | GPT-4o API key — provide via environment or secrets manager |
| `LINKEDIN_EMAIL`     | ✅       | LinkedIn login email                                        |
| `LINKEDIN_PASSWORD`  | ✅       | LinkedIn password                                           |
| `NAUKRI_EMAIL`       | optional | Naukri login email                                          |
| `NAUKRI_PASSWORD`    | optional | Naukri password                                             |
| `TELEGRAM_BOT_TOKEN` | optional | From @BotFather                                             |
| `TELEGRAM_CHAT_ID`   | optional | Your personal chat ID                                       |
| ...                  |          | (see `.env.example` for full list)                          |

---

## Safety & Security (summary)

- Use a secrets manager where possible; `.env` is acceptable only for local dev and must be excluded from VCS and have `600` permissions.
- Do not log secrets or full PII. The app redacts tokens from logs by default.
- `DRY_RUN` defaults to `true` — do not disable until confident in logs and dry-run outputs.
- Use a secondary LinkedIn account for development to reduce the risk of blocks.
- Review the full Security checklist in `PRD.md` before running against real accounts.

---

## Tech stack

| Layer              | Technology                                    |
| ------------------ | --------------------------------------------- |
| UI                 | Gradio 4.x (Blocks)                           |
| LLM                | OpenAI GPT-4o via LangChain                   |
| Browser automation | Playwright (undetected-chromedriver fallback) |
| Database           | SQLite → PostgreSQL                           |
| Scheduler          | APScheduler                                   |
| Notifications      | python-telegram-bot, Twilio                   |
| Config             | Pydantic Settings + python-dotenv             |
| Logging            | Loguru                                        |

---

## Roadmap

(unchanged)

---

## License

MIT
