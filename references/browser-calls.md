# Dashboard Studio – Browser calls

The JavaScript jobs for the user's editor: opening it, discovery, probe and commit, plus the public demo fallback. Read before the first browser call.

## Opening the editor

- `request_access {url: "http://<ip>:5050", scope: "once"}`, then `preview_start` with the editor URL (or `navigate` in the tab the user opened; check `tabs_context` once).
- **Half-loaded page:** if the approval only covered the document, sub-resources may be missing; the page then shows unstyled buttons and `typeof Dashboard === 'undefined'`. Detect this inside the next JS call (return early with that information) and reload once with `navigate`.
- **Start behaviour (verified in `dashboard.js`):** on start the editor loads `dashboards/dashboard.json`; if that file does not exist it falls back to `demo.json` (platform "simulation"). The editor strips all URL parameters except `token` and `test_mode`, so there is no URL parameter to open a specific dashboard in the editor; `?dashboard=<Name>` only works in `viewer.html`. A dashboard saved under the name **`dashboard`** is therefore the editor's start page (and the viewer's default without parameter).

## Discovery call (one browser call)

Switches the editor to the Homey platform, waits, and returns the topic list plus the topic metadata (access flags, flows, moods, variables):

```js
if (typeof Dashboard === 'undefined') { 'NOT_LOADED' } else {
const minimal = {version: window.DASHBOARD_VERSION || "2.2.0", master:{connection:{dataDriver:"homey",driverSettings:{}},
  environment:{canvas:{width:1280,height:800},theme:"default_dark",layout:{mode:"scale",scale:{keepAspectRatio:true}}}},
  globalBindings:{}, widgets:{}, groups:{}, selectionGroups:{}};
if (!Dashboard._isHomeyPlatform()) { await Dashboard.loadDashboardData(minimal); }
await new Promise(r => setTimeout(r, 6000));
const tm = Dashboard.topicMeta || {};
const special = Object.fromEntries(Object.entries(tm)
  .filter(([t, m]) => m && (m.isFlow || m.isMood || m.isVariable))
  .map(([t, m]) => [t, {flow: !!m.isFlow, mood: !!m.isMood, variable: !!m.isVariable,
                        cap: m.capabilityId, title: m.title, set: !!m.setable, get: !!m.getable}]));
JSON.stringify({version: window.DASHBOARD_VERSION, isHomey: Dashboard._isHomeyPlatform(),
  topics: Object.keys(Dashboard.liveData).sort(), special});
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

## Fallback: public demo editor

If the user's editor cannot be reached, test layout and formulas in `https://satoer.com/dashboardstudio/demo/editor.html` with the same loading code, but set `data.master.connection = {dataDriver: 'simulation', driverSettings: {interval: 1000}}` and feed sample values with `Dashboard.applyLocalTopicPayload(topic, value)`. Topic names must still come from the user's list; the demo cannot verify them. The public demo is not a LAN address, so it does not ask per action – use it for layout iterations that need many calls, and the user's editor only for the final probe and the commit.
