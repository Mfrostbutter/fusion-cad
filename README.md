# fusion360-mcp

A Claude Code plugin for driving the [Autodesk Fusion 360 MCP](https://www.autodesk.com/products/fusion-360/) server reliably. Built from real production CAD sessions plus a dedicated stress test of the untested MCP surfaces.

## What it covers

- Parametric constrained sketches (corner-anchored rectangles, ellipses, circles, off-plane and re-anchored entities)
- Parameter management (idempotent adds, reserved math-name guards, resolved-value printing)
- Hole features (simple, counterbore, countersink) via `HoleFeatureInput`
- Fillets and chamfers, including in-place radius edits and edge-finding by geometry
- Construction planes, symmetric / tapered / offset-start extrudes
- Multi-profile cuts, through-cuts of unknown depth, and multi-body cuts
- Multi-body modeling, composite bodies, and Mirror replication
- Components, as-built joints (revolute hinges, sliders), joint limits, motion drive, and assembly reorientation for print
- Print-in-place mechanisms with print-orientation guidance
- Exports to STL, 3MF, and STEP with verified working signatures
- Undo and redo with documented atomicity behavior
- Explicit camera fit-view from script for deterministic screenshots, plus viewport-to-PNG
- Bounding-box and volume verification, partial-height interior features, targeted delete-and-rebuild
- `apiDocumentation` lookup pattern for unfamiliar signatures
- 43 copy-ready patterns and a catalog of 39 failure modes with reproductions and fixes

## Requirements

- Claude Code installed and authenticated
- The Autodesk Fusion MCP server connected (the plugin uses `fusion_mcp_execute`, `fusion_mcp_read`, `fusion_mcp_update`)
- Fusion 360 running with an active document

## Installation

### Local testing (during development)

Clone or download this plugin, then point Claude Code at the local folder:

```bash
claude --plugin-dir /path/to/this/folder
```

Then run `/reload-plugins` inside Claude Code to pick up changes.

### Via the community marketplace (after acceptance)

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install fusion360-mcp@claude-community
```

## How it triggers

The skill is model-invoked. Just talk to Claude about CAD work; the description triggers it automatically. No slash command needed for normal use.

Example prompts that activate the skill:

- "Build me a parametric mounting bracket in Fusion"
- "Add a counterbore to the top face of this part"
- "Convert these bodies to components and add a revolute hinge joint"
- "Lay out a print-in-place box with a captured lid, oriented for the bed"
- "Export the current body to STL"
- "Why does my polygon return profiles.count of 2?"

If you want to invoke it explicitly:

```
/fusion360-mcp:fusion-cad
```

## Files

```
fusion360-mcp/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── fusion-cad/
│       ├── SKILL.md        trigger surface + decision rules + top gotchas
│       ├── patterns.md     43 copy-ready Python snippets
│       └── gotchas.md      39 documented failure modes and fixes
├── LICENSE
└── README.md
```

## Validate before submitting changes

```bash
claude plugin validate .
```

## License

MIT. See [LICENSE](LICENSE).

## Acknowledgments

The patterns in this plugin came from production CAD sessions building real 3D-printed products, supplemented by a stress test of edge cases (screenshot direction behavior, save/close on Untitled docs, export path requirements, apiDocumentation silent failures, camera control). Contributions and corrections welcome via pull request.
