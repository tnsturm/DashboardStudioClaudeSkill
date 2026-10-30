# Dashboard Studio – Data feeds and flow cards

Feeding non-device data (system values, insight histories) and the app's flow cards including calendar handling. Read when a value is not a device capability or a flow card is needed.

## Feeding non-device data (system values, insight histories)

- **"Send Insight" flow card topic (since v2.2.0):** the card has an optional `topic` argument. Without it all time ranges of one insight end up on the SAME topic (`homey/insight/<log name>`); with it the card publishes to `homey/insight/<topic>`, so a 24 h and a 7 day graph of one insight can now coexist (e.g. topics `pv_24h`, `pv_7d`). Before v2.2.0 this needed the HomeyScript feed below. The card "Get Insight Data (Advanced)" only returns tokens (`insight_raw`, `insight_highest`, `insight_lowest`, `insight_average`).
- **Proven pattern: one HomeyScript as data feed.** It reads values and pushes each one as an event:
  ```js
  const DS = 'homey:app:com.satoer.dashboardstudio';
  const send = (name, value) => Homey.flow.runFlowCardAction({uri: DS, id: DS + ':dashboard_send', args: {name, value: String(value)}});
  const info = await Homey.system.getInfo();          // loadavg, cpus, totalmem/freemem, videoCoreTemperature, uptime, undervoltage flags
  const mem  = await Homey.system.getMemoryInfo();    // total, free, swap, types{ 'homey': {name,size}, 'homey:app:<id>': {name,size} }
  const stor = await Homey.system.getStorageInfo();   // total, free, types
  const e = await Homey.insights.getLogEntries({uri: 'homey:manager:system', id: 'homey:manager:system:loadavg', resolution: 'last24Hours'});
  await send('h_cpu_24h', JSON.stringify(e.values.map(p => ({t: p.t, v: Math.round(p.v * 10) / 10}))));
  ```
  Insight `id` must be the FULL id (`homey:manager:system:loadavg`), the short id fails with "Not Found: LogLocal". System insights: `loadavg` (CPU %), `memory` (MB, Homey process memory, not total RAM), `temperature` (°C), `clock` (GHz), `undervoltageratio` (%). Precompute min/avg/max in the script and send them as separate events for legends and scales.
- **Test without persistence:** `homeyscript_run` with inline `code` (a real run only pushes events to the dashboard socket; `dry_run: true` lists the `runFlowCardAction` calls instead). Then read the arrival in the probe call.
- **Persist only after confirmation:** `homeyscript_create` + a trigger flow. Current values every minute, histories every 5 minutes (check `new Date().getMinutes() % 5` in the script).
- **Use Advanced Flows.** A basic flow created via `flows_create` runs, but the Homey web app showed it as an EMPTY canvas without card menu (observed on a Homey Pro 2026 where all other flows were Advanced Flows). Create the trigger with `advanced_flows_create` (cron `homey:manager:cron:every_nth` `{n: 1, type: "minute"}` → `homey:app:com.athom.homeyscript:run` with the autocomplete script object), and verify with `homeyscript_last_runs` after ~70 s.

## Flow cards (verified v2.2.0, read from `Homey.flow.getFlowCard*` on a Homey Pro)

| card id (`homey:app:com.satoer.dashboardstudio:…`) | kind | purpose / arguments |
|---|---|---|
| `dashboard_send` | action | "Send to Dashboard Studio": `name`, `value` → topic `homey/event/<name>` |
| `dashboard_send_insights` | action | "Send Insight": `insight_id`, `time_range` (`today`, `last_hour`, `last_6_hours`, `last_24_hours`, `last_7_days`, `last_30_days`, `last_year`), `datapoints` (2–500, default 20), `topic` (optional, since v2.2.0) |
| `dashboard_send_insights_advanced` | action | "Get Insight Data (Advanced)": same arguments incl. `topic`; also sends, and returns tokens `insight_raw`, `insight_highest`, `insight_lowest`, `insight_average` |
| `fetch_rss_markdown` | action | "Fetch RSS Feed to Markdown" (since v2.2.0): `urls` (one per line), `layout`, `image_width`, `max_items`, `date_format`, `topic` → `homey/event/<topic>`; tokens `markdown`, `article_count`, `latest_headline`, `latest_link`, `latest_summary`, `latest_image_url`. Caches feeds 5 min. Dropdown values not yet verified – read them from the card |
| `dashboard_reload` | action | "Reload Dashboard": `dashboard_name` |
| `load_filter_ical` | action | "Load & Filter iCal": `url`, `target_mode`, `custom_start`, `custom_end` → token **Filtered Events (JSON)** |
| `webcal_to_markdown` | action | "Convert iCal JSON to Markdown": `format`, `topic` (custom topic, since v2.2.0) → token **Calendar Markdown** |
| `combine_ical_data` | action | "Combine Calendar Events": `events_token_a`, `events_token_b` |
| `dashboard_trigger` | trigger | "Dashboard Studio event triggered": `name`. Fires when a widget publishes `homey/event/<name>` or a topic WITHOUT `/` (auto-prefixed, `drivers/homey.js`) |
| `dashboard_loaded` | trigger | "Dashboard Loaded / Refreshed" – since v2.0.13 no longer on browser tab switches |
| `dashboard_saved` | trigger | "Dashboard Saved" |
| `dashboard_page_opened` | trigger | "Dashboard opened page" (since v2.2.0): `dashboard_name` (autocomplete), `page` (number); tokens `dashboard_name`, `page`. Good trigger for heavy feeds that only one page needs |
| `dashboard_is_open` | condition | "Dashboard is open" (since v2.2.0): `dashboard_name` |
| `dashboard_is_open_page` | condition | "Dashboard is open on page" (since v2.2.0): `dashboard_name`, `page` – use it to skip feeds nobody is looking at |

**Calendar (behaviour change in v2.0.13):** `webcal_to_markdown` is always anchored to **today** and shows only days contained in the loaded range. Setups where loader range = displayed range can lose days (e.g. "Yesterday" on the 1st of a month); HomeyScripts that parse the JSON token may break.
- Pattern: ONE `load_filter_ical` with `target_mode: "previous_current_next_month"`, then several `webcal_to_markdown` cards side by side, each to its own event (`homey/event/cal_today`, `homey/event/cal_week`, …), shown in `text` widgets.
- `target_mode`: `previous_current_next_month` (recommended), `today`, `yesterday`, `tomorrow`, `day_after_tomorrow`, `start_this_week`, `start_previous_week`, `start_next_week`, `start_this_month`, `start_previous_month`, `start_next_month`, `custom`.
- `format`: `day_view`, `tomorrow_day_view`, `yesterday_day_view`, `today_tomorrow_list`, `week_view`, `next_week_view`, `previous_week_view`, `workweek_view`, `next_workweek_view`, `previous_workweek_view`, `weekend_view`, `next_weekend_view`, `month_view`, `next_month_view`, `previous_month_view`, `month_workweek_view`, `next_month_workweek_view`, `month_weekend_view`, `next_month_weekend_view`, `days_from_date_1` … `days_from_date_7`.
- Several calendars: load serially (A → B → `combine_ical_data` → convert). Parsing still needs noticeable memory on Homey.
