# Email Triage Assistant: Security, Data, and Deployment Overview

## 1. Security overview

### 1.1 Authentication and authorization

- Users sign in with Google using OAuth 2.0. The web login uses the PKCE
  extension, which protects the authorization code exchange even without a
  client secret being exposed to the browser.
- The one-time PKCE verifier is held in the server side session and used once
  during the callback. It is never sent to the browser.
- The only Gmail data scope requested is `gmail.modify`, the minimum needed to
  read messages and apply labels. The app does not request full account access
  or the ability to permanently delete mail. The web login additionally requests
  the standard OpenID Connect sign-in scopes (`openid` and the user's email
  address) so the app can confirm which account connected.
- The signed-in identity is taken from the verified OpenID Connect ID token, so
  the account that owns the session is confirmed by Google, not self-reported.

### 1.2 Session management

- The logged-in identity is stored in a signed session cookie.
- The signing key comes from the `SESSION_SECRET` environment variable. If it is
  not set, the app generates a random key at startup instead of using any fixed
  built-in value, so there is no predictable secret that could be used to forge
  a session.
- The session cookie is marked `SameSite=Lax`, which blocks it from being sent
  on cross site requests and mitigates cross site request forgery.
- In production the cookie is marked HTTPS only through the `SESSION_HTTPS_ONLY`
  setting, so it is never transmitted over plain HTTP.

### 1.3 Access control and isolation

- Every API route resolves the caller from the session and returns HTTP 401 if
  there is no valid session. Unauthenticated requests cannot reach any triage,
  settings, or search endpoint.
- All stored data is keyed by the connected Gmail address. One user can only
  ever see and act on their own mailbox, tokens, settings, and search index.

### 1.4 Input handling and injection safety

- All database access uses parameterized SQL queries, so user or mail content
  cannot be interpreted as SQL. There is no string concatenation of values into
  queries.
- Request bodies (category lists, dates, scheduled times) are validated on the
  server before use. Invalid input is rejected rather than assumed.
- Content rendered in the dashboard (mail subjects, senders, snippets) is HTML
  escaped before display, which prevents script injection through mail content.

### 1.5 Secrets and configuration

- All secrets live in environment variables or local credential files that are
  excluded from version control. This includes the Groq API key, the session
  secret, the OAuth client files (`credentials.json`, `web_credentials.json`),
  stored user tokens (`token.json`), the local database (`app.db`), and the
  `.env` file.
- A sample configuration file (`.env.example`) documents every setting without
  containing any real values.
- Internal errors are logged on the server but are not returned in full to the
  browser. The user sees a generic failure message instead of raw exception
  text.

### 1.6 Transport security

- For any real deployment the app runs behind HTTPS. Combined with the HTTPS
  only cookie flag, this keeps session cookies and mail content encrypted in
  transit.

### 1.7 Known limitations and hardening roadmap

These are understood and tracked, and are acceptable for the current pilot on a
single trusted machine. They should be closed before a public production launch.

- Stored OAuth tokens and cached mail metadata are currently held in plaintext
  in the local SQLite database. The database file is access controlled and kept
  out of version control, but for production it should be encrypted at rest,
  either through an encrypted volume or by encrypting the token column.
- Some runtime state (triage progress, the pending scheduled run, and the label
  lookup cache) is held in process memory. This is fine for a single server
  process. A multi process or restart tolerant deployment would move this into a
  shared store.

## 2. Data and database processes

### 2.1 Storage engine

The app uses a local SQLite database file named `app.db`. SQLite is sufficient
for the current scale (a few hundred emails per user) and keeps the deployment
simple. All access goes through a small set of helper functions, each of which
opens a connection, runs a parameterized statement, and closes the connection.

### 2.2 Tables and what they hold

The database has three tables, all keyed by the user's Gmail address so accounts
stay isolated.

1. `users`
   - Columns: `email` (primary key), `token_json`, `updated_at`.
   - Holds the OAuth token that lets the app call Gmail on the user's behalf.
   - Written when a user connects, updated when the token is refreshed.

2. `user_settings`
   - Columns: `email` (primary key), `categories_json`, `faq_category`,
     `updated_at`.
   - Holds the user's category list and which category (if any) triggers draft
     replies. The Low Priority category is always kept as a fixed catch all so
     that mail which does not clearly fit any other category has a home and the
     classifier never has to invent a new label.

3. `email_embeddings`
   - Columns: `user_email`, `gmail_id`, `thread_id`, `sender`, `subject`,
     `snippet`, `date`, `embedding`. Primary key is `user_email` plus
     `gmail_id`.
   - Holds per message metadata and a numeric embedding vector used for meaning
     based search.

### 2.3 How data moves through the system

- On connect: the OAuth exchange returns a token, which is saved into `users`.
- On saving settings: the category list and auto reply choice are written into
  `user_settings`. The fixed Low Priority category is enforced on both save and
  load, so it can never be removed by accident or through a direct API call.
- During a triage run: each processed message can be stored in
  `email_embeddings` so it becomes searchable. A hidden internal label named
  AI-Processed is applied in Gmail so the next run skips messages that were
  already handled and never processes the same mail twice.
- During classification and draft generation: the sender, subject, and body of
  each email are sent to the Groq API, a third-party language model service, over
  HTTPS, which returns the category and any draft reply text. This is the one
  point where mail content is processed outside our own server, so it must be
  disclosed in the privacy policy and to the security assessor.
- During search: the query is embedded locally and compared against the stored
  vectors, and the closest matches are returned. No mail content leaves the
  server for search.

### 2.4 Upsert behaviour and integrity

All writes are upserts (insert or update on conflict) keyed by the primary key.
This means reconnecting, re-saving settings, or reprocessing a message updates
the existing row rather than creating duplicates.

### 2.5 Retention and deletion

Because everything is keyed by email, a user's data can be fully removed by
deleting their rows from the three tables. For production this should be exposed
as an explicit account deletion action so that a user request to delete data can
be honoured cleanly, which is also a requirement of Google's user data policy.

## 3. Google verification and the CASA security assessment

### 3.1 Why this applies to us

The `gmail.modify` scope is classified by Google as a restricted scope, because
it gives broad access to a user's mail. Any app that requests a restricted scope
and stores or transmits that data through a server must complete Google's
verification process, which includes an annual independent security assessment
under the CASA framework (Cloud Application Security Assessment, run through the
App Defense Alliance). CASA is built on the OWASP Application Security
Verification Standard and is risk tiered, so the depth of the assessment depends
on the app's risk profile.

### 3.2 Current status

The project is currently in Testing mode with a limited list of test users. In
this mode Google does not require full verification or a security assessment, but
there is a cap on the number of users, testers see an unverified app warning, and
refresh tokens have a limited lifetime. This is the correct and cost free mode
for the pilot and demos. Verification and CASA are only needed when we decide to
open the app to the general public.

### 3.3 The overall sequence to go public

There are three stages, done in order. The security assessment is the final one
and only starts after the earlier verification work is complete.

1. Brand verification. Confirm the OAuth consent screen branding (app name,
   logo, support email, home page, privacy policy). This automated step
   typically completes in about two to three business days.
2. Restricted scope verification. Provide justification for the scope, a public
   home page, a privacy policy hosted on the same domain, and a demonstration
   video showing the consent flow and how the scope is used. This review can take
   several weeks.
3. CASA security assessment. An independent, Google approved assessor validates
   the application against the CASA requirements and issues a letter confirming
   the result. Google's Trust and Safety team contacts you when it is time to
   begin this stage. The letter must be renewed at least every twelve months to
   keep access to the restricted scope.

### 3.4 CASA assurance levels

CASA uses a risk based model with two assurance levels, AL1 and AL2. Google (not
the developer) decides which level applies, based on the sensitivity of the data,
the number of users, and other risk signals. Both levels require the same set of
security controls to be met. The difference is the depth of testing an authorized
lab performs. AL2 is the more thorough assessment and covers the application, the
deployment infrastructure, and any place user data is stored. Once an app has
been validated at AL2, it stays at AL2 in following years.

### 3.5 The CASA process, step by step

This is the practical sequence to actually complete the assessment.

1. Confirmation of trigger. Because the app uses the `gmail.modify` restricted
   scope and stores that data on a server, the assessment is required. In
   practice, Google's Trust and Safety team notifies you to begin it after brand
   and restricted scope verification are done.
2. Internal gap analysis. Before engaging an assessor, review the app against the
   CASA requirements, which are based on the OWASP Application Security
   Verification Standard. Go through each control area (see section 3.8) and note
   what is already met and what needs work. Fixing gaps early lowers cost and
   avoids a failed first assessment.
3. Collect existing certifications. If RootStock already holds recognised
   certifications (for example ISO 27001 or SOC 2), gather them. These can be
   used through the CASA Accelerator to reduce the number of checks (see section
   3.6).
4. Choose an authorized assessor. Pick a lab from the App Defense Alliance list
   of authorized assessors and request quotes (see section 3.7).
5. Scoping with the assessor. Share the application architecture, the data flow,
   the list of endpoints, the hosting and infrastructure details, and where user
   data is stored. The assessor uses this to set the exact test scope and
   confirm the assurance level.
6. Assessment and testing. The assessor runs automated scans and, depending on
   the assurance level, manual testing against the app, its infrastructure, and
   its data storage. They check the controls in section 3.8.
7. Remediation. The assessor returns a list of findings. Fix each one and provide
   evidence (configuration, screenshots, code references, or re-scan results).
   This step can repeat until all findings are cleared.
8. Letter issued. When the app passes, the assessor issues a letter confirming
   the result. Google refers to this as the Letter of Validation, and some Google
   pages call it the Letter of Assessment. It records that the app meets the CASA
   requirements at the assigned assurance level.
9. Submission to Google. The result is associated with the Google verification.
   Once Google has the passing letter and the other verification steps are
   complete, it grants production access to the restricted scope, and the
   unverified app warning is removed for all users.
10. Annual revalidation. The whole assessment repeats at least once every twelve
    months to keep access (see section 3.12).

### 3.6 Using the Accelerator to reduce work

The CASA Accelerator lets you map existing industry certifications to CASA
requirements, so any control already proven by a certification does not need to
be tested again. To use it, share the certifications with the chosen authorized
assessor. The assessor validates them and then customises the remaining checks.
If RootStock holds ISO 27001, SOC 2, or similar, this can meaningfully cut both
the time and the cost of the assessment.

### 3.7 Choosing an assessor

- Use only assessors listed as authorized by the App Defense Alliance.
- Request written quotes from two or three of them so the cost and timeline can
  be compared.
- Confirm what the quote includes: the assurance level, whether manual testing is
  part of it, how many rounds of remediation review are covered, and whether the
  annual renewal is discounted.
- Share the same scoping information with each so the quotes are comparable.

### 3.8 CASA requirements and the evidence to prepare

The requirements follow the OWASP Application Security Verification Standard. The
main control areas, and where the app currently stands, are below. This doubles
as the checklist to prepare evidence for the assessor.

- Authentication and identity. Google OAuth with PKCE and a verified ID token.
  Status: in place.
- Session management. Signed cookies, no fixed built-in secret, SameSite Lax,
  HTTPS only in production. Status: in place.
- Access control. Every API route requires a valid session and data is isolated
  per user. Status: in place.
- Input validation. Server side validation of request bodies and parameterized
  SQL. Status: in place.
- Output handling. HTML escaping of mail content in the dashboard. Status: in
  place.
- Communications security. TLS or HTTPS for all traffic in production. Status: to
  be confirmed at the production host.
- Data protection at rest. Encryption of stored tokens and mail metadata. Status:
  to be implemented before assessment.
- Data deletion. An explicit action to delete a user's data on request. Status:
  to be implemented before assessment.
- Secret management. Secrets kept in environment variables and untracked files.
  Status: in place.
- Logging and error handling. Server side logging, generic error messages to the
  user. Status: in place.
- Third-party data processing. Email content is sent to the Groq API for
  classification and draft generation. Status: in use; for production this must
  be disclosed in the privacy policy and covered by a data processing agreement
  with the provider.
- Dependency and configuration management. Keep dependencies updated and remove
  unused OAuth clients before submitting. Status: routine, to maintain.

The two items marked as to be implemented (encryption at rest and data deletion)
are the priority engineering tasks before starting the assessment.

### 3.9 Indicative time

- Brand verification: about 2 to 3 business days.
- Restricted scope verification: commonly several weeks, depending on how quickly
  review questions are answered.
- CASA assessment: typically a few weeks of assessor engagement, plus time to fix
  anything they flag.
- Realistic end to end estimate for a first time submission: plan for roughly
  six to twelve weeks from a complete submission to full approval, and treat it
  as recurring every year.

### 3.10 Indicative cost

Google and the App Defense Alliance do not publish a fixed price. The assessment
fee is set by the authorized assessor that we choose, and it depends on the
assurance level and how much manual review is required. The figures below are
indicative planning numbers only and should be confirmed with written quotes.

| Item | Indicative annual cost (USD) | Notes |
| --- | --- | --- |
| AL1, lighter assessment through an authorized assessor | Roughly a few hundred to about 2,000 | Automated scan plus assessor validation |
| AL2, deeper assessment with manual review or penetration testing | Roughly 3,000 to 8,000 or more | Depends on scope and complexity |
| Complex applications or higher risk profile | Can reach five figures | Larger surface, deeper review |

Because this recurs every twelve months, it should be treated as an ongoing
operating cost, not a one time fee. Using the Accelerator with existing
certifications can lower these figures.

### 3.11 What we must have ready before submitting

- A public, working home page that clearly describes the app.
- A privacy policy hosted on the same domain, describing exactly how Gmail data
  is accessed, used, stored, and deleted, including any third-party processing
  such as the Groq language model service, and complying with Google's Limited
  Use rules.
- Ownership of the domain verified in Google Search Console.
- A demonstration video of the consent and usage flow.
- Encryption at rest for stored tokens and mail metadata, and an account deletion
  path, both of which the assessor will expect to see.

### 3.12 Annual revalidation

Access to the restricted scope is not permanent. The app must be reassessed at
least every twelve months from the date the assessor's letter is approved. The
required assurance level can rise if the user base or data handling grows, and
once the app reaches AL2 it stays there. Google emails a reminder when renewal is
due, so more than one RootStock account should be listed as Owner or Editor on
the project, and the support and developer contact emails should be kept current,
so the reminder is not missed.

## 4. Migrating the Google Cloud project to RootStock Technology

### 4.1 Current state

The Google Cloud project that holds the OAuth client and consent screen is
currently under a personal Google account. Verification, branding, the privacy
policy domain, and billing are all tied to whoever owns that project.

### 4.2 Why the project needs to move

For a public launch the app must be verified under RootStock's identity. The
consent screen that users see, the domain that hosts the home page and privacy
policy, and the legal entity behind the CASA assessment should all be RootStock,
not an individual. Ownership, billing, and the annual renewal responsibility
should also sit with RootStock. For these reasons the project should live in a
Google Cloud Organization that RootStock controls before verification begins.

### 4.3 Two ways to do the handover

Option A: Transfer the existing project.
Add a RootStock Google account as an Owner on the current project, move the
project into RootStock's Google Cloud Organization, then remove the personal
account once the handover is confirmed. This keeps the existing project history
and settings.

Option B: Recreate the project under RootStock.
RootStock creates a fresh project inside its own Organization, and the OAuth
client, consent screen, and enabled APIs are set up there. The application is
then pointed at the new client credentials. This is the cleanest option when the
public branding and domain are also new, and it avoids carrying over any personal
account settings.

For a product that is going to be verified and launched under RootStock's brand,
Option B (a fresh project owned by RootStock from the start) is usually the
simplest to reason about, with Option A as a fallback if we want to preserve the
current project.

### 4.4 What changes in the application after migration

- A new OAuth client is issued, which means a new client ID and client secret.
  The web client file (`web_credentials.json`) in the app must be replaced with
  the new one.
- The authorized redirect URI must be set to RootStock's production callback URL,
  and the app's `OAUTH_REDIRECT_URI` setting updated to match.
- The consent screen branding (app name, logo, support email, home page, privacy
  policy) is configured under the RootStock project.
- Authorized domains are verified under a RootStock owned Google account in
  Search Console.
- Because the client ID changes, existing users will re-consent the first time
  they connect to the new client. This is expected and is a one time step for
  each user.

### 4.5 Handover checklist

- RootStock creates or receives ownership of the Google Cloud project inside its
  Organization.
- Billing is attached to a RootStock billing account.
- The OAuth consent screen is configured with RootStock branding and contacts.
- The production domain, home page, and privacy policy are live and owned by
  RootStock.
- New OAuth client credentials are generated and placed into the app's
  configuration, and the redirect URI is updated.
- At least two RootStock accounts are added as Owner or Editor so verification
  and annual renewal notices reach the right people.
- Only then is the app submitted for brand verification, restricted scope
  verification, and the CASA assessment.
