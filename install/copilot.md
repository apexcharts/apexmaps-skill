# Installing ApexMaps Skill for GitHub Copilot

## VS Code + Copilot Chat

### Option 1: Add as Workspace Context

1. Clone this repository or download `SKILL.md` into your project
2. In Copilot Chat, reference the file: `@workspace #file:SKILL.md`
3. Ask your map question

### Option 2: Custom Instructions

1. Open VS Code Settings (Cmd/Ctrl + ,)
2. Search for "Copilot Custom Instructions"
3. Paste the contents of `.cursorrules` into the custom instructions field

## Copilot in GitHub.com

When working in GitHub's web editor with Copilot, paste the relevant sections from `SKILL.md` into your prompt for accurate ApexMaps code generation.

## Verification

Ask Copilot to generate a flight-route map with arcs between airports and a marker series with clustering. It should use `type: 'arc'` data with required `from` / `to` endpoints (each `[lon, lat]` or a geometry key), enable clustering as an option on the marker series rather than a separate series type, and await `map.render()`.
