# Data Format: Series, Datums, and Joins

## The series array is a discriminated union

`series` is an array whose `type` field decides which options and datum shape apply:

```ts
type Series =
  | ChoroplethSeriesOptions   // type?: 'choropleth'  (the default)
  | BubbleSeriesOptions       // type: 'bubble'
  | MarkerSeriesOptions       // type: 'marker'
  | ArcSeriesOptions          // type: 'arc'   (licensed)
  | LineSeriesOptions         // type: 'line'  (licensed)
```

- A series without `type` is a **choropleth**. `chart.type` seeds the default for series that omit it.
- An unknown `type` does not throw: the series is skipped with a dev warning naming it.
- Mixing types on one map is normal (bubbles or arcs over a choropleth).
- `arc` and `line` series and marker clustering are licensed: they work without a key for evaluation, with a watermark.

## Coordinate order

Every coordinate **array** is `[lon, lat]`, longitude first, matching GeoJSON. Object datums may use named fields instead: `{ lon, lat }`, with `lng` accepted as a synonym for `lon`. A swapped array renders in the wrong place with no error, so points stacked on a vertical line are the symptom to check first.

## Fields common to every series (`SeriesCommon`)

| Field | Type | Notes |
|---|---|---|
| `name` | `string` | Display name; the choropleth legend title derives from it |
| `visible` | `boolean` | Hide the series without removing it |
| `opacity` | `number` | Series fill opacity |
| `stroke` | `{ color?, width?, opacity?, dashArray? }` | `dashArray` is an SVG dash pattern, e.g. `'4 2'` |
| `labels` | `{ show?, field? }` | `field` is a field name or `(datum) => string` |
| `animation` | `{ type?, duration?, stagger?, ease? }` | `type`: `'grow'` (symbols), `'draw'` (paths), `'fade'` (either), `'none'`; `stagger` is a per-mark delay in ms, applied in render order |
| `valueField` | `string \| (datum) => number \| null` | Field or accessor for the primary value. Default `'value'` |

Accessor options (`valueField`, `labels.field`, `size.field`, `colorField`) take a plain key, a dotted path (`'metrics.total'`, which survives serialization), or a function of the datum.

## Values: `null`, never `undefined`

- Anything that is not a finite number after coercion reads as `null`: `null`, `undefined`, `''`, `NaN`, unparseable strings. Numeric strings are coerced with `Number()`, but pass real numbers. Per the source: a missing value "can never be mistaken for zero".
- A `null` value paints the feature with `scale.nullColor`, names it with `scale.nullLabel` in the legend, and keeps it out of the scale domain.
- Write `null` explicitly for missing data. `undefined` resolves the same at the value layer but does not survive `JSON.stringify`, so a serialized config loses the row's intent.

## Choropleth data rows

```js
series: [{
  name: 'Unemployment rate',
  joinBy: ['iso_a3', 'code'],
  data: [
    { code: 'FRA', value: 7.3 },
    { code: 'DEU', value: 5.7 },
    { code: 'ESP', value: null },   // no data: nullColor, not 0
  ],
}]
```

- A row is a join key plus a value. Extra fields ride along: `tooltip.formatter` receives the whole row as `datum`, and `normalizeBy` and accessors read from it.
- `valueField` redirects the value read: `valueField: 'metrics.total'` or `valueField: (d) => d.a + d.b`.
- `normalizeBy: 'population'` divides the value by that field before the scale maps it (read from the datum first, then from the feature's properties). Rows whose denominator is missing or zero render as no-data, and a dev warning counts them. The legend retitles itself: a series named `Cases` becomes `Cases (per population)`, because a choropleth of raw counts across unequal areas mostly redraws the population map.

## `joinBy`: the three forms

| Form | Meaning |
|---|---|
| `joinBy: 'name'` | Same field on both sides: geometry property `name` against datum field `name` |
| `joinBy: ['iso_a3', 'code']` | `[geoField, dataField]` |
| `joinBy: { geo: 'iso_a3', data: 'code' }` | Same as the tuple, self-documenting; either side may be omitted |

When `joinBy` is omitted, both sides are auto-detected and the fields actually used are reported in the diagnostic:

- **Geometry side**: the key the pack already resolved (its `keyField`). Detection prefers codes over names because names are unstable across datasets: `iso_a3`, `iso3`, `adm0_a3`, `iso_a2`, `hc-key` (Highcharts map geometry works unchanged), `GEOID`, `fips`, `STATEFP`, `id`, `code`, `postal`, then `name` variants.
- **Data side**, first field present of: `id`, `key`, `code`, `iso`, `iso_a3`, `iso3`, `iso_a2`, `iso2`, `fips`, `geoid`, `hc-key`, `region`, `state`, `country`, `name`; else the first string-valued field.

Recommended pack keys: `world/countries` joins on `iso_a3`, `us/states` on postal abbreviations (`'CA'`), `us/counties` on 5-digit FIPS strings, EU packs on `nuts_id`, admin-1 packs on ISO 3166-2 codes (`'JP-13'`).

## Join behavior, repairs, and the diagnostic

Matching is **exact by default**: silent fuzzy matching would trade an obvious failure for a plausible wrong answer. Around nine in ten real-world map failures are join failures, so every mismatch is reported with a suggestion instead of rendered as silent grey.

- **FIPS leading-zero repair (always on).** When every geometry key is a fixed-width numeric string and at least one carries a leading zero (the FIPS/GEOID signature), an all-digit data key that is too short is zero-padded to that width, so `"1001"` matches `"01001"`. Applied even without `fuzzyJoin` because it is a lossless, unambiguous repair of a known spreadsheet defect, and it is always reported.
- **`fuzzyJoin: true` (opt-in).** Applies normalized matches (strip diacritics, lowercase, drop non-alphanumerics) and a curated alias table of variants that actually appear in published datasets: "Ivory Coast" for "Côte d'Ivoire", "Turkey" for "Türkiye", "Burma" for "Myanmar", plus Natural Earth's abbreviations ("W. Sahara", "Eq. Guinea", "Solomon Is."). Every substitution is listed in `applied`, so the convenience stays auditable.
- **Shared keys are reported, not deduplicated.** Published geometry shares keys: Natural Earth gives Australia, the Indian Ocean Territories, and Ashmore and Cartier Islands the same `iso_a3`, and Lord Howe Island carries `AU-NSW` alongside New South Wales. One data row legitimately colours every feature holding the key, and `sharedKeys` says so, which is why 4 rows can light up 7 shapes.
- **Registry packs repair Natural Earth's `-99`.** Natural Earth publishes `ISO_A3` as `-99` for France and Norway (their overseas parts are separate features). The built-in packs repair the key (`ISO_A3`, then `ISO_A3_EH`, then a user-assigned code in common use, then `ADM0_A3`), so joining `world/countries` on `iso_a3` does not silently lose two countries. Raw Natural Earth GeoJSON you load yourself does not get this repair.

`map.diagnoseJoin(seriesIndex = 0)` returns the `JoinResult` on demand (or `null`): `{ matched, totalData, totalFeatures, unmatchedData, unmatchedFeatures, sharedKeys, applied, geoKeyField, dataKeyField, report() }`. Each unmatched row carries up to 3 scored suggestions (`reason`: `'alias'`, `'normalized'`, `'padded'`, or `'similar'` at edit-distance similarity of 0.62 or better). `report()` is the human-readable version:

```
join: 3/6 data rows matched 3/177 features (geometry key "name", data key "name")
  3 data row(s) did not match geometry:
    "Ivory Coast" -> did you mean "Côte d'Ivoire"?
    "United States" -> did you mean "United States of America"?
    "Democratic Republic of the Congo" -> did you mean "Dem. Rep. Congo"?
  174 feature(s) had no data (rendered as no-data): Fiji, Tanzania, W. Sahara, ...
```

The same report prints to the console automatically in dev mode (localhost or `file:`, or `debug: { enabled: true }`) whenever rows went unmatched, repairs were applied, or nothing matched. `debug: { joinDiagnostics: false }` silences it. A series with a `drilldown` is expected to have unmatched rows (they belong to other levels), and the diagnostic says so.

## Bubble series

```js
{
  type: 'bubble',
  name: 'Metro population',
  data: metros,                        // { name, lon, lat, value }
  size: { scale: 'sqrt', range: [4, 34] },
  color: '#d0303f',
  opacity: 0.62,                       // overlap has to stay readable
  stroke: { color: '#fff', width: 1 },
}
```

- Datum: `{ lon?, lat?, lng?, value?, name?, ...extra }`. `lng` is an alias for `lon`.
- **Centroid fallback**: explicit `lon`/`lat` wins; otherwise the datum is joined to a feature via `joinBy` (with optional `fuzzyJoin`) and placed at its centroid, so "revenue by country" needs no coordinates at all. Centroids are computed in lon/lat, so bubbles survive a projection change.
- `size`: `{ field?, scale?, range?, domain? }`. `field` defaults to the series' `valueField`. `scale: 'sqrt'` is the default on purpose: radius proportional to value makes a circle's *area* grow with the square of it, overstating large values. `'linear'` and `'log'` exist; linear warns in dev mode. `range` is `[minRadius, maxRadius]` in screen pixels; the default range derives from the plot size. Radius holds through zoom, because it encodes the value.
- Second encoding: `colorField: 'growth'` plus `colorScale: { palette: 'oranges', classes: 5 }` colours bubbles by another variable; plain `color` is a single fill.
- `sortBySize: true` (default) paints largest first so small bubbles stay on top and clickable. Turning it off is almost always a mistake.

## Marker series

```js
{
  type: 'marker',
  data: capitals,            // { name, lon, lat, bloc }
  shape: 'pin',              // anchored at its point, not its centre
  size: 15,
  colorBy: 'bloc',           // categorical colour and a legend, automatically
  palette: 'capitals',       // a registered palette name
  cluster: { radius: 34, minPoints: 3 },
}
```

- Datum: `{ lon?, lat?, lng?, name?, value?, category?, shape?, color?, size?, ...extra }`. Coordinates or `joinBy`-to-centroid, exactly as bubbles. `category` groups markers for colouring and the legend; `shape`, `color`, `size` are per-point overrides.
- Seven `MarkerShape` values: `circle`, `square`, `diamond`, `triangle`, `star`, `cross`, `pin`. Each is a generated path (no sprite sheet, no CORS). `pin` is the one shape anchored at its point rather than its centre.
- Series `shape` takes a value or `(datum) => MarkerShape`. Series `size` (default 10, the width of the shape's bounding box in px) is fixed on purpose: a marker says "something is here"; size that encodes a quantity is the bubble series.
- `colorBy` names the category field; `palette` names the categorical palette.

`cluster` (licensed; watermark without a key) merges piled-up points. It is an option on the marker series, never a separate series type: the data is identical either way.

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `true` when a `cluster` object is present | |
| `radius` | 60 | Screen-space merge distance, px |
| `maxZoom` | 8 | Above this zoom, individual markers |
| `minPoints` | 2 | Fewer members stay individual markers |
| `size` | | `[min, max]` px radius for the cluster circle, smallest to largest count |
| `showCount` | `true` | Member count inside the circle |
| `zoomOnClick` | `true` | Fly to the members' bounds on click (`clusterClick` emits first) |

## Arc series

```js
{
  type: 'arc',
  name: 'Great-circle distance',
  data: routes,                    // { from, to, value }
  geodesic: true,                  // default
  width: { range: [0.6, 3.4] },    // value drives width
  endpoints: { show: true, radius: 2 },
  flow: true,                      // beads travel from -> to
}
```

- Datum: `{ from, to, value?, name?, id?, ...extra }`. **`from` and `to` are required.** Each is a `[lon, lat]` pair or a geometry-key string resolved against the current map and placed at that feature's centroid, so `{ from: 'SGP', to: 'FRA' }` works on `world/countries` (key `iso_a3`); `joinBy` controls the field strings resolve against. A datum whose endpoint resolves to nothing is dropped and counted in a dev warning: "use [lon, lat] pairs, or geometry keys that exist in the current map".
- `geodesic: true` (default) follows the great circle and is cut at the antimeridian, so a trans-Pacific route re-enters on the other edge instead of streaking across the map. `geodesic: false` is a naive chord through screen space.
- `curvature` (0 to about 1) bulges the arc perpendicular to its chord. Decorative: a curved arc is no longer the true path, so it is off by default and warns in dev mode.
- `width` is the same `SizeOptions` shape bubbles use, driven by `value`; for line width the default scale is `'linear'` (width reads linearly, unlike circle area). `colorScale` colours by value; `color` is a single stroke.
- `endpoints: { show?, radius?, color? }` draws a screen-space dot at each end that does not grow with zoom.

`flow` sends beads along each route so the connection reads as a direction, from `from` towards `to`. `flow: true` takes every default; or pass options:

| Option | Default | Meaning |
|---|---|---|
| `style` | `'dots'` | `'dash'` marches a dashed highlight instead |
| `scale` | `'zoom'` | Beads anchor to the ground and spread with zoom; `'screen'` holds size and spacing fixed |
| `speed` | 90 | Screen px/s at the opening zoom; bounded at twice that under `'zoom'` |
| `spacing` | 56 | Px between beads at the opening zoom; bounded at six times; a route shorter than the spacing carries at most one bead |
| `size` | route width | Bead diameter or dash weight, px; bounded at three times |
| `color` | route colour | |
| `opacity` | 1 | |
| `stagger` | `true` | Offsets each route's phase, so parallel routes read as traffic rather than one synchronised pulse |

Under `prefers-reduced-motion`, with `chart.animations.enabled: false`, or past 600 routes on one series, the beads stay in place and stop travelling: a dotted route still reads as a route.

## Line series

```js
{
  type: 'line',
  name: 'Trade routes',
  data: [{
    name: 'Transpacific (sea)',
    value: 78,
    path: [
      [139.7, 35.6],    // Tokyo Bay
      [160.0, 43.0],
      [-170.0, 50.0],   // crosses the antimeridian: cut at the map edge
      [-140.0, 45.0],
      [-122.4, 37.8],   // San Francisco
    ],
  }],
  width: { range: [1, 5] },
  endpoints: { show: true, radius: 2.5 },
}
```

- Datum: `{ path?, coordinates?, id?, name?, value?, color?, ...extra }`. `path` is the vertex sequence the route passes through, in order; `coordinates` is accepted as a synonym because the data often arrives as GeoJSON. `color` is a per-route override of the series colour.
- Unlike an arc, which derives the great circle between two endpoints, the caller supplies the whole path: a GPS trace, a shipping lane, a transit line. Segments between vertices still follow the sphere and are cut at the antimeridian.
- `width` (driven by `value`), `color`, `colorScale`, `endpoints`, and `flow` behave exactly as on the arc series; flow beads travel in the order the vertices were given.
