# Styling and Interaction: Scales, Legends, Selection, Drilldown, Theming

## Scales (choropleth and hexbin `scale`, bubble/arc/line `colorScale`)

`ScaleOptions` fields:

| Option | Default | Notes |
|---|---|---|
| `type` | `'quantile'` (choropleth default, 5 classes) | See the 10 types below. |
| `classes` | `5` | Class count for classed scales. |
| `palette` | auto (see below) | Registered name, or an explicit color array (taken literally, never resampled). |
| `domain` | data extent | `[min, max]` override. |
| `breaks` | none | Explicit boundaries. REQUIRED for `type: 'threshold'` (missing breaks warns and falls back to quantile). |
| `reverse` | `false` | Flip the ramp. |
| `nice` | `true` for continuous scales | Round the domain outward. |
| `nullColor` | `'#eeeeee'` | Fill for no-data features; unmatched joins get it too. |
| `nullLabel` | `'No data'` | Legend text for the null swatch. |

The 10 `ScaleType` values:

| Type | Kind | Behavior |
|---|---|---|
| `quantile` | classed | Equal counts per class. The default: every class is populated. |
| `quantize`, `equalInterval` | classed | Equal value ranges (the two are computed identically). Legend arithmetic is readable; skewed data leaves classes empty. |
| `jenks`, `naturalBreaks` | classed | Minimize within-class variance (same algorithm). Sampled above 2,000 values, with a warning. |
| `threshold` | classed | Breaks you chose: statutory limits, targets, zero crossings. |
| `linear`, `log`, `sqrt` | continuous | No classes; draws a gradient legend. |
| `ordinal` | categorical | Distinct categories, one color each. |

```js
scale: { type: 'quantile', classes: 5 }
scale: { type: 'threshold', breaks: [5, 25, 100, 400], palette: 'reds' }
scale: { type: 'log' }   // continuous, gradient legend
```

**Automatic diverging selection**: when no `palette` is named and the domain crosses zero, the scale picks `'rdbu'` (diverging, anchored at zero); otherwise `'blues'`. A sequential ramp on signed data hides the sign, which is the single most common palette error.

## Palettes

17 built-in palettes (`ApexMaps.listPalettes()` enumerates them):

| Kind | Names |
|---|---|
| Sequential | `blues`, `greens`, `oranges`, `reds`, `purples`, `greys`, `viridis`, `magma`, `teal` |
| Diverging | `rdbu`, `brbg`, `piyg`, `spectral`, `rdylgn` |
| Categorical | `apex`, `tableau`, `okabeIto` |

- Class colors are the palette's anchor stops resampled in OkLab at render time, so 4-class and 9-class maps of the same data stay perceptually consistent and midpoints do not go muddy grey.
- `okabeIto` is flagged colorblind-safe and is the free tier's answer to color vision, alongside the automatic diverging selection.
- `theme: { palette: 'viridis' }` sets the default palette for series that omit one; `ApexMaps.registerPalette(name, palette)` adds your own, usable anywhere a `PaletteName` is.
- For categories, pass a qualitative set as an explicit array: a classed scale samples a named palette as a ramp, and a blend of two categories is a color that means nothing.

## Size scales (bubble radius, arc and line width)

`SizeOptions` on `bubble.size`, `arc.width`, `line.width`:

| Option | Default | Notes |
|---|---|---|
| `field` | the series' `valueField` | Field name or accessor for the magnitude. |
| `scale` | `'sqrt'` | Readers judge circles by area; sqrt makes area proportional to value. `'linear'` maps value to radius directly (warns in dev mode for symbol radii); `'log'` needs a positive domain. |
| `range` | `[3, 28]` | `[minRadius, maxRadius]` in screen pixels. |
| `domain` | `[min(0, dataMin), dataMax]` | Anchored at zero on purpose: a scale starting at the smallest observed value implies that place has zero magnitude. |

Bubble series get a **nested-circle legend**: three concentric reference circles at round values, sharing a bottom edge so diameters line up and areas can actually be decoded.

## Legend

| Option | Default | Notes |
|---|---|---|
| `show` | `true` | |
| `position` | `'bottom'` | `'top' \| 'left' \| 'right'`. Left/right make the legend a fixed-width column and turn a gradient bar vertical. |
| `width` | `180` | Column width in px, for `left`/`right` only. |
| `align` | `'center'` | `'start' \| 'center' \| 'end'`. |
| `title` | series name | Retitles itself under `normalizeBy`. |
| `interactive` | `true` | Click a class to mute it (fires `legendToggle`). |
| `showNull` | `true` | Include the no-data swatch. |
| `style` | `'auto'` | Continuous scales get `'gradient'`, classed get `'classes'`; set explicitly to force one. |
| `formatter` | none | `(item, index) => string` per `LegendItem` (`{ color, label, from, to, count, isNull, pattern }`). |
| `tickFormatter` | none | `(value) => string` for the numbers under a gradient bar (class boundaries, or the two ends). |
| `marker` | on | Gradient legends only: an arrow that rides the bar and tracks the hovered feature, so the reader sees where it falls on the scale instead of matching colors by eye. `{ label: false }` keeps the arrow, drops the value above it. `false` removes it. |

## Tooltip

| Option | Default | Notes |
|---|---|---|
| `enabled` | `true` | |
| `followCursor` | `true` | |
| `formatter` | built-in | `(context) => string` returning HTML. Responsible for its own escaping. Return `''` to show no tooltip for that mark: the formatter also runs for background geometry, which has no row behind it. |
| `valueFormatter` | none | `(value) => string`, keeps the default layout. |
| `offset` | `[12, 12]` | Pixels from the cursor. |

`TooltipContext`: `{ key, name, value, datum, properties, series }`.

```js
tooltip: { formatter: ({ name, value }) => (value == null ? '' : `<strong>${name}</strong> ${value.toLocaleString()}`) }
```

## Data labels

Off by default. `dataLabels` options: `enabled` (`false`), `field` (defaults to the feature name; string or accessor), `formatter({ value, name, key })`, `collision` (`'hide'` default: drops labels that would overlap; `'none'`), `minFeatureArea` (`240` square px: smaller features get no label), `style` (`{ fontSize: 11, fontWeight: 500, halo: true, haloColor, haloWidth }`). The halo color follows the theme via `--apexmaps-halo`, so it flips in dark mode.

**Annotations always win over generated labels**: a label that would collide with an annotation is dropped, never the annotation, because you placed the annotation deliberately and the label came from a rule.

## States, selection, linked maps

`states` defaults: `hover: { enabled: true, brightness: 0.08 }` (optional `stroke`/`strokeWidth`), `active: { enabled: true, stroke: '#111111', strokeWidth: 1.5 }`, `muted: { opacity: 0.25 }`.

```js
interaction: {
  selection: { enabled: true, multiple: true, rectangle: true, modifier: 'shift' }, // all defaults
},
states: { muted: { opacity: 0.25 } },   // 1 turns dimming off
```

- Shift-drag draws a selection box, Alt adds to the existing selection, Escape abandons the box, a box over nothing clears the selection. `modifier: 'none'` makes every drag a box and therefore requires `pan.enabled: false`: one gesture cannot mean both.
- A box tests each feature's label **anchor**, not its bounding box (Alaska's bbox spans the Pacific). Point marks are tested at their position; the automatic basemap is excluded.
- While anything is selected, everything else dims to `states.muted.opacity`.
- API: `toggleSelection(key)`, `setSelection(keys)`, `clearSelection()`; event `selectionChange: { ids, source }`. Selection does not survive a drilldown level change.

**Linked maps** (licensed): every map declaring the same `link: { group }` shares its selection; brushing one dims non-selected features on all. `filter` controls direction: `'bidirectional'` (default), `'emit'` sends without receiving, `'receive'` follows without leading. A receiver never rebroadcasts, so a bidirectional pair cannot ring. Keys must mean the same thing across the group.

```js
link: { group: 'us-dashboard', filter: 'bidirectional' }
```

## Pattern and image fills (licensed)

`fill` on a choropleth series adds a texture channel over the scale's flat color. Both `fill.pattern` and `fill.image` are licensed (work without a key, with a watermark). `image` wins where both are set. **No-data features are never textured**: an absence has to keep reading as an absence. Eight tile types: `dots`, `squares`, `checks` (filled shapes), `lines`, `grid`, `diagonal`, `crosshatch` (stroked lines), and `custom` with your own `path` drawn in a `size` by `size` box.

| Pattern option | Default | Notes |
|---|---|---|
| `type` | `'dots'` | |
| `size` | `10` | **Spacing between marks, in screen pixels**, not the size of one. The tile is rescaled as the reader zooms so texture holds its size (same reasoning as `non-scaling-stroke`). Marks are small against it (a dot covers about a twelfth of its tile); tightening it makes ink average with the fill into a shade on no scale. |
| `color` | automatic | Ink: white on a dark-enough background, a darkened tint on a pale one, so a whole ramp stays legible unconfigured. |
| `background` | scale color | The pattern is added to the encoding, not a replacement. |
| `strokeWidth` | `size / 5` | Line tiles only. |
| `angle` / `opacity` | `0` / `1` | |

```js
fill: { pattern: { type: 'dots', size: 10 } }
// per feature, for the qualitative case:
fill: { pattern: ({ classIndex }) => ({ type: TILES[classIndex], size: 7 }) }
```

The function form receives `FillContext`: `{ key, name, value, datum, properties, color, classIndex }` (`color` is the resolved scale color; `classIndex` is `-1` on continuous scales). **Legend swatches draw the real tile** off the same builder as the map.

Image fills clip an image to each feature's outline, fitted to its bounding box:

```js
fill: {
  image: {
    src: ({ key }) => `/flags/${key.toLowerCase()}.svg`,  // per feature; null declines
    fit: 'cover',        // default; 'contain' fits it all in; 'fill' stretches
    background: '#eef',  // shows under a 'contain' fit and while loading; defaults to the flat color
  },
}
```

Unlike a texture, an image scales with the region. Cross-origin images taint the canvas and block PNG export (SVG export is unaffected); `fit: 'fill'` with an SVG source only stretches if the file has `preserveAspectRatio="none"`. Image fills are one `<pattern>` def per feature, so they are the wrong tool for thousands of features.

## Drilldown (licensed)

```js
series: [{
  name: 'Adoption', joinBy: { data: 'key' }, data: rows,
  drilldown: { map: 'us/counties', breadcrumb: { rootLabel: 'United States' } },
}]
```

| Option | Default | Notes |
|---|---|---|
| `map` | required | Registry id, URL, or geometry; or a function of `DrilldownContext` (`{ key, name, datum, properties, depth, from }`) so levels can differ. Return `null` to refuse (for maps with children for only some features). |
| `scope` | `'auto'` | `'auto'` finds a child property holding the parent's key (TIGER `state_abbr`, Eurostat `cntr_code`, Natural Earth `adm0_a3`), scoring candidates by how many children match; falls back to a key prefix (FIPS `06037` under `06`, NUTS `DE12` under `DE1`). `'property'` / `'keyPrefix'` force one route; `'all'` draws the whole child map. When neither matches, the drilldown is declined with a dev-mode reason rather than landing on an empty map. |
| `parentField` | detected | Names the child property yourself, skipping detection. |
| `animate` | `'zoom'` | Frames the clicked feature, hands the child that exact screen box, dissolves the old level, and develops the child out of the parent's color from the middle outwards. `'none'` swaps without motion (also what reduced motion gets). |
| `breadcrumb` | `true` | Trail above the map with a way back up; `{ rootLabel }` names the top level. |

Data across levels: keep one array with rows for every level (the join takes whatever matches the level on screen), or fetch per level from the `drilldown` event, which fires after the child renders:

```js
map.on('drilldown', ({ key, name, depth, featureCount }) => {
  fetchCounties(key).then((rows) => map.updateSeries([{ ...series, data: rows }]))
})
map.on('drillup', ({ to, depth }) => {})
map.drillTo('CA')       // programmatic, as a click would
map.drillUp()           // one level; drillUp(Infinity) to the top; never refetches
map.drillDepth          // where you are now
```

Ways back up: the breadcrumb (real, keyboard-reachable buttons), Escape, or `drillUp()`. Enter on a keyboard-focused feature drills exactly where a click would.

## Interaction: zoom, pan, rotate, nearest

| Option | Default | Notes |
|---|---|---|
| `zoom.enabled` | `true` | A pack that declares itself `fixed`, which every hex tile layout does, defaults `zoom.enabled` and `pan.enabled` to `false` instead: a diagram has nothing to zoom into and nothing off-screen to pan to, so the wheel and the drag go back to the page. Asking for them explicitly still wins. |
| `zoom.min` / `max` / `wheel` / `doubleClick` | `0.8` / `4096` / `true` / `true` | |
| `zoom.step` | `1.6` | Scale factor per step: buttons, keyboard, double-click. |
| `zoom.controls` | `{ show: true, position: 'top-right', reset: true }` | `false` removes them. |
| `pan` | `{ enabled: true, inertia: true }` | |
| `rotate.enabled` | `'auto'` | Globe projections (`orthographic`) spin on drag, flat maps pan. `true` forces rotation on any projection that can rotate and invert; `false` gives the drag back to panning. `rotate.inertia` defaults to `pan.inertia`. |
| `nearest` | `{ enabled: true, radius: 20 }` | Proximity hit assist for point marks: a pointer within `radius` px hovers and clicks the nearest mark as if on it (Voronoi-style nearest wins), so a 3px bubble does not demand a 3px hit. Direct hits on other point or path marks take precedence; area features yield. |

**Zoom controls** are on by default wherever zoom is enabled because every other way to change scale is a gesture, and a gesture is not a keyboard path. `position` picks a corner (`'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'`). `reset: true` (default) includes a control returning the opening view: at the default step, eight clicks of `+` pass 40x and no gesture goes back in one move. The group idles at 70% opacity (override with `--apexmaps-zoom-idle-opacity`; `1` disables the fade) and comes to full strength on hover or keyboard focus.

## Accessibility (`a11y`)

Never license-gated, in any tier. Defaults: `enabled: true`, `description: 'auto'` (generated from the spec and data: map type, area count, range, class structure, extremes, no-data count), `dataTable: false` (a real, visually hidden table alongside the map: the reliable fallback for AT that cannot interpret spatial output), `keyboardFeatureLimit: 500`. One tab stop for the whole map; arrow keys walk features in reading order (banded top-to-bottom, then left-to-right), Home/End jump to the ends, Enter selects (and drills where a drilldown exists), Escape leaves. Above `keyboardFeatureLimit` features, walking switches off and the description plus data table remain the accessible path. Announcements replace a polite live region rather than appending, so fast travel does not flood the reader.

## Theming

- `theme: { mode: 'light' | 'dark' | 'auto' }`. Default `'light'`. `'auto'` follows `prefers-color-scheme`; dark is a class (`apexmaps--dark`) on the container, so a dashboard can force it independently of the OS. Dark mode paints its own background; hand it back with `--apexmaps-bg: transparent`. The data palette is unchanged: what changes is chrome, no-data color, and text.
- `theme.palette` names the default scale palette for series that omit one.
- **Family tokens (0.4.0)**: the chrome roles that mean the same thing across the ApexCharts products fall back to the shared `--apx-*` family tokens, so a page can state its brand once on `:root` and the map follows along with the charts beside it. `--apexmaps-fg` falls back to `--apx-fore`, `--apexmaps-surface` to `--apx-surface`, `--apexmaps-border` to `--apx-grid`, `--apexmaps-focus` to `--apx-accent`. Precedence is `--apexmaps-*` set by the host, then the `--apx-*` token, then the built-in default. Two deliberate exceptions: `--apexmaps-bg` stays transparent in light mode so a tinted host card shows through, and the dark palette takes no family tokens at all (one set of `--apx-*` values describes one appearance, and a light brand surface applied to the dark palette is how you get white text on white). Override `--apexmaps-*` to theme dark mode.
- CSS custom properties on the container restyle all chrome with no options: `--apexmaps-bg`, `--apexmaps-fg`, `--apexmaps-fg-muted`, `--apexmaps-surface`, `--apexmaps-border`, `--apexmaps-focus`, `--apexmaps-halo` (label halo), `--apexmaps-font-size` (`12px`), `--apexmaps-radius` (`6px`), `--apexmaps-shadow`, `--apexmaps-legend-bar` (vertical gradient bar height, `140px`), `--apexmaps-legend-width` (`180px`), `--apexmaps-muted-opacity` (`0.25`), `--apexmaps-anim` / `--apexmaps-anim-geom` (transition durations, written by the engine from `chart.animations`), `--apexmaps-zoom-idle-opacity` (`0.7`), and `--apexmaps-flow-duration` / `--apexmaps-flow-delay` / `--apexmaps-flow-travel` (flow bead animation, written by the renderer).
- `chart.context`: `'dashboard'` (default) or `'story'`. Story animates entrances (turns `animations.entrance` on) and is licensed; dashboard does not, because a dashboard reader wants the number now.
- `chart.animations`: `{ enabled: true, speed: 'normal', entrance: false }`. `speed` is `'slow'` (700ms), `'normal'` (350ms), `'fast'` (180ms), `'instant'` (0), or a number in ms. Data updates (`updateSeries`, palette changes, legend toggles) tween fills, radii and stroke widths; camera-driven geometry never animates. Past a few thousand marks the engine degrades on its own: geometry transitions stop first, then everything. `prefers-reduced-motion` disables all of it; flow beads stay in place and stop traveling; drilldown becomes a plain swap. Nothing to configure. The hex layout morph runs at twice the configured speed (it has to be followed, not just noticed) and is the one transition with no cheap version to fall back to, so past the full motion budget it swaps rather than degrading.

## Responsive

```js
responsive: [
  { breakpoint: 560, options: { legend: { position: 'bottom' }, dataLabels: { enabled: false } } },
  { breakpoint: 900, options: { legend: { position: 'right' } } },
]
```

Rules match on the measured container width, the narrowest matching breakpoint wins, so declaration order does not matter. The map keeps the reader's position across a resize rather than refitting.

## Licensing summary

A map that answers a question is free; a map that becomes an application is licensed:

| Free, always | Licensed |
|---|---|
| `choropleth`, `bubble`, `marker` series, automatic basemap | Point clustering (`cluster`) |
| All 16 built-in projections, with spec objects | Self-registered projections (`registerProjection`) |
| Geometry registry, all 26 boundary packs | Drilldown and the breadcrumb |
| Tooltips, legends, labels, data labels, states, themes | Editorial annotations (`annotations`) |
| Zoom, pan, pinch, hover, click, box selection, camera API | `arc` and `line` route series |
| Joins, `fuzzyJoin`, join diagnostics | Linked selection (`link: { group }`) |
| Scales, palettes, size legends, responsive rules | Story mode (`chart: { context: 'story' }`) |
| Flat fills, PNG and SVG export, the accessibility layer | Pattern fills and image fills (`fill`) |
| Registering geometry, layouts, projections and palettes | `hexbin` series: binned point density |
| `chart.context: 'dashboard'`, the default | Hex tile layouts, however you reach them: `geo.layout: 'hex'`, an `@hex` pack id, or your own `registerLayout` table |

The line the two columns follow: a map answering a question is free, and a map becoming an application is licensed. Summarising points into an aggregate the reader cannot get back to the originals from is on the licensed side whether it is done by distance (`cluster`) or by lattice (`hexbin`), and so is drawing the geography as something other than itself (grid layouts). Registering something costs nothing in either case; rendering it is what is gated, which is why a layout table you authored yourself is gated the same as a built-in one. The layout morph has no gate of its own, because it only ever runs on a layout toggle.

Licensed features work without a key, in full, **with a watermark on the map**, so they can be evaluated in your own app. No metering, no seat counting, no network calls.

```js
ApexMaps.setLicense('APEX-xxxxxxxx') // before rendering; applies to every map on the page
```

One key covers every Apex product, but each library needs its own call: `ApexMaps.setLicense()` and `ApexCharts.setLicense()` set different copies of the license manager, because each bundles its own. Accessibility is never gated, and the free tier keeps `okabeIto` plus the automatic diverging selection.
