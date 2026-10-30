# Dashboard Studio – Source research and extras

Reading the app's own source and help via HomeyScript, the Auto-Widget Gallery, variables/flows/moods with a Personal API Key, and further reference sources.

## Source research without the browser (verified v2.2.0)

The editor web server is reachable from HomeyScript on the Homey itself at `http://127.0.0.1:5050/`. This costs the user **no approval click**. Use `homeyscript_run` with inline `code` and `dry_run: true` (GET fetches are allowed in a dry run). Useful files:

| file | content |
|---|---|
| `version.js` | `window.DASHBOARD_VERSION` (use it for the JSON `version` field) and `LATEST_MIGRATION_VERSION` |
| `release-changelog.json` | the app's full release notes as JSON (`[{version, date, changes[]}]`, newest first) – read this first after an update |
| `help/help-content.json` (~415 KB) | the full official help of the installed version (satoer.com blocks automated fetchers) |
| `widgets/<type>.js` | config keys (`c.<field>`), defaults (`static get globalSettings`), `dependsOn` rules |
| `dashboard.js`, `drivers/homey.js`, `save-format-v2.js` | load/save/publish logic, on-disk format |
| `topic-alias.js` | canonical ↔ legacy names for variables, flows, moods (since v2.2.0) |
| `scripts/auto-widget-gallery.js` | the app's own topic → widget classifier (see "Auto-Widget Gallery") |
| `dashboards/<Name>.json` | saved dashboards |

Return only grep hits with line numbers (filter by regex, cap at ~60 lines per file) – never whole files.

```js
const B = 'http://127.0.0.1:5050/';
const t = await (await fetch(B + 'widgets/switch.js')).text();
return t.split('\n').map((l, i) => (i + 1) + ': ' + l.trim())
        .filter(l => /actionType|pageTo|outputTopic/.test(l)).slice(0, 60);
```

## Auto-Widget Gallery (new in v2.1.0)

The app now classifies live topics itself and suggests a pre-configured widget per topic (type, preset, icon, unit, min/max). In the UI: Tools → **Auto-Widget Gallery**, the button above the topic list in the **Data Stream Explorer**, and the last step ("Populate Your Canvas") of the **Reset Canvas** wizard. The author calls the feature not yet finished.

Use it as a **skeleton source** for widget fields instead of guessing names. The script is lazy-loaded; in the discovery call (source-verified, not yet run-tested):

```js
await WidgetScriptLoader.loadScript('scripts/auto-widget-gallery.js');
const {entries} = AutoWidgetGallery.classifyAll();
const skel = entries.map(e => ({name: e.name, type: e.type, rule: e.ruleId, tile: e.tile,
  overrides: e.overrides, bindings: e.bindings}));
```

- `entries[].overrides` come from the first preset of the chosen widget type, with `x: 0, y: 0` and WITHOUT `width`/`height`; position and size are still yours. `bindings` reference the real topic.
- `AutoWidgetGallery.traceTopic(topic)` explains which rule matched one topic; `getRulePipeline()` lists all rules.
- Rule order (first match wins; a pre-pass first groups lights into one light grid, 2x2): write-only action/flow/mood/button → push switch · writable boolean (`onoff`, `alarm_*`, boolean variable) → switch · read-only boolean (contact, motion, alarm) → icon · Markdown/table/calendar → text · SVG markup → image · image URL → image · video URL → video · Phosphor icon slug → icon · color → container · numeric series / insight stream → graph · string list / log → list · battery → battery · fraction capability (`dim`, `volume_set`, `windowcoverings_set`, `light_hue`, `light_saturation`, `light_temperature`) → gauge (read) or slider (write) with `min: 0, max: 1, unit: "%"` · other number → gauge (read) or slider (write) · long text (> 40 chars) → text · short text → label (read) or edit box (write).
- The gallery also knows the widget types `label` and `video`, which are not in the table above; read their fields from the skeleton.
- Review the result: the classifier only sees one sample payload. Report misclassifications to the author with topic name, payload and expected widget type.
- `AutoWidgetGallery.place(specs)` adds widgets to the live canvas (editor history action) – only on the user's explicit request, it changes the open dashboard.

## Variables, Flows and Moods (since v2.0.13, Personal API Key)

- **What it unlocks:** writing Homey Logic variables from `editbox`, `switch` and `slider` widgets (access RW instead of R), starting Standard and Advanced Flows from buttons/switches (capability `start`, access W), and activating zone Moods (capability `set`, access W).
- **Not needed for:** reading variables (read-only works without a key), sensors, and normal device control.
- **The user creates the key, never Claude:** Homey Web App → Settings → API Keys → New API Key. Scopes: Variables `homey.logic`; Flows `homey.flow.start` + `homey.flow.readonly`; Moods `homey.mood.set` + `homey.mood.readonly`. Then Homey app → Settings → Dashboard Studio → Settings → **Personal API Key**; green badges confirm each permission. Never ask the user to paste the key into the chat and never write it with `apps_set_setting`.
- **Publishing items:** Dashboard Studio settings → Configuration → **System & Automation** (below the devices): toggle User Variables, Flows (grouped by folder) and Zone Moods (grouped by zone), then **Save Settings**. Where these selections are stored is not verified, so let the user toggle them in the UI.
- **Topic names (since v2.2.0, `topic-alias.js`):** variables `homey/variables/<name>/value`, flows `homey/flows/<name>/start` (and `/enabled`), moods `homey/moods/<name>/set`. Legacy aliases `<name>/value`, `<name>/start`, `mood-<name>/set` still resolve. `<name>` is still created server-side – take it verbatim from the discovery call, which now lists both forms. `special` there shows `flow`/`mood`/`variable`, `cap` and `set`/`get` (W = set without get).
- **Widget:** write-only flow and mood topics belong on a `switch` with `mode: "push"` (that is also what the Auto-Widget Gallery picks).
- **Cooldowns:** a flow can be started once per **10 s**, a mood once per **3 s**; extra taps are ignored. Never bind a flow or mood to a slider or a constantly changing topic.
- Starting a flow or activating a mood has real effects in the house – test-press only after the user's explicit confirmation.

## Reference sources

- Demo dashboard with many widget examples: `https://satoer.com/dashboardstudio/demo/dashboards/demo.json` (fetch it from inside the demo tab).
- Help content and release notes: preferably the local copies on the Homey (`help/help-content.json`, `release-changelog.json`), read via HomeyScript. Public release notes: `https://satoer.com/dashboardstudio/changelog.json`. Announcements and discussion: Homey Community thread "[APP][Pro] Dashboard Studio" (topic 151169); single posts as JSON via `https://community.homey.app/posts/<post-id>.json`.
- Saved dashboards on the user's Homey: `http://<ip>:5050/dashboards/<Name>.json`.
- Widget source code on the user's Homey: `http://<ip>:5050/widgets/<type>.js` (from HomeyScript: `http://127.0.0.1:5050/…`), startup and save logic in `dashboard.js`, driver in `drivers/homey.js`.
- Documentation pages on satoer.com block automated fetchers (robots.txt). Read them only if the user opens them in their own browser view, and do not work around the block.
