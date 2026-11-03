# Dashboard Studio – Widgets and file format

File schema, widget types and fields, graph widget, live values and formulas in text, binding formulas, colors. Read before generating or editing dashboard JSON.

## File schema

```json
{
  "version": "<installed version from version.js, e.g. 2.3.1>",
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

- Since v2.0.14 the editor saves a nested "V2" format (`save-format-v2.js`): label-based nested keys on disk (e.g. `container.background.startColor`), flat keys at runtime. Empty `groups`/`selectionGroups` are omitted on save. The schema above still imports correctly.
- A minimal `master` is enough. Missing parts (`typography`, `container`, `theme.colors`, `widgetDefaults`) are filled from defaults.
- `dataDriver` is one of `homey`, `mqtt`, `nodered` or `simulation`. Use `homey` for the user's own Homey; it also switches the editor's platform on load.
- `layout.mode`: `fixed` uses exact pixels ("Fixed Resolution"), `scale` scales to the window using the same coordinates ("Scale to Fit Screen"), `responsive` uses columns ("Responsive Columns").
- Themes include `default_dark`, `eink_dark`, `eink_light`, `midnight_blue`, `deep_space`, `executive_suite` and `paper_white` (35+ built in).
- **Widget object:** `type`, `overrides` (position and all properties) and `bindings` (field → topic; can be nested, e.g. graph `series[i].data` since v2.3.0 – walk bindings recursively when collecting topics). Required in `overrides`: `x`, `y`, `width`, `height`, `page` (a number or a string like `"1, 2, 7-12"`). Optional: `metaName` (name shown in the editor), `zIndex`, `visible`, `locked`, and `container: {"show": false}` to hide the card frame.
- Unknown field names: take them from an Auto-Widget Gallery skeleton or read them from the widget source via HomeyScript (see "Source research without the browser" in `source-research.md`), never by trial and error with extra browser calls.

## Widget types and key fields

| type | purpose | fields (overrides) | bindable (bindings) |
|---|---|---|---|
| `text` | Markdown with live values | `text`, `textAlign`, `padding`, `mdTableStyle` (`minimal`, `solid_header`), `contentAnimation`, `mdAutoFitText` (true = scale text to fill the widget) | `text`, `visible` |
| `empty` | container or card with a caption | `label` | `visible` |
| `icon` | Phosphor icon that glows when active | `label`, `size`, `active`, `activeIconMirrorsInactive`, `blink`, `icons:{off:{source:"phosphor",builtin:"ph-sun",weight:"fill",color:"@icon"}, on:{…,color:"@warning"}}` | `active`, `label` |
| `arrow` | line or arrow with flow animation | `fromX`, `fromY`, `toX`, `toY` (absolute canvas coordinates), `customPoints:[{x,y}]` for bends, `pathBendRadius`, `thickness`, `headSize` (0 = no head), `lineRadius`, `headRound`, `mainColor`, `animationEnabled`, `animationSpeed`, `animationReverse`, `flowWidth`, `gradientWidth`, `enableGlow` | `animationEnabled`, `animationReverse`, `thickness`, `visible`, colors |
| `gauge` | round or horizontal gauge | `label`, `unit`, `min`, `max`, `decimals`, `showIcon` (default true, shows a thermometer!), `gaugeCol0`, `gaugeCol100`, `barThickness`, `trackThickness`, `gapRound`, `gaugeSizeOffset` | `value` |
| `battery` | battery bar (no numeric % shown) | `label`, `vertical` | `value` |
| `graph` | line, stepped, bar or big-number chart; several series per widget since v2.3.0 | see "Graph widget" | `series[i].data` (nested), per-series colors, numeric fields such as `min`, `max` |
| `switch` | toggle or push button, also navigation | `buttonLabel`, `mode` (`toggle`/`push`), `style` (`switch`/`button`), `actionType` (`topic`/`page`/`url`), `pageTo` (target page for `page`), `urlTo`, `urlNewTab`, `outputTopic`, `separateOnOffTopics` + `outputTopicOff`, `onPayload`, `offPayload`, `showIcon`, `iconSize`, `icons` | `state` |
| `slider` | dimmer or setpoint | `min`, `max`, `vertical` | `value` |
| `list` | log or feed | – | `data` |
| `image` | URL or SVG image | `url`, `fit`, `liveRefresh`, `refreshInterval` | `svgRaw`, `url` |
| `clock` | analog, flip or text clock | `displayMode` | – |
| `menu` | navigation menu; items with `actionType` `topic`, `page` or `url` (`urlTo`) | – | see the editor |
| `lightgrid`, `editbox` | light tiles, text input | – | see the editor |

**Arrow bounding box:** `x`, `y`, `width` and `height` enclose all points with about 40 px padding. The arrowhead sits at the `to` point. When `animationReverse` is true, the flow runs backwards and the head jumps to the other end. Default colors: flow `@success` (green), reverse flow `@danger` (red). In view mode, arrows let clicks pass through.

**Text size:** `#` headings render small on a 1280 px canvas. For ONE prominent value use `mdAutoFitText: true`. Do not set it on several tiles: auto-fit scales each box by its text length, so "2.78" and "10.27 €" end up in different sizes. For several key figures of equal rank set the same fixed size instead (verified v2.3.1): flat keys `textMdH1Fs` (default 24), `textMdH2Fs` (18), `textMdH3Fs` (14), `textMdParaFs` (12), `textMdThFs` / `textMdTdFs` (table header / cells, 12), e.g. `{"text": "# {{homey/event/x}}", "textMdH1Fs": 64}`. Further text keys: `textVerticalAlign` (`top`, `middle`, `bottom`), `mdTextWrap`, `textMdEqualTableColumnWidths`, `textMdTableCellPadding`. A text widget keeps roughly 12 px inner padding per side, so size short boxes generously. A Markdown table that may grow (e.g. a top-N list) gets its own text widget with `mdAutoFitText: true`; headings and footers go into separate widgets so they are never pushed out.

**Gauge:** set `showIcon: false` unless an icon is wanted; with the default layout the caption overlaps the arc – `gapRound: 6` and `gaugeSizeOffset: 22` (200×140 widget) fix that.

**Navigation buttons (verified):** a `switch` with `mode: "push"`, `style: "button"`, `actionType: "url"`, `urlTo: "viewer.html?dashboard=<Name>"`, `urlNewTab: false`, `buttonLabel`, `showIcon: true`, `icons: {on:{source:"phosphor",builtin:"ph-cpu",weight:"fill",color:"#38bdf8"}, off:{…}}`. Ideal for a start page saved as `dashboard`.

## Graph widget (rewritten in v2.3.0)

Source-verified 2026-11-03 from `widgets/graph.js` and `migrator.js` of v2.3.1. Run-tested in the editor on v2.3.1 (2026-11-03): a `bar` graph with two series bound to `homey/event/…` JSON arrays of `{t, v}` (14 daily points), `timeFormat: "Date"`, `dateFormat: "DD-MM"`, `autoAxisResX: false` + `axisResX: 1`, `dynamicFloor: false` + `fixedFloor: 0`, and a `floating` legend. Items marked "(verified < 2.3)" were run-tested on the old widget only and must be re-checked in the probe call.

- **Several series per widget:** `overrides.series` is an array. Series fields: `id` (unique, e.g. `"s_pv"`), `name`, `nameLocked`, `visible`, `data`, `lineWidth`, `positiveColor`, `negativeColor`, `positiveFillColor`, `negativeFillColor`, `positiveStartOpacity` (default 40), `positiveEndOpacity` (default 0), `negativeStartOpacity`, `negativeEndOpacity`, `pointColor`, `valueMultiplier` (0 = off, e.g. `0.001` for W → kW), `positiveOutlineColor`, `negativeOutlineColor`. The fill is a gradient from start to end opacity; no fill = start opacity 0. Missing colors fall back to the theme marker tints.
- **Binding the data (nested):** `"bindings": {"series": [{"data": "<topic A>"}, {"data": "<topic B>"}]}`, index-aligned with `overrides.series`. Per-series colors bind the same way (`bindings.series[i].positiveColor`).
- **Removed keys** (the widget drops them): top-level `lineColor`, `fillCol`, `negFillCol`, `fillOpacity`, `lineWidth`, `fill`, `barColor`, `pointColor`, `linearTimeScale`, and `bindings.data`. Old dashboards are migrated on load (`migrator.js`, rule 2.3.0 "Graph multi-series"; it also runs for files with a current `version` that still have the old shape). New JSON must use `series`.
- **Data:** unchanged – a JSON string or array of `{t: ISO-timestamp, v: number}` (sorted by `t`), a comma list of numbers, or an object `{label: value}`. Insight entries fit directly.
- **Types (`type`):** `line`, `smooth` (`smoothingAmount` 0–100), `stepped` (new – on/off states, setpoints), `bar` (`barDirection`: `vertical`/`horizontal`, `barGap`, `barRadius`), `text` (big number).
- **Defaults that bite:** `maxPoints` **20** in `bufferMode: "points"` (set e.g. 400); `showLegend` **true** (legend on the right) – set `false` for a single series; `showAxisX`/`showAxisY` false; `label` default "Graph" – set `""`; `timeFormat` default `Datapoints`.
- **Time window instead of point count (new):** `bufferMode: "duration"` with `durationPreset` (`15m`, `1h`, `6h`, `12h`, `24h`, `7d`, `30d`, `custom` + `durationCustomValue` and `durationCustomUnit` `minutes`/`hours`/`days`), `fixedTimeWindow`, `timelineAdvance` (`data` or `time` = glides with the real clock). Also `reverseDirection`, `lineGaps` (`break` / `connect`).
- **Scale:** canonical `dynamicFloor` / `dynamicCeiling` (default true) and `fixedFloor` / `fixedCeiling`. The old `autoMin`/`autoMax` and `min`/`max` are still read as fallback when the new keys are absent, but **do not write them** (verified): as widget overrides they count as the legacy graph shape (`_dashboardHasLegacyGraphSeriesShape` in `migrator.js`), so loading the file runs the migration and opens the blocking "Dashboard Upgraded" notice (see `browser-calls.md`). Binding formulas such as `min_bind_formula` are not affected. Since v2.3.0 the dynamic floor fits the data instead of being forced to 0. Further: `headroom`, `useBaseline` + `baselineVal`, `negMode` (`show_neg`, `show_pos`, `ignore`), `decimals`, `roundingMode` (default `smart`).
- **Axes and grid:** `showAxisX`, `showAxisY`, `axisWidthX`, `axisWidthY` (fixed reserved space), `axisResX`, `axisResY`, `autoAxisResX`, `autoAxisResY`, `axisXFontColor`/`axisYFontColor`, `axisXFontOpacity`/`axisYFontOpacity` (0–100, new), `categoryLabelRotation` (tilted labels), `timeFormat` (`Datapoints`, `TextLabels`, `DateHours`, `Date`, `HH`, `HH:mm`, `HH:mm:ss`), `dateFormat` (default `DD-MM-YYYY`), `time24h`, `timezoneConvert`, `gridHInnerShow`/`Col`/`Op`, `gridHBotShow`, `gridV…`, `padding`, `container`.
- **Legend (new):** `showLegend`, `legendPosition` (`top`, `bottom`, `left`, `right`, `floating`), `legendLayout` (`row`, `column`), `legendSwatch` (`line`, `square`, `circle`), `legendShowContainer`, `legendAutoSize` (true = fit to content; else `legendW`/`legendH`), `legendItem…` typography. Tapping an entry toggles that series.
  - **The legend sits OUTSIDE the widget box** in the fixed and scale layouts (source `_legendFrame`, verified in the editor): `top` places it 8 px *above* the widget, `bottom` below, `left`/`right` beside it. It overlaps whatever is there – in the first test it covered the comparison line of the card above the chart. Only the responsive layout stacks it inside the widget.
  - To put it inside the chart, use `legendPosition: "floating"` with `legendOffX` / `legendOffY` in **absolute canvas coordinates** (verified: `1070, 290` on a 1280 canvas placed it top right inside a chart at `x 24, y 286, width 1232`), plus `legendAutoSize: true` and `legendShowContainer: false`. Otherwise reserve free canvas space where the legend goes.
  - With that placement a built-in legend replaces hand-written color legends in text widgets.
- **Point values:** `showPointValues`, `pointValuesInterval` (with `autoPointValuesInterval`), `pointValuesDecimals`, `pointValuesHoverOnly` (show the value only under the mouse – useful on desktop pages, does nothing on a touch panel), `pointValuesAbbreviate`.
- **Animation (new):** `graphAnimation` (`none` default, `morph` = "Draw In & Morph"), `graphAnimDuration` (0.2–5 s, full update), `graphAnimStreamDuration` (0.1–2 s, streaming). Hover values and animation cannot be checked in a screenshot; ask the user to look in the viewer.
- **X labels (verified < 2.3):** `autoAxisResX: false`, `axisResX: <every n-th point>` (e.g. 24 for 144 points per day or 168 hourly points per week). `linearTimeScale` no longer exists; since v2.3.0 time steps snap to calendar units, so check the labels again.
- `smooth` can overshoot below the floor; use `line` for overlays with a fixed scale.
- **Two measures with the same unit:** two entries in `series` of ONE widget. This replaces the old overlay of two graph widgets.
- **Two measures with different units (second scale, verified < 2.3):** all series of a widget share one Y scale (no per-series axis key in the source), so a second scale still needs two graph widgets with **identical x/y/width/height and identical reserved axes** on top of each other (second one with higher `zIndex`, fill start opacity 0). Keep `showAxisX: true` and the same `axisWidthX` on BOTH, and hide the top one's labels (`axisXFontOpacity`/`axisYFontOpacity: 0`, grid off); otherwise the top graph's plot area is taller and its line runs below the bottom edge. Give the top graph a fixed scale bound to precomputed min/max events: `dynamicFloor: false`, `dynamicCeiling: false`, `headroom: 0`, `bindings: {min: "homey/event/s_x_min", max: "homey/event/s_x_max"}`, `min_bind_formula: "round(value - 0.5)"`, `max_bind_formula: "round(value + 0.5)"` (the bound `min`/`max` are read because `fixedFloor`/`fixedCeiling` are unset). Narrow both graphs by ~46 px and put 4 small text widgets in that strip, one per grid line, showing `{{round((LO + k*(HI-LO)/3)*10)/10}}°` with the same LO/HI expressions. Plot area: top ≈ `y + 5`, bottom = `y + height - axisWidthX`; with `axisResY: 4` and `autoAxisResY: false` the grid lines are at 0, 1/3, 2/3 and 1 of that span. v2.3.0 changed the title layout and axis steps, so re-check this geometry.

Minimal series graph:

```json
{"type": "graph",
 "overrides": {"x": 20, "y": 20, "width": 600, "height": 260, "page": 1, "label": "", "type": "line",
   "showLegend": false, "showAxisX": true, "showAxisY": true, "maxPoints": 400, "timeFormat": "HH:mm",
   "series": [{"id": "s_pv", "name": "Solar", "positiveColor": "@accent", "positiveFillColor": "@accent",
               "positiveStartOpacity": 30, "positiveEndOpacity": 0}]},
 "bindings": {"series": [{"data": "homey/insight/pv_24h"}]}}
```

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

Theme corrections (master theme): since v2.0.13 the sliders are **Color Tone** (formerly Hue Shift, −180° to +180°), **Color Intensity** (formerly Saturation, 0–200 %), **Brightness** and **Contrast**. JSON keys not verified – read them from the source before setting them. Since v2.1.0 themes change only visual styling, never layout or behaviour.

Theme tokens: `@background @backgroundAccent @surface @surfaceAlt @text @textMuted @accent @accentSoft @accentSecondary @icon @iconMuted @success @warning @danger @glow @border @borderAccent @onAccent`. Hex, RGB, HSL and CSS color names also work. For overlays use explicit hex colors per measure and repeat them in the legend text (`[color #38bdf8]━ Last %[/color]`).
