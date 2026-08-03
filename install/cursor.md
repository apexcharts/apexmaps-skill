# Installing ApexMaps Skill for Cursor

## Setup

1. Copy the `.cursorrules` file from this repository into the root of your project:

```bash
cp /path/to/apexmaps-skill/.cursorrules /path/to/your-project/.cursorrules
```

Or download it directly:

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexmaps-skill/main/.cursorrules
```

2. Restart Cursor or open a new Cursor window.

Cursor automatically reads `.cursorrules` files in the project root and uses them as context for AI-assisted coding.

## For Windsurf

Same approach, Windsurf also supports `.cursorrules` files in the project root.

## Verification

Ask Cursor to generate a world choropleth with a drilldown into US counties. It should use registry pack ids (`world/countries`, `us`, `us/counties`), join data with `joinBy`, await `map.render()`, configure drilldown through the choropleth series (not a separate chart), and know that coordinates are always `[lon, lat]` with longitude first.
