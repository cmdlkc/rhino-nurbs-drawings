# rhino-nurbs-drawings

An Agent Skill for Claude Code that makes the agent draw 2D linework directly in Rhino by writing scripts, as true NURBS curves with layers, line weights, hatches and vector export. It is for any line drawing: building elevations, sections and plans, objects and products, diagrams, illustration, botanical studies, patterns, ornament.

It makes the drawing only, not a presentation sheet: no border, title block or scale bar unless you ask. It is a drawing tool, not a modelling tool. It does not build a 3D model and does not use Make2D or hidden-line projection, whose output arrives fragmented and needs cleaning. For 3D modelling see the companion skill, [rhino-nurbs-modelling](https://github.com/cmdlkc/rhino-nurbs-modelling).

It exists because agents connected to Rhino through tool-based MCP servers tend to produce only primitives such as circles and rectangles. The skill removes that shortcut and gives the agent tested helper code, a curve toolkit, numeric checks and a review routine.

## How it draws

1. Picks a route into Rhino and says which one it is using.
2. Researches the subject: sources with dimensions, the most frontal reference image, close-ups of details.
3. Writes an intent file: view, units and scale, setting-out as named numbers, depth bands from back to front, and every element with the rule that generates its curves.
4. Draws on the World XY plane in short Python passes, one script each, back to front.
5. Handles overlap by construction: anything passing behind a nearer closed outline is clipped, and the depth band sets the line weight.
6. Fills cut material and closed regions with hatches, then exports the drawing as PDF, SVG and DWG on a page sized to its extents.
7. Accepts a pass only when its numeric checks pass (flat, closed, no duplicates, counts and extents as expected), then looks at a capture.

The skill includes helpers for tapering ribbons, clipping, fills, layers and checks, and recipes for whiplash stems, branching, tweened line fields, offsets and mirrored or repeated units.

## Requirements

- Claude Code, signed in.
- Rhino 9 or Rhino 8 on Windows, with one of:
  - McNeel's Rhino MCP server. It is built into Rhino 9. On Rhino 8, install `Rhino-MCP-Platform` from the Package Manager. Start it with the `MCPConnect` command and follow the prompt it gives for Claude Code.
  - The `rhinocode` command line tool (Rhino 8.11 or later), after running `StartScriptServer` in Rhino.
- Without Rhino, the skill falls back to ezdxf in a Python virtual environment.

To make the McNeel connector target Rhino 9 from Claude Code, the `rhino` entry under `mcpServers` in `~/.claude.json` needs `"args": ["-v", "9"]`. Without it the connector starts Rhino 8.

## Install

Clone the repository into your personal skills folder, so that `SKILL.md` sits at `~/.claude/skills/rhino-nurbs-drawings/SKILL.md`.

macOS and Linux:

```bash
git clone https://github.com/cmdlkc/rhino-nurbs-drawings ~/.claude/skills/rhino-nurbs-drawings
```

Windows (PowerShell):

```powershell
git clone https://github.com/cmdlkc/rhino-nurbs-drawings "$env:USERPROFILE\.claude\skills\rhino-nurbs-drawings"
```

Then run `/skills` in Claude Code to confirm it is listed.

## Use

```
/rhino-nurbs-drawings Draw the street elevation of a three-bay, four-storey Art Nouveau facade at 1:50: whiplash ironwork balconies, curved window heads, stems rising from each pier. Bay width 3.2 m, storey height 3.4 m.
```

```
/rhino-nurbs-drawings Draw a Thonet No. 14 chair in side view at full size, with the bentwood curves as single tapering outlines.
```

Results are strongest when you give the agent master curves, a setting-out or a reference to elaborate. It executes curves well and judges composition poorly.

Work in a new document in millimetres with a tolerance of 0.01 or finer. The skill tells the agent to keep to its own layer tree and to leave existing objects alone, but a separate file is the safer habit.

## What has been checked

- Every helper and call in `SKILL.md` ran on Rhino 9.0.26300 (Python 3.13.13) on 23 November 2026, including viewport capture and PDF, SVG and DWG export; export at 1:50 measured true in the PDF.
- The ezdxf fallback ran on Linux with ezdxf 1.4.5.

Not yet run: the export page sized from the drawing extents (it replaced a fixed sheet on 23 November 2026), and the `rhinocode` route on Rhino 9. The skill has the agent measure a known dimension in the PDF before stating a scale.

## Limits

- No published study evaluates agents on detailed 2D line drawings. The method transfers evidence from 3D CAD research.
- Agents judge their own drawings poorly by eye. The numeric checks and your own review carry the weight.
- Clipping works against closed outlines. Open or self-crossing elements cannot hide what is behind them.
- Composition and aesthetic decisions stay with you.

## Background

`docs/research-report.md` is the research behind the skill, with sources, source conflicts and unverified items marked, plus an addendum of checks made in Rhino 9. It was written when this skill still derived drawings from a 3D model.

## Provenance and licence

The skill and the report were written with Claude (Anthropic). Released under the MIT licence; see `LICENSE`.
