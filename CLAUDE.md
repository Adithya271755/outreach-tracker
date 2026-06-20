# Outreach Tracker — Project Memory

A static, single-file SPA (`index.html`, ~223KB) for tracking daily SDR
outreach: companies contacted, contacts, email drafts, warm-up automation,
and OAuth-based email account connections. No backend — all data persists
client-side via `localStorage`.

## File layout & sync workflow

- **Source/dev copy**: `/Users/moderninbound/Master Projects Claude/outreach-tracker.html` — edit here.
- **Deployed copy**: `/Users/moderninbound/Master Projects Claude/outreach-deploy/index.html` — this repo,
  pushed to GitHub (`Adithya271755/outreach-tracker`, branch `main`).
- After editing the source, sync with:
  `cp "/Users/moderninbound/Master Projects Claude/outreach-tracker.html" "/Users/moderninbound/Master Projects Claude/outreach-deploy/index.html"`
- Local dev server: `python3 -m http.server 7891 --directory "/Users/moderninbound/Master Projects Claude/outreach-deploy"`
  (config lives in `/Users/moderninbound/.claude/launch.json`, named `outreach-tracker`).
- `font-picker.html` and `.DS_Store` in this repo are pre-existing/unrelated — leave alone.

## Data model

- Global state: `D = {t: 50, d: {}}` — `t` is the daily target (50), `d` is a
  map of `dateKey -> { companies: [...] }`.
- Each company: `{id, domain, contacts: [{email, role, draft}]}`.
- `CAMPAIGN_START = new Date('2026-06-01T00:00:00')`. `dateKey(i)` returns
  `YYYY-MM-DD` (zero-padded) for day index `i` relative to `CAMPAIGN_START`.
  `todayIdx()` computes the current day index.
- Storage keys: `emailStorageKey(email)` → `'ot_v4'` for `OWNER_EMAIL`
  (`adithya@moderninbound.com`), else `'ot_v4_u_' + sanitized(email)`.
  Persisted to `localStorage['ot_v4']`, `'_bak'`, `'_ts'`, and
  `sessionStorage['_sess']`.
- Login gate is purely client-side: checks `localStorage.getItem('ot_user_email')`.
  If `email === OWNER_EMAIL`, login auto-completes as "Adithya".

## Recent work

### Hover-flicker fix (shipped — commit `5e0a04c`)
Hovering just below a company name (to preview its contacts panel) used to
open/close/open/close rapidly. Root cause: a shared mutable flag
(`item._hoveringNow`), toggled independently by `mouseenter`/`mouseleave` on
two elements (`name` and `panel`), got desynced by phantom mouse events that
the browser fires when the panel's CSS expand transition shifts layout under
a stationary cursor.

**Fix**: replaced the flag with a live `:hover` check evaluated at decision
time inside the existing 300ms-open / 150ms-close timeout callbacks:
`isHoveringZone = () => name.matches(':hover') || panel.matches(':hover')`.
Located in `renderCompanyList()`, around line ~2098.

**Lesson**: for hover-driven open/close UI with CSS transitions, don't trust
boolean flags set by enter/leave handlers — check `:hover` live at the
moment you're about to act.

### Find Companies feature — STATUS: PAUSED
User explicitly paused this to focus on bug fixes first. **Do not resume
until told** ("let's get back to building the Find Companies thing").

**Concept** (reverse search / ICP-driven lead discovery):
1. Input own company URL (e.g. moderninbound.com)
2. Scrape homepage, pricing, case studies, customers, testimonials pages
3. Derive ICP via LLM (B2B/B2C, target industries, company size, pricing signal, etc.)
4. Combine ICP with buying-signal criteria (e.g. "B2B, recently VC-funded,
   actively hiring SDRs in recruitment" → expanding sales team → needs more
   leads → good fit for Modern Inbound)
5. Reverse-search (Google Custom Search API — chosen) for ~25 candidate companies
6. Surface results in a "Find Companies" dashboard page

**Build approach chosen**: validate Stage 3 (ICP extraction via Gemini) first
before building the rest of the pipeline or frontend.

**Frontend stub already in place** (in `index.html`): sidebar nav item
`#sbFindCompanies` ("Find Companies"), page `#page-find-companies` with a
"Coming soon." placeholder, and `find-companies` added to `PAGE_IDS` in the
page-navigation IIFE.

**Backend scaffolding** lives in a separate, NOT-yet-git project at
`/Users/moderninbound/Master Projects Claude/find-companies/`:
- `scraper.py` — scrapes homepage + auto-detects pricing/case_studies/customers/testimonials
  subpages via link-keyword matching (`PAGE_KEYWORDS`). Cleans text, caps at
  `MAX_CHARS_PER_PAGE=4000`. `scrape_site(url) -> dict[str,str]`.
- `icp_extractor.py` — sends scraped pages to Gemini 2.5 Flash
  (`generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`)
  with a structured `ICP_SCHEMA` (business_model, target_industries,
  target_company_size, pricing_signal, icp_summary, evidence, etc.) via
  `responseSchema`. `extract_icp(pages) -> dict`.
- `.env` — `GEMINI_API_KEY=` is currently **empty**. Blocked: in Google Cloud
  Console project "Outreach Tracker" (ID `outreach-tracker-498918`), the
  "Gemini API" (formerly "Generative Language API") shows as enabled but
  stays greyed out in the API-key restrictions dropdown, with a banner
  saying "an administrator must verify this account." Next step when
  resuming: try "Verify account" flow (likely needs phone/MFA, similar to
  the parked Azure `Mail.ReadWrite` item).
- `venv/` (Python 3.14.3, has `requests`, `beautifulsoup4`, `python-dotenv`).

**Multi-user billing plan** (see global memory
`project_multiuser_launch_checklist.md`): for now, single-user testing on
Adithya's personal Google Cloud Console key. Revisit billing/rate limits
once paying users use this feature (~₹1,200-3,500/month for 5-6 users on
Gemini 2.5 Flash; bottleneck likely the search API, not Gemini).

## Working conventions / lessons learned

- When testing UI changes via Claude Preview, seed `localStorage` with test
  data via `preview_eval` (set `ot_user_email` to `adithya@moderninbound.com`
  to bypass login, then write `ot_v4`/`ot_v4_bak` with a `D` object).
- Always confirm fixes with the user before pushing to GitHub — don't push
  speculatively.
- Push workflow: commit in `outreach-deploy`, `git push origin main`.
- **No em dashes in user-facing in-app copy** (`—`). Em dashes are a
  giveaway that copy was AI-written and Adithya doesn't want the app to
  read that way. Use a period + new sentence, a comma, or a separator dot
  (` · `) instead. The middle-dot is fine because the app already uses it
  for pace/verdict strings (e.g. `38 to go · finish by 6:28 PM`). This
  rule applies to in-app strings (verdicts, empty states, button labels,
  chip text); chat replies are not affected.
- **After ANY round of UI changes, always end the chat reply with the
  updated dev localhost link** so Adithya can click straight into it.
  Standard pair: dev at http://localhost:7896 (current changes), baseline
  at http://localhost:7895 (frozen revert point). Format as clickable
  markdown links, not bare URLs. Do this even when the change was small
  (one CSS tweak, one string fix) — the click should never require him to
  remember which port maps to which copy.

## Efficiency lessons (mistakes / wasted paths — don't repeat)

After finishing a task, ask "how could this have been done faster, with
fewer tokens/tool calls?" and log the answer here. Keep entries short and
concrete — what happened, what to do instead.

- **`preview_start` / `preview_screenshot` / `preview_eval` always need IDs
  up front**: `preview_start` requires a `name` param, and
  `preview_screenshot`/`preview_eval` require the `serverId` returned by
  `preview_start`. Don't call these without them — it costs a failed
  round-trip every time. Always pass `name` on first call, and capture/reuse
  `serverId` for all follow-ups.
- **Check what's already on a port before starting a preview server**: when
  `preview_start` reports the port is in use by a non-preview process, it's
  often the user's own existing dev server for the same project (e.g.
  `python3 -m http.server` per `~/.claude/launch.json`). Quickly confirm with
  `ps -p <pid> -o command` before killing — but if it matches the expected
  command for this project, it's safe to kill and restart under the preview
  tool without a lot of back-and-forth.
- **Gemini API naming**: in Google Cloud Console, search the API Library for
  "**Gemini API**" directly — "Generative Language API" is the old name and
  won't surface it. If a freshly-enabled API stays greyed out in the API-key
  restrictions dropdown with an "administrator must verify this account"
  banner, that's an account-verification gate (likely needs phone/MFA), not
  a propagation-delay issue — go straight to the "Verify account" flow
  instead of retrying enable/refresh cycles.
- **Don't assume a path is stable mid-session**: if the user reorganizes
  their filesystem while you're working (e.g. moving project folders into
  `~/Master Projects Claude/`), a path that worked earlier in the session
  can silently 404 later. If a Read/Edit on a path you just wrote to fails,
  `find ~ -iname <filename>` before troubleshooting further — don't assume
  the write failed.
- **Don't pass `model: "sonnet" / "opus"` to the bare `Agent` tool with
  `subagent_type: "general-purpose"` — it gets rejected**. To get
  multi-model fan-out, use a skill that handles model routing internally
  (`multi-agent-consensus`, `council-of-claude`), or spawn plain
  general-purpose agents and accept the default. Never write 4 long agent
  prompts before confirming the tool actually accepts the params.
- **When delegating to a QA / verification agent, hand it the diff + exact
  line ranges of the change, not "read the file and verify"**. The 223KB
  outreach-tracker `index.html` cost a QA agent ~11 minutes and 41 tool
  calls because it had to discover where the edits were. Always include:
  (1) the file path, (2) line ranges of each edit, (3) the diff itself or
  the specific function names to inspect, (4) the precise behavioral claims
  to verify. Cuts QA time by ~5×.
- **Don't repeat heavy context across N parallel agents — reference a file
  path instead**. When fanning out 3-5 ideation agents, including a 700-word
  buyer persona verbatim in every prompt wastes ~5× the same tokens. Either
  write the persona to a file once (`/tmp/persona.md`) and tell each agent
  to read it, OR include a tight 2-sentence summary inline. The full text
  goes once.
- **Default to 3 ideation agents, not 5**. For "what features should we
  add" with this persona, 3 agents covered all of the consensus themes; the
  4th and 5th were confirming, not surfacing new signal. Reserve 5+ for
  genuinely high-variance open-ended problems. Smaller fan-outs = faster
  consensus, easier to read.
- **Read big files in one wide pass, not 5 drip-reads**. When you need to
  understand the structure of a large single-file SPA, do one `grep -n` to
  find anchors, then ONE wide Read covering the relevant range, not six
  100-line reads chasing references. Same applies to "where does this
  function get called" — `grep` first, Read once.
- **Ship the minimum-viable shape first, layer extras in later
  iterations**. Tonight's Today page shipped at ~140 lines of JS in one
  pass (ring + 3 stats + sparkline + resume + leftovers + jump button).
  Iteration 1 should have been just ring + 3 stats (~40 lines); sparkline,
  resume, and leftovers could have been iteration 2/3. Smaller diffs = QA
  finishes in 1-2 minutes instead of 11, and rollback is trivial.
- **Batch preview ops into one `preview_eval`**. `preview_resize` + seed
  localStorage + `location.reload()` can be one eval call instead of three
  round trips. Same for any verification dance: combine the screenshot
  trigger, the data setup, and the navigation into one JS block.
- **The multi-agent fan-out pattern is structurally expensive and should
  NOT be the default for every iteration**. Each Sonnet sub-agent uses
  ~30-40K tokens; each Opus QA uses 40-80K. A full "3-agent fan-out + Opus
  QA" iteration costs ~190-330K tokens. At a 3h51m cron cadence that's
  ~1.5M tokens/day, which blows past the Pro session pool. ROI of N agents
  is real for genuinely open-ended decisions (Iter 1 ideation, Iter 3
  persona refinement) but already diminishing by Iter 2 (the feature was
  pre-decided). Default rule going forward: **routine iterations =
  Claude works directly (no sub-agents); fan out only when the decision
  is genuinely contested OR Adithya explicitly says "council this" /
  "consensus this"**. Reserve Opus QA for genuinely risky changes too.
  Cuts per-iteration cost ~5-10×.
- **Iterations must ship RADICAL features, not menial polish.** Adithya
  is explicit: each cron iteration should propose ONE separate
  sidebar-level feature (Find Companies scope) that the persona would
  actually buy for. Not a 4th dashboard tile, not a chip cleanup, not a
  copy tweak. Show "the face of it" — UI shell with mock data — and let
  Adithya approve. If approved, he and Claude pair on the API key setup
  in a normal session. Menial polish (alignment, copy, color) only
  happens AFTER the radical features are stacked up and approved. Rule of
  thumb: if the iteration's headline isn't a new SIDEBAR ITEM, it's too
  small.
- **Reply style: terse, lead with the point.** Adithya doesn't want
  multi-paragraph build summaries. Format: one-line headline of what
  was shipped, then 2-3 bullets of detail, then localhost links.
  No prose. No "I" sentences. No restating the request back. The diff
  speaks for itself; the reply is the headline.
- **Don't add UI elements just because the data exists.** The
  "yesterday's leftovers" chip and "last added yesterday" fallback on
  the Today page were technically functional but added noise that
  Adithya didn't want. Default behavior: if a feature requires the user
  to ask for it ("would be nice to see X..."), build it. Don't pre-empt
  utility from the data model.
- **Codex mailbox/contacted handoff (June 20, 2026)**: Adithya connected
  primary Gmail (`adithya@moderninbound.com`) because the actual outbound
  Outlook account (`adithya@moderninbounds.com`) requires Rishabh/Azure
  verification. Requirement: if a tracked lead email appears in the
  connected mailbox, show the existing green `contacted` chip under the
  company/contact area. This must be scalable for any user with connected
  mailboxes, not hardcoded to Adithya.

  Codex changed the mailbox scanner in `outreach-tracker.html` and synced it
  to `outreach-deploy/index.html`: Gmail now reads `To`/`Cc` headers from
  inbox and sent metadata, excludes connected/login mailbox addresses, and
  falls back to Gmail search for every tracked contact email because
  forwarded/HubSpot/tracking copies can make Gmail UI search find a lead
  even when the lead is not exposed in message headers. Outlook sent scanning
  now also excludes connected/login mailbox addresses. The scan badge can be
  clicked to force a fresh scan, and `window.otDebugMailScan(email)` reports
  whether a specific email is in the contacted/replied sets.

  Known status: scan badge shows Gmail scanning works (example seen:
  `Scanned · 170 contacted · 3 replied`), but the frontend still did not show
  `contacted` for specific rows like `ashutosh.taparia@credable.in` /
  `harshit@intugine.com`. Next Claude step should diagnose whether the exact
  row email is absent from `_mailScan.contacted`, whether Gmail query syntax
  needs another shape, or whether `renderCompanyList()` is only showing
  company-level chips beside the `N contacts` count instead of the desired
  contact-row location. Use `window.otDebugMailScan("email@domain.com")` after
  a forced rescan, then inspect `renderCompanyList()` around the `coStatus`
  and `statusChip` logic.
