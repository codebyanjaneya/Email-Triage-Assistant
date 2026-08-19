# Email Triage Assistant

An AI assistant that keeps a busy Gmail inbox under control. It reads your unread
email, sorts each message into a clear category, labels it in Gmail, pulls out
deadlines, points you to the mail that actually needs your attention, and can
draft (but never send) replies. It runs both as a Gmail add-on that lives inside
your inbox and as a multi-user web dashboard.

The goal is simple. People lose a large part of their day just deciding which
emails matter. This tool makes that decision for them and gets better the more
they use it.

## What it does

- **Sorts every email into a category.** Each message is classified as Needs
  Action, Red Flag, FAQ, Spam/Newsletter, or Others. Fast keyword and domain
  rules run first, and anything they do not catch is sent to a language model
  (Groq's `llama-3.3-70b-versatile`) in batches. If the model is unavailable, the
  message falls back to a safe category instead of being dropped.
- **Learns from you.** When you move a message to a different category, that
  correction is remembered immediately, along with the sender and the words in
  the subject. Similar mail from then on is handled the same way without asking
  the model again. Repeated model decisions also turn into gentle guidance after
  a few consistent observations. Corrections stay specific, so fixing one subject
  does not blindly reroute everything from that sender.
- **Shows you what needs attention.** Unread Red Flag and Needs Action mail is
  surfaced in a short list, ranked by category, how recent it is, whether it is a
  reply in an ongoing thread, and whether you know the sender. Once you open a
  message it drops off the list.
- **Finds deadlines on its own.** Due dates are pulled straight from the email
  text, including plain dates, formatted dates, and relative ones like tomorrow.
  Only real future deadlines with a clear cue are kept. Promotional and spam mail
  never produce a deadline.
- **Summarizes long email in one click.** Open a message and the add-on turns it
  into a few factual bullet points, keeping the requests, dates, and amounts.
- **Drafts replies without sending them.** For a category you choose (FAQ by
  default), the app writes a short, polite reply grounded in your own templates
  and saves it as a Gmail draft. It never calls send. A person always reviews and
  sends the message.
- **Protects real conversations from spam.** Anyone you have replied to is
  remembered as a known contact and is never left in a spam bucket.
- **Runs on a schedule.** Turn on auto-sort and triage happens quietly in the
  background at the interval you pick, so the inbox stays organized on its own.
- **Searches by meaning.** Every processed email is embedded locally, so you can
  search your mail by what it is about rather than by exact words. No external
  vector database is needed.
- **Works inside Gmail.** The Apps Script add-on gives you date and range
  controls, alerts, deadlines, one-click summaries, category corrections, undo,
  and a link back to the full web dashboard, on both web and mobile Gmail.

## How it is built

Two entry points share one pipeline. The web app (FastAPI) is multi-user. Each
account signs in with Google using OAuth with PKCE, and its token is stored
separately in a local SQLite database, keyed by the email from the verified
OpenID Connect token. The command line version uses the classic installed-app
flow for a single account.

The core pipeline fetches mail that has not been processed yet, classifies it
with deterministic rules, then learned memory, then the language model, applies
the category label plus a hidden processing label so the same mail is never
handled twice, optionally drafts a reply, scores priority, extracts deadlines,
remembers conversation contacts, and stores a local embedding. A background
scheduler runs recurring and one-time triage.

```
app/
  config.py        Environment and configuration
  gmail_client.py  OAuth and Gmail access, applies the AI-Processed label
  web_auth.py      Multi-user web OAuth (PKCE), one token per account
  db.py            SQLite: tokens, settings, learned rules, contacts,
                   priority, deadlines, embeddings, schedules
  classifier.py    Rules first, learned memory next, LLM last
  rules.py         High-confidence keyword and domain rules
  auto_reply.py    Draft replies saved to Gmail Drafts only
  priority.py      Deterministic attention scoring with reasons
  deadlines.py     Deadline extraction with an LLM fallback for relative dates
  summarize.py     Bullet summaries for the add-on
  search.py        Local embeddings and cosine-similarity search
  main.py          Triage pipeline and CLI entry point
  server.py        FastAPI app, routes, sessions, scheduler
static/            Web dashboard
gmail-addon/       Apps Script Gmail add-on (Code.gs and manifest)
tests/             Isolated regression suite
```

## Tech stack

- Python, FastAPI, and Uvicorn for the backend
- APScheduler for recurring and one-time triage
- Groq for classification, summaries, and relative-date resolution
- SQLite for per-user storage
- fastembed for local semantic search
- Google Apps Script for the Gmail add-on

## Getting started

### 1. Install

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

The first semantic search downloads a small embedding model into a local cache.

### 2. Configure

Copy the example environment file and fill it in.

```powershell
Copy-Item .env.example .env
```

| Variable | Purpose |
| --- | --- |
| `GROQ_API_KEY` | Key used for classification, summaries, and reply drafting |
| `APP_ENV` | Set to `production` on a hosted service to enable strict checks |
| `SESSION_SECRET` | Signs the web session cookie; use a long random value |
| `SESSION_HTTPS_ONLY` | Set true in production so cookies are HTTPS only |
| `OAUTH_REDIRECT_URI` | OAuth callback URL; the public HTTPS callback in production |
| `ADDON_SHARED_SECRET` | Shared secret required by every add-on API call |
| `ADDON_GOOGLE_AUDIENCE` | OAuth client ID used to verify signed add-on identity tokens |
| `APP_DB_PATH` | SQLite path; put it on persistent storage in production |
| `HOST` / `PORT` | Bind address and port |
| `REMINDER_EMAILS_ENABLED` | Set true to send periodic self-reminder emails |

You also need two Google OAuth clients from the Google Cloud console. A Desktop
client saved as `credentials.json` for the command line flow, and a Web client
saved as `web_credentials.json` for the dashboard. The only Gmail data scope
requested is `gmail.modify`, which allows reading mail and applying labels and
drafts. It cannot permanently delete mail and it never sends.

### 3. Run

```powershell
.venv\Scripts\python.exe run_server.py
```

Open http://localhost:8000, connect your Gmail account, and run a triage. For the
single-account command line version, run `python -m app.main`.

## Testing

```powershell
.venv\Scripts\python.exe -m unittest discover -s tests -v
```

The suite uses a temporary database and mocked Gmail and Groq clients, so it
never touches a real inbox. It covers classification order, learning and manual
corrections, public-domain protection, deadline extraction, reply priority,
conversation memory, reminders, OAuth handling, identity binding, schedule
persistence, and the add-on API contract.

## Security and privacy

- Sign-in uses Google OAuth with PKCE, and the account is confirmed by Google
  through the OpenID Connect token rather than being self-reported.
- Only the `gmail.modify` scope is requested. The app never deletes mail and
  never sends on your behalf. Replies are always saved as drafts for review.
- All data is isolated per user and keyed by the verified email.
- Database access uses parameterized queries, and mail content shown in the
  dashboard is HTML escaped.
- Secrets live in environment variables and untracked credential files.
- The only point where mail content leaves the server is the language model
  provider, reached over HTTPS for classification and summaries. For strict data
  residency this can be replaced with a self-hosted model.
- The backend can run on the client's own infrastructure, so their email data
  and database stay within their control.

## The unverified app screen

While the OAuth app is still in testing, Google shows a warning during sign-in
because the app is not yet verified. This is expected for a self-hosted project.
Choose Advanced and continue. Testing mode allows a limited number of listed test
users. Lifting the warning for the general public requires Google verification
and the associated security review, which is only needed for a public launch.

## Deployment notes

- Run a single backend worker. Progress and run locks are process-local.
- Put `APP_DB_PATH` on durable storage. It holds tokens, learned memory,
  deadlines, priority, and settings.
- Use a stable HTTPS backend for real use. Temporary tunnels change their
  hostname and are only suitable for quick testing.
