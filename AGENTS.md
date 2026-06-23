# Outreach Tracker - Codex Project Memory

Static single-file SPA for tracking SDR outreach: companies contacted, contacts, email drafts, reply/contacted detection, warm-up archive automation, and OAuth-based mailbox connections.

## Canonical Files

- Edit source first: `/Users/moderninbound/Master Projects Claude/outreach-tracker.html`
- Sync to deployed repo: `/Users/moderninbound/Master Projects Claude/outreach-deploy/index.html`
- Deploy repo: `/Users/moderninbound/Master Projects Claude/outreach-deploy/`
- GitHub repo: `Adithya271755/outreach-tracker`, branch `main`
- Existing Claude memory: `/Users/moderninbound/Master Projects Claude/outreach-deploy/CLAUDE.md`
- Leave `font-picker.html` and `.DS_Store` alone unless explicitly asked.

Sync command after source edits:

```sh
cp "/Users/moderninbound/Master Projects Claude/outreach-tracker.html" "/Users/moderninbound/Master Projects Claude/outreach-deploy/index.html"
```

Do not push to GitHub speculatively. Ask/confirm before pushing.

## Local Preview

- Existing Claude launch config: `/Users/moderninbound/.claude/launch.json`
- Standard deploy preview: `python3 -m http.server 7891 --directory "/Users/moderninbound/Master Projects Claude/outreach-deploy"`
- Recent dev/baseline pair:
  - Dev: `http://localhost:7896`, directory `/Users/moderninbound/Master Projects Claude/outreach-tracker-dev`
  - Baseline: `http://localhost:7895`, directory `/Users/moderninbound/Master Projects Claude/outreach-tracker-baseline`

For browser automation, follow the global rule: kill stale Chrome processes and clear stale profiles before starting. If the task changes UI, finish the chat reply with clickable localhost links.

## Data Model

- Global state: `D = { t: 50, d: {} }`
- `t` is the daily target. `d` maps `YYYY-MM-DD` to `{ companies: [...] }`.
- Company shape: `{ id, domain, contacts: [{ email, role, draft }] }`
- Campaign start: `CAMPAIGN_START = new Date('2026-06-01T00:00:00')`
- Owner email: `adithya@moderninbound.com`
- Client-side login gate reads `localStorage.getItem('ot_user_email')`.
- Storage:
  - owner data key: `ot_v4`
  - other user key: `ot_v4_u_` + sanitized email
  - backup/timestamp/session variants: `_bak`, `_ts`, `_sess`
  - connected mailbox accounts: `ot_email_accounts`
  - compose draft: `ot_compose_draft`
  - session share IDs/tokens: `ot_session_id`, `ot_owner_token`

## APIs And Integrations

Read current official docs before modifying any API/OAuth integration.

- Supabase:
  - Script: `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js`
  - URL in app: `https://kndxvhtjtawmfbkfhakf.supabase.co`
  - Publishable key is embedded in the static app.
  - Tables referenced: `user_data`, `sessions`, `user_profiles`
  - Used for user data sync, session sharing, profile list/update/delete.
- Google Identity/Gmail:
  - Script: `https://accounts.google.com/gsi/client`
  - Client ID: `551201525725-c2pjulu9ta3kqpmhmvut4192ik43gg5r.apps.googleusercontent.com`
  - OAuth token client: `google.accounts.oauth2.initTokenClient`
  - Scopes: `gmail.readonly`, `gmail.send`, `userinfo.email`
  - Endpoints: Gmail messages list/metadata and OAuth userinfo.
- Microsoft OAuth/Graph:
  - MSAL script: `https://cdn.jsdelivr.net/npm/@azure/msal-browser@3.28.1/lib/msal-browser.min.js`
  - Client ID: `4b8af517-792a-43a9-84a2-96713f25b66d`
  - Authority: `https://login.microsoftonline.com/common`
  - Cache: `localStorage`
  - Scopes: `Mail.ReadWrite`, `Mail.Send`, `User.Read`
  - Endpoints: Graph `/me/mailFolders/.../messages`, `/me/messages/{id}/move`, draft update/delete/send helpers.
- Google utility endpoints:
  - Favicons: `https://www.google.com/s2/favicons`
  - Translate link: `https://translate.google.com/`
- Paused Find Companies API scaffold:
  - Folder: `/Users/moderninbound/Master Projects Claude/find-companies/`
  - Gemini model: `gemini-2.5-flash`
  - Endpoint: `generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
  - `.env` currently has empty `GEMINI_API_KEY=`
  - Planned search API: Google Custom Search API
  - Google Cloud project noted in Claude memory: `Outreach Tracker`, ID `outreach-tracker-498918`

Security note: this is still a static client app with OAuth tokens and Supabase publishable key visible client-side. Before onboarding real users, review Supabase RLS, OAuth app verification/redirect origins, token storage risk, and whether mailbox operations need a backend.

## Current Feature Notes

- Find Companies is paused. Do not resume until Adithya explicitly says something like "let's get back to building the Find Companies thing."
- Frontend stub exists: `#sbFindCompanies`, `#page-find-companies`, `find-companies` in `PAGE_IDS`.
- Hover-flicker fix shipped in prior Claude work: `renderCompanyList()` uses live `:hover` checks instead of mutable hover flags.
- Warm-up archive is Microsoft-focused and detects Reach Inbox watermark text including `Aggressive-Chamois`.
- Contacted detection supports CC-copy workflows: Gmail inbox scans add visible `To`/`Cc` recipients to the contacted set, while excluding connected/login mailbox addresses. It also falls back to Gmail search for tracked contact emails, because forwarded/tracking copies can make Gmail search find a lead even when the lead is not exposed in message headers. The scan badge can be clicked to force a fresh mailbox scan, and `window.otDebugMailScan(email)` reports whether a specific email is currently in the contacted/replied sets.
- Do not infer `contacted` from the day being in the past. A June 23, 2026 bug showed every June 22 company as contacted because `renderCompanyList()` set `coStatus = 'contacted'` for `curKey < today`. Contacted/replied UI must come only from mailbox evidence. If a mailbox scan fails, clear mailbox-derived chips instead of showing partial/stale matches.

## Claude/Codex Handoff Log

Decisions one agent made that the other later overruled — read this before touching contacted/scan logic.

### priorityScanVisible() — do NOT remove (Claude overruled Codex, commit de04ee3, 2026-06-23)

Codex removed the `priorityScanVisible()` call from `scanMailboxesIfNeeded()` as part of the false-positive fix (commit 822fc2d). Claude restored it one session later.

- `priorityScanVisible()` fires parallel Gmail searches for only the contacts visible on the current date. Results come back in ~500ms. Without it, the full scan harvests up to 500 sent + 500 inbox messages sequentially — takes 4-5 minutes before any chips appear.
- The false-positive bug (all past-day contacts showing contacted) was caused by the `isPastDay` heuristic in `renderCompanyList()`, NOT by `priorityScanVisible()`. Those are independent.
- Rule going forward: `priorityScanVisible()` must always be called concurrently at the start of `scanMailboxesIfNeeded()`. If you remove it for any reason, document why here first.

### isPastDay heuristic — permanently banned (Claude introduced, Codex correctly removed, 2026-06-23)

Claude added `const isPastDay = curKey < dateKey(todayIdx())` and used it to auto-show `contacted` for all past-day companies. Codex correctly removed it. Do not re-add any date-based inference for contacted status. Contacted/replied must come only from `_mailScan.contacted` / `_mailScan.replied`. Reason: users log companies before sending — past day in tracker does not mean email was sent.

### Outcome dot on contact rows — removed by user request (Claude added, user rejected, 2026-06-20)

Claude added a clickable outcome chip (`·` / `S` / `I` / etc.) to each contact row in `renderCompanyList()` as a manual "mark sent" workaround. User did not want the dot UI. Removed same session. The `OUTCOME_RING` / `OUTCOME_META` / `renderOutcomeChip()` scaffolding still exists in the codebase but is not rendered in contact rows. Do not re-add UI to contact rows without user asking.

## Working Rules

- Keep changes scoped. This is a large single-file app, so use `rg` to find anchors and read wide relevant ranges before editing.
- For UI changes, seed local storage for testing with `ot_user_email = adithya@moderninbound.com` and a representative `ot_v4` data object.
- No em dashes in user-facing in-app copy. Use a period, comma, or ` · `.
- Log meaningful project-specific lessons back into this file after implementation. Do not promote lessons to global `AGENTS.md` unless Adithya asks for a consolidation pass.
- Routine iterations should be direct, not multi-agent fan-outs. Use council/consensus only when the user explicitly asks or the decision is genuinely high-variance.

## Reply Style For Adithya

Keep project replies terse: one-line shipped/fixed headline, 2-3 bullets if needed, then localhost links when UI changed.
