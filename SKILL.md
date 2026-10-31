---
name: apexmaps
description: >
  AI skill for building ApexMaps geographic data visualizations. Use whenever
  the user asks to create, configure, render, update, or troubleshoot a
  choropleth map, pixel / dot-lattice map, bubble / proportional-symbol map,
  marker or point map, flow map (arcs, great-circle routes,
  origin-destination), route/line map, hexbin density map, hex tile map
  (honeycomb, tilegram, cartogram), or a map over raster street tiles built
  with `apexmaps`. Covers the seven series types (choropleth, pixel, bubble,
  marker, hexbin, arc, line), the built-in geometry registry (world countries,
  US states and counties, EU NUTS 0-3, admin-1 for 15 countries) and its seven
  hex tile layouts, data joins via `joinBy`, projections, the raster tile
  underlay (`geo.tiles`), classed and continuous scales, palettes, drilldown,
  marker clustering, the camera API and `viewChange`, annotations, theming,
  `locale` for the map's built-in text, and framework integration (React /
  Vue / Angular). In React / Vue / Angular projects, prefer the framework
  wrapper packages (`react-apexmaps`, `vue-apexmaps`, `ngx-apexmaps`) over the
  core API.
metadata:
  author: ApexCharts
  version: "1.3.1"
  library_version: "1.2.1"
  category: data-visualization
  tags: [maps, choropleth, pixel-map, hexbin, tilegram, geojson, topojson, geographic, charts, apexmaps]
  docs: https://apexcharts.com/docs/apexmaps/
  npm: apexmaps
  github: https://github.com/apexcharts/apexmaps
---

# ApexMaps AI Skill

ApexMaps is a standalone geographic visualization library in the ApexCharts
ecosystem (it does NOT require the `apexcharts` package or a global). It draws
choropleths (as flat fills or as a lattice of dots), proportional-symbol
bubbles, markers, binned point density, great-circle arcs, and routes over a
built-in geometry registry, so there is no GeoJSON to find, host, or parse for
the common maps. It can also draw a region set as a hex tile map instead of its
real boundaries, or put raster street tiles under a Web Mercator map.

> **Status: stable since 1.0.0.** The public API follows semantic versioning,
> so breaking changes wait for a major version. The engine, all seven series
> types, hex tile layouts and the morph between them, projections, joins,
> scales, fills, legend, tooltip, labels, annotations, camera, geometry
> registry, drilldown, clustering, selection, the accessibility layer, `locale`
> and the raster tile underlay are shipped and tested. The story engine is not
> built yet.

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
   when the first map mounts, into the root the map renders in: at the start
   of `<head>` (so your own rules at equal specificity win), or into the
   shadow root of a web component the map lives in. Importing
   `apexmaps/apexmaps.css` explicitly (as the wrapper READMEs show) is
   harmless and useful for CSP-strict setups.
3. **Geometry comes from the registry**: `geo: { map: 'world/countries' }`.
   Use pack ids or their aliases (`'us'`, `'jp/prefectures'`, `'eu/nuts2'`)
   before reaching for a GeoJSON URL. Custom geometry: pass GeoJSON/TopoJSON
   inline or via `ApexMaps.registerMap(id, source)`.
4. **Coordinates are `[lon, lat]`**, longitude first, matching GeoJSON. A map
   whose points all sit on a vertical line through Africa has them swapped.
5. **Choropleth is the default series type.** A series without `type` is a
   choropleth (`chart.type` can seed a different default). A `pixel` series
   is the same choropleth drawn as dots: change the type, keep the data.
6. **Join data with `joinBy`**: `'name'`, `['geoField', 'dataField']`, or
   `{ geo, data }`. When omitted, the join key is auto-detected. Each registry
   pack carries a recommended key (`us/states` joins on postal abbreviations
   like `'CA'`, `us/counties` on FIPS, admin-1 packs on ISO 3166-2 codes,
   `world/countries` on `iso_a3`).
7. **Use `null`, never `undefined`, for missing values.** No-data features get
   `scale.nullColor` and stay out of the scale.
8. **Values must be numbers** (or `null`). The default value field is `value`;
   override with `valueField` or per-series accessors.
9. **Licensed features render with a watermark until a key is set**: `pixel`,
   `arc`, `line` and `hexbin` series, grid layouts (`geo.layout: 'hex'`, an
   `@hex` pack id, or your own `registerLayout` table), the raster tile
   underlay (`geo.tiles` with a `url`, gated as configured even while no tile
   is drawn), pattern/image fills, marker clustering, drilldown, linked
   selection (`link`), annotations, story context, and self-registered
   projections. Everything else renders without a watermark, including the
   `viewChange` event and `locale`. No watermark is not the same as no
   licence needed: the terms in the package's `LICENSE.md` decide who needs
   one, so never tell a user a map is free to use because it renders clean.
   `ApexMaps.setLicense('APEX-...')` before rendering removes the watermark.
   One Apex key covers every product, and the license is page-wide:
   `ApexCharts.setLicense(key)` licenses ApexMaps too, and clears the
   watermark from maps already on screen.
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
| `pixel` | the choropleth's rows, unchanged: `{ <joinKey>: 'FRA', value: 7.3, ... }` | joined to geometry by key, each region drawn as the lattice points inside it |
| `bubble` | `{ lon, lat, value, name? }` (`lng` accepted for `lon`) | coordinates, or `joinBy` to feature centroids |
| `marker` | `{ lon, lat, name?, value?, category?, shape?, color?, size? }` | coordinates, or `joinBy` to centroids |
| `arc` | `{ from, to, value?, name? }`, `from`/`to` REQUIRED, each `[lon, lat]` or a geometry key like `'JFK'`/`'FRA'` | endpoints |
| `line` | `{ path: [[lon,lat], ...], name? }` (`coordinates` accepted for `path`) | the vertex sequence itself |
| `hexbin` | `{ lon, lat, value?, name? }` (`lng` accepted; `coordinates: [lon, lat]` accepted) | coordinates only, binned onto a lattice. There is NO `joinBy` |

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
- **pixel** (licensed): a choropleth drawn as a lattice of marks. Same
  `joinBy`, `fuzzyJoin`, `normalizeBy`, `scale`, `valueField` and
  `drilldown`, same legend, tooltip, selection and `featureClick`; no `fill`
  (the dots are the texture) and no outline by default. `shape`: `'circle'`
  (default), `'square'`, `'hexagon'`, `'diamond'`. `grid: 'geo'` (default)
  fixes the dots to the ground, so zooming magnifies them; `grid: 'screen'`
  keeps an even screen spacing and refines on zoom. `spacing` is DEGREES on a
  geo grid (default about 120 dots across the map: 3 on a world map) and
  screen PIXELS on a screen grid (default 8). `size` is a share of the
  spacing (`'80%'` default; `'100%'` tiles edge to edge) or a number of
  pixels. On an `orthographic` globe the lattice sits on the sphere and turns
  with it; `grid: 'screen'` is refused there.
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
  geometry keys, and `joinBy` names the geometry field they match
  (`joinBy: 'name'` lets an endpoint read `'France'` on `world/countries`).
- **line**: caller supplies the whole path (GPS trace, shipping lane); same
  `width`, `colorScale`, `endpoints`, `flow` options as arc.
- **hexbin** (licensed): the right mark when there are more points than pixels.
  Bins points onto a hexagonal lattice and colours each cell by an
  `aggregate` (`'count'` default, plus `'sum' | 'mean' | 'min' | 'max'`).
  `radius` is in SCREEN pixels (default 14), so the lattice refines as the
  reader zooms instead of magnifying, and the colour domain follows the bins
  unless `scale.domain` or `scale.breaks` pins it. `minCount` drops thin
  cells, `gap` (default 0) shrinks each cell, `orientation` is `'pointy'`
  (default) or `'flat'`. Points only: a hexbin has no `joinBy`.

Common to every series: `name`, `visible`, `opacity`, `stroke`, `labels`,
`animation` (`grow` / `draw` / `fade`), `valueField`.

Full details: `references/data-format.md`.

## 3. Top-Level Options (`ApexMapsOptions`)

| Key | Purpose |
|---|---|
| `chart` | width, height, default series `type`, background, fontFamily, `context: 'story' \| 'dashboard'`, `animations`, `events` |
| `geo` | `map` (registry id / URL / GeoJSON / TopoJSON), `layout` (`'hex'` draws the region set as a hex tile map; licensed), `object`, `keyField`, `nameField`, `projection`, `view: { fit, padding }`, `graticule`, `sphere`, `fill`, `repairWinding`, `tiles` (raster `{z}/{x}/{y}` underlay; needs `projection: 'webMercator'`; licensed) |
| `series` | array of the union above |
| `legend` | `position` (`bottom` default), `style: 'auto' \| 'classes' \| 'gradient'`, `interactive` (click to mute a class), `marker` (hover arrow on gradient bars), formatters |
| `tooltip` | `formatter(context)` returning HTML, `valueFormatter`, `followCursor` |
| `dataLabels` | collision-avoiding labels: `field`, `formatter`, `minFeatureArea`, halo style |
| `states` | `hover`, `active`, `muted` appearance |
| `theme` | `{ mode: 'light' \| 'dark' \| 'auto', palette }` |
| `interaction` | `zoom` (wheel, doubleClick, `controls`), `pan` (+ inertia), `rotate` (globe drag on orthographic, `'auto'`), `selection` (box select with `modifier: 'shift'`), `nearest` (proximity hit assist) |
| `a11y` | on by default: generated description, keyboard navigation, optional `dataTable` |
| `locale` | the text the map writes itself: `zoomControls`, `breadcrumb`, `dataTable`, `noData`, `sizeLegend`, `drilldown` announcements. Each key falls back to English |
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
| `getView()` | what is on screen now: `{ bounds, center, zoom, k }`, the `viewChange` payload |
| `diagnoseJoin(seriesIndex?)` | the join mismatch report, on demand |
| `toSpec()` | current options as a serializable object |
| `getSvgString()`, `exportSVG()`, `exportPNG()`, `dataURI()` | export |
| `destroy()` | teardown |

Statics: `ApexMaps.setLicense(key)`, `registerMap(id, source, meta?)`,
`registerLayout(id, pack, meta?)` (your own grid layout table),
`registerProjection(name, factory)`, `registerPalette(name, palette)`,
`setGeoSource(urlOrFetcher)` (self-host geometry), `listMaps()`,
`catalogue()`, `mapMeta(id)`, `listProjections()`, `listPalettes()`,
`getInstance(id)`, `version`.

Events (`ApexMapsEventMap`): `rendered`, `updated`, `resized`,
`featureClick`, `featureHover`, `featureFocus`, `markClick`, `markHover`,
`clusterClick`, `drilldown`, `drillup`, `selectionChange`, `legendToggle`,
`zoom`, `panEnd`, `rotate`, `rotateEnd`, `viewChange`.

`viewChange` fires on every pan, zoom and globe-turn frame and on resize,
synchronously with the frame, with `{ bounds: [[west, south], [east, north]]
| null, center: [lon, lat] | null, zoom, k }`. `zoom` is the view's web-map
zoom (fractional; with 256px tiles it is also the tile level), and `null` on
anything but an unturned Web Mercator. It is not gated, and is computed only
while something listens.

## 6. Geometry Registry

26 built-in boundary packs plus 7 hex tile layouts, fetched lazily (one
request per pack). Canonical id is `region/level@detail`; the detail-free form
resolves to the lightest pack.

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

### Hex tile layouts (licensed)

`geo.layout: 'hex'` draws the region set as one equal hexagon per region
instead of its real boundaries: a honeycomb, tilegram, or hex cartogram.

```js
geo: { map: 'us', layout: 'hex' }   // one hexagon per state
geo: { map: 'us/states@hex' }       // the same layout, named directly
```

- Seven ship: `us/states@hex`, `au/admin1@hex`, `ca/admin1@hex`,
  `de/admin1@hex`, `br/admin1@hex`, `jp/admin1@hex`, `eu/nuts0@hex` (aliases
  `us/hex`, `au/hex`, ... also resolve). A `layout: 'hex'` on a map with no
  layout throws and names the ones that exist; it never falls back to real
  boundaries.
- **The join does not change.** Cells carry the same key the boundary pack
  carries (`abbr` for US states, `iso_3166_2` for admin-1, `nuts_id` for
  NUTS), so one dataset works against either representation.
- A layout is a diagram, not geography, and says so about itself: it is drawn
  with the `identity` projection, zoom and pan default OFF, and data labels
  default to the KEY (`CA`, not `California`, which does not fit a cell).
- Toggling the layout through `updateOptions` **morphs** each region between
  its boundary and its cell, which is how a reader learns which cell is which.
- Layouts are a subset by design: `us/states@hex` places 51 of the pack's 56
  features (the five territories are unplaced) and reports that as a dev
  warning. Rows keyed to an unplaced region do not appear.
- Licensed however you reach it: `geo.layout`, an `@hex` id, or your own
  `ApexMaps.registerLayout(id, { keyField, cells })` table.

### Raster tiles under the map (licensed)

`geo.tiles` draws `{z}/{x}/{y}` raster tiles (any slippy-map server) under
every series, in the same transform as the geometry. Nothing is requested
without a `url`.

```js
geo: {
  map: 'us/states',
  projection: 'webMercator',   // REQUIRED: no pack defaults to it
  tiles: {
    url: 'https://tiles.example.com/{z}/{x}/{y}.png',
    attribution: '© Example contributors',
    minZoom: 4,                // no tiles until the reader zooms in this far
  },
}
```

- Tiles draw only on `'webMercator'` (or `'mercator'` / `'epsg:3857'`) with no
  rotation, angle or reflection. On any other projection, including a pack's
  own default such as `albersUsa`, the layer stays empty with no error; debug
  mode says why.
- Other keys: `subdomains` (`{s}`, default `['a', 'b', 'c']`), `maxZoom`
  (the server's highest `{z}`, default 19, stretched past it), `opacity`
  (default 1), `tileSize` (default 256; set it to what the server cuts, since
  it decides which `{z}` is requested: 512px tiles come from one level
  shallower), and `request(url, { z, x, y })` returning a URL or
  `{ url, headers }` (may be async) to sign, key, or proxy each tile.
  `minZoom` is the view's zoom, as `viewChange` reports it.
- While tiles show, the automatic basemap's fill gives way; a choropleth keeps
  its fill, so give it `opacity` if the streets should show through. Exports
  leave the tiles out.

Full pack table, the layout table, custom geometry, projections, tiles, and
the camera: `references/geo-and-projections.md`.

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
11. **Hexbinning regions instead of points**
    - Wrong: `{ type: 'hexbin', joinBy: ['abbr', 'code'], data: stateRows }`.
      A hexbin has no `joinBy`, and binning region centroids bins the
      geometry: the answer would be decided by how the regions were drawn.
    - Correct: hexbin takes `{ lon, lat }` points. Rows keyed to regions are a
      choropleth.
12. **Expecting `layout: 'hex'` on any map**
    - Wrong: `geo: { map: 'in', layout: 'hex' }` and expecting real boundaries
      when no layout exists. It throws, on purpose: a silent fallback looks
      exactly like the option being ignored.
    - Correct: use one of the seven, or register your own table with
      `ApexMaps.registerLayout()`.
13. **Tiles on the pack's default projection**
    - Wrong: `geo: { map: 'us', tiles: { url } }`. The US packs default to
      `albersUsa`, and every other map to its pack's projection or
      `equalEarth`, never a Mercator, so the tile layer silently stays empty
      (and is still watermarked without a key).
    - Correct: `geo: { map: 'us', projection: 'webMercator', tiles: { url } }`.
14. **Feeding points to a pixel map**
    - Wrong: `{ type: 'pixel', data: [{ lon, lat, value }] }`. A pixel series
      is a choropleth: it joins rows to regions and has no point positions.
    - Correct: region-keyed rows for `pixel`; raw points want `hexbin`.

## 8. Theming

- `theme: { mode: 'auto' }` follows `prefers-color-scheme`; `'light'` /
  `'dark'` force it. `theme.palette` names the default scale palette.
- CSS custom properties (`--apexmaps-bg`, `--apexmaps-fg`,
  `--apexmaps-surface`, `--apexmaps-focus`, `--apexmaps-font-size`, ...)
  restyle chrome without options. Set them on the map's element or on any
  ancestor (`:root`, a dashboard panel, a shadow host). A value applies in
  light and dark mode alike; give dark its own under `.apexmaps--dark`. The
  library only reads these names, so your own CSS reading an unset
  `--apexmaps-*` gets nothing: give it a fallback.
- 17 built-in palettes (`blues`, `viridis`, `rdbu`, `okabeIto`, `apex`, ...);
  diverging palettes are picked automatically when the domain crosses zero.
- `prefers-reduced-motion` disables animations, flow beads keep their dots in
  place, and drilldown swaps without motion. Nothing to configure.

Details: `references/styling-and-interaction.md`.

## 9. Reference Routing Table

| Topic | Reference File |
|---|---|
| Datum shapes per series type, joins (`joinBy`, `fuzzyJoin`, diagnostics), `normalizeBy`, FIPS repair, pixel lattices (shape, grid, spacing, size), hexbin binning and aggregates | `references/data-format.md` |
| Registry packs and aliases, hex tile layouts and the morph, `registerMap`, `registerLayout`, custom GeoJSON/TopoJSON, `setGeoSource`, projections and spec objects, raster tiles (`geo.tiles`), camera API, `viewChange` / `getView()` | `references/geo-and-projections.md` |
| Scales, palettes, size scales, legends, tooltips, data labels, pattern/image fills, states, selection, linked maps, drilldown, zoom controls, a11y, `locale`, theming, responsive, licensing | `references/styling-and-interaction.md` |
| React / Vue / Angular wrappers | `references/framework-wrappers.md` |
