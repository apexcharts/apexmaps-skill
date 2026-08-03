# ApexMaps AI Skill

AI coding skill for building [ApexMaps](https://apexcharts.com/docs/apexmaps/) geographic data visualizations. Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant that can read project context.

> **Separate skill, one of the ApexCharts ecosystem skills.** This is the dedicated skill for **ApexMaps** (`apexmaps`), shipped as its own `apexmaps-skill` package and repo, distinct from the core `apexcharts-skill` and the other product skills. Each product has its own library and skill; use the one that matches yours:
>
> | Product | npm library | Skill package & repo |
> |---|---|---|
> | ApexCharts, charts | `apexcharts` | [`apexcharts-skill`](https://github.com/apexcharts/apexcharts-skill) |
> | ApexGantt, Gantt / timeline | `apexgantt` | [`apexgantt-skill`](https://github.com/apexcharts/apexgantt-skill) |
> | ApexTree, hierarchy / org charts | `apextree` | [`apextree-skill`](https://github.com/apexcharts/apextree-skill) |
> | ApexSankey, flow / Sankey | `apexsankey` | [`apexsankey-skill`](https://github.com/apexcharts/apexsankey-skill) |
> | Apex Grid, data grid | `apex-grid` | [`apexgrid-skill`](https://github.com/apexcharts/apexgrid-skill) |
> | ApexStock, financial / stock | `apexstock` | [`apexstock-skill`](https://github.com/apexcharts/apexstock-skill) |
> | **ApexMaps**, geographic / choropleth · *this skill* | `apexmaps` | `apexmaps-skill` |

## What This Does

AI models routinely get map code wrong: swapping `[lon, lat]` into `[lat, lon]`, hunting for GeoJSON files the geometry registry already ships, joining US states on full names when the pack key is the postal code, coloring choropleths by raw counts instead of rates, or treating clustering as a separate series type. This skill ships structured reference files so the assistant generates correct ApexMaps code on the first try.

### Coverage

- **The five series types**: choropleth (default), bubble, marker, arc, line, and each one's datum shape
- **Data joins**: `joinBy` forms, key auto-detection, `fuzzyJoin`, the join diagnostic, FIPS repair
- **The geometry registry**: 26 built-in packs (world, US states/counties, EU NUTS 0-3, admin-1 for 15 countries), aliases, recommended join keys
- **Projections**: 16 built-ins (20 accepted names with aliases), spec objects, per-pack defaults like `albersUsa`, `registerProjection`
- **Scales and palettes**: quantile / Jenks / threshold and more, 17 palettes, automatic diverging selection
- **Interaction**: camera moves (`flyTo`, `fitBounds`), selection and linked maps, drilldown, marker clustering, globe rotation
- **Styling**: legends, tooltips, data labels, pattern and image fills, annotations, theming via `--apexmaps-*` tokens
- **Licensing**: which features are free vs licensed, and the evaluate-with-watermark model
- **Framework wrappers**: `react-apexmaps`, `vue-apexmaps`, `ngx-apexmaps`

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexmaps-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexmaps-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### Generic AI Assistant

Paste the contents of `SKILL.md` into the system prompt or attach it as context.

### As an npm dependency

For tools that build on top of this skill (MCP servers, custom AI agents):

```bash
npm install apexmaps-skill
```

```js
import { skillFile, referencesDir, referencePath } from 'apexmaps-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const geo = await readFile(referencePath('geo-and-projections.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                            # Main entry point, read this first
├── .cursorrules                        # Self-contained version for Cursor / Windsurf
├── references/
│   ├── data-format.md                  # datum shapes per series type, joins, normalizeBy
│   ├── geo-and-projections.md          # registry packs, custom geometry, projections, camera
│   ├── styling-and-interaction.md      # scales, legends, selection, drilldown, theming, licensing
│   └── framework-wrappers.md           # React, Vue, Angular
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [ApexMaps Documentation](https://apexcharts.com/docs/apexmaps/)
- [ApexMaps GitHub](https://github.com/apexcharts/apexmaps)
- [npm: apexmaps](https://www.npmjs.com/package/apexmaps)
- [react-apexmaps](https://www.npmjs.com/package/react-apexmaps)
- [vue-apexmaps](https://www.npmjs.com/package/vue-apexmaps)
- [ngx-apexmaps](https://www.npmjs.com/package/ngx-apexmaps)

## License

MIT
