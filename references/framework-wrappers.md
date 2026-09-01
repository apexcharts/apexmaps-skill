# Framework Wrappers: React, Vue, Angular

Official wrappers live in the apexmaps repo (`wrappers/react`, `wrappers/vue`, `wrappers/angular`) and are versioned in step with the core (`0.4.0`, peer dependency `apexmaps ^0.4.0`). Each is built for its own framework's change model rather than adapted from the React one, and all are typed against the core package's own types (`ApexMapsOptions`, `Series`, `ApexMapsEventMap`), so there is no second options schema to learn. A Svelte wrapper is planned but does not exist yet: in Svelte, use the core API directly.

## Which package

| Project uses | Package | Import |
|---|---|---|
| React >= 18 | `react-apexmaps` | `import ApexMaps from 'react-apexmaps'` (default export; also named `ApexMapsReact`) |
| Vue 3 >= 3.3 | `vue-apexmaps` | `import ApexMaps from 'vue-apexmaps'` (default export; also named `ApexMapsVue`) |
| Angular >= 20 (`@angular/core`) | `ngx-apexmaps` | `import { ApexMapsComponent } from 'ngx-apexmaps'`, selector `<apx-map>` |
| Svelte | none yet | Svelte wrapper is planned, not built; use the core API |
| Anything else | `apexmaps` core | `new ApexMaps(el, options)` then `await map.render()` |

No CSS import is needed in any of them: styles are injected by the core (`apexmaps/apexmaps.css` exists for CSP-strict setups only).

## Shared surface

All three wrappers take the same inputs:

| Prop / input | Type | Notes |
|---|---|---|
| `options` | `ApexMapsOptions` | Required. The same options object the core takes. |
| `series` | `Series[]` | Optional shorthand for `options.series`; takes precedence over it. Both routes reach `updateSeries`, so both tween. |
| `width` / `height` | `number \| string` | Shorthand for `options.chart.width` / `options.chart.height`. |

Anything else you pass (`class`/`className`, `style`, `id`, `aria-*`) lands on the outer element the framework owns; the inner element belongs to the map. Size with the `width`/`height` props, not CSS: `chart.height` defaults to `400` and an explicit number wins over the container, while `chart.width` defaults to `'100%'` and follows the container. `height="100%"` plus a styled container hands the height to the container.

All 17 core events are forwarded: `rendered`, `updated`, `resized`, `featureClick`, `featureHover`, `featureFocus`, `markClick`, `markHover`, `clusterClick`, `drilldown`, `drillup`, `selectionChange`, `legendToggle`, `zoom`, `panEnd`, `rotate`, `rotateEnd`. Each handler receives the payload the core emits, fully typed (e.g. `featureClick` gets `{ key, name, value, datum }`). Every wrapper destroys the instance on unmount; you never call `destroy()` yourself.

One core behavior to remember in declarative code: `updateOptions` merges, so removing a key from `options` does not reset it. Pass the off value (`dataLabels: { enabled: false }`) instead of omitting the key.

## React: `react-apexmaps`

```bash
npm install apexmaps react-apexmaps
```

```jsx
import ApexMaps from 'react-apexmaps'

const options = { geo: { map: 'world/countries' } }
const series = [
  {
    type: 'choropleth',
    name: 'Unemployment rate',
    joinBy: ['iso_a3', 'code'],
    data: [
      { code: 'FRA', value: 7.3 },
      { code: 'DEU', value: 5.7 },
      { code: 'ESP', value: null }, // no data: nullColor, not 0
    ],
  },
]

export default function WorldMap() {
  return (
    <ApexMaps
      options={options}
      series={series}
      height={480}
      onFeatureClick={({ key, name, value }) => console.log(key, name, value)}
    />
  )
}
```

- Extra prop: `mapRef` (`{ current: ApexMaps | null }`, e.g. a `useRef(null)`) receives the live instance for the imperative API: `map.current?.frameFeature('IND')`, `camera.flyTo(...)`, `drillTo(...)`, `exportPNG()`, `diagnoseJoin()`. Nulled on unmount.
- Events are props named `on` + capitalized event name: `onRendered` ... `onFeatureClick` ... `onRotateEnd`. Handlers are read at emit time, so inline arrow functions cost nothing and never miss an event.
- Update model: props are compared deeply, so a fresh but equal `options` object on every parent render is not a redraw. Functions (formatters) compare by source, not identity, so a formatter reading changed state through its closure is not seen: lift such values into `options` as data. `geo.map` is compared by identity and never walked: pass a stable reference (module constant, `useMemo`, or a pack id string), or every render reprojects. Series-only changes route to `updateSeries` and tween; anything else goes to `updateOptions`. Camera position, selection, and drilldown depth survive an options change.
- Ships a `'use client'` banner, so it works in Next.js App Router; the module is importable without a DOM but the map is not server rendered (the core has no HTML output path).

## Vue 3: `vue-apexmaps`

```bash
npm install apexmaps vue-apexmaps
```

```vue
<script setup>
import ApexMaps from 'vue-apexmaps'

const options = { geo: { map: 'world/countries' } }
const series = [
  {
    type: 'choropleth',
    name: 'Unemployment rate',
    joinBy: ['iso_a3', 'code'],
    data: [
      { code: 'FRA', value: 7.3 },
      { code: 'DEU', value: 5.7 },
      { code: 'ESP', value: null },
    ],
  },
]
const onClick = ({ key }) => console.log(key)
</script>

<template>
  <ApexMaps :options="options" :series="series" :height="480" @feature-click="onClick" />
</template>
```

- Events are emitted under the core's own camelCase names, so both template spellings work: `@feature-click` and `@featureClick`, `@selection-change`, `@rotate-end`, etc.
- Imperative API through a template ref: the component exposes `map`, so `ref="mapEl"` gives `mapEl.value.map.frameFeature('IND')`, `.exportPNG()`, `.drillTo(...)`, and so on. The instance is held in a `shallowRef`, never a reactive proxy.
- Update model: built for Vue's mutation style. Mutating reactive `options` in place (`options.legend.position = 'top'`) is detected, because the component compares against a structural snapshot of the last applied options, not a reference. No reactive proxy ever reaches the map: options are snapshotted to plain objects and `geo.map` is unwrapped with `toRaw`, so a county topology is not proxied coordinate by coordinate. Still prefer `markRaw` on imported geometry so Vue never proxies it at all. Deep watching stops at the geometry: `geo.map` is watched by reference alone, so rebuilding an equal topology object every render reprojects every time. Series-only changes tween via `updateSeries`; inline formatters compare by source, so template-inline formatters do not cause redraws.

## Angular: `ngx-apexmaps`

```bash
npm install apexmaps ngx-apexmaps
```

```ts
import { Component, signal } from '@angular/core'
import { ApexMapsComponent } from 'ngx-apexmaps'

@Component({
  selector: 'app-world-map',
  standalone: true,
  imports: [ApexMapsComponent],
  template: `
    <apx-map
      [options]="options"
      [series]="series()"
      [height]="480"
      (featureClick)="selected.set($event.key)"
    />
  `,
})
export class WorldMapComponent {
  readonly options = { geo: { map: 'world/countries' } }
  readonly series = signal([
    {
      type: 'choropleth',
      name: 'Unemployment rate',
      joinBy: ['iso_a3', 'code'],
      data: [
        { code: 'FRA', value: 7.3 },
        { code: 'DEU', value: 5.7 },
        { code: 'ESP', value: null },
      ],
    },
  ])
  readonly selected = signal<string | null>(null)
}
```

- Standalone component with signal inputs (`options` is `input.required`), `OnPush`, no NgModule, no rxjs or zone.js requirement of its own. Peer deps: `apexmaps` and `@angular/core >= 20`.
- One `output()` per core event under the core's own name: `(featureClick)`, `(selectionChange)`, `(rotateEnd)`, etc.
- Zone contract: the map is constructed, rendered, and updated outside the Angular zone, so pointermove listeners never trigger change detection; outputs are emitted back inside the zone, so a handler that sets state repaints in zoned and zoneless apps alike (under zoneless, both calls degrade to plain function calls).
- Imperative API: the component exposes the live instance as a `map` signal. With a template ref `#map`, call `map.map()?.frameFeature('IND')` or `map.map()?.exportPNG()`. It is `null` until the first render and after destroy, so `effect()` on it to run something once the map exists.
- Update model: bindings are compared deeply against a snapshot (a template expression like `[options]="build()"` hands over a new object every change detection cycle, so reference inequality cannot mean "changed"). Formatters compare by source; `geo.map` by identity, never walked. Signal inputs do not fire on in-place mutation, so prefer replacing objects; an earlier in-place mutation is picked up when any other input later changes. SSR-safe: the container renders on the server, the map is built in `afterNextRender`, which only runs in the browser.

## Common pitfalls in framework code

| Wrong | Correct |
|---|---|
| Rebuilding `geo.map` as a fresh object every render | Stable reference (module constant, `useMemo`, `markRaw`) or a registry id string like `'world/countries'` |
| Omitting an options key to turn a feature off | Pass the off value: `updateOptions` merges, a missing key resets nothing |
| Sizing with CSS only (`style="height: 600px"`) | Use the `height` prop/input; `chart.height` defaults to 400 and wins over the container. `height="100%"` defers to the container |
| Reading state inside a formatter closure (React/Angular) | Formatters compare by source; move changing values into `options` as data |
| Calling `destroy()` or `render()` yourself through the wrapper | The wrapper owns the lifecycle; use the instance (mapRef / exposed `map`) only for imperative calls like `camera.flyTo`, `frameFeature`, `drillTo`, `exportPNG`, `diagnoseJoin` |
| `ApexMaps.setLicense(key)` per component | Call it once at app startup, on the core `apexmaps` import, before the first render |
| Waiting for a Svelte wrapper | It is planned but not built; use the core constructor + `await map.render()` with `onMount`/destroy semantics |
