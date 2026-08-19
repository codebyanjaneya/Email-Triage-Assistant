# Email Triage Assistant — Runbook

A quick reference of every command needed to set up, run, and manage the app.
Run all commands from the project root (the folder that contains `run_server.py`).

> Platform: Windows PowerShell. On macOS/Linux, replace `.venv\Scripts\...`
> with `.venv/bin/...` and use `cp` instead of `Copy-Item`.

---

## 1. One-time setup

| Step | Command | What it does |
| ---- | ------- | ------------ |
| Create virtual environment | `python -m venv .venv` | Makes an isolated Python environment in `.venv/` so project packages don't touch your system Python. |
| Activate the environment | `.venv\Scripts\Activate.ps1` | Turns the venv on for the current terminal. You'll see `(.venv)` in the prompt. |
| Install dependencies | `pip install -r requirements.txt` | Installs FastAPI, uvicorn, Groq, fastembed, APScheduler, Google auth libraries, etc. |
| Create the env file | `Copy-Item .env.example .env` | Copies the template so you can fill in your own keys. |
| Generate a session secret | `python -c "import secrets; print(secrets.token_hex(32))"` | Prints a random value to paste into `SESSION_SECRET` in `.env`. |

After copying `.env`, open it and fill in:
- `GROQ_API_KEY` — your Groq API key (needed for classification + reply drafting).
- `APP_ENV` — leave `development` locally; use `production` on the hosted service.
- `SESSION_SECRET` — paste the value generated above.
- `ADDON_SHARED_SECRET` — a random shared secret used by the Gmail add-on.
- `OAUTH_REDIRECT_URI` — the exact public HTTPS OAuth callback in deployment.
- `APP_DB_PATH` — a path on persistent storage in deployment.

You also need two Google OAuth files in the project root (see README → Google Cloud setup):
- `credentials.json` — Desktop client (used by the CLI).
- `web_credentials.json` — Web client (used by the dashboard). Redirect URI must be `http://localhost:8000/auth/callback`.

---

## 2. Running the app

| Command | What it does |
| ------- | ------------ |
| `.venv\Scripts\python.exe run_server.py` | **Recommended.** Starts the web dashboard on http://127.0.0.1:8000. Auto-reload is OFF on purpose (a triage runs as a long background task and reload would interrupt it). |
| `uvicorn app.server:app --port 8000` | Alternative way to start the same server. |
| `uvicorn app.server:app --reload --port 8000` | Dev mode with auto-reload (use only when editing code and no triage is running). |
| `python -m app.main` | Runs the single-account **CLI** version: classifies/labels up to 200 unread emails and prints a summary. |

Then open **http://localhost:8000** in a browser and click **Connect Gmail**.

---

## 3. Using the dashboard (what each button does)

| Action | What happens |
| ------ | ------------ |
| **Connect Gmail** | Starts the Google OAuth login. The account must be added as a Test User in Google Cloud (while the app is unverified). |
| **Run Triage** (with date + range) | Fetches all inbox mail in the chosen window (Up to 1 day / Up to 1 week), classifies each email, and applies Gmail labels. No count limit. |
| **Stop** | Cancels an in-progress triage run. |
| **Categories** | Edit your category list and pick which category triggers auto-reply drafting. "Others" is always on as a catch-all. |
| **Search** | Finds already-processed emails by meaning (semantic search). Only works after at least one triage run. |
| **Schedule / Run on** | Schedules a one-time triage for a chosen date/time. Backed by APScheduler; you can view or cancel it. |
| **Label guide** | Lists every label the tool applies and how it decides where each email goes. |

---

## 4. Managing the server

| Command | What it does |
| ------- | ------------ |
| `Ctrl + C` (in the server terminal) | Stops the running server. |
| `Get-NetTCPConnection -LocalPort 8000 -State Listen -ErrorAction SilentlyContinue \| Select-Object -ExpandProperty OwningProcess -Unique \| ForEach-Object { Stop-Process -Id $_ -Force }` | Force-kills whatever process is holding port 8000 (use if the server is stuck or a previous instance is still running). |
| `try { Invoke-WebRequest 'http://localhost:8000/' -TimeoutSec 2 -UseBasicParsing \| Out-Null; 'up' } catch { 'down' }` | Quick check whether the server is running. |

> **Note:** because auto-reload is off, after changing any Python code you must
> stop the server (`Ctrl + C` or the kill command above) and start it again for
> the change to take effect. Frontend (`static/`) changes only need a browser
> hard-refresh (`Ctrl + F5`).

---

## 5. Files you must NOT commit or share publicly

These hold secrets and are git-ignored. Keep them out of any public repo:
`.env`, `credentials.json`, `web_credentials.json`, `token.json`, `app.db`.

---

## 6. Regression testing

| Check | Command | Expected result |
| ----- | ------- | --------------- |
| Python syntax | `.venv\Scripts\python.exe -m compileall -q app run_server.py` | Exit code 0, no output. |
| Automated regression | `.venv\Scripts\python.exe -m unittest discover -s tests -v` | All tests `OK`; uses temp DB + mocks. |
| Add-on manifest | `Get-Content gmail-addon\appsscript.json -Raw \| ConvertFrom-Json \| Out-Null` | No JSON error. |
| Local server | `Invoke-WebRequest http://127.0.0.1:8000/ -UseBasicParsing` | HTTP 200. |

The automated suite covers deterministic/LLM classification, temporary
`Others`, adaptive rules, manual corrections, remembered contacts, deadline
extraction/storage, reply priority, unread-alert filtering, summarization, and
the add-on manifest/function contract. It never calls the real inbox.

For the live smoke test, verify these non-destructive calls from the add-on:

1. Home card loads and reports `Ready`/`idle`.
2. Digest returns priority, only-still-unread alerts, deadlines, learned counts.
3. Opening a mail shows `Summarize this email`; clicking it returns bullets.
4. Do not test Undo or Run Triage on a production inbox unless label mutation is intended.

---

## 7. Gmail add-on setup

1. In Apps Script, paste `gmail-addon/Code.gs` and `gmail-addon/appsscript.json`.
2. Set Script Properties: `BACKEND_URL=<stable HTTPS backend>` and
	`ADDON_SECRET=<same value as ADDON_SHARED_SECRET>`.
3. Install a Gmail test deployment. Manifest changes require uninstall/install
	and re-authorization because the message-read scope is contextual.
4. Never use a `trycloudflare.com` quick tunnel for the final deployment; its URL
	changes after network/process restarts.

---

## 8. KT walkthrough (recommended order)

1. **Request flow:** Gmail/dashboard → `app.server` → authenticated user/token.
2. **Triage pipeline:** `app.main.triage_until_empty` → fetch → classify → label →
	priority/deadline/contact/embedding persistence.
3. **Classification order:** deterministic `rules.py` → active learned rule →
	batched Groq in `classifier.py`; `Others` remains temporary.
4. **Memory:** `db.py` stores learned sender/domain rules and known contacts per
	user. Three specific LLM hits activate a rule; manual correction is immediate.
5. **Safety:** drafts are never sent, every user is isolated by email, add-on APIs
	require a shared secret, and undo records applied category labels per run.
6. **Add-on:** `Code.gs` renders cards and calls `/api/addon/*`; the manifest adds
	homepage/contextual triggers and current-message read scope.
7. **Operations:** restart after Python edits, run regression tests, inspect logs,
	verify OAuth redirect URI, persistent DB volume, and HTTPS health.

---

## 9. Sunday deployment checklist

- [ ] Host FastAPI on a stable HTTPS service; use `run_server.py` or equivalent.
- [ ] Set `APP_ENV=production`, `HOST=0.0.0.0`, and run exactly one worker.
- [ ] Configure `GROQ_API_KEY`, strong `SESSION_SECRET`, `SESSION_HTTPS_ONLY=true`,
	  `ADDON_SHARED_SECRET`, and public HTTPS `OAUTH_REDIRECT_URI` as secrets.
- [ ] Add the public `/auth/callback` URI to the Google OAuth web client.
- [ ] Set `APP_DB_PATH` to a persistent volume; otherwise tokens/memory disappear.
- [ ] Restrict/backup the database because it stores OAuth tokens and mail metadata.
- [ ] Re-paste `gmail-addon/Code.gs`, update Apps Script `BACKEND_URL`, reinstall,
	  and authorize the test deployment.
- [ ] Run syntax, automated regression, local/public health, digest, and summarize checks.
- [ ] Confirm logs contain no secrets and restart/recovery behavior works.

Current pilot limitations: one-time schedules do not survive restart, runtime
progress/cancellation is process-local, and the add-on shared secret is not
cryptographically bound to a specific Google user. These require shared durable
job state and Google identity-token verification before public multi-tenant use.
