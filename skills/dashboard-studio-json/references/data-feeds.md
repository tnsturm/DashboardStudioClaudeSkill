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

## System health, logs and timeline (run-tested 2026-11-04, bridge 5.2.8)

Sources for a "Homey system state" dashboard, all readable from ONE HomeyScript feed (no browser):

| source | HomeyScript call | notes |
|---|---|---|
| Simple (Sys) Log app (`nl.nielsdeklerk.log`) | `(await (await Homey.apps.getApp({id: 'nl.nielsdeklerk.log'})).get({path: '/'})).logs` | Array, newest first: `{id, timestamp (UTC ISO), severity, app, message}`. `severity` follows syslog (0 emergency … 3 error, 4 warning, 6 info); entries of the app itself ("Removed log data older than …") have none. Other paths (`/logs`, `/getlogs`) return 404. App settings: `removeLogOlderThanDays`, `groupInTimeline`. |
| Timeline | `Object.values(await Homey.notifications.getNotifications())` | `{excerpt, ownerUri, dateCreated}`; holds only the newest **250** entries. With Simple Log's `groupInTimeline: true` the log app can fill most of it (80 % in the test), so the timeline then covers only ~3 days. |
| System | `Homey.system.getInfo()`, `getMemoryInfo()`, `getStorageInfo()` | `rebootReason` (e.g. `unexpected`), `bootDate`, `uptime`, `videoCoreTemperature`, `videoCoreUndervoltageOccured` / `…ThrottleOccured` (since boot) and `…Currently`; memory and storage `types` per app (incl. swap). |
| System insights | see the feed example above | `loadavg`, `memory`, `temperature`, `undervoltageratio` as 7-day hourly histories (`resolution: 'last7Days'`, 168 points). |
| Apps | `Object.values(await Homey.apps.getApps())` | `state`, `enabled`, `crashed`, `crashedCount`, `crashedMessage`. |
| Devices | `Object.values(await Homey.devices.getDevices())` | `available`, `unavailableMessage`, `capabilitiesObj[cap].value / .lastUpdated` – enough for offline, low battery, active `alarm_*` and "silent for > 24 h". Inverters with a home battery report its state of charge as `measure_battery`; exclude them from low-battery warnings. |
| Radio | `Homey.zigbee.getState()`, `Homey.zwave.getState()` | ready flags, node lists. |

- **Not reachable:** the internal Homey system log (the developer log of Homey itself) – neither HomeyScript nor the bridge exposes it. Say so on the dashboard instead of leaving a gap.
- **Severity is only as good as the flows that write it.** In the test a flow logged routine fan on/off messages with severity 0 and a status message as "Pool Error"; give the feed an ignore list (sources and message patterns) and classify technical failures by message (`timeout`, `unreach`, `refused`, `5xx`, `crash`) rather than by severity alone.
- **Patterns beat raw lines:** group the problem entries of the last 7 days by source plus the first ~24 characters of the message with digits replaced, count them, and report the rhythm ("mostly 03:00", "only on 30.10."). Simple Log timestamps are UTC; convert to local time before computing hours and days.
- **Size:** 45 events incl. four 168-point histories (~7 KB each) stayed below the 16 KB limit of *State Caching: Optimized* per event. A cron Advanced Flow every 5 minutes (`{n: 5, type: "minute"}`) ran the feed reliably.

## Energy and fair comparisons from power insights (run-tested 2026-11-03)

Many devices report power (kW or W) but no energy counter. A feed can integrate the power insight itself:

```js
const H = 3600000;
const hr = await Homey.insights.getLogEntries({ id, resolution: 'last7Days' });   // hourly averages, 7 days
const sx = await Homey.insights.getLogEntries({ id, resolution: 'last31Days' });  // 6-hour averages, 31 days
const hStart = hr.values.length ? +new Date(hr.values[0].t) : Date.now();
const buckets = [];                         // {t: start, len: ms, kwh: energy over the whole bucket}
for (const p of sx.values) if (p.v != null && +new Date(p.t) < hStart) {
  const t = +new Date(p.t), len = Math.min(t + sx.step, hStart) - t; buckets.push({ t, len, kwh: p.v * len / H }); }
for (const p of hr.values) if (p.v != null) buckets.push({ t: +new Date(p.t), len: hr.step, kwh: p.v * hr.step / H });
// energy in [a, b): cut buckets count pro rata (assumes even use inside a bucket)
const sumRange = (a, b) => buckets.reduce((s, x) => { const ov = Math.min(x.t + x.len, b) - Math.max(x.t, a); return ov > 0 ? s + x.kwh * ov / x.len : s; }, 0);
```

- `v` is the bucket's average power; for a W insight divide by 1000. The running bucket is the average over the elapsed part, so `sumRange(midnight, now)` returns the energy so far.
- **Compare like with like** (design rule 4): "yesterday" for a comparison with "today so far" is `sumRange(yesterdayMidnight, yesterdayMidnight + (now - todayMidnight))`; "previous week, same point" is `sumRange(prevWeekStart, now - 7 days)`. An independent check on hourly values matched to 0.01 kWh.
- Older than 7 days only 6-hour buckets exist, so day boundaries are pro-rated estimates there.
- Compute local midnights with `Intl.DateTimeFormat(…, {timeZone})`, not with fixed UTC offsets (DST).
- Format numbers in the feed for the reader's locale (`toLocaleString('de-DE', …)`); formulas in text widgets only produce `.` decimals.
- A ratio such as COP is energy-weighted (`Σ heat / Σ electricity`), not the mean of the live ratio, which is 0 whenever the device is idle.

## Flow cards (verified v2.3.1, read from `Homey.flow.getFlowCard*` on a Homey Pro; unchanged since v2.2.0)

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
- Since v2.3.0 published calendar links from Outlook.com and Microsoft Exchange work as `url` too; recurring events keep their times across DST changes.
- Several calendars: load serially (A → B → `combine_ical_data` → convert). Parsing still needs noticeable memory on Homey.
