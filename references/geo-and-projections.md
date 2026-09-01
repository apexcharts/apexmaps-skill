# Geometry and Projections: Registry, Custom Maps, Camera

## The built-in registry: 26 boundary packs, 7 hex layouts

Name a pack in `geo.map` and the geometry is fetched lazily: nothing loads until a pack is used, and one pack is one HTTP request no matter how many aliases or maps on the page ask for it. The dataset ships as a separate package (`apexmaps-geo`), versioned independently of the library, fetched by default from `https://cdn.jsdelivr.net/npm/apexmaps-geo@1/`. That URL is pinned to the major version, so it serves the current 1.x and a caller on the default needs no configuration for either the boundary packs or the hex layouts. For offline or self-hosted setups, `npm install apexmaps-geo` and point `ApexMaps.setGeoSource()` at a loader function or your own URL; **the seven hex layout files are new in `apexmaps-geo@1.1.0`**, so a self-hosted copy from an earlier release has to be refreshed before `layout: 'hex'` resolves.

| Canonical id | Aliases | `keyField` | One feature is | Features |
|---|---|---|---|---|
| `world/countries@110m` | `world/countries`, `world` | `iso_a3` | Countries | 177 |
| `world/countries@50m` | none (bare form goes to `@110m`) | `iso_a3` | Countries | 242 |
| `world/land@110m` | `world/land` | none (backdrop) | Landmasses | 127 |
| `world/land@50m` | none | none (backdrop) | Landmasses | 1,420 |
| `us/states@10m` | `us/states`, `us` | `abbr` (postal, `'CA'`) | States | 56 |
| `us/counties@10m` | `us/counties` | `fips` (5-digit string) | Counties | 3,231 |
| `us/nation@10m` | `us/nation` | none (backdrop) | Nation | 1 |
| `eu/nuts0@20m` | `eu/nuts0`, `eu` | `nuts_id` | Countries | 37 |
| `eu/nuts1@20m` | `eu/nuts1`, `eu/regions` | `nuts_id` | Major regions | 125 |
| `eu/nuts2@20m` | `eu/nuts2` | `nuts_id` | Basic regions | 334 |
| `eu/nuts3@20m` | `eu/nuts3` | `nuts_id` | Small regions | 1,514 |
| `cn/admin1@10m` | `cn/admin1`, `cn/provinces`, `cn` | `iso_3166_2` | Provinces | 32 |
| `in/admin1@10m` | `in/admin1`, `in/states`, `in` | `iso_3166_2` | States | 36 |
| `jp/admin1@10m` | `jp/admin1`, `jp/prefectures`, `jp` | `iso_3166_2` | Prefectures | 47 |
| `de/admin1@10m` | `de/admin1`, `de/states`, `de` | `iso_3166_2` | States | 16 |
| `gb/admin1@10m` | `gb/admin1`, `gb/districts`, `gb` | `iso_3166_2` | Districts and unitary authorities | 232 |
| `fr/admin1@10m` | `fr/admin1`, `fr/departments`, `fr` | `iso_3166_2` | Departments | 101 |
| `it/admin1@10m` | `it/admin1`, `it/provinces`, `it` | `iso_3166_2` | Provinces | 110 |
| `ca/admin1@10m` | `ca/admin1`, `ca/provinces`, `ca` | `iso_3166_2` | Provinces and territories | 13 |
| `br/admin1@10m` | `br/admin1`, `br/states`, `br` | `iso_3166_2` | States | 27 |
| `ru/admin1@10m` | `ru/admin1`, `ru/regions`, `ru` | `iso_3166_2` | Federal subjects | 86 |
| `mx/admin1@10m` | `mx/admin1`, `mx/states`, `mx` | `iso_3166_2` | States | 33 |
| `au/admin1@10m` | `au/admin1`, `au/states`, `au` | `iso_3166_2` | States and territories | 12 |
| `kr/admin1@10m` | `kr/admin1`, `kr/provinces`, `kr` | `iso_3166_2` | Provinces | 17 |
| `es/admin1@10m` | `es/admin1`, `es/provinces`, `es` | `iso_3166_2` | Provinces | 52 |
| `id/admin1@10m` | `id/admin1`, `id/provinces`, `id` | `iso_3166_2` | Provinces | 33 |

Provenance travels with each pack and is rendered as attribution automatically when required: world and admin-1 packs are Natural Earth 5.1.1 (public domain, 2022, de facto boundaries), US packs are Census TIGER/Line via us-atlas 3.0 (public domain, 2023, legal boundaries), NUTS packs are Eurostat GISCO NUTS 2021 at 1:20 million (CC BY 4.0, EuroGeographics attribution). Packs ship repaired identifiers: Natural Earth publishes `iso_a3` as `-99` for France and Norway, and the packs fix that.

### Per-pack defaults and caveats

A pack's recommended projection and bounds apply automatically; an explicit `geo.projection` or `geo.view.fit` always wins.

| Pack(s) | Default projection / bounds | Why |
|---|---|---|
| `us/states`, `us/counties`, `us/nation` | `albersUsa` | Insets Alaska and Hawaii. Without it the Aleutians cross the antimeridian and the fitted map spans the whole world. |
| `eu/nuts0` through `eu/nuts3` | `{ name: 'azimuthalEqualArea', rotate: [-10, -52] }`, bounds `[-25, 32, 45, 72]` | ETRS89-LAEA (EPSG:3035), the projection Eurostat publishes in; equal-area matters because a choropleth encodes value as area fill. Bounds crop the overseas regions (NUTS reaches French Guiana and Réunion); Cyprus and the Canaries stay in. |
| `ca/admin1` | `{ name: 'conicConformal', rotate: [95, 0], parallels: [49, 77] }` | Custom conic for Canada. |
| `ru/admin1` | `{ name: 'conicEqualArea', rotate: [-100, 0], parallels: [50, 70] }` | Chukotka crosses the antimeridian; a projection centred on 0 degrees tears the country in half and the fitted extent becomes the whole world. |
| `fr/admin1` | bounds `[-5.5, 41, 10, 51.5]` | Default view is metropolitan France; Guadeloupe, Martinique, French Guiana, Réunion and Mayotte are in the data (fitting all of them spans 118 degrees of longitude). Pass `geo.view.fit` to see them. |
| `au/admin1` | bounds `[112, -44.5, 154.5, -9]` | Excludes the Heard and McDonald Islands, 4,000 km southwest of Perth. |
| `us/counties` | note | FIPS codes are five digits and keep the leading zero; a numeric data key is repaired automatically and the repair is reported. |
| `gb/admin1` | note | Natural Earth admin-1 for the UK is the district tier, not the four countries or the twelve regions. For those use `eu/nuts1@20m`. |
| `fr/admin1`, `it/admin1`, `es/admin1` | note | Departments / provinces, not regions. For the 13 French regions use `eu/nuts1@20m`; for the 20 Italian regions or 17 Spanish autonomous communities use `eu/nuts2@20m`. |

## How ids resolve

Canonical form is `region/level@detail`. The detail-free form always resolves, and resolves to the pack declared first, which is the lightest one: `world/countries` gives you the 110m file, not the 800 kB 50m one. Aliases use the country's own term for the tier the source actually provides (`fr/departments`, not "regions": Natural Earth admin-1 for France is the 101 departments).

```js
geo: { map: 'world/countries@110m' }  // canonical id
geo: { map: 'world/countries' }       // detail-free: the lightest pack
geo: { map: 'us' }                    // alias for us/states@10m
geo: { map: 'jp/prefectures' }        // alias for jp/admin1@10m
```

Introspection statics:

```js
ApexMaps.listMaps()        // every registered id and alias
ApexMaps.catalogue()       // GeoPack[]: the built-in packs with provenance (excludes registerMap entries)
ApexMaps.mapMeta('us')     // source, license, vintage, boundaries, keyField, levelName,
                           // projection, bounds, note; alias entries carry aliasOf
```

`listMaps()` includes the hex layout ids and their aliases; `catalogue()` is boundary packs only. A layout's `mapMeta` carries three extra fields that say it is a diagram rather than a place: `labelField` (the key), `fixed: true`, and `layout: { grid, of, unplaced }`.

`mapMeta(id).boundaries` records whose boundary view a pack carries (Natural Earth de facto, US Census legal, Eurostat NUTS 2021). The software licence does not cover the data; all three sources are permissively licensed.

## Hex tile layouts (licensed)

A layout draws the region set as a grid of equal cells instead of its real boundaries: a hex tile map, honeycomb, or tilegram. Every region becomes one identical hexagon, so the smallest unit is as legible as the largest and land area stops deciding how loud a value looks.

```js
geo: { map: 'us', layout: 'hex' }     // one hexagon per state
geo: { map: 'us/states@hex' }         // the same layout, named directly
```

It is a cartogram, not geography. Reach for it when the story is about people, votes or money and land area is actively misleading, or when every unit has to be legible including the ones too small to see. The cost is that a reader who does not already know the country cannot navigate it, so it is a poor default and a strong deliberate choice.

| Layout id | Aliases | Layout of | `keyField` | Cells |
|---|---|---|---|---|
| `us/states@hex` | `us/hex`, `us/states/hex` | `us/states@10m` | `abbr` | 51 (50 states plus DC) |
| `au/admin1@hex` | `au/hex`, `au/states/hex` | `au/admin1@10m` | `iso_3166_2` | 8 |
| `ca/admin1@hex` | `ca/hex`, `ca/provinces/hex` | `ca/admin1@10m` | `iso_3166_2` | 13 |
| `de/admin1@hex` | `de/hex`, `de/states/hex` | `de/admin1@10m` | `iso_3166_2` | 16 |
| `br/admin1@hex` | `br/hex`, `br/states/hex` | `br/admin1@10m` | `iso_3166_2` | 27 |
| `jp/admin1@hex` | `jp/hex`, `jp/prefectures/hex` | `jp/admin1@10m` | `iso_3166_2` | 47 |
| `eu/nuts0@hex` | `eu/hex`, `eu/countries/hex` | `eu/nuts0@20m` | `nuts_id` | 37 |

- **The join does not change.** Cells carry the same key the boundary pack carries, so one dataset, one `joinBy`, and one scale work against either representation.
- `layout: 'hex'` resolves through aliases, so `map: 'us'` and `map: 'us/states@10m'` both find `us/states@hex`. Only `'hex'` is accepted: square grids exist in the generator and are reachable through `registerLayout` with `grid: 'square'`, but no built-in square layout ships.
- **A map with no layout throws**, naming the ones that exist, rather than falling back to real boundaries. A silent fallback draws a perfectly good map that is not the one that was asked for, which looks exactly like the option being ignored.
- A layout is distributed as a table of grid positions, not as baked polygons, so the file is a fraction of the boundary pack it replaces: `us-states-hex.json` is 51 cells of two integers each, about 4 kB against 111 kB for `us-states-10m.json`. It also resolves independently of the boundaries, so this form never downloads geometry it will not draw.

### What a layout changes about the map

Three recommendations travel with the pack, and all three are overridable:

| Behaviour | Why |
|---|---|
| `projection: 'identity'` | The coordinates are a flat grid, so projecting them would be meaningless. |
| Zoom and pan default **off** (`fixed: true`) | A diagram has no detail that sharpens on zoom and nothing off-screen to pan to, so both gestures go back to the page. Set `interaction: { zoom: { enabled: true } }` explicitly to take them back. |
| Data labels default to the **key**, not the name | "Rhode Island" does not fit a cell sized for Rhode Island, which is also why layout keys are the postal abbreviation. Data labels are still off until you turn them on: `dataLabels: { enabled: true, minFeatureArea: 0, collision: 'none' }` is the setting that labels every cell. |

Layouts are a subset by design, and say so. `us/states@hex` places 51 of the boundary pack's 56 features: the five inhabited territories (`AS`, `GU`, `MP`, `PR`, `VI`) are unplaced because no published one-hex-per-unit layout includes them, and `au/admin1@hex` leaves out three external-territory codes nobody reports state-level data for. A dev warning names them on render, because rows keyed to an unplaced region are dropped by the join like any other unmatched key and would otherwise just fail to appear.

### The morph: geography and back

Turning the layout on or off through `updateOptions` **morphs** each region from its real boundary to its cell rather than swapping the two pictures:

```js
await map.updateOptions({ geo: { map: 'us', layout: 'hex' } })   // states walk to their cells
await map.updateOptions({ geo: { map: 'us', layout: null } })    // and back
```

- It runs whenever a map change lands on two representations of the **same region set** (a layout and the boundary pack it is a layout of). Changing `map` to unrelated geography shares no keys, so that stays a swap.
- The point of it is legibility, not decoration: the hardest thing about reading a cartogram is knowing which cell is which region, and watching Texas walk to its cell is how a reader keeps the map they already knew.
- It runs off `chart.animations`, so `enabled: false`, `prefers-reduced-motion`, and `speed: 'instant'` all turn it off with everything else. It is the one renderer doing vertex work every frame, so it stands down above the motion budget instead of degrading, and the map swaps.
- Labels and annotations fade out for the flight and back at the end: they are anchored to the geometry that has already arrived, and a cell's label text differs from a region's.

### `registerLayout`: your own table

```js
ApexMaps.registerLayout('nl/provinces@hex', {
  keyField: 'code',                                   // must match the boundary pack's key
  cells: { 'NL-GR': [4, 0], 'NL-FR': [3, 1], 'NL-DR': [4, 1] },   // key -> [col, row]
  names: { 'NL-GR': 'Groningen' },                    // so a tooltip has something to say
})

// geo: { map: 'nl/provinces@hex' }
```

`cells` is `key -> [col, row]`, row 0 north and col 0 west. Optional pack fields: `grid` (`'hex'` default, `'square'`), `orientation` (`'pointy'` default, `'flat'`), `offset` (`'odd-r'`/`'even-r'` for pointy, `'odd-q'`/`'even-q'` for flat; mismatching orientation and offset throws rather than being reinterpreted), `gap` (default `0.06`, a property of the pack so two maps sharing a layout cannot disagree), `names`, `unplaced`, `compromises`, and the usual provenance fields. Two regions on one cell throws, because one of them would be invisible.

Curating the table is the whole job: there is no canonical layout for any country, and a good one is a judgement about which real adjacencies matter most. Registering is free; **rendering** a grid layout is licensed, your own table included, because the gate is on drawing a region set as a grid rather than on whose table it is.

## Geometry beyond the registry

`geo.map` is a `MapSource`: a registry id, a URL, or inline geometry (`GeoInput` = FeatureCollection, single Feature, bare Geometry, `Feature[]`, or a TopoJSON topology). Related `geo` options:

| Option | Default | Purpose |
|---|---|---|
| `object` | undefined | TopoJSON object name, when the topology holds several |
| `keyField` | pack's recommended key | Force the geometry join-key property |
| `nameField` | undefined | Force the label property |
| `repairWinding` | `true` | Normalise ring winding on ingest (the failure mode is silent and catastrophic; repairs are reported) |

### `registerMap`: your own geometry under an id

`ApexMaps.registerMap(id, geometry, meta?)` puts anything into the registry with the same provenance fields the built-in packs carry. The second argument is `GeoInput` or a lazy loader `() => Promise<GeoInput>`. Nothing about the engine assumes the earth: a floor plan, stadium, or wafer map works like a country, drawn with the `identity` projection because the coordinates are already a flat plan, not lon/lat.

```js
ApexMaps.registerMap('demo/floor',
  { type: 'FeatureCollection', features: zones },
  { source: 'hand-drawn', license: 'n/a', vintage: '2026', keyField: 'id', levelName: 'Zones' })

new ApexMaps(el, {
  geo: { map: 'demo/floor', projection: 'identity', keyField: 'id' },
  series: [{ joinBy: ['id', 'key'], data: rows }],
})
```

### `setGeoSource`: self-hosting the packs

Point the catalogue at your own copy of the dataset: a base URL, or a loader function (bundler imports, air-gapped paths, an authenticated fetch). A source change drops previously cached loads, and a failed load is never cached.

```js
ApexMaps.setGeoSource('https://cdn.example.com/apexmaps-geo/')
ApexMaps.setGeoSource((file) => import(`apexmaps-geo/${file}`).then((m) => m.default)) // no network
```

A self-hosted copy needs refreshing to `apexmaps-geo@1.1.0` before hex layouts resolve: that release adds seven files (`us-states-hex.json`, `au-admin1-hex.json`, `ca-admin1-hex.json`, `de-admin1-hex.json`, `br-admin1-hex.json`, `jp-admin1-hex.json`, `eu-nuts0-hex.json`) to the 26 boundary packs. Callers on the default source get them with no configuration, because that URL is pinned to `apexmaps-geo@1`.

## Projections

Default is `equalEarth`: a world thematic map in Web Mercator overstates high-latitude area by an order of magnitude, and most callers never choose a projection at all. All 20 accepted name strings (16 projections plus 4 aliases), all free:

| Name | Aliases | Notes |
|---|---|---|
| `equalEarth` | | Default. Equal-area world. |
| `mercator` | `webMercator`, `epsg:3857` | Conformal. |
| `equirectangular` | `plateCarree`, `epsg:4326` | Raw lon/lat layout. |
| `naturalEarth` | | Compromise world. |
| `orthographic` | | The globe. Drag rotates by default. |
| `albers`, `albersUsa` | | `albersUsa` is a fixed composite (AK and HI insets): `rotate`/`center` are ignored on it. |
| `conicConformal`, `conicEqualArea`, `conicEquidistant` | | Take `parallels`. |
| `azimuthalEqualArea`, `azimuthalEquidistant`, `gnomonic`, `stereographic` | | Azimuthal: a camera `center` move rotates the sphere. |
| `transverseMercator` | | |
| `identity` | | No projection: planar coordinates (floor plans). |

A projection is a name or a `ProjectionSpec` object: `{ name, rotate: [lambda, phi, gamma?], center: [lon, lat], parallels: [a, b], angle, clipAngle, clipExtent, reflectX, reflectY }`. Rotating is how a Pacific-centred world or an antimeridian-crossing country is drawn without tearing. Unknown names throw (listing what is registered) rather than silently falling back.

```js
geo: { projection: { name: 'conicEqualArea', rotate: [-100, 0], parallels: [50, 70] } }
geo: { projection: { name: 'orthographic', rotate: [53, 10], clipAngle: 90 } }
```

The rest of the `d3-geo-projection` catalogue (Robinson, Mollweide, Winkel Tripel, ...) is registered by the consumer. Registering is free; rendering a map with a self-registered projection is a licensed feature (works without a key, with a watermark). Re-registering a built-in name over the built-in stays free, since the gate is by name.

```js
import { geoRobinson } from 'd3-geo-projection'
ApexMaps.registerProjection('robinson', () => geoRobinson())
ApexMaps.listProjections()          // now includes 'robinson'
// geo: { projection: 'robinson' }
```

## View, graticule, sphere, basemap

| Option | Default | Purpose |
|---|---|---|
| `view.fit` | `'data'` | `'data'` fits the geometry, `'world'` the full projection, `'none'` skips fitting, `[west, south, east, north]` fits a bbox. Pack `bounds` supply the default view for packs that declare one. |
| `view.padding` | `16` | Number or `{ top, right, bottom, left }`. |
| `graticule` | `{ show: false, step: 20, color: 'rgba(128,128,128,0.25)', width: 0.5 }` | Meridian/parallel grid. |
| `sphere` | `{ show: false, fill: 'none', stroke: 'rgba(128,128,128,0.4)', width: 0.5 }` | Outline (and fill) of the projected sphere; the disc edge on a globe. |
| `fill` | | Fill for the no-data basemap drawn when no series is configured: a map with `geo` and no `series` still renders, as a basemap. |
| `boundaries` | `'neutral-dashed'` | Disputed-territory policy. Declared but NOT applied yet; setting it warns in dev diagnostics. Do not rely on it. |

## Camera

`map.camera` is the imperative controller (available after `await map.render()`). Every move is interruptible and retargeting: a new move starts from the current interpolated state instead of queueing or snapping, and a hand pan or zoom stops an in-flight move. All of it honours `prefers-reduced-motion` by jumping instead of animating.

`CameraTarget`: `{ center: [lon, lat], zoom, bounds, padding, maxZoom, k, x, y }` (`bounds` wins, then `center`, then `zoom`, then raw `k`/`x`/`y`; `k`/`x`/`y` are advanced raw scale/translate). Transitions add `{ duration, ease }`.

```js
map.camera.jumpTo({ center: [2.35, 48.86], zoom: 6 })              // no animation
await map.camera.easeTo({ center: [2.35, 48.86], zoom: 8, duration: 400, ease: 'cubicInOut' })
await map.camera.flyTo({ center: [139.69, 35.69], zoom: 8 })       // Van Wijk path
await map.camera.fitBounds(worldBounds, { padding: 40, maxZoom: 12, transition: 'fly' })
```

- `flyTo` follows the Van Wijk and Nuij zoom-and-pan path: it arcs out to a wider zoom on long moves and derives its duration from the distance, so crossing a continent takes longer than nudging to a neighbour. Extra knobs: `speed` (divides the natural duration) and `curve` (Van Wijk rho; 0 disables the arc).
- `easeTo` is fixed-duration and eased: for short mechanical moves (a legend re-fit, a drilldown) where an arc would be theatrical.
- `fitBounds` takes world-space bounds `[[x0, y0], [x1, y1]]` (project geographic corners via `map.viewport.project([lon, lat])`). To frame a feature by key, prefer the instance method.

Instance-level camera helpers:

```js
await map.frameFeature('France', { padding: 40, transition: 'fly' }) // frame a feature by key
await map.resetView()      // opening fit, and on a globe the opening rotation too
map.zoomIn(); map.zoomOut() // one step (interaction.zoom.step, default 1.6) about the plot centre
map.zoom                    // current scale, 1 at the opening fit
map.rotateTo([-25, -18])    // absolute [lambda, phi, gamma?], reprojects; no-op if not rotatable
map.rotation                // [lambda, phi, gamma]; [0, 0, 0] on a projection that cannot rotate
```

## Globe behaviour: rotation, not panning

On an azimuthal projection (`orthographic`, `stereographic`, `gnomonic`, `azimuthalEqualArea`, `azimuthalEquidistant`) a camera move to a `center` turns the sphere instead of panning: the camera is a screen-space transform and cannot reach the far side of a globe by translating. The turn interpolates as a quaternion slerp: short way round (crossing the antimeridian goes 20 degrees, not 340), steady angular pace, no sideways swing over the poles. `flyTo` derives the turn's duration from the angle covered, and when a move both turns and zooms the longer of the two sets the pace so they land together; `easeTo` keeps its fixed duration, rotation included. Flat projections (Equal Earth, Mercator, the conics, `albersUsa`) pan exactly as before; the conics are deliberately flat because they are defined by standard parallels, not a centre.

Drag-to-spin is versor-based (the point under the cursor stays under the cursor at any latitude) and configured via:

```js
interaction: {
  rotate: {
    enabled: 'auto',  // default: on for orthographic, flat maps pan.
                      // true forces it on any projection that can rotate and invert; false gives the drag back to panning
    inertia: true,    // defaults to pan.inertia
  },
}
```

Wheel zoom, pinch, and double-click still belong to the camera, so zoom and spin compose. A rotation is a projection change, not a camera transform: it reprojects every coordinate, coalesced to one pass per frame. Events: `rotate` fires while turning, `rotateEnd` once the glide settles, both with `{ rotate }`.
