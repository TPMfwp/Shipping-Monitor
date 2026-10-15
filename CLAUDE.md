# Shipping Monitor — context for Claude

Multi-store shipment tracking dashboard. ParcelPanel tracking updates flow into Supabase; a static
dashboard on GitHub Pages reads them and shows which orders need action; Slack gets notified once per
order per action section.

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
`order_date`, `fulfillment_date`, `last_updated`, `last_checkpoint_time`, `tracking_link`.
`tracking_link` = the store's own tracking page, set by `webhook` (`findTrackingLink()`: payload `tracking_link`/
`tracking_url`/`tracking_page_url`, else `https://<store.domain>/apps/parcelpanel?nums=`), never overwritten with
null. E&R rows were backfilled with `ellisonrowe.com/apps/parcelpanel`. Dashboard `trackHtml()` and Slack alerts
fall back to `https://t.17track.net/en#nums=<tracking>` when it's empty. When a store link exists, both are shown
(dashboard: number → store page + small "17track" link; Slack: number + "Store page · 17track"). Customers never see
these links — they only get ParcelPanel's own tracking page via the store.
`status` is a ParcelPanel status: `PENDING, INFO_RECEIVED, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERED,
EXCEPTION, EXPIRED, FAILED_ATTEMPT, CANCELLED, UNKNOWN, READY_FOR_PICKUP, RESOLVED`.

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

### `perf_weekly_log`
Records which weeks have already had the weekly performance Slack message, so Monday runs don't double-send.

### `app_settings` — key/value config
`webhook_token`: secret that `webhook` and `perf-check` require as `?token=`. Readable only after login, so the
dashboard loads it at runtime (`loadWebhookUrl()`) — **never hardcode it in `index.html`** (public repo + public
Pages). Rotating it = update the row, then re-paste the URL in ParcelPanel for every store.

## Edge Functions (Supabase)

| Function | Role |
|---|---|
| `webhook` | Receives ParcelPanel `shipment_status / any_update` webhooks → rejects with 401 unless `?token=` matches `app_settings.webhook_token` → upserts `shipments` → checks action sections → sends Slack alert via `slack-notify` (once per section, tracked in `action_log.slack_notified_sections`). |
| `slack-notify` | Thin proxy: `POST {webhook_url, blocks, text}` → posts to the Slack incoming webhook. Requires either `?token=` (used by `webhook`/`perf-check`) or a logged-in session `Authorization: Bearer` (dashboard `sendSlack()` for the "Test" buttons; browser can't post to Slack directly due to CORS). Only accepts `https://hooks.slack.com/` URLs. |
| `perf-check` | Requires `?token=`. Monday 07:00 UTC via pg_cron `perf-check-weekly` (exits early on other days). Sends ONE "Delivery report" per store (weekly, measured over the last 30 days; context line says so) to `perf_webhook` (no separate alert any more): header block, goals context line, verdict (🟢 on track / 🟡 almost / 🔴 off track / ⚪ <20 orders — worst of the two metrics), On time + Delivery problems with "1 in N orders" wording and "X% better/worse than last week", Where the time goes (processing order→shipped vs transit shipped→delivered), Open problems by type, "What stands out" and "Where to improve" (rule-based, max 3 lines, fixed tip per problem type in `PROBLEM_INFO`). Last 30 days; on time/avg over orders delivered in the period, problems over orders shipped in the period that ever had a problem status (incl. `action_log` exception history). Snapshot per week stored in `perf_weekly_log` (ontime/problem %, avg/proc/transit days, counts) for next week's comparison and "best in N weeks". Test params: `&store=<name>&force_weekly=1` (sends outside Monday, never writes a snapshot). |
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

**Supplier Performance** (`renderPerformance()`, per store or all):
- delivery time = `last_checkpoint_time − fulfillment_date` for DELIVERED rows, kept if 0–120 days.
- Avg delivery, On time % (≤ `perf_norm`), Exceptions % (of rows with a known status), Delivered count,
  this-month vs last-month avg delta.
- Settings panel (hidden on "All stores") saves `perf_*` to `stores`; Test sends a sample block to `perf_webhook`.

**Manage stores modal** — tabs: Add (ParcelPanel setup instructions + webhook URL with token), Remove (type store name to confirm;
deletes that store's rows in `shipments`, `dismissed`, `action_log`), Slack (per-store `slack_webhook_url`
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

## Supplier Performance redesign (2026-10-15) — step 1 (Slack/perf-check) DONE, step 2 (dashboard) and step 3 (skip weekly when idle) open

Current calc problems (dashboard `renderPerformance()` and `perf-check`, duplicated):
1. All-time figures → barely move, weekly message ~identical every week, alerts late or stuck on.
2. "This month" uses only already-delivered orders → slow ones still in transit, so it always looks better.
3. Exception % counts only orders whose *current* status is an exception; resolved ones drop out. `action_log`
   (`section='exception'`) has the history. `EXPIRED` isn't always a real problem.
4. Mean instead of median; a few 60-day orders skew it.

Options discussed:
- Period: per week alone is too noisy (E&R had 2–224 deliveries/week). Leaning towards: alerts on rolling
  30 days (by delivery date), weekly Slack message showing last 7 days + last 30 days with order counts,
  7/30/90/all selector in dashboard, minimum ~20 delivered orders before an alert fires.
- Exception alert stays an absolute per-store % (not deviation from average) — per-store thresholds already cover that.
- Rewrite performance Slack texts for the supplier: explain "on time" as "delivered within N days", hide
  internal "threshold" wording or phrase it as agreed target, add a request ("Could you look into…"), use
  period counts instead of all-time "Delivered: 1151". Drafts were shown to the merchant in-session.
- Weekly message currently sends every Monday even when a store is idle — maybe only send if something was delivered.
- Separate idea: action section for `READY_FOR_PICKUP` (parcel at pickup point, may be returned).
- Separate idea: drop customer email from Slack alerts if the channel is shared with the supplier (privacy).
- Tracking links: BUILT 2026-10-15 (see `shipments.tracking_link` / `webhook_samples`). Still open: verify the
  real ParcelPanel payload fields once a new store sends its first webhook.
