# Dashboard Studio – Browser calls

The JavaScript jobs for the user's editor: opening it, discovery, probe and commit, plus the public demo fallback. Read before the first browser call.

## Opening the editor

- `request_access {url: "http://<ip>:5050", scope: "once"}`, then `preview_start` with the editor URL (or `navigate` in the tab the user opened; check `tabs_context` once).
- **Half-loaded page (verified 2026-11-03):** if the approval only covered the document, sub-resources may be missing; the page then shows unstyled buttons, the header says `OFFLINE` and `typeof Dashboard === 'undefined'`. In `performance.getEntriesByType('resource')` every script and stylesheet (even `version.js`) is listed with 0 bytes. Detect this inside the next JS call (wait up to ~20 s for `Dashboard`, then return early with that information) and reload once with `navigate`; that fixed it.
- **"Dashboard Upgraded" dialog blocks the editor (verified v2.3.1):** `loadDashboardData` awaits `CustomDialog.alert(…, "Dashboard Upgraded")` whenever `DashboardMigrator.migrate()` changes the file. This happens (a) at editor start when `dashboards/dashboard.json` is older than `LATEST_MIGRATION_VERSION`, and (b) when a generated draft still contains a legacy key (for graphs: `autoMin`, `min`, `autoMax`, `max`, `linearTimeScale`, `bindings.data`, … – see `widgets.md`), even if its `version` is current. Until someone clicks **OK**, the data driver stays offline (`Dashboard._dataDriverConnected === false`, `liveData` empty) and the `loadDashboardData` promise never resolves, so a JS call that awaits it times out after 45 s. Clicking OK only closes the notice; it does not save. Close it in the discovery call (below), wrap every `loadDashboardData` in a timeout, and keep generated JSON free of legacy keys. To test a saved file without loading it: `DashboardMigrator.migrate(JSON.parse(JSON.stringify(raw))).migrated` (load the migrator first with `WidgetScriptLoader.ensureMigratorScript()`); files saved by v2.3.1 return `false`.
- **Start behaviour (verified in `dashboard.js`):** on start the editor loads `dashboards/dashboard.json`; if that file does not exist it falls back to `demo.json` (platform "simulation"). The editor strips all URL parameters except `token` and `test_mode`, so there is no URL parameter to open a specific dashboard in the editor; `?dashboard=<Name>` only works in `viewer.html`. A dashboard saved under the name **`dashboard`** is therefore the editor's start page (and the viewer's default without parameter).

## Discovery call (one browser call)

Closes a pending "Dashboard Upgraded" notice, switches the editor to the Homey platform, waits for the data driver, and returns the topic list plus the topic metadata (access flags, flows, moods, variables). Run a data feed once via `homeyscript_run` beforehand, so its `homey/event/…` topics already appear in the list.

```js
(async () => {
const sleep = ms => new Promise(r => setTimeout(r, ms)), vis = el => el && el.offsetParent !== null;
for (let i = 0; i < 40 && typeof Dashboard === 'undefined'; i++) await sleep(500);
if (typeof Dashboard === 'undefined') return 'NOT_LOADED';           // half-loaded page → navigate once
const closeUpgrade = () => {                                         // OK only closes the notice, nothing is saved
  const h = [...document.querySelectorAll('.ds-dialog-header')].find(e => vis(e) && /Dashboard Upgraded/.test(e.innerText));
  const b = h && [...document.querySelectorAll('.ds-dialog-footer button')].find(b => vis(b) && b.textContent.trim() === 'OK');
  if (b) { b.click(); return true; } return false; };
const upgradeClosed = closeUpgrade();
const minimal = {version: window.DASHBOARD_VERSION || "2.3.1", master:{connection:{dataDriver:"homey",driverSettings:{}},
  environment:{canvas:{width:1280,height:800},theme:"default_dark",layout:{mode:"scale",scale:{keepAspectRatio:true}}}},
  globalBindings:{}, widgets:{}, groups:{}, selectionGroups:{}};
if (!Dashboard._isHomeyPlatform()) await Promise.race([Dashboard.loadDashboardData(minimal), sleep(12000)]);
for (let i = 0; i < 30 && !Dashboard._dataDriverConnected; i++) await sleep(500);
await sleep(6000);
const tm = Dashboard.topicMeta || {};
const special = Object.fromEntries(Object.entries(tm)
  .filter(([t, m]) => m && (m.isFlow || m.isMood || m.isVariable))
  .map(([t, m]) => [t, {flow: !!m.isFlow, mood: !!m.isMood, variable: !!m.isVariable,
                        cap: m.capabilityId, title: m.title, set: !!m.setable, get: !!m.getable}]));
return JSON.stringify({version: window.DASHBOARD_VERSION, upgradeClosed, connected: Dashboard._dataDriverConnected,
  isHomey: Dashboard._isHomeyPlatform(), topics: Object.keys(Dashboard.liveData).sort(), special});
})()
```

`connected: false` with an empty topic list means the driver did not start; look for a dialog (as above) before suspecting the bridge or the network. The list contains event topics twice, as `homey/event/<name>` and as bare `<name>`; use the `homey/event/…` form.

## Probe call (one browser call per layout iteration)

Loads the draft (gzip + base64), waits, and checks everything that can be checked without looking:

```js
(async () => {
const sleep = ms => new Promise(r => setTimeout(r, ms));
const bin = Uint8Array.from(atob(B64), c => c.charCodeAt(0));
const data = JSON.parse(await new Response(new Blob([bin]).stream().pipeThrough(new DecompressionStream('gzip'))).text());
// never await the load bare: a migration notice would block it until the 45 s tool timeout
const load = await Promise.race([Dashboard.loadDashboardData(data).then(() => 'loaded'), sleep(12000).then(() => 'TIMEOUT – dialog open? (see "Opening the editor")')]);
await sleep(5000);
const ld = Dashboard.liveData || {};
const used = new Set();
for (const w of Object.values(data.widgets)) {
  const walk = b => { if (typeof b === 'string') { if (b) used.add(b); } else if (b && typeof b === 'object') Object.values(b).forEach(walk); };
  walk(w.bindings);   // bindings can be nested (graph series[i].data since v2.3.0)
  for (const m of String(w.overrides.text || '').matchAll(/\{\{(homey\/[^{}]+?|[a-z0-9-]+\/[^{}\s]+?)\}\}/g)) used.add(m[1]);
}
const missingLive = [...used].filter(t => !t.startsWith('local/') && !(t in ld));
// v2.3.1 (verified): the widget's DOM lives in w.elements (container, innerContent, wrapper, …)
const elOf = w => { const e = w.elements || {}; return e.container || e.wrapper || Object.values(e).find(x => x instanceof HTMLElement) || w.el || w.element || null; };
const R = el => { const r = el.getBoundingClientRect(); return [r.left, r.top, r.right, r.bottom].map(Math.round); };
const MIN_FONT = 16;   // canvas px; 16 for wall panels, 13 for phone/desktop pages
const problems = [], hints = [];
for (const [id, w] of Object.entries(Dashboard.widgets)) {
  const el = elOf(w);
  if (!el) { problems.push(id + ': NO ELEMENT – checks did not run'); continue; }   // never treat as a pass
  const type = data.widgets[id]?.type;
  if (type === 'graph' && w.legendEl && getComputedStyle(w.legendEl).display !== 'none') {
    const [bl, bt, br, bb] = R(el), [ll, lt, lr, lb] = R(w.legendEl);
    if (ll < bl || lt < bt || lr > br || lb > bb) problems.push(id + ': legend outside the widget (needs free canvas space or legendPosition floating)');
  }
  if (type !== 'text') continue;
  const inner = (w.elements && w.elements.innerContent) || el;
  if (/(^|\s)-(\s|$)/.test(inner.innerText || '')) problems.push(id + ': "-" (formula?)');
  // overflow is only a hint: small boxes report it although nothing is clipped (inner padding counts)
  if (inner.scrollHeight > inner.clientHeight + 2 || inner.scrollWidth > inner.clientWidth + 2) hints.push(id + ': overflow ' + inner.scrollWidth + 'x' + inner.scrollHeight + ' > ' + inner.clientWidth + 'x' + inner.clientHeight);
  let minFont = Infinity;
  for (const n of inner.querySelectorAll('*')) {
    if (n.childElementCount || !n.textContent.trim()) continue;
    minFont = Math.min(minFont, parseFloat(getComputedStyle(n).fontSize));
    if (/\d/.test(n.textContent) && n.getClientRects().length > 1) hints.push(id + ': number wraps');
  }
  if (minFont < MIN_FONT) hints.push(id + ': font ' + Math.round(minFont) + 'px');
}
return JSON.stringify({load, connected: Dashboard._dataDriverConnected, unsaved: Dashboard.hasUnsavedChanges,
  widgets: Object.keys(Dashboard.widgets).length, missingLive, problems, hints});
})()
```

Fix every entry in `missingLive` and `problems` before the screenshot. `unsaved: true` right after loading means the migrator changed the draft – a legacy key is still in it. `hints` are not proof of a defect: in the first v2.3.1 test 11 of 11 "overflow" hints were false (labels, hero values and a table that all rendered cleanly), while the two real defects (a table one row too short, a legend outside the chart) were only visible in the screenshot. The screenshot is therefore **mandatory** before handing over. Take it as a full `screenshot` at `scale` 0.6–0.7; a `zoom` region refers to the latest coordinate frame, which changes when the pane is resized, so a zoom planned from an older screenshot can miss the canvas. Note that `getComputedStyle` reports the font size before any canvas scaling.

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

Verified details: the save dialog is `#file-modal` (class `show`), the input is `#save-filename`, the primary button text is exactly `Save` (class `ds-dialog-button primary`). After saving, a dialog "Success – Saved successfully!" with an `OK` button appears and must be closed. The file list in the dialog (`.ds-file-list-name`) is not proof of success; only `fetch('dashboards/<Name>.json')` = 200 is. A file saved by v2.3.1 contains `"dynamicFloor": {"autoMin": true}` and `"dynamicCeiling": {"autoMax": true}` inside `master.widgetDefaults`: that is the nested save format, not a legacy key, and the file reopens without a migration notice (checked with `DashboardMigrator.migrate`).

**File names:** use a clear name without spaces, e.g. `Title_yyyymmdd_V#`, and save changes as a new version (`_V2`, `_V3`, …) instead of overwriting, unless the user asks otherwise. Exception: the start page must be called exactly `dashboard`.

## Fallback: public demo editor

If the user's editor cannot be reached, test layout and formulas in `https://satoer.com/dashboardstudio/demo/editor.html` with the same loading code, but set `data.master.connection = {dataDriver: 'simulation', driverSettings: {interval: 1000}}` and feed sample values with `Dashboard.applyLocalTopicPayload(topic, value)`. Topic names must still come from the user's list; the demo cannot verify them. The public demo is not a LAN address, so it does not ask per action – use it for layout iterations that need many calls, and the user's editor only for the final probe and the commit.
