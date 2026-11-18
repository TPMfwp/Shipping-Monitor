# Shipping Monitor — context for Claude

Multi-store shipment tracking dashboard. ParcelPanel tracking updates flow into Supabase; a static
dashboard on GitHub Pages reads them and shows which orders need action; Slack gets notified once per
order per action section.

## Status (2026-10-16): DONE — ready for the next store

The merchant confirmed the system is finished. Everything below is live, tested and documented in this file.
Ellison & Rowe is the only store; it has been idle since June, so it gets no reports or alerts until orders flow again.

Built/fixed 2026-10-15/16:
- **Security**: `webhook` + `perf-check` require `?token=` (`app_settings.webhook_token`, never in the repo);
  `slack-notify` needs token or login and only posts to hooks.slack.com; `esc()`/`jsq()` escaping in the dashboard.
- **Alerts**: per-store day thresholds (Alerts tab), tracking links (store page + 17track), one alert per order per
  section, and a **daily sweep** (07:00 UTC) so stuck/delayed alerts also fire when ParcelPanel sends no more updates.
- **Reports**: weekly report every Monday (week vs last week) + monthly report on the first Monday (previous month);
  facts only, positive framing ("on time", "without problems"), no verdict under 20 orders, idle stores skipped.
- **Dashboard**: Supplier Performance uses the same calculation as the reports, with a 7d/30d/90d/All selector;
  goals entered as "% on time" / "% without problems"; Test sends the real weekly report.
- **Remove store**: RPC `remove_store` deletes everything of a store in one transaction (client deletes used to fail
  silently); confirm button reacts while typing.
- **Free-plan safety**: `keep-alive` cron every 6h (prevents pausing), incremental refresh + no refresh in hidden tabs
  (egress), usage line in the header (database exact, data estimate; orange 60%, red 80%).
- **Test data** (TEST STORE, DEMO stores) fully removed.

Scheduled jobs (pg_cron): `perf-check-weekly` (Mon 07:00 UTC), `alert-sweep` (daily 07:00 UTC), `keep-alive` (every 6h).

When the next store goes live: set its Slack webhooks (alerts: Manage stores → Slack; reports: Supplier Performance →
Settings), check the first rows in `webhook_samples` for the tracking-link fields, and update E&R's old `perf_webhook`
if that store comes back. Open ideas only (see the end of this file): READY_FOR_PICKUP section, customer email out of
Slack for shared channels, egress plan when the usage line turns orange.

## Working agreement

- **Only edit `index.html`.** No frameworks, no build step, no npm — plain vanilla HTML/CSS/JS in one file.
- After a change: commit and push to `main`. GitHub Pages deploys automatically.
- **Pages has a deploy delay of a few minutes** after push. The raw file on `main` updates instantly,
  the served site does not. Wait (or poll the live HTML for the new code) before concluding something is broken.
- Supabase Edge Functions are **not in this repo**. They are edited in the Supabase dashboard editor.
  Read them via the Supabase MCP (`list_edge_functions` / `get_edge_function`) before reasoning about them.
- UI copy is mostly English; some Slack-settings copy and status messages are Dutch. Keep the language
  of the surrounding text.
- Code style: dense, minified-ish one-liners, inline styles in the HTML, short names. Match it.
- **Slack messages** (alerts, reports, any new message) — the merchant's fixed rules:
  - Always a blank line directly after the main header.
  - Scannable: two columns (Slack section `fields`), no blocks of prose, neutral tone, facts only.
  - The dividers above and below an alert stay.
  - Approved pattern: header → Store/Order/Customer/Amount in two columns → question to the supplier.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole dashboard (~1090 lines: CSS, markup, JS). |
| `import.html` | Legacy one-off order import form; posts to the `Import` Edge Function, which is no longer active. No login screen. Leave alone. |
| `README.md` | Stub. |

## Infra

- Supabase project: `https://frxyfmfjtadyxhfogdvt.supabase.co` (org "ForwardPath")
- Publishable key is hardcoded in `index.html` (`SUPABASE_KEY`). The repo is **public** (Pages on a free
  account requires that), so the key is public. **RLS is the only real security layer.**
- supabase-js v2 loaded from jsDelivr UMD (`window.supabase.createClient`).

### Auth

- Login screen asks only for a password. `checkPassword()` calls
  `sb.auth.signInWithPassword({email: AUTH_EMAIL, password})` with a fixed hidden account
  (`AUTH_EMAIL` const in `index.html`). Change the password via Supabase Dashboard → Authentication → Users.
- Session persists in localStorage; on `DOMContentLoaded` an existing session skips the login screen.
- All app tables have RLS policies `TO authenticated` (no public access). `stores` has no DELETE policy;
  `perf_weekly_log` has RLS on with no policies (only Edge Functions with the service role touch it).
  `app_settings` is SELECT-only for `authenticated`.

## Data model (Supabase tables)

### `shipments` — one row per order/tracking, written by the `webhook` function
Columns used by the dashboard: `order_number`, `customer_name`, `tracking_number`, `status`, `store_name`,
`order_date`, `fulfillment_date`, `last_updated`, `last_checkpoint_time`, `tracking_link`, `pickup_date`.
`pickup_date` (added 2026-11-16) = when the carrier actually picked the parcel up, from ParcelPanel's `pickup_date` field.
`fulfillment_date` is NOT the shipping moment: the supplier creates the label (= Shopify fulfilled) right after the order
comes in, and the parcel can sit at the supplier until pickup. `webhook` stores `pickup_date` only when it is a valid date
(never overwritten with null). Rows before 2026-11-16 don't have it. Verify on the first real webhook of a new store
(`webhook_samples`) that ParcelPanel really fills it.
`tracking_link` = the store's own tracking page, set by `webhook` (`findTrackingLink()`: payload `tracking_link`/
`tracking_url`/`tracking_page_url`, else `https://<store.domain>/apps/parcelpanel?nums=`), never overwritten with
null. E&R rows were backfilled with `ellisonrowe.com/apps/parcelpanel`. Dashboard `trackHtml()` and Slack alerts
fall back to `https://t.17track.net/en#nums=<tracking>` when it's empty. When a store link exists, both are shown
(dashboard: number → store page + small "17track" link; Slack: number + "Store page · 17track"). Customers never see
these links — they only get ParcelPanel's own tracking page via the store.
`status` is a ParcelPanel status: `PENDING, INFO_RECEIVED, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERED,
EXCEPTION, EXPIRED, FAILED_ATTEMPT, CANCELLED, UNKNOWN, READY_FOR_PICKUP, RESOLVED`.
Added 2026-11-16 (webhook v17): `substatus` / `substatus_label` (ParcelPanel reason, e.g. `Exception_004` "Address issue";
dashboard shows it under the badge for problem statuses, red for Returned/Damaged/Lost), `status_since` (set when the
status changes; used for the pickup alert), `alerts` jsonb (which extra Slack alerts were sent and when:
`exception_at`, `delayed_at`, `reminder_at`, `pickup_at`, `reason_at`, `reasons[]`).

### `webhook_samples` — first 5 raw ParcelPanel payloads per store
Written by `webhook` (service role only, RLS on, no policies). Purpose: verify which fields ParcelPanel really
sends (tracking link? store domain?) — **check this after the first real webhook from a new store** and adjust
`findTrackingLink()` if needed.

### `action_log` — orders that entered an action section (primary key `id` = `rowId`)
`id, order_number, customer_name, tracking_number, store_name, order_date, fulfillment_date, section,
resolved_at, resolved_status, slack_notified` (+ server-side `slack_notified_sections` array).
- `section`: `label_not_shipped` | `delayed` | `exception` | `deleted` (permanent delete marker).
- Two notification fields coexist on purpose:
  - `slack_notified_sections` (array) is the real one, maintained by the `webhook` function: each order gets
    **one Slack message per action section, ever**.
  - `slack_notified` (boolean) is legacy; the dashboard still writes `slack_notified:false` on new entries.
    Don't remove either without checking the `webhook` function.

### `dismissed` — orders the user dismissed from the action sections (primary key `id` = `rowId`)
Orders manually clicked away (×) in the dashboard are stored here so they no longer appear in the action
sections. Snapshot of the shipment + `dismissed_at`. Restore deletes the row (and its `action_log` row).

### `stores` — per-store settings (primary key `store_name`)
`slack_webhook_url` (shipping alerts), `perf_norm` (on-time norm in days, default 14), `perf_webhook`
(performance alerts), `perf_alert_ontime` (alert if on-time % below), `perf_alert_exc` (alert if exception % above),
`alert_not_shipped_days` (default 5) and `alert_delayed_days` (default 15) — the per-store action-section thresholds,
edited in Manage stores → Alerts. A store without a `stores` row uses 5/15.
`sheet_url` (2026-11-16) = web-app `/exec` link of the store's Google Sheet, set in Slack settings → "Google Sheet link".

### Google Sheet per store (2026-11-16, webhook v19)
Each store can have its own Google Sheet with a **bound** Apps Script (local clasp copy: `~/Projects/Shipping-Monitor Sheet (bound clasp)/`,
clasp logged in as the sheet owner; after `clasp push` run `clasp deploy -i <deploymentId>` so the `/exec` link stays the same).
5 tabs = the alert types: Info Received (was "Stuck", renamed 2026-11-17; script `RENAMED` renames an old Stuck tab automatically, webhook v24 sends 'Info Received'), Not moving (Last Scan column, script @7), Delayed (+ 30-day reminder), Exception (+ Reason changed), Pickup point, plus **Solved** (script @9): as soon as Current Status becomes Delivered (status update from the webhook, or Setup/Test), `moveDone_()` (was moveDelivered_) moves the row from the open tab to Solved (top, newest first; columns Solved Date + From Tab + all alert columns). Status updates also reach rows in Solved. A new alert for a solved order creates a fresh row in its open tab. **CS ↔ Supplier (script @12, 2026-11-17):** black bar in every open tab = Requested Solution (key cs_request) / Supplier Answer / Issue Status (dropdown Open/Pending/Solved). Two CS-filled tabs: **Customer issues** (Date, Issue dropdown Damaged/Wrong item/Missing/Wrong size/Other, Order, Customer, Email, Product, Description, Evidence) and **RTC** (return to China, Return Status dropdown from the CS sheet). Menu: ➕ New customer issue / ➕ New RTC return (row on top, today, Open). Issue Status → Solved: row STAYS in its tab and turns green; `onEdit` (simple trigger) writes "Solved on yyyy-MM-dd" as a note on that cell (back to Open/Pending clears it). `moveDone_()` moves a row to Solved when Current Status is Delivered, or Issue Status has been Solved for `SOLVED_DAYS` (7) days (Solved without a note gets today's date). It runs after every doPost, on open and on Setup (no time trigger, no extra permissions). Requested Solution + Supplier Answer move along (script @14). **RTC automatic (script @20–@21, 2026-11-17):** an Exception alert whose reason contains "return" (e.g. Returned to sender) goes straight to RTC instead of Exception; a status update whose checkpoint says returning/returned to sender/shipper/origin moves the order out of any auto tab into RTC (`toRtc_`: one RTC row per order, conversation + Issue Status move along, Return Status 'Open / In transit', Description = reason + checkpoint, deduped). CS only fills Return Status and the conversation; ➕ New RTC return stays for manual returns. RTC header = purple 'AUTOMATED (CS can add returns)' (needs Setup once). Test data (@19+): per-tab sample conversations, test returns go into RTC. **Customer mails + Today (script @24, 2026-11-18):** every open tab has a teal 'CUSTOMER MAILS' block after the black bar: 1st/2nd/3rd Mail checkboxes (ticking writes 'Sent on yyyy-MM-dd' as a cell note via onEdit) + Next Step (script-filled). Schedule from the course SCM SOP: 2nd mail 3 days after the 1st, 3rd after 5, day 8 (6 days after the 1st) no answer → refund. `refreshToday_()` rewrites Next Step in all open tabs and rebuilds the first tab **Today** (what to send today/overdue, with a link to the row); runs on open, on mail/Issue Status edits, after every doPost, Setup and Test. Mail ticks move along to RTC. Solved has no mail block. Agreed direction (2026-11-18): 2 sheets — this order sheet (CS + supplier, no money) and the merchant's CS sheet (money only); later: Not shipped tab, Issues tab replacing Customer issues, customer returns with photo in RTC, Supplier claims in the CS sheet (CS sheet only when the merchant says so). Open questions for the merchant: keep the 7-day Solved rule?, Description/Evidence in Solved, money-free RTC statuses (merchant does not want money matters visible to the supplier). The webhook never writes in CS tabs (writeAlert_ ignores them); updateStatus_ only touches auto tabs + Solved. Direction: this sheet = only place for supplier communication; the separate CS sheet keeps numbers only and is not touched. A missing tab is created automatically on the first alert; Setup puts the tabs in this order. Columns differ per
tab (`LAYOUT` in the script; the script maps payload keys to columns): Reason only in Delayed + Exception, Previous Reason
only in Exception (Reason cell orange when it's filled). Row 1 purple `#a26e87` "AUTOMATED (Do not edit)" over the auto
columns, black `#2c2c33` bar over 3 manual columns (content not decided yet), 2 rows frozen. Current Status + Last Checkpoint
follow every ParcelPanel update (v21: also right after a new alert, so a row in an older tab stays current). One row per order per tab (matched on tracking #): a follow-up alert updates the same row; a new order is inserted at the top (row 3, newest first). The script never
writes the black columns. `webhook` POSTs `{source:'shipping-monitor', tab, alert, ...}` after each Slack alert (also from the
sweep), and `{type:'status', ...}` (Current Status + Last Checkpoint) in the background on later updates of orders that
already had an alert. No secret: the long `/exec` link is the secret; the script ignores bodies without `source`.
Sheet menu: Setup (builds tabs, safe to re-run), Make live (opens the Apps Script editor + steps: Deploy → New deployment →
Web app, Me, Anyone → copy URL; automating it via the Apps Script API fails because a bound script's default GCP project
can't enable that API — the merchant wants no Google Cloud; never use `ScriptApp.getService().getUrl()` from the menu, it
returns a wrong `/exec` link that 404s), Test. New store: duplicate an empty template sheet → Setup → Make live → paste the
`/exec` link in the dashboard. E&R live deployment: `AKfycbzXUp2w…` @2.
`shipments.products` (text, "x1 Title - Variant" per line) is stored from ParcelPanel `products[]` since v19.

### `perf_weekly_log`
Dedup + record: one row per store per sent report. `week` = Monday date (`2026-10-19`) for the weekly report, `month-YYYY-MM` (the month reported on) for the monthly report. Holds that report's figures (ontime/problem %, proc/transit days, counts). Comparisons in the reports are computed live from `shipments`, NOT from this table.

### `app_settings` — key/value config
`webhook_token`: secret that `webhook` and `perf-check` require as `?token=`. Readable only after login, so the
dashboard loads it at runtime (`loadWebhookUrl()`) — **never hardcode it in `index.html`** (public repo + public
Pages). Rotating it = update the row, then re-paste the URL in ParcelPanel for every store.

## Edge Functions (Supabase)

| Function | Role |
|---|---|
| `webhook` | Receives ParcelPanel `shipment_status / any_update` webhooks → rejects with 401 unless `?token=` matches `app_settings.webhook_token` → upserts `shipments` → checks action sections → sends Slack alert via `slack-notify` (once per section, tracked in `action_log.slack_notified_sections`). **Daily sweep** (`?sweep=1`, pg_cron `alert-sweep` `0 7 * * *`): stuck/delayed depend on days, but the webhook only checks when ParcelPanel sends an update — an order that never gets another update (label never picked up) never alerted. The sweep runs the same `alertIfNeeded()` over open orders (shipped/ordered in the last 60 days, `SWEEP_DAYS`), max 100 alerts per run, 1.1s apart. `&dry=1` only lists what it would send. Note: dashboard Restore deletes the `action_log` row, so a restored order can get its alert again via the sweep. **Extra alerts (v17/v18, 2026-11-16)**, each at most once per order, tracked in `shipments.alerts`, no suggestions in the question: 🔄 *Exception Changed* — order already alerted as exception gets a new reason from `REASON_UPDATES` (Exception_003 Returned to sender, _004 Address issue, _005 Damaged, _006 Lost, _007 Held at customs), fields Before/Now, only on a real webhook update; 🐌 *Still Not Delivered — 30+ Days* — still in transit at 2× `alert_delayed_days` after the Delayed alert, never the same day as that alert; 📍 *Waiting at Pickup Point — 7+ Days* (`PICKUP_DAYS`) — info only, no question. ⏸️ *Not Moving — No Tracking Update 5+ Days* (v23, 2026-11-17, `NOT_MOVING_DAYS`) — status IN_TRANSIT/OUT_FOR_DELIVERY and `shipments.last_checkpoint_time` (newest carrier scan, `checkpoints[0]`, verified on 698 delivered orders) 5+ days old; every new scan resets the counter; one alert per standstill (`alerts.not_moving_cp` = the scan it fired on, a new alert only after a newer scan stalls again); mostly fired by the daily sweep; sheet tab *Not moving* with Last Scan column. Since v23 `last_checkpoint`/`last_checkpoint_time` are only overwritten when the update contains checkpoints. Exception alert also shows Reason + Carrier. Closing sentences (v22, 2026-11-17): Exception Changed → "The exception status of this shipment has changed. Please provide an update. 🙏"; Still Not Delivered → "This shipment has still not been delivered. Please provide an update. 🙏". Sweep runs the extra checks too (except Exception Changed). |
| `slack-notify` | Thin proxy: `POST {webhook_url, blocks, text}` → posts to the Slack incoming webhook. Requires either `?token=` (used by `webhook`/`perf-check`) or a logged-in session `Authorization: Bearer` (dashboard `sendSlack()` for the "Test" buttons; browser can't post to Slack directly due to CORS). Only accepts `https://hooks.slack.com/` URLs. |
| `perf-check` | Requires `?token=`. pg_cron `perf-check-weekly` Monday 07:00 UTC. Every Monday a **📊 Weekly delivery report** (last 7 days vs the 7 days before); on the first Monday of the month also a **📅 Monthly delivery report** (previous calendar month vs the month before). Both built by `buildReport(kind)`, sent to `perf_webhook`. Layout: header, context line (period + goals), verdict (🟢 On goal / 🟡 Close to goal / 🔴 Below goal / ⚪ no verdict when <20 orders — worst of the two metrics, neutral wording), fields On time (% delivered within `perf_norm` of orders DELIVERED in the period, vs previous period) + Without problems (positive framing, merchant's choice: 100 − % of orders SHIPPED that ever had a problem status; goal shown as 100 − `perf_alert_exc`, so the setting stays "max % problems" incl. `action_log` exception history; weekly report uses orders shipped 14–21 days ago because fresh shipments haven't had time to get stuck — the date range is shown), Where the time goes (processing order→label, waiting for pickup label→`pickup_date` — only shown when orders have it —, transit pickup→delivered or label→delivered without pickup; on time stays label→delivered, vs previous period), Stuck right now (current problem status, shipped in last 60 days, per type; orders deleted in the dashboard are excluded unless the carrier had a new scan (`last_checkpoint_time`) after the delete — same idea as the Slack alerts; dismissed orders still count), What stands out (facts only, no advice, max 3, each with its own threshold: where late orders lose time vs on-time orders (processing, waiting for pickup or transit), on time ±10pt or processing ±2 days vs previous period, carrier gap ≥15pt with ≥20 delivered orders each (percentages, so unequal volumes are still fair), back on goal, best on-time period in N (≥3 earlier periods with ≥20 orders), else "Steady week" when green). Skips a store with nothing delivered/shipped/stuck. Test params: `&store=<name>&force_weekly=1` and/or `&force_monthly=1` (sends any day, never writes `perf_weekly_log`). |
| `Import` | Old Shopify+ParcelPanel bulk import used by `import.html`. **Inactive.** |

New stores connect by adding a ParcelPanel webhook (Professional plan) pointing at
`https://frxyfmfjtadyxhfogdvt.supabase.co/functions/v1/webhook?token=<webhook_token>` (shown with Copy button in
Manage stores → Add store); the store appears after the first update. One URL serves all stores — the store is
identified by `store.name` in the payload.

> **Section logic lives in two places — keep it in sync.** The thresholds themselves come from `stores`
> (per store), but the section *logic* (`getSection()`, statuses, CUTOFF, the 5/15 fallback) is duplicated in
> `index.html` (`thr()`/`getSection()`) and the `webhook` Edge Function. Change one = change both.

## Dashboard logic (`index.html`)

Global state: `all` (all shipments), `dismissedCache`, `actionLogCache`, `perfSettingsCache`, `storesCache`,
`curStore`, `curTableFilter`, `curPage`, `sortCol/sortDir`.

**Load cycle** — `loadData()` (on login + every 60s): fetch all `shipments` in 1000-row pages ordered by
`order_date desc` → `loadDismissed()` → `loadActionLog()` → `loadStores()` → `loadPerfSettings()` →
render everything.

**Row identity** — `rowId(d) = order_number + '|' + (tracking_number || 'NO_TRACK')`. Used as the key in
`dismissed` and `action_log`.

**Action sections** — `getSection(d)` returns 1/2/3 or null:
- excluded: status `CANCELLED`/`RESOLVED`, order before `CUTOFF` (2026-03-10), no tracking, dismissed,
  or `action_log.section === 'deleted'`.
- days = days since `fulfillment_date || order_date`.
- **3 Exception**: `EXCEPTION`, `EXPIRED`, `FAILED_ATTEMPT` (any age).
- **1 Label created, not shipped**: `PENDING`/`INFO_RECEIVED` and days ≥ `thr(store).ns` (default 5).
- **2 Delayed in transit**: `IN_TRANSIT`/`OUT_FOR_DELIVERY` and days ≥ `thr(store).dl` (default 15).
- Section subtitles (`sub1`/`sub2`) show the selected store's numbers; on "All stores" only when all stores match.
- Day colouring (section rows + table) uses the same per-store thresholds.

`renderSections()` also writes to `action_log` from the client: `logAction()` inserts new action items,
`resolveAction()` sets `resolved_at/resolved_status` when an item leaves all sections. "Recently solved"
= `resolved_at` within 48h.

**Dismiss / Restore / Delete** — × button → `dismissById()` writes to `dismissed`. In the Dismissed panel:
Restore removes the `dismissed` and `action_log` rows (item can re-enter and re-notify); Delete upserts
`action_log.section='deleted'` so it never comes back.

**Stats & table** — 4 clickable stat tiles (Action required / In transit / Delivered / Total) set the table
filter; table is sortable, 100 rows per page. Store tabs are derived from distinct `store_name` in shipments.

**Supplier Performance** (`renderPerformance()`, per store or all) — same definitions as `perf-check`:
- Period selector 7d / 30d / 90d / All (`perfPeriod`, always starts on All — idle stores looked broken on 30d); each figure shows
  the previous period of the same length next to it ("last week" / "prev 30d").
- On time % (orders DELIVERED in the period, ≤ that store's `perf_norm`), Without problems % (orders SHIPPED in
  the period shifted 14 days back, `PERF_LAG`; problem = current problem status or `action_log` exception history),
  Delivery time (avg transit, + processing), Delivered count. Footnote `perf-note` only shows "No orders in the last N days" for an empty period (explanation text removed — too busy).
- Colour like Slack (green on goal / orange within 10pt resp. 5pt / red below), only per store and with ≥20 orders.
- Settings panel (hidden on "All stores") saves `perf_*` to `stores`; goal fields are "% on time" and "% without
  problems" (stored as `perf_alert_exc = 100 − input`). Test saves first, then calls `perf-check?force_weekly=1&store=`
  (no-cors, token from `app_settings`) so it sends the real weekly report to the saved `perf_webhook`.

**Refresh** (`loadData(full)`): full table load on open, "Refresh now", after Remove store and every 15 min
(`FULL_EVERY`, safety net); otherwise every 60s only rows with `last_updated >= shipCursor` (max `last_updated` in the data
minus 2 min), merged into `all` by `id`. Paused while the tab is hidden (`document.hidden`), refreshes on return
(`visibilitychange`). Built 2026-10-16 to keep egress low (free plan 5 GB; was 0.09 GB). Rows changed outside the webhook
without touching `last_updated` only show up on the next full load.

**Usage line** (`usage-line`, under the header buttons): "Database X/500 MB · Data this month ~Y/5 GB" (free-plan
limits). DB = RPC `usage_stats()` (sum of `pg_database_size` over all DBs, matches Supabase's figure). Data = estimate:
after each refresh `trackUsage()` gzips what was just fetched (shipments + `action_log` + `dismissed`) in the browser and
adds it via RPC `add_egress()` to `usage_egress` (per calendar month, all devices). Grey < 60%, orange < 80%, red above.
Supabase's own Usage page stays the source of truth (billing cycle ≠ calendar month, other egress not counted).
Header label is just "Live" (refresh details above).

**Manage stores modal** — tabs: Add (ParcelPanel setup instructions + webhook URL with token), Remove (type store name to confirm;
calls RPC `remove_store(p_store)` — SECURITY DEFINER, authenticated only — which deletes the store's rows in `shipments`,
`action_log`, `dismissed`, `perf_weekly_log`, `webhook_samples` and `stores` in one transaction. Before 2026-10-16 it did
separate client deletes, which silently removed 0 shipments (no DELETE policy) and the reload re-created `action_log` rows), Slack (per-store `slack_webhook_url`
save + Test, which sends 3 example alerts via `slack-notify`), Alerts (per-store not-shipped/delayed days).

`notifySlack()` is a no-op — notifications moved server-side into the `webhook` function. `buildBlocks()`
is still used by the Slack Test button and shows the alert format.

## Known quirks (not bugs to fix unless asked)

- Action thresholds (`alert_*_days`) are separate from the performance `perf_norm` (default 14).
- Delete puts the order in "Recently solved" for 48h — intentional (lets the merchant double-check a delete).
- `webhook` ignores dismissed/deleted orders: a deleted order can still get a Slack alert when it later enters
  a *new* section (e.g. a wrong label that ParcelPanel marks EXPIRED). Intentional — the merchant sees it as an
  extra "this is an old one" confirmation. Don't "fix" without asking.
- `testWebhook()` sets two status messages in a row ("Test meldingen verstuurd." then "Test melding verstuurd.").

User-supplied strings go through `esc()` before `innerHTML`, and through `jsq()` when passed as an argument in an
inline `onclick`. Keep doing that for new markup.

## Supplier Performance redesign (2026-10-15) — Slack reports + dashboard DONE (perf-check v20)

Decisions by the merchant (2026-10-15):
- Weekly report is purely about the week, compared with last week (a 30-day figure next to it was confusing).
  Monthly view = separate monthly report on the first Monday + the dashboard period selector.
- No advice/tips in the report ("Where to improve" removed): only facts from the data, neutral, not talking anyone down.
  Nothing may be made up or suggested: a line only appears when the comparison is fair. "Stuck orders mostly at
  carrier X" was removed because it ignores how many orders each carrier ships.
- Fewer than 20 orders → numbers shown but no colour/verdict.
- Positive framing: report shows "% without problems" (goal 95%) instead of "% delivery problems" (max 5%).
  Dashboard Performance settings input "Goal: % without problems" (e.g. 95) — saved as `perf_alert_exc = 100 − input`, so
  the DB column and `perf-check` keep the "max % problems" meaning (`9d5915b`).
- Dashboard actions don't change the % figures; only "Stuck right now" excludes deleted orders (see perf-check).
- Merchant wants minimal maintenance: the report must stay correct without manual work in the dashboard.

Step 2 (dashboard aligned + period selector) DONE 2026-10-15.
Keep-alive: free-plan projects pause after ~7 days of low activity (Supabase: "a few user requests to the database
each day"). pg_cron job `keep-alive` (`0 */6 * * *`) does `net.http_get` on `/rest/v1/stores?limit=1` with the public
publishable key — anon has no SELECT policy, so it returns `[]` (no data exposed) but counts as API activity.
Added 2026-10-16 because E&R is idle and the weekly cron alone isn't enough. Paused projects can be resumed within 90 days.

Status 2026-10-16: test data (TEST STORE, DEMO GOOD/ALMOST/BAD) fully removed; only Ellison & Rowe remains. E&R is idle
(last shipment 2026-06-01), so `perf-check` skips it (no weekly report on 19 Oct, no monthly on 2 Nov) until orders
flow again. Its `perf_webhook` still points at the old `#dayone-supply-chain` webhook — update when a store goes live.

Ideas for later (not started):
- Scale estimate (2026-10-16) for 3 stores × 2,000 orders/month: DB ±6 MB/month (≈6 years to 500 MB), function
  invocations fine, but egress grows because full loads fetch ALL shipments ever — roughly 5 GB/month after ~10–12
  months (rough; depends on compression and how long the dashboard is open). Fix when egress > 3 GB: dashboard loads
  only the last ~120 days by default; Supplier Performance "All" fetches older rows on demand.
- Separate idea: action section for `READY_FOR_PICKUP` (parcel at pickup point, may be returned).
- Separate idea: drop customer email from Slack alerts if the channel is shared with the supplier (privacy).
- Tracking links: BUILT 2026-10-15 (see `shipments.tracking_link` / `webhook_samples`). Still open: verify the
  real ParcelPanel payload fields once a new store sends its first webhook.

### Today-tab @25–@26 (2026-11-18)
- "Go to row" = RichText-link (`#gid=..&range=A..`), geen HYPERLINK-formule (NL-locale geeft #ERROR!).
- @26: één tabel met inklapbare rijgroepen per taak (Send 2nd mail / Send 3rd mail / Refund — day 8). Kopregel met aantal, daaronder rijen: Order #, Customer Name, Tab, Late, → Open. +/− staat links (BEFORE).
- @27 (2026-11-18): Today-tab op verzoek van de merchant helemaal weggehaald (setup verwijdert de oude tab). Next Step-kolom blijft (refreshNext_). solvedOn_ was in @26 per ongeluk weg (brak ook de 7-dagen-Solved-regel), teruggezet.
- @28 (2026-11-18): hele klantmail-functie (1st/2nd/3rd Mail, Next Step, CUSTOMER MAILS-balk) weer verwijderd op verzoek van de merchant. Code = stand van @23 + dropMails_() in Setup die de oude kolommen en Today-tab weghaalt.
- @29: dropMails_ steviger (breakApart rij 1-2, alle tabs, aantal in Setup-melding). Merchant bevestigd: kolommen weg.
- @30–@31 (2026-11-18): tabvolgorde RTC vóór Customer issues; kop Supplier Answer → "Supplier Update / Answer:" (key blijft supplier_answer).
- @32: tab "Customer issues" heet nu "Customer Order Issue" (RENAMED hernoemt de oude tab bij Setup; menu-tekst ongewijzigd).
- @33: Solved zonder blauwe FILLED IN BY CS-balk; Description/Evidence onder paars AUTOMATED. Solved houdt alle kolommen (merchant: niks verwijderen).
