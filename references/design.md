# Dashboard Studio – Design rules

How to lay out content so a dashboard is understood at a glance: page plan, hierarchy, labels, colors, limits and automatic checks. Read before planning the pages and widgets of a new dashboard, and before reworking an existing one. The technique (schema, fields, formulas) is in `widgets.md`; this file decides **what goes where**.

Credit: sections 1–3 and 5 adapt the principles of [rubenhassid1/grenade](https://github.com/rubenhassid1/grenade) by Ruben Hassid (MIT license), which were written for Claude artifacts and reports. They are rewritten here for Homey dashboards. Section 4 is not from that source.

## 1. Agree on the target picture before building

Ask these as clickable questions if the request does not answer them, and ask nothing else:

1. **Who looks at it?** Only the builder, the whole household, guests? What do they already know?
2. **Where and how?** Wall tablet seen from 2–3 m, phone in the hand, desktop browser? Touch or mouse?
3. **What should they do afterwards?** Check that everything is fine, operate something, review a trend, or look something up?

Then propose a **page plan**: one line per page with its job and one sentence of the form "after 3 seconds the reader knows X / can do Y". That sentence is the spine of the page. Get the page plan approved before generating JSON; layout iterations in the editor cost browser approvals, a page plan in chat costs none.

## 2. One page per job

A "comprehensive" dashboard does not mean everything on one page. Split it by job and link the pages with navigation buttons or a `menu` widget (see `widgets.md`). Pages differ in layout; do not reuse one tile grid for every job.

| The reader needs to… | Page | It looks like |
|---|---|---|
| Check that all is fine (start page) | Status page | A status sentence on top. At most 4 big values. Exceptions as colored words or glowing icons. Everything else one tap away. |
| Operate (lights, shading, scenes) | Control page | Only switches, sliders, light grid. Big touch targets. Each control shows its current state. No charts. |
| Review (energy, heating, history) | Report page | At most 4 key figures, each with its comparison. ONE main chart, the series that matters in `@accent`, the rest grey. |
| Follow a flow (energy, water) | Flow page | One diagram with arrows. A number with unit on every node and every active arrow. |
| Look up (device status, batteries, readings) | Reference page | Cards grouped by topic, at most 5 items each, at most one table. |

When a page mixes two jobs, lead with the main one.

## 3. Rules

1. **Show it, label it.** Every visual carries a readable number with unit. On graphs the axis title (`axisYLabel`, since v2.3.5) can carry the unit. A gauge shows its value; a graph shows a Y axis or a current-value label; the `battery` widget shows no number, so add the value as text or use a text widget instead. Icons show state (glow when active), never decoration. Test: hide all static text – the page still makes sense.
2. **The top says the state.** The first element on the status page is one sentence built from formulas: "All fine" or "2 things need attention: window bathroom open · pool pH 7.8". Build it from several short `if()` expressions next to each other (length limit, see `widgets.md`).
3. **Plain labels.** Write labels the way you would say them: "Pool water", "Grid import now" – not "Power pulse" or "Aqua status". One label per value. No subtitle under a heading, no caption under a number.
4. **Every changing number has a comparison – and it compares like with like.** "12.4 kWh · +2.1 vs yesterday until 18:17", "21.3 °C · target 21.0", never a bare delta. A running period (today so far, this week so far) is compared with the same span of the reference period up to the same time, never with the whole previous day or week: "today until 17:00" against "all of yesterday" almost always looks like a saving (in the first test −2.4 kWh, while the fair comparison was +1.0 kWh). If only the whole reference period is available, the label says so ("vs. all of yesterday"). Averages of complete days are fine when the label names them ("Ø 7 days"). Comparisons need data (insight histories, computed values) – plan them with `data-feeds.md`. If no comparison is available, show the target or limit instead.
5. **The biggest thing is the most important thing.** At most 4 big numbers per page; at most 5 text sizes on the whole dashboard. A single hero value may use `mdAutoFitText: true`. Several key figures of equal rank (a report page) all get the same fixed size instead (`textMdH1Fs`, see "Text size" in `widgets.md`), because auto-fit scales each box by its text length and makes equal figures look unequal. The eye must know where to go first, second, third.
6. **Color means something.** `@success`, `@warning`, `@danger` only for good, watch and bad. `@accent` only for the one thing to look at and the highlighted chart series. Everything else uses `@text`, `@textMuted` or grey. A graph with several series (since v2.3.0) colors each series from the theme marker tints unless set; set `positiveColor` explicitly so that only the important series uses `@accent`. Arrows default to `@success` green – on a flow page set an explicit `mainColor` per measure and keep green for "good". Give each measure one color and use it everywhere (solar is the same color on the flow page and in the chart legend). One theme for the whole dashboard; no per-widget restyling.
7. **Hard limits per page:** at most 4 sections, at most 5 items per card or list, at most 1 table, at most 4 big numbers, no grid with more than 6 columns. More than that? Keep what the reader acts on and move or cut the rest.
8. **Say each thing once per page.** The same value does not appear as a gauge, a text value and a graph on one page. Helper bindings (arrow thickness, animation, `visible`) do not count. Totals must add up: values from different devices are sampled at slightly different times, so compute the remainder from the others instead of showing three raw values that never sum.
9. **Cut, don't hide.** Moving detail to another page is fine only for reference detail. An exception that needs action stays on the status page.
10. **Nothing spills.** Every number fits on one line at every scale. Size boxes for the longest expected value ("-12.5 °C", "10 000 W"), not the current one, and for the height of their content: a Markdown table needs about (rows + 1) × 2.3 × font size plus ~24 px padding (header + 3 rows at 16 px ≥ 170 px; 150 px cut off the last row in the first test). A graph legend in the fixed or scale layout is drawn outside the graph widget and covers its neighbours unless it is `floating` inside the chart or has free space (see `widgets.md`).
11. **Mark estimates.** A value computed with an assumption (a price taken from another day, a gap filled by interpolation) gets a visible `*` at the number and one line in the footnote saying what was assumed. Remove the mark as soon as the real data exists.
12. **Numbers the reader knows.** Graph axes abbreviate large values by default (`abbreviateLargeValues`: 12.5k). For readers who do not think in "k", switch it off or convert with the series `valueMultiplier` (W → kW) and name the unit in the axis title.

## 4. Home-dashboard specifics

These go beyond the source above, because a wall dashboard is read hundreds of times in passing rather than once at a desk.

- **Readable from a distance.** As a rule of thumb, on a wall panel no text below about 16 canvas px on a 1280-px-wide canvas; 13 px is acceptable for phone or desktop pages. Check on the real device.
- **Color plus word.** A state is never shown by color alone ("Open" in `@danger`, not just a red dot).
- **Calm when normal.** When everything is fine, the status page is quiet. `[blink]` and `@danger` are reserved for real alarms, so they still stand out.
- **Back to the status page.** A wall panel should return to the status page after inactivity (environment settings `inactivityPageEnabled`, `inactivityReturnPage`, `inactivityTimeoutSeconds`; set them in the editor). Swipe gestures (v2.3.0, beta) can replace navigation buttons on touch panels, but keep a visible way back for people who do not know the gesture.
- **Night.** Use a dark theme; if the panel runs at night, consider a dimmed night page or hide non-essential widgets with `visible` bindings.
- **Touch targets.** Controls at least about 60 canvas px in their short dimension, with space between neighbouring controls.

## 5. Slop signs – fix before showing

- A row of identical tiles, each with its own mini-chart. Use one main chart with plain numbers around it.
- Visuals without numbers: gauges without value, graphs without axis or value, battery bars, icons standing in for data.
- More than 4 numbers fighting for attention at the top.
- Clever labels instead of plain ones; a subtitle or caption under every heading or number.
- A change without comparison ("+12 %"), or a running period compared with a complete one ("today so far" vs. "all of yesterday").
- An estimated value that looks exactly like a measured one.
- The same value shown two or three times on one page; totals that do not add up.
- Green used both for "power flows" and for "good".
- Every page in the same tile skeleton, whatever its job.
- Numbers broken onto two lines, text spilling out of its card, grids with 7+ columns.
- Everything on one long page instead of one page per job.
- Stacked bars whose segments do not add up to the shown total, or stacked series in different units.

## 6. Automatic checks

**Static check (build script, no browser):** run after generating the JSON and before the probe call; report the warnings to the user together with the page plan, or fix them.

```python
import re, collections

TOPIC = re.compile(r'\{\{(homey/[^{}]+?|[a-z0-9-]+/[^{}\s]+?)\}\}')
DISPLAY_BINDINGS = {'value', 'data', 'text', 'state'}   # fields that show a value

def pages_of(p):
    out = set()
    for part in str(p).split(','):
        part = part.strip()
        if '-' in part:
            a, b = part.split('-'); out.update(range(int(a), int(b) + 1))
        elif part:
            out.add(int(part))
    return out

def static_words(txt):
    while re.search(r'\{\{[^{}]*\}\}', txt):            # strip formulas from the inside out
        txt = re.sub(r'\{\{[^{}]*\}\}', ' ', txt)
    txt = re.sub(r'\[/?[a-z]+[^\]]*\]|:ph-[^:]+:|[#|*_>]|-{3,}', ' ', txt)
    return len(txt.split())

def design_check(dash, start_page=1, max_start_words=60):
    warn, per_page = [], collections.defaultdict(list)
    for wid, w in dash['widgets'].items():
        for p in pages_of(w['overrides'].get('page', 1)):
            per_page[p].append((wid, w))
    for p, ws in sorted(per_page.items()):
        shown, hero, tables, sections, words = collections.defaultdict(set), 0, 0, 0, 0
        sizes = set()
        for wid, w in ws:
            o, t, txt = w['overrides'], w['type'], str(w['overrides'].get('text', ''))
            hero += bool(o.get('mdAutoFitText')) or (o.get('textMdH1Fs') or 0) >= 40   # auto-fit or fixed big size
            for k, v in o.items():                                          # text sizes (rule 5)
                if re.search(r'(Fs|FontSize)$', k) and isinstance(v, (int, float)): sizes.add(v)
            sections += t == 'empty' and bool(o.get('label'))
            if t == 'text':
                rows = len(re.findall(r'^\s*\|.*\|\s*$', txt, re.M)) - 1      # static table rows (minus separator)
                fs = o.get('textMdTdFs') or 12
                if rows > 1 and o['height'] < rows * 2.3 * fs + 24:
                    warn.append(f'page {p} {wid}: {rows} table rows need ~{round(rows * 2.3 * fs + 24)} px, box is {o["height"]} px')
                tables += len(re.findall(r'^\s*\|?\s*:?-{3,}', txt, re.M))
                words += static_words(txt)
                for m in TOPIC.finditer(txt): shown[m.group(1)].add(wid)
            for field, topic in (w.get('bindings') or {}).items():
                if field in DISPLAY_BINDINGS and isinstance(topic, str): shown[topic].add(wid)
            for s in (w.get('bindings') or {}).get('series') or []:    # graph series since v2.3.0
                if isinstance(s, dict) and isinstance(s.get('data'), str): shown[s['data']].add(wid)
            if t == 'battery':
                warn.append(f'page {p} {wid}: battery widget shows no number')
            if t == 'graph' and o.get('type') != 'text' and not o.get('showAxisY'):
                warn.append(f'page {p} {wid}: graph without Y axis – no readable numbers')
            if t == 'graph' and o.get('showLegend', True) and o.get('legendPosition', 'right') != 'floating':
                warn.append(f'page {p} {wid}: legend "{o.get("legendPosition", "right")}" is drawn OUTSIDE the widget – reserve space or use floating')
            if t == 'text' and any(k in o for k in ('fontSize', 'textColor', 'lineHeight')):
                warn.append(f'page {p} {wid}: fontSize/textColor/lineHeight are ignored by the text widget – use textMdParaFs/…Col/…LineHeight')
            old = [k for k in ('lineColor', 'fill', 'fillOpacity', 'linearTimeScale', 'autoMin', 'autoMax', 'min', 'max',
                               'categoryLabelRotation', 'valueLabelRotation', 'timezoneConvert', 'pointValuesInterval') if k in o]
            if t == 'graph' and old:
                warn.append(f'page {p} {wid}: outdated graph keys {old} – see "Graph widget" in widgets.md')
        if hero > 4: warn.append(f'page {p}: {hero} big numbers (max 4)')
        if len(sizes) > 5: warn.append(f'page {p}: {len(sizes)} text sizes {sorted(sizes)} (max 5 on the whole dashboard)')
        if tables > 1: warn.append(f'page {p}: {tables} tables (max 1)')
        if sections > 4: warn.append(f'page {p}: {sections} labelled sections (max 4)')
        for topic, ids in shown.items():
            if len(ids) > 1: warn.append(f'page {p}: {topic} shown {len(ids)}x ({", ".join(sorted(ids))})')
        if p == start_page and words > max_start_words:
            warn.append(f'start page: {words} static words (aim <= {max_start_words})')
    return warn
```

`max_start_words` is a starting value; adjust it once the user has seen the first version. The table check only sees tables written into the widget text; a table delivered by a data feed (e.g. `{{homey/event/x_table}}`) must be sized by hand from the feed's row count. Like-for-like comparisons (rule 4) and estimate marks (rule 11) live in the data feed, not in the JSON: check them in the feed code.

**Runtime check (probe call):** the probe call in `browser-calls.md` reports `problems` (missing element, failed formula, legend outside the widget) and `hints` (overflow, smallest font, wrapped numbers). Run-tested on v2.3.1: a widget without a found element means the checks did not run – that is a failure, never a pass. The overflow hint is unreliable for small text boxes (11 false hints in the first test), and neither check saw the two real defects. So the screenshot before handover is mandatory, and the static check above, not the runtime check, is the first line of defence.
