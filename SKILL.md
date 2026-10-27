---
name: "dashboard-studio-json"
description: "Builds, changes and repairs importable JSON dashboards for the Homey app Dashboard Studio (com.satoer.dashboardstudio) via the MCP AI Bridge. Use when the user wants a Dashboard Studio dashboard, widget, data feed or layout fix. Only for this app – not for the bridge's own dashboard or native Homey dashboards."
---

# Dashboard Studio – generating JSON dashboards

**Skill version 2.1.0** – matches the Dashboard Studio version it was verified against (`com.satoer.dashboardstudio` 2.1.0).
**Requires MCP AI Bridge** (`community.mcp-ai-bridge`) – tested with **5.2.1**. If the installed Dashboard Studio or bridge version differs, mention it to the user; for a different bridge version check the bridge-dependent steps (tool names, `homeyscript_run` with `dry_run`, `apps_get_settings`).

Applies ONLY to the Homey app **Dashboard Studio** by Satoer (app ID `com.satoer.dashboardstudio`). Do not use it for the MCP AI Bridge's own dashboard (`dashboard_generate`) or for native Homey dashboards (`homey_dashboards_*`).

Knowledge verified against Dashboard Studio **v2.1.0** (test channel, released 2026-10-19) on a Homey Pro (2026), checked 2026-10-21. If the installed version is newer, read the app's own release notes first (`release-changelog.json`, see "Source research without the browser" in `references/source-research.md`) and re-check anything they touch.

Tool names below are given without their MCP prefix (e.g. `apps_get` = `<prefix>homey-bridge__apps_get`, `javascript_tool` = `<prefix>Claude_Browser__javascript_tool`).

## Reference files (read only what the task needs)

| file | read when |
|---|---|
| `references/browser-calls.md` | before the first browser call (opening the editor, discovery, probe, commit, demo fallback) |
| `references/widgets.md` | before generating or editing the dashboard JSON (schema, widget fields, graphs, text formulas, bindings, colors) |
| `references/data-feeds.md` | a value is not a device capability, or flow cards / calendars are involved |
| `references/source-research.md` | after an app update, for unknown widget fields, the Auto-Widget Gallery, or variables/flows/moods |

## Browser budget – the most important rule

On the user's LAN address the built-in browser asks the user for approval **for every single browser call** (load, JavaScript, screenshot, click). An entry in the user's allow list does not change that; even `request_access` with `scope: "site"` is answered with "ONE load or action". Every browser call therefore costs the user one click. Rules:

1. **Do everything that does not need the browser via the bridge first**: devices, capabilities, insights, flows, HomeyScript tests, data-feed tests (`homeyscript_run`), app settings. None of this asks the user.
2. **Do all research and the layout preparation in the cloud workspace** (Python build script, topic check against the saved topic list, gzip+base64 encoding) before the next browser call.
3. **One browser call = one complete job.** Never split "load", "wait", "check" and "read back" into separate calls. Never use `read_page`, `get_page_text`, `find` or `form_input` in the editor – do the same in the JavaScript call. Use `browser_batch` only when a screenshot must follow a JS step directly (each item still asks).
4. **Check rendering in JavaScript, not by eye** (overflowing text, `-` formula errors, missing live values, resolved widget config – see "Probe call" in `references/browser-calls.md`). A screenshot is only needed once at the end for the user's visual review, preferably `computer` → `zoom` on the canvas region at `scale` ≤ 0.6.
5. **Do source-code research without the browser** via HomeyScript (see "Source research without the browser" in `references/source-research.md`). Only if that is impossible, fold it into the discovery or probe call.
6. **Save in the same call that does the final check** once the user has confirmed name and saving (see "Commit call" in `references/browser-calls.md`).

Target for a new dashboard: **4 browser calls** – 1 open editor, 1 discovery call (switch to Homey platform + read topics + optional source research), 1 probe call per layout iteration, 1 screenshot, 1 commit call. Tell the user up front roughly how many approvals to expect.

## Step 0 – Preconditions (always run first, bridge only)

1. **MCP AI Bridge reachable?** Check that Homey bridge tools are available in this session, then call `system_get_info`. If the tools are missing or the call fails, stop and tell the user: the Homey app **MCP AI Bridge** (`community.mcp-ai-bridge`) must be installed, running and connected to Claude.
2. **Bridge app running?** `apps_get` with `community.mcp-ai-bridge` → `state` must be `running` and `enabled: true`. Report the version.
3. **Dashboard Studio installed?** `apps_get` with `com.satoer.dashboardstudio` → `state: running`. Note `version`. If it is not installed, tell the user to install it from the Homey App Store and stop.
4. **Find the editor URL.** First `apps_get_settings` on `com.satoer.dashboardstudio` and look for any URL, host or port key. Up to v2.1.0 the readable settings hold only `enabled_capabilities` (a Personal API Key, if set, appears as `<redacted>`), so build it from `system_get_info`: `ethernetAddress` (or `wifiAddress` when ethernet is not connected) plus port **5050** (verified default). Editor `http://<ip>:5050/editor.html`, viewer `http://<ip>:5050/viewer.html?dashboard=<name>`.
5. **Browser available?** The editor runs on a private LAN address, so it is reachable only from a browser on the user's network. Use the **built-in Claude browser** (`Claude_Browser__*` tools; load them all with one ToolSearch on the full prefix, and read the `built-in-browser` skill first). Claude in Chrome also works when it is connected, but it has proven unstable (background tabs block screenshots, the extension disconnects). If neither browser is reachable, fall back to handing over the JSON for manual import.

## Workflow

1. **Collect devices and values** via the bridge (`devices_list`, `devices_get`, `devices_get_capability`, `insights_list`, `insights_get_entries`): available capabilities, units, sign conventions (e.g. grid meters often report export as negative `measure_power`), sub-capabilities such as `measure_power.pv1input`, value ranges.
2. **Enable capabilities in Dashboard Studio.** Without this the app sends no data for them. They are stored in the app setting `enabled_capabilities` = `{ "<deviceId>": ["cap", "cap.sub"] }`. Read the current value with `apps_get_settings`, MERGE (never drop existing entries), show the user what will be added, and write with `apps_set_setting` only after the user confirms. The app picks up changes immediately.
   For values that are not device capabilities (system data, insight histories, computed values) use a data feed instead – see "Feeding non-device data" in `references/data-feeds.md`.
3. **Read the exact topic names from the editor – MANDATORY** in ONE discovery call (see "Discovery call" in `references/browser-calls.md`), or ask the user to paste the Data Stream Explorer list. Do NOT generate the JSON before this list is available. Never derive, guess or construct device topic names (not from device IDs, not from device names). Every device topic used must appear verbatim in that list. Save the list to a file in the workspace for the topic check.
4. **Generate the JSON** (schema in `references/widgets.md`) with a script, and run the topic check ("Topic names").
5. **Test in the user's editor** with ONE probe call per iteration, fix the build script, repeat. Then one zoomed screenshot.
6. **Hand over for approval.** Show the screenshot and ask whether and under which name to save. Never click *Save to Homey* without explicit confirmation in this session. Also deliver the JSON file.
7. **Save** only after confirmation with ONE commit call, which also verifies.

## Topic names (critical)

- The only valid source for device topic names is the Data Stream Explorer / live topic list of the user's editor. Copy names verbatim (case, dots and hyphens included).
- Background only, not for building names: topics have the form `<slug of the device name>/<capability>`. The slug is lower-case, umlauts and other special characters become `-`, and trailing spaces in the device name produce a trailing `-` (e.g. device "Wetter Poolhütte " → `wetter-poolh-tte-/measure_co2`). Renaming a device can change its topic, so after a rename read the list again.
- If a needed value is missing from the list, the capability is not enabled yet: go back to workflow step 2.
- Other topic kinds: `local/<name>` lives only inside the browser and is useful for UI wiring between widgets. `homey/event/<name>` is sent by the Flow card "Send to Dashboard Studio" (`dashboard_send`, args `name`, `value`). `homey/insight/<name>` is sent by the Flow card "Send Insight", where `<name>` is the full autocomplete name of the insight (e.g. `homey/insight/system - loadavg (CPU Load)`). Event names may be chosen freely, but verify in the probe call that they actually arrive (`missingLive`).
- Before testing or handing over, check programmatically that every `{{topic}}` in texts (including nested ones inside formulas) and every value in `bindings` is contained in the saved topic list (device topics) or is a known event topic. Report any mismatch instead of delivering.

A wrong topic shows an empty field without any error.

## Pitfalls

- Every browser call on the LAN editor costs the user an approval click – plan calls, bundle work, see "Browser budget".
- Topic names not taken verbatim from the Data Stream Explorer leave fields empty without any error.
- The editor starts with `dashboard.json`, else the demo in `simulation`; reading topics there yields only `sim/...`.
- Capabilities that are not enabled send no data, so always check `enabled_capabilities`.
- Over-long `{{…}}` expressions render `-` without an error.
- Graph `maxPoints` defaults to 20 and silently truncates histories.
- Overlaid graphs with different reserved axis space do not line up.
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
- Never save to Homey or overwrite an existing dashboard name without explicit confirmation; create scripts and flows on Homey only after confirmation.
