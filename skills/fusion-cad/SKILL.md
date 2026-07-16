---
name: fusion-cad
description: "Use this skill whenever the user wants to do CAD work through the Autodesk Fusion MCP (fusion_mcp_execute, fusion_mcp_read, fusion_mcp_update, fusion_mcp_electronics_read). Covers parametric constrained sketches, extrudes, holes, fillets, multi-body modeling, assemblies and components, as-built joints, motion (hinges, sliders), print-in-place mechanisms and print orientation, exports to STL/3MF/STEP, undo/redo, screenshots, the apiDocumentation lookup pattern, explicit camera control, and current API-stability guidance from the local Fusion API corpus. Trigger on any mention of Fusion 360, the Fusion MCP, .f3d/.f3z/.step files, parametric CAD design, building or editing 3D models programmatically, sketches, extrudes, fillets, counterbore holes, components, joints, hinges, sliders, assemblies, print-in-place parts, print orientation, or when the user asks to model a bracket, tray, mount, panel, organizer, rack mount, or any functional 3D-printed product. Use even if the user does not name the MCP explicitly: if they ask Claude to design something in Fusion or build a part, this skill applies."
---

# Fusion 360 MCP

> Created from production Fusion 360 MCP sessions plus a dedicated stress test. Free to use and modify.

This skill teaches Claude to drive the Autodesk Fusion MCP server safely and efficiently. The MCP exposes four tools that wrap the Fusion Python API:

- `fusion_mcp_execute` runs Python in the active document, or does file ops
- `fusion_mcp_read` does screenshots, API docs lookup, document and project queries
- `fusion_mcp_update` does undo/redo
- `fusion_mcp_electronics_read` reads Schematic / Board / Library data from an active Electronics design (read-only; Autodesk tags the underlying Electronics API surface as Preview, so do not depend on it in distributed workflows without explicit preview opt-in per the rule below)

Patterns in this skill were verified across multiple real product design sessions plus a dedicated stress test of the untested MCP surfaces. Alternatives have specific failure modes documented in `gotchas.md`.

## Decision principles

These six rules sit above any individual snippet. Follow them and most of the gotchas in `gotchas.md` never trigger.

1. **Parametric first.** Every dimension that might change goes in `userParameters`. Hardcoded numbers in sketches are forbidden. Re-running the script should be safe.
2. **One feature per `execute` call** during build. Faster debugging, cleaner timeline, scoped failures.
3. **Name every entity.** Sketches, features, bodies, construction planes. Required for the targeted delete-and-rebuild pattern (`patterns.md` section 19) and for timeline readability when something breaks.
4. **Assert before extrude.** `assert sk.profiles.count == <expected>` immediately after `sk.isComputeDeferred = False`. Profile count is the only signal for self-intersection (returns 2, not an error).
5. **Verify with bounding box AND volume.** Screenshots can deceive about orientation; `body.boundingBox` per axis is ground truth. `body.volume` (returns cm^3) is a fast spec-vs-actual cross-check. Pair them after every body-adding feature. Volume is a stronger check than it looks: across 35+ features on a real part it tracked intended geometry (`intended_volume - overlap_with_prior_features`) to under 1 mm^3 every time, catching missed cuts and wrong `participantBodies` that screenshots hid (`patterns.md` section 24).
6. **Constrain sketches with the corner-anchor mechanism.** Build rectangles via `addCenterPointRectangle` plus explicit horizontal/vertical constraints and positive distance dimensions to a SPECIFIC corner (`patterns.md` section 2). Avoid diagonal-midpoint and symmetric-axis centering (section 43); they leave the UI under-constrained even when the API disagrees. Confirm with the lock badge in the browser, not `isFullyConstrained` alone, which can read True while the sketch is still under-constrained (`gotchas.md`).
7. **Treat Preview API badges as release gates.** If the local API corpus page says Preview, do not use it in a public/distributed workflow unless the user explicitly opts into preview risk. As of the 2026-05-31 corpus review: ray-mesh and curved-face construction planes are stable; UCS, Electronics, AssemblyConstraint, and VolumetricModel pages are Preview.

## When to reach for each tool

| Tool | When to use |
|---|---|
| `execute` (script) | Any model change. Param adds, sketches, extrudes, fillets, hole features, edits to existing features, body cleanup. About 80% of calls. |
| `read` (screenshot) | Verifying geometry after a non-trivial change. ALWAYS run the explicit fit-view block from `patterns.md` section 20 before screenshotting; named directions do not auto-fit and even `direction: "current"` is unreliable on Untitled docs. |
| `read` (apiDocumentation) | BEFORE writing a script that uses an unfamiliar method. Always set `apiCategory`. Omitting it returns success with empty data (silent failure). |
| `read` (document, projects) | Listing open or recent docs, searching by name, getting project IDs. Use `search` (fuzzy, cross-project) when looking for a specific design. |
| `update` (undo, redo) | Last resort. Undo treats the prior `execute` as ONE atomic transaction; if it added params AND geometry, undo wipes both silently. Prefer the delete-loop cleanup pattern in `patterns.md`. |
| `execute` (document) | File ops: open, save, close. NEVER call without explicit user instruction. Save on Untitled docs is refused by the MCP; initial SaveAs must happen in the Fusion UI. Close on dirty docs requires `userConfirmedSaveAndClose` or `userConfirmedCloseWithoutSave`; surface the choice to the user. |

## Script anatomy

Every `execute` script body must use this shape. No other entry point works.

```python
import adsk.core, adsk.fusion

def run(_context: str):
    app = adsk.core.Application.get()
    design = adsk.fusion.Design.cast(app.activeProduct)
    root = design.rootComponent
    # ... work here ...
    print(f"summary: bodies={root.bRepBodies.count}")  # tool returns this as message
```

Rules:

1. Define exactly `def run(_context: str):`. No other name works.
2. Do NOT wrap `run()` in `try/except`. Exceptions return as the tool error with full traceback. Catching kills the traceback and turns a 30-second fix into a 10-minute debug.
3. Use `print()` for any data you need back. Tool result `message` captures stdout.
4. Print the bounding box after any extrude. Screenshots can deceive on orientation; bounding box does not.

## Default workflow for new geometry

1. If the doc state is uncertain, run a tiny read script first: print `bRepBodies.count`, `sketches.count`, `features.count`, `design.unitsManager.defaultLengthUnits`.
2. Add user parameters first, idempotently. See `patterns.md`, "Idempotent user-parameter add". Reference them in both feature inputs (extrude distances, fillet radii, plane offsets) AND in sketch dimensions as needed — parametric sketches work (verified 2026-05-31 via live round-trip in V3). A lock badge may not appear on a script-built sketch until you enter edit mode once; the constraints are fine regardless, see `gotchas.md` G9 lock-badge display lag.
3. Build geometry one feature per `execute` call. Name every feature (`feat.name = 'main_solid'`). Makes the timeline readable when something goes wrong, and lets `patterns.md` section 19 (targeted delete-and-rebuild) work later.
4. Assert profile counts before extruding. Self-intersecting polygons return `profiles.count == 2`; closed simple polygons return `1`; broken polygons return `0`.
5. Print the bounding box to confirm orientation matches intent.
6. Verify visually: run the fit-view block from `patterns.md` section 20, then screenshot with `direction: "current"`.

## Default workflow for unfamiliar APIs

Before writing a script that touches `HoleFeatureInput`, `ExportManager`, `RevolveFeatures`, `LoftFeatureInput`, or any method whose signature is not memorized:

1. If working from this repo, check `src/api-reference/pages/<Object>_<member>.md` first; it contains the scraped Fusion API page, including Preview and Introduced metadata.
2. In a shipped plugin session, call `fusion_mcp_read` with `queryType: "apiDocumentation"`, `apiCategory: "member"`, and the method name as `searchPattern`.
3. Read the signature, arg types, and Preview status.
4. Then write the script. If the page is Preview, ask before using it.

One read call is cheaper than a failed execute round-trip plus debugging.

## Top gotchas

Full catalog in `gotchas.md`. These are the ones that bite first:

0. **Lock-badge display lag on script-built sketches (G9).** A sketch built by a script can be fully constrained AND have a working parametric link to user parameters, while its browser-tree icon shows no lock badge until you enter edit mode once. `isFullyConstrained` is honest; parametric expressions on sketch dims work. Trust the API flag plus a parametric round-trip, not the badge. See `gotchas.md` G9. This entry replaces an earlier (wrong) version that recommended using only numeric mm literals; corrected `discovery-2026-05-31-constraints-corrected.md` supersedes the 2026-05-30 doc.

1. **Sketch plane orientation.** XY plane is the safe default (extrude up world Z). XZ plane maps sketch Y to NEGATIVE world Z; negate Y in vertex lists for Z-up geometry. YZ plane extrudes along world X and INVERTS sketch X to negative world Z (sketch Y maps to positive world Y); see `gotchas.md` for the exact verified mapping. For any non-XY plane, drop one test `sketchPoint` and read `worldGeometry` before committing coordinates. When in doubt, sketch on XY.

2. **`profiles.count == 2` after closing a 4+ vertex polygon means self-intersection.** The polygon looks like a figure-8 internally. Always `assert sk.profiles.count == <expected>` before extruding.

3. **Undo is dangerous on mixed-content scripts.** It wipes the last `execute` as one atomic transaction. If that script added params AND geometry, both vanish silently. Make scripts idempotent and prefer the delete-loop cleanup over undo.

4. **`apiDocumentation` with no `apiCategory` returns empty silently.** Always set it. Use `"member"` for one function, `"class"` for exploration, `"all"` when unsure.

5. **Internal geometry is always cm, not mm.** `userParameters.itemByName(name).value` returns cm. `Point3D.create(x, y, z)` takes cm. Pass `ValueInput.createByString('30 mm')` for human-readable expressions; Fusion handles the conversion. Only multiply by 10 when PRINTING dimensions for display.

6. **Screenshots need an explicit fit-view first.** Use `patterns.md` section 20 (`vp.fit()` + orientation set + `vp.refresh()`) before EVERY screenshot. `direction: "current"` auto-fits in most cases but is unreliable on Untitled docs and right after script-driven feature additions. Named directions never auto-fit on their own.

7. **Cut through unknown depth: use `setAllExtent(direction)`, not fixed distance.** `setDistanceExtent` stops short when gussets, dividers, or flanges sit below the sketch plane. `setAllExtent(NegativeExtentDirection)` cuts through everything in that direction. Valid for cut and intersect operations only.

8. **Save on Untitled docs fails via MCP.** Initial SaveAs must happen in the Fusion UI. After that, MCP `save` works for revisions. Surface this when the user asks Claude to save a brand-new doc.

9. **Close on dirty docs requires explicit confirmation flag.** Never auto-pick `userConfirmedSaveAndClose` vs `userConfirmedCloseWithoutSave`. Ask the user.

10. **Export paths must be absolute with forward slashes; relative paths fail.** Always `os.makedirs(dirname, exist_ok=True)` before export. Overwrite is silent.

## Reference files

Load on demand:

- `patterns.md` reusable Python snippets: params, constrained sketches (corner-anchored rectangle, ellipse, circle, yZ-plane, edge-anchored, re-anchor a pinned entity), extrudes, cuts (multi-profile, setAllExtent, symmetric, tapered, multi-body via participantBodies), OffsetStartDefinition, planes, fillets (UI selection + edit + edge-find by geometry), Mirror replication, composite-via-Join, hole API (simple, counterbore, countersink), exports (STL, 3MF, STEP), idempotent rebuild loop, bounding box check, screenshot defaults, document search, partial-height interior features, targeted delete-and-rebuild, explicit camera fit-view, viewport-to-PNG, doc state sanity-check, stepped solid via Join extrude, volume sanity check, print resolved param values, components and as-built joints (hinge, slider), joint limits, print-in-place travel stop, assembly reorientation for print, and sketch constraint anti-patterns.
- `gotchas.md` full failure mode catalog with reproductions and fixes.
- `api-reference/` repo-local scraped API corpus for development and review only. It is not copied into the public plugin artifact and must not be redistributed as Autodesk documentation content.

## Quick reference: complete first-call template

For a brand-new design, this is the canonical opening script:

```python
import adsk.core, adsk.fusion

def run(_context: str):
    app = adsk.core.Application.get()
    design = adsk.fusion.Design.cast(app.activeProduct)
    root = design.rootComponent

    # 1. Sanity-check doc state
    print(f"bodies={root.bRepBodies.count} sketches={root.sketches.count} "
          f"features={root.features.count} units={design.unitsManager.defaultLengthUnits}")

    # 2. Add params idempotently
    param_defs = [
        ('length', '100 mm', 'mm', ''),
        ('width',  '50 mm',  'mm', ''),
        ('height', '20 mm',  'mm', ''),
    ]
    params = design.userParameters
    for name, expr, unit, comment in param_defs:
        if not params.itemByName(name):
            params.add(name, adsk.core.ValueInput.createByString(expr), unit, comment)

    # 3. Constrained sketch (corner-anchored, see patterns.md section 2) + extrude
    P = adsk.core.Point3D.create
    DO = adsk.fusion.DimensionOrientations
    sk = root.sketches.add(root.xYConstructionPlane)
    sk.name = 'outer'
    sk.isComputeDeferred = True
    rect = sk.sketchCurves.sketchLines.addCenterPointRectangle(
        P(0, 0, 0), P(2.5, 1.25, 0))   # initial size; dimensions drive the final size
    L_top, L_left, L_bot, L_right = rect.item(0), rect.item(1), rect.item(2), rect.item(3)
    sk.isComputeDeferred = False

    # find SW corner (shared by left + bottom edges)
    sw_pt = None
    for sp in [L_left.startSketchPoint, L_left.endSketchPoint]:
        for sp2 in [L_bot.startSketchPoint, L_bot.endSketchPoint]:
            if sp.geometry.x == sp2.geometry.x and sp.geometry.y == sp2.geometry.y:
                sw_pt = sp; break
        if sw_pt: break

    gc = sk.geometricConstraints
    gc.addHorizontal(L_top); gc.addHorizontal(L_bot)
    gc.addVertical(L_left);  gc.addVertical(L_right)

    sd = sk.sketchDimensions
    dim_w = sd.addDistanceDimension(L_top.startSketchPoint, L_top.endSketchPoint,
        DO.HorizontalDimensionOrientation, P(0, 2, 0))
    dim_d = sd.addDistanceDimension(L_left.startSketchPoint, L_left.endSketchPoint,
        DO.VerticalDimensionOrientation, P(-3, 0, 0))
    dim_x = sd.addDistanceDimension(sk.originPoint, sw_pt,
        DO.HorizontalDimensionOrientation, P(-2, -2, 0))
    dim_y = sd.addDistanceDimension(sk.originPoint, sw_pt,
        DO.VerticalDimensionOrientation, P(-3, -1, 0))
    # Parametric sketch dims: parameter-name expressions work and propagate when the param changes.
    # If you want a baked-in literal, write "70 mm" instead. See gotchas.md G9 for the
    # lock-badge-display-lag gotcha (cosmetic; constraints are real either way).
    dim_x.parameter.expression = 'length / 2'
    dim_y.parameter.expression = 'width / 2'
    dim_w.parameter.expression = 'length'
    dim_d.parameter.expression = 'width'
    assert sk.isFullyConstrained
    assert sk.profiles.count == 1, f"expected 1 profile, got {sk.profiles.count}"

    extrudes = root.features.extrudeFeatures
    ext_in = extrudes.createInput(sk.profiles.item(0),
        adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
    ext_in.setDistanceExtent(False, adsk.core.ValueInput.createByString('height'))
    feat = extrudes.add(ext_in)
    feat.name = 'body_solid'

    # 4. Bounding box check
    body = feat.bodies.item(0)
    body.name = 'main'
    bb = body.boundingBox
    print(f"bbox X:{bb.minPoint.x*10:.2f} to {bb.maxPoint.x*10:.2f} mm")
    print(f"bbox Y:{bb.minPoint.y*10:.2f} to {bb.maxPoint.y*10:.2f} mm")
    print(f"bbox Z:{bb.minPoint.z*10:.2f} to {bb.maxPoint.z*10:.2f} mm")
```

Use this as the scaffold for any new design. Replace param defs, sketch geometry, and feature operations as needed.
