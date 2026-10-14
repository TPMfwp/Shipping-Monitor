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

## Data model (Supabase tables)

### `shipments` — one row per order/tracking, written by the `webhook` function
Columns used by the dashboard: `order_number`, `customer_name`, `tracking_number`, `status`, `store_name`,
`order_date`, `fulfillment_date`, `last_updated`, `last_checkpoint_time`.
`status` is a ParcelPanel status: `PENDING, INFO_RECEIVED, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERED,
EXCEPTION, EXPIRED, FAILED_ATTEMPT, CANCELLED, UNKNOWN, READY_FOR_PICKUP, RESOLVED`.

### `action_log` — orders that entered an action section (primary key `id` = `rowId`)
`id, order_number, customer_name, tracking_number, store_name, order_date, fulfillment_date, section,
resolved_at, resolved_status, slack_notified` (+ server-side `slack_notified_sections` array).
- `section`: `label_not_shipped` | `delayed` | `exception` | `deleted` (permanent delete marker).
- `slack_notified_sections` (array) is maintained by the `webhook` function: each order gets **one Slack
  message per action section, ever**. The dashboard itself still writes the legacy boolean `slack_notified:false`
  on insert and does not touch the array.

### `dismissed` — orders the user dismissed from the action sections (primary key `id` = `rowId`)
Snapshot of the shipment + `dismissed_at`. Restore deletes the row (and its `action_log` row).

### `stores` — per-store settings (primary key `store_name`)
`slack_webhook_url` (shipping alerts), `perf_norm` (on-time norm in days, default 14), `perf_webhook`
(performance alerts), `perf_alert_ontime` (alert if on-time % below), `perf_alert_exc` (alert if exception % above).

### `perf_weekly_log`
Records which weeks have already had the weekly performance Slack message, so Monday runs don't double-send.

## Edge Functions (Supabase)

| Function | Role |
|---|---|
| `webhook` | Receives ParcelPanel `shipment_status / any_update` webhooks → upserts `shipments` → checks action sections → sends Slack alert via `slack-notify` (once per section, tracked in `action_log.slack_notified_sections`). On Mondays also triggers the weekly performance update (deduped via `perf_weekly_log`). |
| `slack-notify` | Thin proxy: `POST {webhook_url, blocks, text}` → posts to the Slack incoming webhook. Also called directly from the dashboard for "Test" buttons (browser can't post to Slack directly due to CORS). |
| `perf-check` | Computes performance vs. the per-store thresholds in `stores` and sends alerts to `perf_webhook`. |
| `Import` | Old Shopify+ParcelPanel bulk import used by `import.html`. **Inactive.** |

New stores connect by adding a ParcelPanel webhook (Professional plan) pointing at
`https://frxyfmfjtadyxhfogdvt.supabase.co/functions/v1/webhook`; the store appears after the first update.

> The server-side section logic must stay in sync with `getSection()` in `index.html`. When changing
> thresholds, change both (and check `perf-check`).

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
- **1 Label created, not shipped**: `PENDING`/`INFO_RECEIVED` and days ≥ 5.
- **2 Delayed in transit**: `IN_TRANSIT`/`OUT_FOR_DELIVERY` and days ≥ 15.

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

**Manage stores modal** — tabs: Add (ParcelPanel setup instructions), Remove (type store name to confirm;
deletes that store's rows in `shipments`, `dismissed`, `action_log`), Slack (per-store `slack_webhook_url`
save + Test, which sends 3 example alerts via `slack-notify`).

`notifySlack()` is a no-op — notifications moved server-side into the `webhook` function. `buildBlocks()`
is still used by the Slack Test button and shows the alert format.

## Known quirks (not bugs to fix unless asked)

- Action thresholds (5/15 days) are hardcoded and separate from the configurable `perf_norm` (default 14).
- `testWebhook()` sets two status messages in a row ("Test meldingen verstuurd." then "Test melding verstuurd.").
- Table "days" colouring uses 7/15 thresholds, section rows use 5/15.
- User-supplied strings (customer names, store names) are interpolated into `innerHTML` without escaping.
