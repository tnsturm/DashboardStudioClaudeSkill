---
name: "dashboard-studio-json"
description: "Builds importable JSON dashboards for the Homey app Dashboard Studio (com.satoer.dashboardstudio) via the MCP AI Bridge. Only for this app – not for bridge or native Homey dashboards."
---

# Dashboard Studio – generating JSON dashboards

Applies ONLY to the Homey app **Dashboard Studio** by Satoer (app ID `com.satoer.dashboardstudio`). Do not use it for the MCP AI Bridge's own dashboard (`dashboard_generate`) or for native Homey dashboards (`homey_dashboards_*`).

Knowledge verified against Dashboard Studio v2.0.12 on a Homey Pro (2026), October 2026. If the installed version is much newer, re-check the file format against the public demo (see "Reference sources").

Tool names below are given without their MCP prefix (e.g. `apps_get` = `<prefix>homey-bridge__apps_get`, `javascript_tool` = `<prefix>Claude_Browser__javascript_tool`).

## Browser budget – the most important rule

On the user's LAN address the built-in browser asks the user for approval **for every single browser call** (load, JavaScript, screenshot, click). An entry in the user's allow list does not change that; even `request_access` with `scope: "site"` is answered with "ONE load or action". Every browser call therefore costs the user one click. Rules:

1. **Do everything that does not need the browser via the bridge first**: devices, capabilities, insights, flows, HomeyScript tests, data-feed tests (`homeyscript_run`), app settings. None of this asks the user.
2. **Do all research and the layout preparation in the cloud workspace** (Python build script, topic check against the saved topic list, gzip+base64 encoding) before the next browser call.
3. **One browser call = one complete job.** Never split "load", "wait", "check" and "read back" into separate calls. Never use `read_page`, `get_page_text`, `find` or `form_input` in the editor – do the same in the JavaScript call. Use `browser_batch` only when a screenshot must follow a JS step directly (each item still asks).
4. **Check rendering in JavaScript, not by eye** (overflowing text, `-` formula errors, missing live values, resolved widget config – see "Probe call"). A screenshot is only needed once at the end for the user's visual review, preferably `computer` → `zoom` on the canvas region at `scale` ≤ 0.6.
5. **Fold source-code research into the probe call** (`fetch('widgets/<type>.js')` and return snippets) instead of separate calls.
6. **Save in the same call that does the final check** once the user has confirmed name and saving (see "Commit call").

Target for a new dashboard: **4 browser calls** – 1 open editor, 1 discovery call (switch to Homey platform + read topics + optional source research), 1 probe call per layout iteration, 1 screenshot, 1 commit call. Tell the user up front roughly how many approvals to expect.

## Step 0 – Preconditions (always run first, bridge only)

1. **MCP AI Bridge reachable?** Check that Homey bridge tools are available in this session, then call `system_get_info`. If the tools are missing or the call fails, stop and tell the user: the Homey app **MCP AI Bridge** (`community.mcp-ai-bridge`) must be installed, running and connected to Claude.
2. **Bridge app running?** `apps_get` with `community.mcp-ai-bridge` → `state` must be `running` and `enabled: true`. Report the version.
3. **Dashboard Studio installed?** `apps_get` with `com.satoer.dashboardstudio` → `state: running`. Note `version`. If it is not installed, tell the user to install it from the Homey App Store and stop.
4. **Find the editor URL.** First `apps_get_settings` on `com.satoer.dashboardstudio` and look for any URL, host or port key. In v2.0.12 settings only hold `enabled_capabilities`, so build it from `system_get_info`: `ethernetAddress` (or `wifiAddress` when ethernet is not connected) plus port **5050** (verified default). Editor `http://<ip>:5050/editor.html`, viewer `http://<ip>:5050/viewer.html?dashboard=<name>`.
5. **Browser available?** The editor runs on a private LAN address, so it is reachable only from a browser on the user's network. Use the **built-in Claude browser** (`Claude_Browser__*` tools; load them all with one ToolSearch on the full prefix, and read the `built-in-browser` skill first). Claude in Chrome also works when it is connected, but it has proven unstable (background tabs block screenshots, the extension disconnects). If neither browser is reachable, fall back to handing over the JSON for manual import.

## Workflow

1. **Collect devices and values** via the bridge (`devices_list`, `devices_get`, `devices_get_capability`, `insights_list`, `insights_get_entries`): available capabilities, units, sign conventions (e.g. grid meters often report export as negative `measure_power`), sub-capabilities such as `measure_power.pv1input`, value ranges.
2. **Enable capabilities in Dashboard Studio.** Without this the app sends no data for them. They are stored in the app setting `enabled_capabilities` = `{ "<deviceId>": ["cap", "cap.sub"] }`. Read the current value with `apps_get_settings`, MERGE (never drop existing entries), show the user what will be added, and write with `apps_set_setting` only after the user confirms. The app picks up changes immediately.
   For values that are not device capabilities (system data, insight histories, computed values) use a data feed instead – see "Feeding non-device data".
3. **Read the exact topic names from the editor – MANDATORY** in ONE discovery call (see "Discovery call"), or ask the user to paste the Data Stream Explorer list. Do NOT generate the JSON before this list is available. Never derive, guess or construct device topic names (not from device IDs, not from device names). Every device topic used must appear verbatim in that list. Save the list to a file in the workspace for the topic check.
4. **Generate the JSON** (schema below) with a script, and run the topic check ("Topic names").
5. **Test in the user's editor** with ONE probe call per iteration, fix the build script, repeat. Then one zoomed screenshot.
6. **Hand over for approval.** Show the screenshot and ask whether and under which name to save. Never click *Save to Homey* without explicit confirmation in this session. Also deliver the JSON file.
7. **Save** only after confirmation with ONE commit call, which also verifies.

## Opening the editor

- `request_access {url: "http://<ip>:5050", scope: "once"}`, then `preview_start` with the editor URL (or `navigate` in the tab the user opened; check `tabs_context` once).
- **Half-loaded page:** if the approval only covered the document, sub-resources may be missing; the page then shows unstyled buttons and `typeof Dashboard === 'undefined'`. Detect this inside the next JS call (return early with that information) and reload once with `navigate`.
- **Start behaviour (verified in `dashboard.js`):** on start the editor loads `dashboards/dashboard.json`; if that file does not exist it falls back to `demo.json` (platform "simulation"). The editor strips all URL parameters except `token` and `test_mode`, so there is no URL parameter to open a specific dashboard in the editor; `?dashboard=<Name>` only works in `viewer.html`. A dashboard saved under the name **`dashboard`** is therefore the editor's start page (and the viewer's default without parameter).

## Discovery call (one browser call)

Switches the editor to the Homey platform, waits, and returns the topic list – plus any source code you need for unknown widget fields:

```js
if (typeof Dashboard === 'undefined') { 'NOT_LOADED' } else {
const minimal = {version:"2.0.12", master:{connection:{dataDriver:"homey",driverSettings:{}},
  environment:{canvas:{width:1280,height:800},theme:"default_dark",layout:{mode:"scale",scale:{keepAspectRatio:true}}}},
  globalBindings:{}, widgets:{}, groups:{}, selectionGroups:{}};
if (!Dashboard._isHomeyPlatform()) { await Dashboard.loadDashboardData(minimal); }
await new Promise(r => setTimeout(r, 6000));
const src = {};
for (const f of [/* e.g. 'widgets/graph.js' */]) {
  const t = await (await fetch(f)).text();
  src[f] = [...new Set([...t.matchAll(/\bc\.(\w+)/g)].map(m => m[1]))];   // config keys
}
JSON.stringify({isHomey: Dashboard._isHomeyPlatform(), topics: Object.keys(Dashboard.liveData).sort(), src});
}
```

## Probe call (one browser call per layout iteration)

Loads the draft (gzip + base64), waits, and checks everything that can be checked without looking:

```js
const bin = Uint8Array.from(atob(B64), c => c.charCodeAt(0));
const data = JSON.parse(await new Response(new Blob([bin]).stream().pipeThrough(new DecompressionStream('gzip'))).text());
await Dashboard.loadDashboardData(data);
await new Promise(r => setTimeout(r, 5000));
const ld = Dashboard.liveData || {};
const used = new Set();
for (const w of Object.values(data.widgets)) {
  Object.values(w.bindings || {}).forEach(t => used.add(t));
  for (const m of String(w.overrides.text || '').matchAll(/\{\{(homey\/[^{}]+?|[a-z0-9-]+\/[^{}\s]+?)\}\}/g)) used.add(m[1]);
}
const missingLive = [...used].filter(t => !t.startsWith('local/') && !(t in ld));
// rendering problems: overflowing text widgets and '-' (failed formula)
const problems = [];
for (const [id, w] of Object.entries(Dashboard.widgets)) {
  const el = w.el || w.element || w.container || document.querySelector(`[data-widget-id="${id}"]`);
  if (!el) continue;
  if (el.scrollHeight > el.clientHeight + 2 || el.scrollWidth > el.clientWidth + 2) problems.push(id + ': overflow');
  if (/(^|\s)-(\s|$)/.test(el.innerText || '') && data.widgets[id]?.type === 'text') problems.push(id + ': "-" (formula?)');
}
const cfg = id => { const w = Dashboard.widgets[id]; const c = w && (w.config || w.c); return c ? {min: c.min, max: c.max, type: c.type} : null; };
JSON.stringify({isHomey: Dashboard._isHomeyPlatform(), widgets: Object.keys(Dashboard.widgets).length,
  missingLive, problems /*, resolved: cfg('<widgetId>') */});
```

If the element lookup returns nothing for a widget type, inspect `Object.keys(Dashboard.widgets['<id>'])` once inside the same call and adapt. Only when `missingLive` and `problems` are empty, take the single zoomed screenshot.

## Commit call (one browser call, only after explicit confirmation)

Saves the currently loaded dashboard under `NAME`, handles the overwrite and success dialogs, and verifies. Optionally load the final B64 first in the same call so nothing has to be repeated.

```js
const vis = el => el && el.offsetParent !== null, sleep = ms => new Promise(r => setTimeout(r, ms));
const NAME = '<Name>';            // case-sensitive file name without .json
const ALLOW_OVERWRITE = false;    // true only if the user explicitly asked to replace this name
const existing = (await fetch('dashboards/' + NAME + '.json', {cache: 'no-store'})).ok;
if (existing && !ALLOW_OVERWRITE) { 'EXISTS – ask the user first' } else {
  [...document.querySelectorAll('button')].find(b => vis(b) && /Save to Homey/i.test(b.textContent)).click();
  await sleep(1500);
  const modal = document.querySelector('#file-modal');
  const inp = document.querySelector('#save-filename');
  inp.value = NAME; inp.dispatchEvent(new Event('input', {bubbles: true}));
  await sleep(300);
  [...modal.querySelectorAll('button')].find(b => vis(b) && b.textContent.trim() === 'Save').click();
  await sleep(1500);
  const clicked = [];
  for (let i = 0; i < 3; i++) {     // overwrite confirmation (if any) and "Saved successfully!" → OK
    const b = [...document.querySelectorAll('button')].find(b => vis(b) && /^(Overwrite|Replace|Yes|Confirm|OK)$/i.test(b.textContent.trim()));
    if (!b) break; clicked.push(b.textContent.trim()); b.click(); await sleep(1500);
  }
  try { ServerFileManager.close(); } catch (e) {}
  const r = await fetch('dashboards/' + NAME + '.json', {cache: 'no-store'});
  JSON.stringify({clicked, status: r.status, widgets: r.ok ? Object.keys((await r.json()).widgets || {}).length : null});
}
```

Verified details: the save dialog is `#file-modal` (class `show`), the input is `#save-filename`, the primary button text is exactly `Save` (class `ds-dialog-button primary`). After saving, a dialog "Success – Saved successfully!" with an `OK` button appears and must be closed. The file list in the dialog (`.ds-file-list-name`) is not proof of success; only `fetch('dashboards/<Name>.json')` = 200 is.

**File names:** use a clear name without spaces, e.g. `Title_yyyymmdd_V#`, and save changes as a new version (`_V2`, `_V3`, …) instead of overwriting, unless the user asks otherwise. Exception: the start page must be called exactly `dashboard`.

## Topic names (critical)

- The only valid source for device topic names is the Data Stream Explorer / live topic list of the user's editor. Copy names verbatim (case, dots and hyphens included).
- Background only, not for building names: topics have the form `<slug of the device name>/<capability>`. The slug is lower-case, umlauts and other special characters become `-`, and trailing spaces in the device name produce a trailing `-` (e.g. device "Wetter Poolhütte " → `wetter-poolh-tte-/measure_co2`). Renaming a device can change its topic, so after a rename read the list again.
- If a needed value is missing from the list, the capability is not enabled yet: go back to workflow step 2.
- Other topic kinds: `local/<name>` lives only inside the browser and is useful for UI wiring between widgets. `homey/event/<name>` is sent by the Flow card "Send to Dashboard Studio" (`dashboard_send`, args `name`, `value`). `homey/insight/<name>` is sent by the Flow card "Send Insight", where `<name>` is the full autocomplete name of the insight (e.g. `homey/insight/system - loadavg (CPU Load)`). Event names may be chosen freely, but verify in the probe call that they actually arrive (`missingLive`).
- Before testing or handing over, check programmatically that every `{{topic}}` in texts (including nested ones inside formulas) and every value in `bindings` is contained in the saved topic list (device topics) or is a known event topic. Report any mismatch instead of delivering.

A wrong topic shows an empty field without any error.

## Feeding non-device data (system values, insight histories)

- **"Send Insight" flow card limitation:** all time ranges of the same insight end up on the SAME topic (`homey/insight/<name>`), so a 24 h and a 7 day graph of one insight cannot coexist. The card "Get Insight Data (Advanced)" only returns tokens (`insight_raw`, `insight_highest`, `insight_lowest`, `insight_average`).
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

## File schema

```json
{
  "version": "<installed Dashboard Studio version, e.g. 2.0.12>",
  "master": {
    "connection": {"dataDriver": "homey", "driverSettings": {}},
    "environment": {
      "editorMode": "advanced", "editorZoom": 100, "widgetLabelMode": "hover",
      "canvas": {"width": 1280, "height": 870, "preset": "",
        "background": {"startColor": "@background", "startOpacity": 100, "endColor": "@backgroundAccent", "endOpacity": 100, "type": "radial", "angle": 0, "overlay": false, "image": "", "size": "cover"}},
      "grid": {"size": 10, "show": false, "snap": true},
      "paging": {"currentPage": 1},
      "theme": "default_dark",
      "layout": {"mode": "scale", "scale": {"keepAspectRatio": true}}
    }
  },
  "globalBindings": {},
  "widgets": { "<id>": {"type": "…", "overrides": {…}, "bindings": {…}} },
  "groups": {},
  "selectionGroups": {}
}
```

- A minimal `master` is enough. Missing parts (`typography`, `container`, `theme.colors`, `widgetDefaults`) are filled from defaults.
- `dataDriver` is one of `homey`, `mqtt`, `nodered` or `simulation`. Use `homey` for the user's own Homey; it also switches the editor's platform on load.
- `layout.mode`: `fixed` uses exact pixels, `scale` scales to the window using the same coordinates, `responsive` uses columns.
- Themes include `default_dark`, `eink_dark`, `eink_light`, `midnight_blue`, `deep_space`, `executive_suite` and `paper_white` (35+ built in).
- **Widget object:** `type`, `overrides` (position and all properties) and `bindings` (field → topic). Required in `overrides`: `x`, `y`, `width`, `height`, `page` (a number or a string like `"1, 2, 7-12"`). Optional: `metaName` (name shown in the editor), `zIndex`, `visible`, `locked`, and `container: {"show": false}` to hide the card frame.
- Unknown field names: read them from the widget source (`widgets/<type>.js`, keys used as `c.<field>`, defaults in `static get globalSettings`) inside the discovery call, never by trial and error with extra browser calls.

## Widget types and key fields

| type | purpose | fields (overrides) | bindable (bindings) |
|---|---|---|---|
| `text` | Markdown with live values | `text`, `textAlign`, `padding`, `mdTableStyle` (`minimal`, `solid_header`), `contentAnimation`, `mdAutoFitText` (true = scale text to fill the widget) | `text`, `visible` |
| `empty` | container or card with a caption | `label` | `visible` |
| `icon` | Phosphor icon that glows when active | `label`, `size`, `active`, `activeIconMirrorsInactive`, `blink`, `icons:{off:{source:"phosphor",builtin:"ph-sun",weight:"fill",color:"@icon"}, on:{…,color:"@warning"}}` | `active`, `label` |
| `arrow` | line or arrow with flow animation | `fromX`, `fromY`, `toX`, `toY` (absolute canvas coordinates), `customPoints:[{x,y}]` for bends, `pathBendRadius`, `thickness`, `headSize` (0 = no head), `lineRadius`, `headRound`, `mainColor`, `animationEnabled`, `animationSpeed`, `animationReverse`, `flowWidth`, `gradientWidth`, `enableGlow` | `animationEnabled`, `animationReverse`, `thickness`, `visible`, colors |
| `gauge` | round or horizontal gauge | `label`, `unit`, `min`, `max`, `decimals`, `showIcon` (default true, shows a thermometer!), `gaugeCol0`, `gaugeCol100`, `barThickness`, `trackThickness`, `gapRound`, `gaugeSizeOffset` | `value` |
| `battery` | battery bar (no numeric % shown) | `label`, `vertical` | `value` |
| `graph` | line, bar or big-number chart, ONE series | see "Graph widget" | `data`, plus numeric fields such as `min`, `max` |
| `switch` | toggle or push button, also navigation | `buttonLabel`, `mode` (`toggle`/`push`), `style` (`switch`/`button`), `actionType` (`topic`/`page`/`url`), `urlTo`, `urlNewTab`, `outputTopic`, `onPayload`, `offPayload`, `showIcon`, `iconSize`, `icons` | `state` |
| `slider` | dimmer or setpoint | `min`, `max`, `vertical` | `value` |
| `list` | log or feed | – | `data` |
| `image` | URL or SVG image | `url`, `fit`, `liveRefresh`, `refreshInterval` | `svgRaw`, `url` |
| `clock` | analog, flip or text clock | `displayMode` | – |
| `menu` | navigation menu; items with `actionType` `topic`, `page` or `url` (`urlTo`) | – | see the editor |
| `lightgrid`, `editbox` | light tiles, text input | – | see the editor |

**Arrow bounding box:** `x`, `y`, `width` and `height` enclose all points with about 40 px padding. The arrowhead sits at the `to` point. When `animationReverse` is true, the flow runs backwards and the head jumps to the other end. Default colors: flow `@success` (green), reverse flow `@danger` (red). In view mode, arrows let clicks pass through.

**Text size:** `#` headings render small on a 1280 px canvas. For one prominent value use `mdAutoFitText: true`. Do not set it on all tiles: each tile then scales differently and the layout looks uneven. A Markdown table that may grow (e.g. a top-N list) gets its own text widget with `mdAutoFitText: true`; headings and footers go into separate widgets so they are never pushed out.

**Gauge:** set `showIcon: false` unless an icon is wanted; with the default layout the caption overlaps the arc – `gapRound: 6` and `gaugeSizeOffset: 22` (200×140 widget) fix that.

**Navigation buttons (verified):** a `switch` with `mode: "push"`, `style: "button"`, `actionType: "url"`, `urlTo: "viewer.html?dashboard=<Name>"`, `urlNewTab: false`, `buttonLabel`, `showIcon: true`, `icons: {on:{source:"phosphor",builtin:"ph-cpu",weight:"fill",color:"#38bdf8"}, off:{…}}`. Ideal for a start page saved as `dashboard`.

## Graph widget

- **Data:** a JSON string or array of `{t: ISO-timestamp, v: number}` (sorted by `t`), a comma list of numbers, or an object `{label: value}`. Insight entries fit directly.
- **Defaults that bite:** `maxPoints` defaults to **20** (set e.g. 400), `showAxisX`/`showAxisY` default false, `label` shows a caption "GRAPH" unless set to `""`.
- Key fields: `type` (`line`, `smooth`, `bar`, `text`), `lineColor`, `fillCol`, `fill`, `fillOpacity`, `lineWidth`, `autoMin`/`min`, `autoMax`/`max`, `headroom`, `decimals`, `showAxisX`, `showAxisY`, `axisWidthX`, `axisWidthY` (fixed reserved space), `axisResY` (number of Y labels), `axisXFontColor`, `axisYFontColor`, `timeFormat` (`HH:mm`, `Date`, `DateHours`, `HH`, `Datapoints`, `TextLabels`), `dateFormat` (`DD-MM`, …), `time24h`, `timezoneConvert`, `gridHInnerShow`/`Col`/`Op`, `gridHBotShow`, `padding`, `container`.
- **X labels:** use `linearTimeScale: false`, `autoAxisResX: false`, `axisResX: <every n-th point>` (e.g. 24 for 144 points per day or 168 hourly points per week). `linearTimeScale: true` together with `axisResX` produced unreadable, overlapping labels.
- `smooth` can overshoot below `min`; use `line` for overlays with a fixed scale.
- **Combining two measures in one chart (verified):** a graph shows one series, so place two graph widgets with **identical x/y/width/height and identical reserved axes** on top of each other (second one with higher `zIndex`, `fill: false`). Keep `showAxisX: true` and the same `axisWidthX` on BOTH, and make the top one's labels transparent (`axisXFontColor`/`axisYFontColor: "rgba(0,0,0,0)"`, grid off); otherwise the top graph's plot area is taller and its line runs below the bottom edge.
- **Second scale on the right (verified):** give the top graph a fixed scale bound to precomputed min/max events: `autoMin: false`, `autoMax: false`, `headroom: 0`, `bindings: {min: "homey/event/s_x_min", max: "homey/event/s_x_max"}`, `min_bind_formula: "round(value - 0.5)"`, `max_bind_formula: "round(value + 0.5)"`. Narrow both graphs by ~46 px and put 4 small text widgets in that strip, one per grid line, showing `{{round((LO + k*(HI-LO)/3)*10)/10}}°` with the same LO/HI expressions. Plot area: top ≈ `y + 5`, bottom = `y + height - axisWidthX`; with `axisResY: 4` the grid lines are at 0, 1/3, 2/3 and 1 of that span.

## Live values in text (Markdown)

- Insert a value: `{{topic}}`, e.g. `{{my-inverter/measure_power}} W`
- Formula: `{{round({{topic}}, 0)}}`; for one decimal: `{{round({{topic}} * 10, 0) / 10}}`
- Floor/ceil (no such functions): `round(x - 0.5)` / `round(x + 0.5)`.
- Condition: `{{if({{topic}} < 0, "[color @success]Export[/color]", "[color @danger]Import[/color]")}}`
- Value inside a condition: `{{if({{t}} >= 1000, "[color @danger]"+round({{t}}, 0)+" ppm[/color]", "[color @success]"+round({{t}}, 0)+" ppm[/color]")}}` (verified)
- Allowed functions: `round min max abs if number and or not xor nor contains containsanycase`. Comparisons: `< <= > >= ==`. `+` also concatenates text.
- Markup: `[color @danger]…[/color]`, `[blink]…[/blink]`, icons `:ph-sun:` or `:ph-flame|32|@danger:` (name, size, color), tables, `#` headings.
- **Length limit:** a single `{{…}}` expression of roughly 500+ characters fails silently and renders `-`. Long topic names repeated many times hit this fast. Split long logic into several short, independent expressions placed next to each other.
- Verified pattern, compass direction from an angle (one short expression per sector, only one prints):
  `{{if(A<22.5,"N",if(A>=337.5,"N",""))}}{{if(A>=22.5,if(A<67.5,"NO",""),"")}}…{{if(A>=292.5,if(A<337.5,"NW",""),"")}}` with `A` = `{{topic}}`. A single 8-level nested `if` with `or()` failed.

## Binding options and field formulas

These are stored in `overrides` next to the field, using the pattern `<field>_bind_<option>`:

- **Boolean field from a number** (e.g. `animationEnabled`, `active`, `visible`): `"animationEnabled_bind_transformMode": "numericFormula"` and `"animationEnabled_bind_formula": "value > 20"`. A result other than 0 means true.
- **Number field** (e.g. `thickness`, graph `min`/`max`): `"thickness_bind_formula": "min(max(abs(value) / 300, 3), 18)"`, with no transformMode.
- **Map text to a boolean:** `"visible_bind_mapTrue": "on"`, `"visible_bind_mapFalse": "off"`.
- In formulas, `value` (or `x`) is the bound payload. Other topics are referenced as `{{topic}}`. Maximum length is 512 characters.

Proven patterns for energy flows:
- Animate a line only when power flows: `bindings.animationEnabled = topic` with formula `value > 20`.
- Line width scaled to power: `bindings.thickness = topic` with formula `min(max(value / 300, 3), 18)`.
- Grid direction: draw the line from the house to the grid, so export runs forward. Then `bindings.animationReverse = <grid meter>/measure_power` with formula `value > 0`, so import runs backwards. Check the meter's sign convention first, and use `abs(value)` for the width.
- Glowing icon: `bindings.active = topic`, `active_bind_transformMode = numericFormula`, formula `value > 20` (or `value > 0`, e.g. rain).

## Colors

Theme tokens: `@background @backgroundAccent @surface @surfaceAlt @text @textMuted @accent @accentSoft @accentSecondary @icon @iconMuted @success @warning @danger @glow @border @borderAccent @onAccent`. Hex, RGB, HSL and CSS color names also work. For overlays use explicit hex colors per measure and repeat them in the legend text (`[color #38bdf8]━ Last %[/color]`).

## Fallback: public demo editor

If the user's editor cannot be reached, test layout and formulas in `https://satoer.com/dashboardstudio/demo/editor.html` with the same loading code, but set `data.master.connection = {dataDriver: 'simulation', driverSettings: {interval: 1000}}` and feed sample values with `Dashboard.applyLocalTopicPayload(topic, value)`. Topic names must still come from the user's list; the demo cannot verify them. The public demo is not a LAN address, so it does not ask per action – use it for layout iterations that need many calls, and the user's editor only for the final probe and the commit.

## Reference sources

- Demo dashboard with many widget examples: `https://satoer.com/dashboardstudio/demo/dashboards/demo.json` (fetch it from inside the demo tab).
- Help content as JSON: `https://satoer.com/dashboardstudio/demo/help/help-content.json`.
- Saved dashboards on the user's Homey: `http://<ip>:5050/dashboards/<Name>.json`.
- Widget source code on the user's Homey: `http://<ip>:5050/widgets/<type>.js`, startup and save logic in `dashboard.js`, driver in `drivers/homey.js`.
- Documentation pages on satoer.com block automated fetchers (robots.txt). Read them only if the user opens them in their own browser view, and do not work around the block.

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
- Never save to Homey or overwrite an existing dashboard name without explicit confirmation; create scripts and flows on Homey only after confirmation.
