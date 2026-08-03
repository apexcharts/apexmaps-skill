---
name: apexmaps
description: >
  AI skill for building ApexMaps geographic data visualizations. Use whenever
  the user asks to create, configure, render, update, or troubleshoot a
  choropleth map, bubble / proportional-symbol map, marker or point map, flow
  map (arcs, great-circle routes, origin-destination), route/line map, or any
  map built with `apexmaps`. Covers the five series types (choropleth, bubble,
  marker, arc, line), the built-in geometry registry (world countries, US
  states and counties, EU NUTS 0-3, admin-1 for 15 countries), data joins via
  `joinBy`, projections, classed and continuous scales, palettes, drilldown,
  marker clustering, the camera API, annotations, theming, and framework
  integration (React / Vue / Angular). In React / Vue / Angular projects,
  prefer the framework wrapper packages (`react-apexmaps`, `vue-apexmaps`,
  `ngx-apexmaps`) over the core API.
metadata:
  author: ApexCharts
  version: "1.0.0"
  library_version: "0.3.0"
  category: data-visualization
  tags: [maps, choropleth, geojson, topojson, geographic, charts, apexmaps]
  docs: https://apexcharts.com/docs/apexmaps/
  npm: apexmaps
  github: https://github.com/apexcharts/apexmaps
---

# ApexMaps AI Skill

ApexMaps is a standalone geographic visualization library in the ApexCharts
ecosystem (it does NOT require the `apexcharts` package or a global). It draws
choropleths, proportional-symbol bubbles, markers, great-circle arcs, and
routes over a built-in geometry registry, so there is no GeoJSON to find, host,
or parse for the common maps.

> **Status: pre-alpha (0.3.0).** The engine, all five series types,
> projections, joins, scales, fills, legend, tooltip, labels, annotations,
> camera, geometry registry, drilldown, clustering, selection, and the
> accessibility layer work and are tested. The story engine and map tiles are
> not built yet. Expect option-level changes between minor versions.

## Framework wrapper detection: check `package.json` first

Before generating code, check what the project uses:

- `react-apexmaps` or React project: use the `<ApexMaps />` component (props
  are compared deeply; a series-only change tweens instead of rebuilding)
- `vue-apexmaps` or Vue 3 project: use the `<ApexMaps />` component (in-place
  mutation of reactive options is detected)
- `ngx-apexmaps` or Angular project: use `<apx-map />` (standalone component,
  signal inputs, zoneless-ready)
- Otherwise: core API, `new ApexMaps(element, options)`

See `references/framework-wrappers.md`.

## 1. Critical Rules

1. **Constructor + async render**: `const map = new ApexMaps(el, options)` then
   `await map.render()`. `render()` is async because geometry packs load
   lazily. An instance is not reusable after `destroy()`.
2. **No CSS import needed**: styles are inlined into the bundle and injected
   automatically. Importing `apexmaps/apexmaps.css` explicitly (as the wrapper
   READMEs show) is harmless and useful for CSP-strict setups.
3. **Geometry comes from the registry**: `geo: { map: 'world/countries' }`.
   Use pack ids or their aliases (`'us'`, `'jp/prefectures'`, `'eu/nuts2'`)
   before reaching for a GeoJSON URL. Custom geometry: pass GeoJSON/TopoJSON
   inline or via `ApexMaps.registerMap(id, source)`.
4. **Coordinates are `[lon, lat]`**, longitude first, matching GeoJSON. A map
   whose points all sit on a vertical line through Africa has them swapped.
5. **Choropleth is the default series type.** A series without `type` is a
   choropleth (`chart.type` can seed a different default).
6. **Join data with `joinBy`**: `'name'`, `['geoField', 'dataField']`, or
   `{ geo, data }`. When omitted, the join key is auto-detected. Each registry
   pack carries a recommended key (`us/states` joins on postal abbreviations
   like `'CA'`, `us/counties` on FIPS, admin-1 packs on ISO 3166-2 codes,
   `world/countries` on `iso_a3`).
7. **Use `null`, never `undefined`, for missing values.** No-data features get
   `scale.nullColor` and stay out of the scale.
8. **Values must be numbers** (or `null`). The default value field is `value`;
   override with `valueField` or per-series accessors.
9. **Licensed features work without a key, with a watermark**: `arc` and
   `line` series, pattern/image fills, marker clustering, drilldown, linked
   selection (`link`), annotations, story context, and self-registered
   projections. Everything else renders clean without a key.
   `ApexMaps.setLicense('APEX-...')` before rendering removes the watermark;
   one Apex key covers every product, but each library needs its own
   `setLicense` call.
10. **Defaults are publishable.** Do not pile on options: no projection,
    palette, classification, legend, or tooltip configuration is required.
    Add options only when the user asks for the behavior they change.

## 2. Series Data Formats

`series` is an array of a **discriminated union**: the `type` decides which
fields exist. Mixing series types on one map is normal (e.g. choropleth +
bubble).

| Type | Datum shape | Position source |
|---|---|---|
| `choropleth` (default) | `{ <joinKey>: 'FRA', value: 7.3, ... }` | joined to geometry by key |
| `bubble` | `{ lon, lat, value, name? }` (`lng` accepted for `lon`) | coordinates, or `joinBy` to feature centroids |
| `marker` | `{ lon, lat, name?, value?, category?, shape?, color?, size? }` | coordinates, or `joinBy` to centroids |
| `arc` | `{ from, to, value?, name? }`, `from`/`to` REQUIRED, each `[lon, lat]` or a geometry key like `'JFK'`/`'FRA'` | endpoints |
| `line` | `{ path: [[lon,lat], ...], name? }` (`coordinates` accepted for `path`) | the vertex sequence itself |

```js
const map = new ApexMaps(document.querySelector('#map'), {
  geo: { map: 'world/countries@110m' },
  series: [
    {
      name: 'Unemployment rate',
      joinBy: ['iso_a3', 'code'],
      data: [
        { code: 'FRA', value: 7.3 },
        { code: 'DEU', value: 5.7 },
        { code: 'ESP', value: null }, // no data: gets nullColor, not 0
      ],
    },
  ],
})
await map.render()
```

Key per-type notes:

- **choropleth**: `scale` (classification + palette), `normalizeBy` (divide
  value by another field: counts across unequal areas mostly redraw the
  population map), `fuzzyJoin` (opt-in alias/normalized matching), `fill`
  (pattern/image, licensed), `drilldown` (licensed).
- **bubble**: `size: { scale: 'sqrt' }` is the default on purpose (area, not
  radius, should encode the value); `colorScale` + `colorField` add a second
  encoding; `sortBySize: true` (default) keeps small bubbles clickable.
- **marker**: seven shapes (`circle`, `square`, `diamond`, `triangle`,
  `star`, `cross`, `pin`; `pin` anchors at its point), `colorBy` a category
  field for categorical color + legend, `cluster: { ... }` (licensed) merges
  piled-up points; clustering is an option, never a separate series type.
- **arc**: `geodesic: true` (default) follows the great circle and handles the
  antimeridian; `curvature` is decorative and bypasses geodesic accuracy;
  `flow: true` (or options) sends beads along routes to show direction;
  `width`/`colorScale` scale by `value`; string endpoints resolve against
  geometry keys (`joinBy` controls the field).
- **line**: caller supplies the whole path (GPS trace, shipping lane); same
  `width`, `colorScale`, `endpoints`, `flow` options as arc.

Common to every series: `name`, `visible`, `opacity`, `stroke`, `labels`,
`animation` (`grow` / `draw` / `fade`), `valueField`.

Full details: `references/data-format.md`.

## 3. Top-Level Options (`ApexMapsOptions`)

| Key | Purpose |
|---|---|
| `chart` | width, height, default series `type`, background, fontFamily, `context: 'story' \| 'dashboard'`, `animations`, `events` |
| `geo` | `map` (registry id / URL / GeoJSON / TopoJSON), `object`, `keyField`, `nameField`, `projection`, `view: { fit, padding }`, `graticule`, `sphere`, `fill`, `repairWinding` |
| `series` | array of the union above |
| `legend` | `position` (`bottom` default), `style: 'auto' \| 'classes' \| 'gradient'`, `interactive` (click to mute a class), `marker` (hover arrow on gradient bars), formatters |
| `tooltip` | `formatter(context)` returning HTML, `valueFormatter`, `followCursor` |
| `dataLabels` | collision-avoiding labels: `field`, `formatter`, `minFeatureArea`, halo style |
| `states` | `hover`, `active`, `muted` appearance |
| `theme` | `{ mode: 'light' \| 'dark' \| 'auto', palette }` |
| `interaction` | `zoom` (wheel, doubleClick, `controls`), `pan` (+ inertia), `rotate` (globe drag on orthographic, `'auto'`), `selection` (box select with `modifier: 'shift'`), `nearest` (proximity hit assist) |
| `a11y` | on by default: generated description, keyboard navigation, optional `dataTable` |
| `annotations` | editorial layer: `points` (at `[lon, lat]`), `features` (by key, optional `outline`), `areas` (bounds or geometry). Licensed. |
| `link` | `{ group }` cross-map linked selection. Licensed. |
| `debug` | `joinDiagnostics` |
| `responsive` | `[{ breakpoint, options }]` |

## 4. Lifecycle

```js
const map = new ApexMaps(el, options) // does not draw yet
await map.render()                    // fetches packs, draws, resolves when ready

map.updateSeries(newSeries)           // data update: tweens fills/radii
await map.updateOptions(partial)      // deep-merged; reprojects when needed

map.on('featureClick', ({ key, name, value, datum }) => { ... })
map.off('featureClick', handler)

map.destroy()                         // removes DOM, listeners, fetch handles; not reusable
```

- `updateSeries` for data-only changes (it animates value transitions).
- `updateOptions` for anything else; pass `{ redrawGeometry: true }` to force
  reprojection.
- Handlers can also be declared in options: `chart.events.featureClick`.

## 5. Public API

Instance methods and getters:

| Member | Purpose |
|---|---|
| `render()` | async initial draw |
| `updateSeries(series)`, `updateOptions(options, flags?)` | updates |
| `on(event, fn)`, `off(event, fn?)` | events (see below) |
| `camera.flyTo(target)`, `camera.easeTo(target)`, `camera.jumpTo(target)`, `camera.fitBounds(bounds)` | camera moves (flyTo is the Van Wijk zoom-out-and-in path; on azimuthal projections a move turns the globe instead of panning) |
| `frameFeature(key, opts?)`, `resetView(opts?)` | frame a feature / return to the opening view |
| `zoomIn()`, `zoomOut()`, `zoom`, `rotateTo(angles)`, `rotation` | zoom and globe rotation |
| `drillTo(key)`, `drillUp(levels?)`, `drillDepth` | drilldown navigation |
| `toggleSelection(key)`, `setSelection(keys)`, `clearSelection()` | selection |
| `diagnoseJoin(seriesIndex?)` | the join mismatch report, on demand |
| `toSpec()` | current options as a serializable object |
| `getSvgString()`, `exportSVG()`, `exportPNG()`, `dataURI()` | export |
| `destroy()` | teardown |

Statics: `ApexMaps.setLicense(key)`, `registerMap(id, source, meta?)`,
`registerProjection(name, factory)`, `registerPalette(name, palette)`,
`setGeoSource(urlOrFetcher)` (self-host geometry), `listMaps()`,
`catalogue()`, `mapMeta(id)`, `listProjections()`, `listPalettes()`,
`getInstance(id)`, `version`.

Events (`ApexMapsEventMap`): `rendered`, `updated`, `resized`,
`featureClick`, `featureHover`, `featureFocus`, `markClick`, `markHover`,
`clusterClick`, `drilldown`, `drillup`, `selectionChange`, `legendToggle`,
`zoom`, `panEnd`, `rotate`, `rotateEnd`.

## 6. Geometry Registry

26 built-in packs, fetched lazily (one request per pack). Canonical id is
`region/level@detail`; the detail-free form resolves to the lightest pack.

| Id (alias) | Join key (`keyField`) | Features |
|---|---|---|
| `world/countries@110m` (`world/countries`, `world`) | `iso_a3` | 177 |
| `world/countries@50m` | `iso_a3` | 242 |
| `world/land@110m`, `world/land@50m` | none (backdrop) | coastline |
| `us/states@10m` (`us/states`, `us`) | `abbr` (postal, `'CA'`) | 56 |
| `us/counties@10m` (`us/counties`) | `fips` (5-digit string) | 3,231 |
| `us/nation@10m` | none (backdrop) | 1 |
| `eu/nuts0@20m` (`eu`) ... `eu/nuts3@20m` | `nuts_id` | 37 / 125 / 334 / 1,514 |
| `cn`, `in`, `jp`, `de`, `gb`, `fr`, `it`, `ca`, `br`, `ru`, `mx`, `au`, `kr`, `es`, `id` admin-1 (`jp/prefectures`, `fr/departments`, ...) | `iso_3166_2` (`'JP-13'`) | varies |

Packs ship repaired identifiers (Natural Earth publishes `iso_a3` as `-99`
for France and Norway; the packs fix that), a recommended join key, per-pack
default projections (`albersUsa` for US packs, ETRS89-LAEA for NUTS, custom
conics for Canada and Russia), and provenance/attribution metadata.

Full pack table, custom geometry, projections, and the camera:
`references/geo-and-projections.md`.

## 7. Pitfalls: Wrong vs Correct

1. **Swapped coordinates**
   - Wrong: `{ lat: 48.85, lon: 2.35 }` written as `[48.85, 2.35]`
   - Correct: arrays are `[lon, lat]`: `[2.35, 48.85]`. Object datums may use
     named `lon`/`lat` (or `lng`) fields instead.
2. **Hosting your own world GeoJSON**
   - Wrong: `geo: { map: 'https://cdn.example.com/world.json' }` for a
     standard map
   - Correct: `geo: { map: 'world/countries' }`. URLs are for geography the
     registry does not have.
3. **Joining US states on full names when the pack key is postal**
   - Wrong: `data: [{ state: 'California', value: 1 }]` with no `joinBy`
     against `us`
   - Correct: join on `'CA'` (the pack's `abbr` key), or set
     `joinBy: ['name', 'state']`, or enable `fuzzyJoin: true` and check the
     reported substitutions.
4. **`undefined` for missing data**
   - Wrong: `{ code: 'ESP', value: undefined }`
   - Correct: `{ code: 'ESP', value: null }` (renders as no-data, not as 0).
5. **Counts instead of rates on a choropleth**
   - Wrong: coloring by raw `population`-correlated counts
   - Correct: `normalizeBy: 'population'` (the legend retitles itself), or
     precompute rates.
6. **Clustering as a separate series**
   - Wrong: `{ type: 'cluster', data }`
   - Correct: `{ type: 'marker', data, cluster: { radius: 60 } }`.
7. **`curvature` with `geodesic` accuracy expectations**
   - `curvature` bulges the arc for looks and is no longer the flown path;
     leave `geodesic: true` (default) for real routes.
8. **Selection modifier conflicts**
   - `selection.modifier: 'none'` makes every drag a selection box and
     requires `pan.enabled: false`; one gesture cannot mean both.
9. **Expecting `render()` to be synchronous**
   - Wrong: `map.render(); map.camera.flyTo(...)`
   - Correct: `await map.render()` first; packs load over the network.
10. **Re-instantiating to change data**
    - Wrong: `destroy()` + `new ApexMaps(...)` per update
    - Correct: `map.updateSeries(next)`; it tweens and keeps the camera.

## 8. Theming

- `theme: { mode: 'auto' }` follows `prefers-color-scheme`; `'light'` /
  `'dark'` force it. `theme.palette` names the default scale palette.
- CSS custom properties (`--apexmaps-bg`, `--apexmaps-fg`,
  `--apexmaps-surface`, `--apexmaps-focus`, `--apexmaps-font-size`,
  `--apexmaps-legend-width`, ...) restyle chrome without options.
- 17 built-in palettes (`blues`, `viridis`, `rdbu`, `okabeIto`, `apex`, ...);
  diverging palettes are picked automatically when the domain crosses zero.
- `prefers-reduced-motion` disables animations, flow beads keep their dots in
  place, and drilldown swaps without motion. Nothing to configure.

Details: `references/styling-and-interaction.md`.

## 9. Reference Routing Table

| Topic | Reference File |
|---|---|
| Datum shapes per series type, joins (`joinBy`, `fuzzyJoin`, diagnostics), `normalizeBy`, FIPS repair | `references/data-format.md` |
| Registry packs and aliases, custom GeoJSON/TopoJSON, `registerMap`, `setGeoSource`, projections and spec objects, camera API | `references/geo-and-projections.md` |
| Scales, palettes, size scales, legends, tooltips, data labels, pattern/image fills, states, selection, linked maps, drilldown, zoom controls, a11y, theming, responsive | `references/styling-and-interaction.md` |
| React / Vue / Angular wrappers | `references/framework-wrappers.md` |
