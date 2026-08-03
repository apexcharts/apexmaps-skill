# Installing ApexMaps Skill for Claude Code

## Installation

```bash
# Navigate to your project's Claude config
mkdir -p .claude/skills
cd .claude/skills

# Clone the skill
git clone https://github.com/apexcharts/apexmaps-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on ApexMaps code.

## Verification

Ask Claude to build a map:

> Create an ApexMaps choropleth of unemployment by US state, joined on the state postal code, with a quantile scale, then add a bubble series sized by population.

Claude should generate code that:
- Calls `new ApexMaps(el, options)` followed by `await map.render()`
- Uses a registry pack id (`geo: { map: 'us' }`), not a hand-hosted GeoJSON file
- Joins data with `joinBy` (for `us/states` the recommended key is the postal code, e.g. `'CA'`)
- Puts choropleth rows in `series[i].data` as plain objects with a join key and a `value`
- Gives bubble data `{ lon, lat, value }` coordinates (lon first), or a `joinBy` to resolve centroids
- Uses `null`, never `undefined`, for missing values
