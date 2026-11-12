---
name: "dashboard-studio-json"
description: "Builds, changes and repairs importable JSON dashboards for the Homey app Dashboard Studio (com.satoer.dashboardstudio) via the MCP AI Bridge. Use when the user wants a Dashboard Studio dashboard, widget, data feed or layout fix. Only for this app – not for the bridge's own dashboard or native Homey dashboards."
---

# Dashboard Studio – generating JSON dashboards

**Skill version 2.3.5** – matches the Dashboard Studio version it was verified against (`com.satoer.dashboardstudio` 2.3.5).
**Requires MCP AI Bridge** (`community.mcp-ai-bridge`) – tested with **6.1.0** (test channel) and 5.3.0. If the installed Dashboard Studio or bridge version differs, mention it to the user; for a different bridge version check the bridge-dependent steps (tool names, `homeyscript_run` with `dry_run`, `apps_get_settings`).

Applies ONLY to the Homey app **Dashboard Studio** by Satoer (app ID `com.satoer.dashboardstudio`). Do not use it for the MCP AI Bridge's own dashboard (`dashboard_generate`) or for native Homey dashboards (`homey_dashboards_*`).

Knowledge verified against Dashboard Studio **v2.3.5** (test channel, released 2026-11-07) on a Homey Pro (2026), checked from the app's own source and release notes via HomeyScript (v2.3.3–2.3.5 changes on 2026-11-12, not yet run-tested). Run-tested in the editor on v2.3.1 (2026-11-03: report dashboard with two-series bar graph, floating legend, fixed-size text figures, data feed via `homey/event/…`) and on v2.3.2 (2026-11-04: four-page system-health dashboard with a two-series 7-day line graph, horizontal bars with text labels, grouped bars, two `stepped` graphs, a page navigation and one HomeyScript feed with 45 events). If the installed version is newer, read the app's own release notes first (`release-changelog.json`, see "Source research without the browser" in `references/source-research.md`) and re-check anything they touch.

Tool names below are given without their MCP prefix (e.g. `apps_get` = `<prefix>homey-bridge__apps_get`, `javascript_tool` = `<prefix>Claude_Browser__javascript_tool`).

## Reference files (read only what the task needs)

| file | read when |
|---|---|
| `references/design.md` | before planning the pages and widgets of a new dashboard or reworking one (page plan, hierarchy, labels, colors, limits, design check) |
| `references/browser-calls.md` | before the first browser call (opening the editor, discovery, probe, commit, demo fallback) |
| `references/widgets.md` | before generating or editing the dashboard JSON (schema, widget fields, graphs, text formulas, bindings, colors) |
| `references/data-feeds.md` | a value is not a device capability, or flow cards / calendars are involved |
| `references/source-research.md` | after an app update, for unknown widget fields, the Auto-Widget Gallery, or variables/flows/moods |

## Changes in v2.3.2 – v2.3.5 (take precedence over the reference files if they still say otherwise)

- **Graph widget, new fields** (details in "Graph widget" in `references/widgets.md`): stacked bars (`barGroupMode: "stacked"`, `barStackGapStep`), axis titles (`axisXLabel`, `axisYLabel` + typography), line/bar opacity per series (`positiveColorOpacity`, `negativeColorOpacity`), `abbreviateLargeValues` (default on: 12.5k), point value distribution (`pointValuesDistribution`, `pointValuesStep`, `pointValuesTimeWidth`), labels of negative values below the point (`pointValInvertYNegative`).
- **Graph keys renamed:** `categoryLabelRotation`/`valueLabelRotation` → `axisXRotation`/`axisYRotation` (not migrated), `timezoneConvert` → `convertToLocalTime` + `timeOffset`, `pointValuesInterval` → `pointValuesStep`/`pointValuesTimeWidth`. `LATEST_MIGRATION_VERSION` is still 2.3.0.
- **Time zone (2.3.3):** graphs treat timestamps as UTC and convert to local time by default (`convertToLocalTime: true`; before, the old `timezoneConvert` defaulted to off). Feeds must send UTC ISO timestamps; a graph that runs an hour ahead or behind gets local times without offset.
- **2.3.3:** setting tooltips in the editor; Freeze Dynamic Values also freezes `{{…}}` placeholders in text widgets; smoother streaming graphs; fewer redraws on kiosk screens. **2.3.4:** image auto-refresh and camera snapshots update again.
- The changelog shipped with 2.3.5 already lists **2.4.0** (regional/language settings, `countdown()` formula). Neither is in the 2.3.5 code – do not use them before the installed `version.js` says 2.4.0.
- Unchanged (checked on 2.3.5, 2026-11-12): flow cards (14, see `references/data-feeds.md`) and readable app settings (`enabled_capabilities`).
- **MCP AI Bridge 5.3.0 / 5.2.9:** each API key only sees and uses its own tool selection, also for scheduled tasks and approved plans. With strict guardrails on a lock rule, `homeyscript_run` is refused – dry runs included – because a script can reach any device. Then the source research and data-feed tests in this skill need the browser instead, or the user relaxes strict mode for that rule. 
- **MCP AI Bridge 6.1.0** (test channel, tested 2026-11-12): every tool this skill uses works unchanged – `system_get_info`, `apps_get`, `apps_get_settings`, `apps_set_setting`, `homeyscript_run` incl. `dry_run` (writes are listed in `would_have`), `devices_list`, `devices_get_capability`, `insights_list`, `insights_get_entries`, `homeyscript_create`, Advanced Flows. Per its author 6.x puts an API key on every install. `insights_get_entries` needs the FULL log id from `insights_list` (`homey:device:<id>:measure_temperature`); the short `measure_temperature` fails with "Not Found: LogLocal". It returns every raw point (720 per hour at 5 s resolution), so fetch histories in a HomeyScript feed or dry run and send only what the graph needs.
- **The bridge's own skill `homey-dashboards`** (`skills_list`, since 6.x) covers only the bridge's web dashboard (`dashboard_generate`) and native Homey dashboards. When the user means Dashboard Studio, follow THIS skill, not that one; ask when it is unclear which of the three is meant.

## Changes in v2.3.2 and run-test findings of 2026-11-04 (still valid)

- **2.3.2 (release notes):** fixed page switches, visibility rules and audio triggers on bare Homey event topic names (aliases); faster dashboard load and group repaint; settings-reset and inspector fixes. No change to the JSON schema, the graph widget, the flow cards or `LATEST_MIGRATION_VERSION` (still 2.3.0). Keep writing full `homey/event/<name>` topics.
- **Text size and color only via the Markdown keys.** The text widget deletes `fontSize`, `textColor` and `lineHeight` from its config (`widgets/text.js`, verified 2.3.2): JSON with these keys renders in the default 12 px. Use `textMdParaFs` / `textMdParaCol` / `textMdParaLineHeight` (paragraphs, lists, bold lines), `textMdH1Fs`… for headings and `textMdThFs` / `textMdTdFs` for tables (details in "Text size" in `references/widgets.md`).
- **Graph run-test on v2.3.2** (details in `references/widgets.md`): two `line` series with `bufferMode: "duration"` + `durationPreset: "7d"` (168 hourly points each), horizontal bars from `[{l, v}]` with `timeFormat: "TextLabels"`, grouped vertical bars with two series, `stepped` with `dynamicFloor/dynamicCeiling: false` + `fixedFloor: 0` / `fixedCeiling: 5` all render. A `top` or `bottom` legend again landed outside the widget; leaving ~35 px free below the graph fixed it.
- **Simple (Sys) Log and timeline as a data source** (details in `references/data-feeds.md`): the app `nl.nielsdeklerk.log` answers `Homey.apps.getApp({id}).get({path: '/'})` in HomeyScript with all entries incl. severity; the timeline is `Homey.notifications.getNotifications()` (max. 250 entries). The internal Homey system log is not reachable via HomeyScript or the bridge.
- **Fewer approvals for the visual check:** lay all pages side by side on one oversized canvas and take ONE screenshot ("Contact sheet" in `references/browser-calls.md`).

## Changes in v2.3.0 / v2.3.1 (still valid)

- **Graph widget rewritten** (details in "Graph widget" in `references/widgets.md`): several series per widget in `overrides.series`, data bound as nested `bindings.series[i].data`; new type `stepped`, horizontal bars, legend (on by default), time-window buffer, value multiplier, animations, axis font opacity. Old top-level keys (`lineColor`, `fill`, `fillOpacity`, `lineWidth`, `linearTimeScale`, `bindings.data`, …) are dropped; old dashboards are migrated on load (migration rule 2.3.0), new JSON must use `series`. Scale keys are now `dynamicFloor`/`dynamicCeiling`/`fixedFloor`/`fixedCeiling` (old `autoMin`/`min`… are still read as fallback, but writing them makes the file count as legacy: loading it runs the migration and opens a blocking "Dashboard Upgraded" notice). The dynamic floor now fits the data instead of starting at 0.
- **Run-test findings (2026-11-03, v2.3.1):** the graph legend is drawn *outside* the widget for `top`/`bottom`/`left`/`right` and covers its neighbours; place it with `legendPosition: "floating"` + `legendOffX`/`legendOffY` (canvas coordinates) or reserve space. A "Dashboard Upgraded" notice (old `dashboard.json` at editor start, or a draft with a legacy key) blocks `loadDashboardData` and keeps the data driver offline until OK is clicked. Widget DOM is in `widget.elements.container`. Details in `references/widgets.md` and `references/browser-calls.md`.
- **Nested bindings:** collect topics recursively for the topic check and the probe call (done in `references/browser-calls.md` and `references/design.md`).
- `LATEST_MIGRATION_VERSION` is **2.3.0**; take the JSON `version` from `version.js`.
- **Swipe gestures (beta):** environment keys `gestureTouchOnly`, `swipe{Left,Right,Up,Down}{Enabled,Action,Page,Topic,Payload}`, `swipeIgnorePages`, `swipeWrapAround`; actions `next_page`, `prev_page`, `specific_page`, `topic` (from `master-layout.js` / `dashboard.js`, not run-tested). Let the user set them in the editor unless their location in the saved JSON has been checked.
- Fixed: widgets with a `page` bound to a topic move as soon as the value arrives; custom canvas sizes survive upgrades; arrow and line colors with light/color topics; analog clock at midnight. New: off-canvas editing, Outlook.com/Exchange calendar links, Widget Info tab.
- **2.3.1:** fixed slow or missing updates of widgets and feeds fed by Homey flow events (`homey/event/…`). On 2.3.0 an event that seems not to arrive may be this bug, not a wrong topic.
- Unchanged (checked 2026-11-03): flow cards (14, see `references/data-feeds.md`), readable app settings (`enabled_capabilities`), editor port 5050, topic naming.

## Changes in v2.2.0 (still valid)

- **Send Insight** (`dashboard_send_insights`, `…_advanced`) has an optional `topic` argument → `homey/insight/<topic>`. Several time ranges of one insight can now coexist; the HomeyScript feed is no longer needed just for that.
- **`webcal_to_markdown`** also takes a custom `topic`.
- New flow cards: action `fetch_rss_markdown` (`urls`, `layout`, `image_width`, `max_items`, `date_format`, `topic` → `homey/event/<topic>`; Markdown table for a `text` widget), trigger `dashboard_page_opened` (`dashboard_name`, `page`), conditions `dashboard_is_open` (`dashboard_name`) and `dashboard_is_open_page` (`dashboard_name`, `page`). Use the conditions to skip heavy feeds while no dashboard is open.
- Variables/flows/moods use canonical `homey/variables|flows|moods/…` topics with legacy aliases (see "Topic names"); source in `topic-alias.js`.
- New app settings: **State Caching** (Full / Optimized) and **Capability & Topic Cleanup** (see "Pitfalls"). Calendar settings tab renamed to **Flow Cards**.
- Fixed: all-day calendar events no longer shift a day at a DST change (author: not fully verifiable until the next DST change).
- Unchanged in v2.2.0: JSON schema, widget types, import/export, device topic naming, editor port 5050, `LATEST_MIGRATION_VERSION` 2.0.0.

## Browser budget – the most important rule

On the user's LAN address the built-in browser asks the user for approval **for every single browser call** (load, JavaScript, screenshot, click). An entry in the user's allow list does not change that; even `request_access` with `scope: "site"` is answered with "ONE load or action". Every browser call therefore costs the user one click. Rules:

1. **Do everything that does not need the browser via the bridge first**: devices, capabilities, insights, flows, HomeyScript tests, data-feed tests (`homeyscript_run`), app settings. None of this asks the user.
2. **Do all research and the layout preparation in the cloud workspace** (Python build script, topic check against the saved topic list, gzip+base64 encoding) before the next browser call.
3. **One browser call = one complete job.** Never split "load", "wait", "check" and "read back" into separate calls. Never use `read_page`, `get_page_text`, `find` or `form_input` in the editor – do the same in the JavaScript call. Use `browser_batch` only when a screenshot must follow a JS step directly (each item still asks).
4. **Check rendering in JavaScript, not by eye** (overflowing text, `-` formula errors, missing live values, resolved widget config – see "Probe call" in `references/browser-calls.md`). The JavaScript checks miss layout defects (a legend outside its widget, a table one row too short), so ONE full `screenshot` at `scale` 0.6–0.7 before handing over is mandatory. Prefer it to `zoom`: the coordinate frame changes when the pane is resized, and a zoom region planned from an older screenshot can miss the canvas.
5. **Do source-code research without the browser** via HomeyScript (see "Source research without the browser" in `references/source-research.md`). Only if that is impossible, fold it into the discovery or probe call.
6. **Save in the same call that does the final check** once the user has confirmed name and saving (see "Commit call" in `references/browser-calls.md`).

Target for a new dashboard: **4 browser calls** – 1 open editor, 1 discovery call (switch to Homey platform + read topics + optional source research), 1 probe call per layout iteration, 1 screenshot, 1 commit call. Tell the user up front roughly how many approvals to expect. Each `request_access` plus the following call counts as one approval. Realistic in the first v2.3.1 run: 14 approvals for a new dashboard (half-loaded page, upgrade notice, one probe timeout, two layout iterations) and 5 for a later layout change – the discovery and probe snippets in `references/browser-calls.md` now handle those cases.

## Step 0 – Preconditions (always run first, bridge only)

1. **MCP AI Bridge reachable?** Check that Homey bridge tools are available in this session, then call `system_get_info`. If the tools are missing or the call fails, stop and tell the user: the Homey app **MCP AI Bridge** (`community.mcp-ai-bridge`) must be installed, running and connected to Claude.
2. **Bridge app running?** `apps_get` with `community.mcp-ai-bridge` → `state` must be `running` and `enabled: true`. Report the version.
3. **Dashboard Studio installed?** `apps_get` with `com.satoer.dashboardstudio` → `state: running`. Note `version`. If it is not installed, tell the user to install it from the Homey App Store and stop.
4. **Find the editor URL.** First `apps_get_settings` on `com.satoer.dashboardstudio` and look for any URL, host or port key. Up to v2.2.0 the readable settings hold only `enabled_capabilities` (a Personal API Key, if set, appears as `<redacted>`), so build it from `system_get_info`: `ethernetAddress` (or `wifiAddress` when ethernet is not connected) plus port **5050** (verified default). Editor `http://<ip>:5050/editor.html`, viewer `http://<ip>:5050/viewer.html?dashboard=<name>`.
5. **Browser available?** The editor runs on a private LAN address, so it is reachable only from a browser on the user's network. Use the **built-in Claude browser** (`Claude_Browser__*` tools; load them all with one ToolSearch on the full prefix, and read the `built-in-browser` skill first). Claude in Chrome also works when it is connected, but it has proven unstable (background tabs block screenshots, the extension disconnects). If neither browser is reachable, fall back to handing over the JSON for manual import.

## Workflow

0. **Agree on the target picture** for a new dashboard or a rework (sections 1–2 of `references/design.md`): who looks at it, on which device, and what they should do afterwards; then a page plan with one job per page. Get the page plan approved before generating JSON. Skip this for small fixes to a single widget.
1. **Collect devices and values** via the bridge (`devices_list`, `devices_get`, `devices_get_capability`, `insights_list`, `insights_get_entries` with the full log id): available capabilities, units, sign conventions (e.g. grid meters often report export as negative `measure_power`), sub-capabilities such as `measure_power.pv1input`, value ranges.
2. **Enable capabilities in Dashboard Studio.** Without this the app sends no data for them. They are stored in the app setting `enabled_capabilities` = `{ "<deviceId>": ["cap", "cap.sub"] }`. Read the current value with `apps_get_settings`, MERGE (never drop existing entries), show the user what will be added, and write with `apps_set_setting` only after the user confirms. The app picks up changes immediately.
   For values that are not device capabilities (system data, insight histories, computed values) use a data feed instead – see "Feeding non-device data" in `references/data-feeds.md`.
3. **Read the exact topic names from the editor – MANDATORY** in ONE discovery call (see "Discovery call" in `references/browser-calls.md`), or ask the user to paste the Data Stream Explorer list. Do NOT generate the JSON before this list is available. Never derive, guess or construct device topic names (not from device IDs, not from device names). Every device topic used must appear verbatim in that list. Save the list to a file in the workspace for the topic check.
4. **Generate the JSON** (schema in `references/widgets.md`, layout rules in `references/design.md`) with a script, and run the topic check ("Topic names") and the static design check ("Automatic checks" in `references/design.md`).
5. **Test in the user's editor** with ONE probe call per iteration, fix the build script, repeat. Then the mandatory screenshot; fix what it shows and add a static check for it, so it cannot come back.
6. **Hand over for approval.** Show the screenshot and ask whether and under which name to save. Never click *Save to Homey* without explicit confirmation in this session. Also deliver the JSON file.
7. **Save** only after confirmation with ONE commit call, which also verifies.

## Topic names (critical)

- The only valid source for device topic names is the Data Stream Explorer / live topic list of the user's editor. Copy names verbatim (case, dots and hyphens included).
- Background only, not for building names: topics have the form `<slug of the device name>/<capability>`. The slug is lower-case, umlauts and other special characters become `-`, and trailing spaces in the device name produce a trailing `-` (e.g. device "Wetter Poolhütte " → `wetter-poolh-tte-/measure_co2`). Renaming a device can change its topic, so after a rename read the list again.
- If a needed value is missing from the list, the capability is not enabled yet: go back to workflow step 2.
- **Variables, flows and moods (since v2.2.0):** canonical topics are `homey/variables/<name>/value`, `homey/flows/<name>/start` (or `/enabled`) and `homey/moods/<name>/set`. The pre-2.2.0 names (`<name>/value`, `<name>/start`, `mood-<name>/set`) keep working as aliases (`topic-alias.js`); the live topic list contains BOTH forms. Use the canonical `homey/…` form in new dashboards, never mix both forms for the same item, and leave existing dashboards as they are unless the user asks.
- Other topic kinds: `local/<name>` lives only inside the browser and is useful for UI wiring between widgets. `homey/event/<name>` is sent by the Flow card "Send to Dashboard Studio" (`dashboard_send`, args `name`, `value`). `homey/insight/<name>` is sent by the Flow card "Send Insight": without the optional `topic` argument `<name>` is the full autocomplete name of the insight (e.g. `homey/insight/system - loadavg (CPU Load)`); with `topic` set (since v2.2.0) it is `homey/insight/<topic>`. Event names may be chosen freely, but verify in the probe call that they actually arrive (`missingLive`).
- Before testing or handing over, check programmatically that every `{{topic}}` in texts (including nested ones inside formulas) and every value in `bindings` (walk nested bindings such as graph `series[i].data`) is contained in the saved topic list (device topics) or is a known event topic. For variables, flows and moods accept the canonical and the legacy form. Report any mismatch instead of delivering.

A wrong topic shows an empty field without any error.

## Pitfalls

- A "comprehensive" dashboard crammed onto one page is unreadable: split it into one page per job and respect the per-page limits in `references/design.md`.
- Every browser call on the LAN editor costs the user an approval click – plan calls, bundle work, see "Browser budget".
- Topic names not taken verbatim from the Data Stream Explorer leave fields empty without any error.
- The editor starts with `dashboard.json`, else the demo in `simulation`; reading topics there yields only `sim/...`. If that `dashboard.json` is older than `LATEST_MIGRATION_VERSION`, a "Dashboard Upgraded" notice waits for OK and the editor stays offline with an empty topic list; the discovery call closes it (OK does not save).
- Never `await Dashboard.loadDashboardData()` without a timeout: a migration notice blocks the promise and the JS tool call fails after 45 s.
- Capabilities that are not enabled send no data, so always check `enabled_capabilities`.
- Over-long `{{…}}` expressions render `-` without an error.
- Graph `maxPoints` defaults to 20 and silently truncates histories; since v2.3.0 `showLegend` defaults to true and shows a legend even for a single series.
- Graph JSON in the pre-2.3.0 shape (`lineColor`, `bindings.data`, and also `autoMin`/`min`/`autoMax`/`max`/`linearTimeScale` as widget overrides) still loads through the migrator, but do not generate it: write `series` and `dynamicFloor`/`fixedFloor`/`dynamicCeiling`/`fixedCeiling`. Assert it in the build script.
- A graph legend (on by default since v2.3.0) is drawn outside the widget box in the fixed and scale layouts; use `floating` inside the chart or reserve space.
- The text widget ignores `fontSize`, `textColor` and `lineHeight`; use `textMdParaFs`, `textMdParaCol`, `textMdParaLineHeight` (and `textMdTdFs`/`textMdThFs` for tables). The static check in `references/design.md` warns about it.
- Comparing a running period with a complete one ("today so far" vs. "all of yesterday") misleads; compare up to the same time (`references/design.md`, rule 4).
- Overlaid graph widgets (since v2.3.0 only needed for a second Y scale) with different reserved axis space do not line up.
- Basic flows created via the bridge appear empty in the Homey web app on Advanced-Flow setups; use Advanced Flows.
- Claude in Chrome cannot screenshot a tab that is in the background, and may disconnect after a timeout.
- Some daily counters report whole numbers only (e.g. some inverters' `meter_power.daily`).
- Values combined from different devices are sampled at slightly different times, so small mismatches in sums are normal.
- Never overwrite `enabled_capabilities` without merging the existing entries, and confirm with the user before writing.
- Fraction capabilities (`dim`, `volume_set`, `windowcoverings_set`, `light_hue`, `light_saturation`, `light_temperature`) run from 0 to 1, not 0 to 100 – use `min: 0`, `max: 1` and format as %.
- Disabled widgets (since v2.0.13) block touch, click and keyboard input completely.
- Flow starts and moods have cooldowns (10 s / 3 s); taps within that time are silently ignored.
- Calendar views since v2.0.13 are relative to today: the loader range must cover every displayed day.
- Fixed in v2.1.0: formulas that reference the same topic several times evaluate reliably; earlier versions could show wrong values there. Since v2.0.15 interactive widgets no longer overwrite their bound input topic when clicked.
- State caching (since v2.2.0, app setting **State Caching**: *Full* or *Optimized (< 16 KB events)*): in Optimized mode large event payloads (Markdown tables, graph histories, SVG) are not cached, so those widgets stay empty after a reload until the data feed sends again. Before diagnosing an empty Text/Graph widget, ask which mode is set or let the feed resend; for heavy feeds prefer a trigger on `dashboard_loaded` or `dashboard_page_opened`.
- New in v2.2.0: the **Capability & Topic Cleanup** tool (app settings → Configuration) disables capabilities that no saved dashboard references. A dashboard only built as a draft or JSON file is not "saved", so its capabilities can be offered for removal – tell the user before they run it.
- Never save to Homey or overwrite an existing dashboard name without explicit confirmation; create scripts and flows on Homey only after confirmation.
