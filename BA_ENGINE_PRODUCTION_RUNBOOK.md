# BA Job Intelligence — Production Runbook

**Last verified:** 06 Oct 2026 (Asia/Kolkata)  
**Purpose:** Single source of truth for operating, troubleshooting, and safely refining the Business Analyst job-search engine and Control Centre without depending on a ChatGPT conversation.

---

## 1. System at a glance

The production system is split into four independent layers:

```text
ChatGPT scheduled tasks (optional/separate)
            |
            | separate automation
            v
+--------------------------------------------------+
| SUPABASE: job-vault                              |
|                                                  |
| pg_cron                                          |
|   -> search engine Edge Function                 |
|   -> dedup / quality / URL verification          |
|   -> EOD finalization                            |
|   -> AgentMail emailer                           |
|                                                  |
| PostgreSQL = system of record                    |
+-------------------------+------------------------+
                          |
                          v
                  Email mailbox

Control Centre:
GitHub Pages frontend -> Supabase read-only dashboard API/RPC -> live DB data
```

**Production backend:** Supabase project `job-vault`  
**Project ref:** `eybjpjhzwskfgskvnfbj`  
**Region:** `ap-southeast-1`  
**Control Centre repository:** `rianmercer/ba-job-intelligence-control-centre`  
**Live Control Centre:** `https://rianmercer.github.io/ba-job-intelligence-control-centre/`

Do not treat this ChatGPT conversation as the production system. The backend, cron jobs, Edge Functions, GitHub repository, and secrets live independently.

---

## 2. Production Edge Functions

| Function | Responsibility |
|---|---|
| `ba-job-engine-core-v2` | Source ingestion, normalization, matching, data-quality gates, direct URL verification, deduplication, digest generation |
| `ba-job-emailer-agentmail` | Reads `notification_outbox`, sends EOD email through AgentMail, records delivery state |
| `ba-control-api-v3` | Secure Control Centre API / backend control operations and dashboard data contract |

**Verified snapshot on 06 Oct 2026:**
- `ba-job-engine-core-v2` — ACTIVE, version 82
- `ba-job-emailer-agentmail` — ACTIVE, version 31
- `ba-control-api-v3` — ACTIVE, version 32

Versions will change after future deployments; use the live Supabase function list as the current source of truth.

---

## 3. Database tables that matter

| Table | Purpose |
|---|---|
| `job_sources` | Source registry and source health |
| `job_vacancies` | Canonical vacancy records |
| `run_matches` | Per-run candidate/match records |
| `search_runs` | Run-level telemetry |
| `source_fetches` | Per-source fetch telemetry |
| `daily_digests` | Consolidated daily EOD result |
| `notification_outbox` | Email queue / send / delivery state |
| `notification_attempts` | Individual email attempts and errors |
| `applications` | User application/pipeline state |
| `job_alert_ledger` | Durable record of vacancies already included in an alert |
| `engine_config` | Runtime search, schedule, and notification configuration |
| `engine_health` | Engine heartbeat/run/email health |
| `job_dedup_ledger` | Existing engine-level deduplication support |

### Critical dedup rule

An emailed vacancy is **not** automatically an application.

The engine uses two separate suppression concepts:

1. **Alert ledger:** vacancy was already included in an EOD alert -> suppress from future alerts.
2. **Application state:** vacancy is already in the application pipeline -> suppress from future alerts.

Do not replace either with a fake `applications.status='applied'` record merely because an email was sent.

---

## 4. Search schedule (Asia/Kolkata / IST)

### Search windows

| Window | Cron UTC | IST |
|---|---|---|
| Run 1 | `30 2 * * *` | 08:00 |
| Run 2 | `30 6 * * *` | 12:00 |
| Run 3 | `30 12 * * *` | 18:00 |
| Run 4 | `0 16 * * *` | 21:30 |

### EOD sequence

| Task | Current IST time | Cron UTC | Purpose |
|---|---:|---:|---|
| `ba-job-engine-2130` | 21:30 | `0 16 * * *` | Final search window |
| `ba-job-eod-finalizer-2131` | 21:31 | `1 16 * * *` | Ensure final run/digest is current |
| `ba-job-eod-recovery-5m` | 21:xx–23:59 | `*/5 16-17 * * *` | Recovery guard |
| `ba-job-recovery-2140` | 21:40 | `10 16 * * *` | Missing-window recovery |
| `ba-job-emailer-2200` | 22:00 | `30 16 * * *` | Primary EOD email attempt |
| `ba-job-emailer-2205` | 22:05 | `35 16 * * *` | Retry |
| `ba-job-emailer-2250` | 22:50 | `20 17 * * *` | Retry |
| `ba-job-emailer-2310` | 23:10 | `40 17 * * *` | Final retry |

All of these were verified active on 06 Oct 2026.

**Important:** Run 4 is **21:30**, not 22:00. The EOD email target is **22:00**. Keep those concepts separate.

---

## 5. Current runtime configuration snapshot

`engine_config` currently indicates:

- Timezone: `Asia/Kolkata`
- Search windows: `08:00`, `12:00`, `18:00`, `21:30`
- Minimum match score: `55`
- Strong match score: `72`
- Direct URL required: `true`
- Direct output only: `true`
- Source policy: `direct_only` for final output
- Discovery enabled: `true` internally
- Included regions: India, Europe, Gulf, Other
- Included countries: configured broad international list including India, Europe/UK, Gulf, Canada, Australia/NZ, Singapore/Malaysia
- Freshness window: `21` days (with additional engine freshness safeguards)
- Max direct URL verification: `120`
- Notification provider: `agentmail`
- Notification enabled: `true`
- Retry enabled: `true`
- Max email attempts: `5`
- EOD target: `22:00`
- Full-day requirement: `true`

### Current production recipient

Supabase currently uses:

`ptainds5@googlemail.com`

This is intentional because AgentMail suppressed the `ptainds5@gmail.com` address. A successful delivery to the `@googlemail.com` alias has already been confirmed.

AgentMail support was contacted to review/clear the original `@gmail.com` suppression. Do not switch production back to `@gmail.com` until a successful provider test confirms that the suppression is cleared.

---

## 6. Job source policy

The engine is not intended to be limited to Freshworks, Talan, or any single company/job board.

Current registry snapshot includes:

- Direct employer / ATS sources
- Discovery sources used to find candidate vacancies
- ATS/provider types such as Greenhouse, Lever, Ashby, SmartRecruiters and others

### Non-negotiable URL rule

For a job to reach an EOD email, the final link must be the exact live vacancy/application URL for that specific requisition and must be directly verifiable on the employer's official careers domain or official ATS.

Do not include:

- Company homepage
- Generic careers page
- Search-results page
- Generic ATS portal landing page
- LinkedIn job URL when an official direct vacancy URL is unavailable
- Indeed / Glassdoor / other job-board URL as the final application link
- Guessed or constructed URLs

Third-party/discovery sources are useful for discovery, but they must not weaken the final URL-verification rule.

---

## 7. Daily email decision flow

```text
Source discovery
    |
    v
Normalize vacancy
    |
    v
Profile match
    |
    v
Data-quality checks
    |
    v
Direct URL verification
    |
    v
Cross-source / identity dedup
    |
    v
Suppress already-applied / pipeline records
    |
    v
Suppress job_alert_ledger records
    |
    v
Same-day duplicate guard
    |
    v
EOD digest
    |
    v
notification_outbox
    |
    v
AgentMail
    |
    v
Delivered mailbox
```

A smaller result is preferable to padding the email with weak, stale, duplicate, or unverifiable roles.

---

## 8. How to diagnose a missing EOD email

Do this in order. Do not immediately change cron jobs.

### Check 1 — `search_runs`

For today's date, confirm all four windows exist and inspect:

- `status`
- `source_count`
- `successful_sources`
- `fetched_jobs`
- `eligible_jobs`
- `strong_matches`
- `moderate_matches`
- `error_count`
- `verification_attempts`

Healthy pattern: completed run(s), successful sources, no unexplained errors.

### Check 2 — `daily_digests`

Confirm today's digest exists and inspect:

- `status`
- `total_sources`
- `total_fetched`
- `total_new`
- `total_eligible`
- `strong_matches`
- `moderate_matches`
- `notification_status`
- `notification_error`

### Check 3 — `notification_outbox`

Find the row whose subject starts with:

`Daily BA Job Alerts | YYYY-MM-DD`

Inspect:

- `status`
- `recipient`
- `sent_at`
- `delivered_at`
- `provider_message_id`
- `attempt_count`
- `last_error`

### Check 4 — `notification_attempts`

Look at the newest attempt for the same outbox row. Classify the result:

- `2xx` -> provider accepted message
- `403` / permanent provider rejection -> investigate recipient suppression/policy
- `409` -> usually idempotency conflict; regenerate a unique key only when safely replaying a distinct message
- `429` -> retry/rate-limit path
- network/timeout -> inspect retry behaviour and function logs

### Check 5 — `engine_health`

Inspect:

- last run timestamps
- `last_email_sent_at`
- `last_email_delivered_at`
- `consecutive_failed_runs`
- `last_error`
- heartbeat

Never clear a real error just to make the dashboard green. Only clear stale diagnostic fields after the underlying failure has been resolved and verified.

---

## 9. Common failure patterns and correct action

### “I keep getting Freshworks/Talan jobs again.”

Check `job_alert_ledger` and `applications` first. Do not solve this by simply tightening the freshness window.

The engine should suppress jobs already alerted or already in the application pipeline even when the same vacancy is encountered again from another source.

### “A source has jobs but they never appear in the email.”

Check URL verification, freshness, relevance, and dedup suppression before changing source enablement.

A discovered job with an unverifiable direct URL is supposed to be rejected.

### “No jobs today.”

Do not assume the engine is broken. Check whether:

- searches completed successfully
- candidates were found
- candidates were suppressed because already alerted/applied
- exact URLs failed verification
- the final digest has zero qualifying fresh records

A zero-email-match day is valid, but the production EOD policy should still send the daily status message when configured to require a full-day digest.

### “AgentMail rejected my Gmail address.”

Inspect `notification_outbox` and `notification_attempts`. A provider suppression may be the cause. Do not create endless plus-address aliases; AgentMail may canonicalize them back to the suppressed mailbox.

Current working recipient: `ptainds5@googlemail.com`.

### “Control Centre says API error.”

Check:

1. GitHub Pages deployment status.
2. Live `index.html` frontend contract.
3. `ba-control-api-v3` status/version.
4. Dashboard RPC/API HTTP response.
5. Browser cache query string (`?v=N`) if necessary.

The Overview dashboard is intended to be passwordless and read-only, with live data from Supabase. Do not put service-role secrets into the frontend.

---

## 10. Control Centre architecture

**Repository:** `rianmercer/ba-job-intelligence-control-centre`

**Frontend:** GitHub Pages

**Data flow:**

```text
index.html
   |
   +--> Supabase publishable API/RPC
             |
             +--> PostgreSQL functions / tables
                      |
                      +--> search_runs
                      +--> job_vacancies
                      +--> run_matches
                      +--> job_sources
                      +--> applications
                      +--> job_alert_ledger
                      +--> daily_digests
                      +--> engine_health
                      +--> engine_config
```

The frontend is a visualization layer. The database/backend remains the source of truth.

### Frontend safety

Allowed in the frontend:

- Supabase publishable/anon key
- read-only public dashboard endpoint/RPC

Never place in the frontend:

- service-role key
- AgentMail API key
- Vault secrets
- engine trigger secret
- control password

---

## 11. Changing configuration safely

### Change matching threshold

Modify `engine_config.search`.

Example request to a future maintainer/ChatGPT:

> “Change the minimum BA match score from 55 to 60. Inspect the current production configuration, update only that setting, then verify the engine function and next scheduled run remain healthy.”

### Change included countries/regions

Modify `engine_config.search.include_regions` and/or `include_countries`.

### Change a source

Modify `job_sources` carefully. Confirm the source's URL/API remains live before enabling it for production output.

### Change run times

Modify the relevant `pg_cron` jobs. Do not change a UI label alone. The backend schedule is the source of truth.

### Change email behaviour

Modify the notification configuration or `ba-job-emailer-agentmail`. Test with a unique, clearly labelled test email before touching production EOD behaviour.

### Change matching/deduplication logic

Modify `ba-job-engine-core-v2` and deploy it. Then run a controlled test/search and inspect `search_runs`, `run_matches`, `job_alert_ledger`, `daily_digests`, and `notification_outbox`.

---

## 12. Safe deployment rules

1. Inspect live production state before editing anything.
2. Make one logical change at a time.
3. Never expose secrets in chat, source code, logs, screenshots, or the runbook.
4. Preserve existing cron jobs unless the change explicitly requires them.
5. Never disable a scheduled job just because one run failed.
6. Deploy Edge Functions through a controlled deployment path.
7. Track database changes as versioned migrations whenever possible.
8. After deployment, run a read-only verification query or controlled smoke test.
9. Check Supabase advisors after meaningful production changes.
10. Never declare a fix successful until the live endpoint/function or deployment has been tested.

---

## 13. ChatGPT scheduled tasks — do not confuse with Supabase

ChatGPT scheduled tasks are a separate automation layer.

As of 06 Oct 2026, the important enabled tasks are:

- `Daily BA Job Digest`
- `BA Email Delivery Guard`

There are also older BA tasks that are already disabled/paused. Do not re-enable them just because an email was missing unless explicitly requested.

### Important distinction

Supabase's production EOD address is currently:

`ptainds5@googlemail.com`

The currently enabled ChatGPT `Daily BA Job Digest` task is configured for:

`ptainds5@gmail.com`

This is a configuration mismatch that should be consciously resolved later; do not silently assume the two automation layers are identical.

**Do not disable or alter ChatGPT scheduled tasks during Supabase troubleshooting unless explicitly requested.**

---

## 14. Current health baseline (06 Oct 2026)

The last verified backend state showed:

- 08:00 run: completed, 57/57 sources successful, 0 errors
- 12:00 run: completed, 57/57 sources successful, 0 errors
- 18:00 run: completed, 57/57 sources successful, 0 errors
- 21:30 run: scheduled for EOD completion on the daily cycle
- EOD delivery path: previously verified successful
- Engine consecutive failed runs: `0`
- Engine last error: `null`

A healthy dashboard does not mean every discovered job becomes an email. Deduplication and quality gates intentionally reduce the final output.

---

## 15. Database migration / version-control policy

Future database changes should be represented as versioned migrations wherever practical.

Recommended repository structure:

```text
ba-job-intelligence-control-centre/
├── index.html
├── BA_ENGINE_PRODUCTION_RUNBOOK.md
└── supabase/
    └── migrations/
```

The runbook should be updated whenever a production architectural change is made.

Do not store secret values in migrations or this runbook. Store only secret **names/references**, e.g.:

- `agentmail_api_key`
- `agentmail_from_address`
- `ba_job_engine_trigger`
- `ba_control_password_v2`

---

## 16. “New ChatGPT conversation” recovery procedure

If the original ChatGPT conversation disappears, start a new conversation and say:

> “Work on my BA Job Intelligence production system. The production runbook is in the GitHub repo `rianmercer/ba-job-intelligence-control-centre`. Read `BA_ENGINE_PRODUCTION_RUNBOOK.md` first. Do not disable existing ChatGPT scheduled tasks. Inspect live Supabase state before changing anything.”

Then provide the symptom, for example:

> “Today's EOD email did not arrive.”

or:

> “Control Centre is showing stale data.”

or:

> “Add a new region/source and keep direct-URL verification mandatory.”

That is enough to reconstruct the operational context without this conversation.

---

## 17. What to ask for in future

Use plain English. You do not need to write SQL or code.

Good examples:

> “Exclude SAP functional consulting unless the role is clearly BA-led.”

> “Add Singapore to the search and keep the same URL verification rules.”

> “Increase the strong-match threshold from 72 to 75.”

> “Today's EOD email didn't arrive. Diagnose it end to end without disabling any scheduled task.”

> “The Control Centre is showing old numbers. Compare the frontend with live Supabase telemetry and fix the mismatch.”

> “Do a fresh test run, perform all duplicate/data-quality/direct-URL checks, and show me only genuinely fresh matches.”

---

## 18. Golden rules

**1. Backend is the source of truth.**  
Frontend numbers must come from Supabase, not hardcoded values.

**2. Fresh means fresh.**  
Previously alerted or already-applied jobs must not be emailed again.

**3. Verify the exact URL.**  
No generic careers pages or unverifiable links in final email output.

**4. Quality beats quantity.**  
Zero qualifying matches is better than sending stale or weak jobs.

**5. Do not break the scheduler while troubleshooting.**  
Inspect first; change only the failing component.

**6. Keep ChatGPT automation separate from Supabase production.**  
Never disable one because the other is being investigated.

**7. Never expose secrets.**  
Vault and provider credentials remain backend-only.

**8. Test after every meaningful change.**  
A deployment is not a verification.

---

## 19. Useful live locations

**GitHub repository**  
`https://github.com/rianmercer/ba-job-intelligence-control-centre`

**Control Centre**  
`https://rianmercer.github.io/ba-job-intelligence-control-centre/`

**Supabase project dashboard**  
`https://supabase.com/dashboard/project/eybjpjhzwskfgskvnfbj`

---

## 20. Change log

### 06 Oct 2026

- Diagnosed AgentMail suppression of `ptainds5@gmail.com`.
- Confirmed `ptainds5@googlemail.com` delivery works.
- Replayed and delivered the missed 05 Oct EOD digest.
- Added durable `job_alert_ledger`.
- Added automatic ledger recording when an EOD alert is sent/delivered.
- Updated engine suppression to check alert ledger and application pipeline.
- Improved cross-source duplicate suppression.
- Corrected Himalayas discovery query/pagination handling while keeping exact URL verification mandatory.
- Repaired Control Centre API/frontend contract.
- Rebuilt Overview dashboard as passwordless live Supabase read-only visualization.
- Corrected Control Centre Run 4 label from 22:00 to 21:30 IST.
- Corrected EOD finalizer cron from 22:01 to 21:31 IST so finalization precedes the 22:00 emailer.
- Verified GitHub Pages deployment and live frontend.
- Verified Supabase search runs and email health.

---

**End of runbook.**