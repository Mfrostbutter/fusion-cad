# Fusion MCP Patterns

Copy-ready Python for `fusion_mcp_execute` scripts. Every snippet assumes the standard `run()` wrapper:

```python
import adsk.core, adsk.fusion

def run(_context: str):
    app = adsk.core.Application.get()
    design = adsk.fusion.Design.cast(app.activeProduct)
    root = design.rootComponent
    # paste snippet here
```

Internal units are cm. `ValueInput.createByString('30 mm')` accepts unit suffixes and Fusion converts.

## Table of contents

1. Idempotent user-parameter add
2. Parametric box from a sketched rectangle
3. Construction plane offset from origin plane
4. Multi-profile cut (n cells in one feature)
5. Cut through unknown depth
6. Symmetric extrude (centered on sketch plane)
7. Tapered extrude (legacy countersink approach)
8. Holes via HoleFeatureInput (preferred)
9. Fillet by current UI selection
10. Edit existing fillet radius (no rebuild needed)
11. Read parameter values for sketch math
12. Bounding box sanity check
13. Clean rebuild without losing parameters
14. Export to STL / 3MF / STEP
15. API documentation lookup (before guessing signatures)
16. Screenshot defaults
17. Document search
18. Partial-height interior features via top-shave cut
19. Targeted delete-and-rebuild
20. Force camera fit-view from script
21. Save viewport snapshot to disk
22. Doc state sanity-check (read-only opener)
23. Stepped solid via Join extrude (body + lip flange)
24. Volume sanity check (spec vs actual)
25. Countersink hole via HoleFeatureInput
26. Print all resolved parameter values
27. Constrained parametric ellipse
28. Off-center constrained rectangle (edge-anchored)
29. Constrained parametric circle
30. Constrained rectangle on yZ plane (G5-aware)
31. Re-anchor an origin-pinned sketch entity
32. Edge finding by geometry (fillet/chamfer without UI selection)
33. Replicate features via Mirror (vs Rectangular Pattern)
34. Parametric clearance via composed expressions
35. OffsetStartDefinition for extrude start offset
36. Multi-body cut via participantBodies
37. Composite body via NewBody + Join with overlap
38. Bodies to components, preserving world position
39. As-built joints (revolute hinge + slider)
40. Joint limits
41. Internal travel-stop for a print-in-place slider
42. Reorient a grounded jointed assembly for print/export
43. Sketch constraint anti-patterns

## 1. Idempotent user-parameter add

Make every script safe to re-run.

```python
param_defs = [
    ('length', '220 mm', 'mm', ''),
    ('width',  '148 mm', 'mm', ''),
    ('height', '38 mm',  'mm', ''),
    ('wall',   '3 mm',   'mm', ''),
    ('floor',  '3 mm',   'mm', ''),
    ('total_height',
     'base_height + air_gap + 2 * floor_thickness',
     'mm', 'COMPUTED'),
]

params = design.userParameters
for name, expr, unit, comment in param_defs:
    if not params.itemByName(name):
        params.add(name, adsk.core.ValueInput.createByString(expr), unit, comment)
```

Expressions can reference other params directly.

## 2. Parametric box from a sketched rectangle

A fully constrained centered rectangle: every line shows BLACK in the UI, the browser shows a lock badge, and the box resizes correctly when `length`/`width` change. `addCenterPointRectangle` creates only the four lines, NOT the constraints, so add them explicitly. Never ship a raw-vertex sketch for geometry that should stay parametric (see Sketch constraint anti-patterns, section 43).

```python
P = adsk.core.Point3D.create

sk = root.sketches.add(root.xYConstructionPlane)
sk.name = 'outer'
sk.isComputeDeferred = True

# 1. Geometry only (the API adds no constraints here)
rect = sk.sketchCurves.sketchLines.addCenterPointRectangle(
    P(0, 0, 0), P(2.5, 2.5, 0))   # initial size; dimensions will drive the final size
L_top, L_left, L_bot, L_right = rect.item(0), rect.item(1), rect.item(2), rect.item(3)
sk.isComputeDeferred = False

# 2. Find SW corner (point shared by the left and bottom edges)
sw_pt = None
for sp in [L_left.startSketchPoint, L_left.endSketchPoint]:
    for sp2 in [L_bot.startSketchPoint, L_bot.endSketchPoint]:
        if sp.geometry.x == sp2.geometry.x and sp.geometry.y == sp2.geometry.y:
            sw_pt = sp; break
    if sw_pt: break

# 3. Geometric constraints lock the rectangle shape
gc = sk.geometricConstraints
gc.addHorizontal(L_top); gc.addHorizontal(L_bot)
gc.addVertical(L_left);  gc.addVertical(L_right)

# 4. Size + position-from-origin dimensions
sd = sk.sketchDimensions
DO = adsk.fusion.DimensionOrientations

dim_w = sd.addDistanceDimension(L_top.startSketchPoint, L_top.endSketchPoint,
    DO.HorizontalDimensionOrientation, P(0, 3, 0))
dim_w.parameter.expression = 'length'

dim_d = sd.addDistanceDimension(L_left.startSketchPoint, L_left.endSketchPoint,
    DO.VerticalDimensionOrientation, P(-4, 0, 0))
dim_d.parameter.expression = 'width'

dim_sw_x = sd.addDistanceDimension(sk.originPoint, sw_pt,
    DO.HorizontalDimensionOrientation, P(-2, -3, 0))
dim_sw_x.parameter.expression = 'length / 2'   # math expr binds first try

dim_sw_y = sd.addDistanceDimension(sk.originPoint, sw_pt,
    DO.VerticalDimensionOrientation, P(-4.5, -1, 0))
dim_sw_y.parameter.expression = 'width / 2'

# 5. G6 workaround: re-assign bare param names so they bind parametrically
dim_w.parameter.expression = 'length'
dim_d.parameter.expression = 'width'

assert sk.isFullyConstrained
for d in sk.sketchDimensions:
    assert d.parameter.dependencyParameters.count > 0, \
        f"dim {d.parameter.expression} did not bind to a user param"
assert sk.profiles.count == 1, f"expected 1 profile, got {sk.profiles.count}"

extrudes = root.features.extrudeFeatures
ext_in = extrudes.createInput(sk.profiles.item(0),
    adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
ext_in.setDistanceExtent(False, adsk.core.ValueInput.createByString('height'))
feat = extrudes.add(ext_in)
feat.name = 'body_solid'
feat.bodies.item(0).name = 'main'
```

Four geometric constraints + four dimensions produce a sketch that survives a parametric cascade (verified: `length` 70 to 90 mm updated all dependent geometry with no re-solve). The re-assign in step 5 is the bare-name binding workaround (see "may not bind on first assignment" in gotchas.md). Always assert `isFullyConstrained` AND check `dependencyParameters` after closing the sketch; the API flag alone can read True while the UI still treats the sketch as under-constrained. `isComputeDeferred = True` before bulk line adds matters for >10 vertices; below that it is optional.

## 3. Construction plane offset from origin plane

```python
pi = root.constructionPlanes.createInput()
pi.setByOffset(root.xYConstructionPlane,
               adsk.core.ValueInput.createByString('height'))
plane = root.constructionPlanes.add(pi)
plane.name = 'top_plane'
```

## 4. Multi-profile cut (n cells in one feature)

After sketching multiple closed rectangles on a plane:

```python
profs = adsk.core.ObjectCollection.create()
for i in range(sk.profiles.count):
    profs.add(sk.profiles.item(i))

cut_in = extrudes.createInput(profs,
    adsk.fusion.FeatureOperations.CutFeatureOperation)
ext_def = adsk.fusion.DistanceExtentDefinition.create(
    adsk.core.ValueInput.createByString('height - floor'))
cut_in.setOneSideExtent(ext_def,
    adsk.fusion.ExtentDirections.NegativeExtentDirection)
cut_in.participantBodies = [body]
feat = extrudes.add(cut_in)
feat.name = 'pockets'
```

## 5. Cut through unknown depth

When geometry below the sketch plane has variable depth (gussets, flanges, dividers):

```python
cut_in.setAllExtent(adsk.fusion.ExtentDirections.NegativeExtentDirection)
```

Valid for cut and intersect operations only. This pattern fixes counterbore cuts that get blocked by intermediate geometry the script does not know about.

## 6. Symmetric extrude (centered on sketch plane)

```python
ext_in.setSymmetricExtent(
    adsk.core.ValueInput.createByString('length'),
    True)  # True = value is TOTAL extent, not half
```

## 7. Tapered extrude (cone, legacy countersink approach)

```python
ext_def = adsk.fusion.DistanceExtentDefinition.create(
    adsk.core.ValueInput.createByString('cs_depth'))
taper_vi = adsk.core.ValueInput.createByString('-cs_angle / 2')  # negative = inward
cut_in.setOneSideExtent(ext_def,
    adsk.fusion.ExtentDirections.NegativeExtentDirection,
    taper_vi)
```

For real countersinks, prefer `HoleFeatures.createCountersinkInput` (see section 8).

## 8. Holes via HoleFeatureInput (preferred over sketch+cut)

HoleFeature is parametric, named, editable in the timeline, and cleaner than two extruded cuts. Use over sketch+extrude-cut for any standard hole.

### 8a. Simple through-hole

```python
holes = root.features.holeFeatures
hi = holes.createSimpleInput(adsk.core.ValueInput.createByString('4.2 mm'))
center = adsk.core.Point3D.create(2.5, 2.5, 1.0)  # cm
hi.setPositionByPoint(top_face, center)
hi.setDistanceExtent(adsk.core.ValueInput.createByString('10 mm'))
holes.add(hi)
```

### 8b. Counterbored hole

```python
holes = root.features.holeFeatures
hi = holes.createCounterboreInput(
    adsk.core.ValueInput.createByString('4 mm'),   # hole dia
    adsk.core.ValueInput.createByString('8 mm'),   # cbore dia
    adsk.core.ValueInput.createByString('3 mm'))   # cbore depth
center = adsk.core.Point3D.create(2.5, 2.5, 1.0)
hi.setPositionByPoint(top_face, center)
hi.setDistanceExtent(adsk.core.ValueInput.createByString('10 mm'))
holes.add(hi)
```

Direction is implicit from the face normal passed to `setPositionByPoint`. No explicit `ExtentDirections` argument needed for `setDistanceExtent` here.

## 9. Fillet by current UI selection

When the user has selected edges in Fusion UI before asking Claude to fillet them:

```python
sel = adsk.core.Application.get().userInterface.activeSelections
edges = [sel.item(i).entity for i in range(sel.count)
         if isinstance(sel.item(i).entity, adsk.fusion.BRepEdge)]

fi = root.features.filletFeatures.createInput()
fi.isRollingBall = True  # matches user expectation for blends
ec = adsk.core.ObjectCollection.create()
for e in edges:
    ec.add(e)
fi.addConstantRadiusEdgeSet(ec,
    adsk.core.ValueInput.createByString('8 mm'),
    False)  # False = no tangent chain propagation
feat = root.features.filletFeatures.add(fi)
feat.name = 'edge_fillet_8mm'
```

## 10. Edit existing fillet radius (no rebuild needed)

```python
for f in root.features.filletFeatures:
    if 'edge_fillet' in f.name:
        f.edgeSets.item(0).radius.expression = '10 mm'
        f.name = 'edge_fillet_10mm'
        break
```

Inline radius edit. Cheaper than delete and recreate.

## 11. Read parameter values for sketch math

```python
def p(name):
    return design.userParameters.itemByName(name).value  # returns CM

reach = p('arm_reach')   # if param=30mm, reach == 3.0 (cm)
# Point3D.create(reach, 0, 0) also takes cm. Math is consistent.
```

Only multiply by 10 when printing dimensions for human display.

## 12. Bounding box sanity check (always after extrudes)

```python
bb = body.boundingBox
print(f"X: {bb.minPoint.x*10:.2f} to {bb.maxPoint.x*10:.2f} mm")
print(f"Y: {bb.minPoint.y*10:.2f} to {bb.maxPoint.y*10:.2f} mm")
print(f"Z: {bb.minPoint.z*10:.2f} to {bb.maxPoint.z*10:.2f} mm")
```

Screenshots can deceive on orientation. Bounding box per axis is ground truth.

## 13. Clean rebuild without losing parameters

```python
while root.bRepBodies.count > 0:  root.bRepBodies.item(0).deleteMe()
while root.sketches.count > 0:    root.sketches.item(0).deleteMe()
while root.features.count > 0:    root.features.item(0).deleteMe()
```

Use this instead of `update(undo)` to reset geometry while keeping `userParameters`. Safer than undo because parameters are preserved. For changes that only touch a subset of features (e.g. re-laying out cavities while keeping the body+lip), prefer the targeted delete-and-rebuild in section 19.

## 14. Export to STL / 3MF / STEP

All four formats verified working. Always absolute paths with forward slashes; ensure parent directory exists.

### 14a. STL

```python
import os
out_dir = 'C:/path/to/your/exports'
os.makedirs(out_dir, exist_ok=True)

body = design.rootComponent.bRepBodies.itemByName('main')
em = design.exportManager
opts = em.createSTLExportOptions(body, f'{out_dir}/part.stl')
opts.meshRefinement = adsk.fusion.MeshRefinementSettings.MeshRefinementHigh
em.execute(opts)
```

Refinement options: `MeshRefinementLow`, `MeshRefinementMedium`, `MeshRefinementHigh`. Only affects curved geometry; flat boxes produce identical files at any setting. Use `High` for production exports; the cost on flat-dominated geometry is zero, the benefit on curves is real.

### 14b. 3MF (color + multi-body capable)

```python
opts = em.createC3MFExportOptions(body, f'{out_dir}/part.3mf')
em.execute(opts)
```

Note the capital `C` in `createC3MFExportOptions`. Common signature-guess miss.

### 14c. STEP (component-scoped, not body-scoped)

```python
opts = em.createSTEPExportOptions(f'{out_dir}/part.step', design.rootComponent)
em.execute(opts)
```

Pass a sub-component to scope down; pass `design.rootComponent` for the whole doc.

### Export warnings

- Overwrite is SILENT. If preserving prior exports matters, add a version suffix.
- Relative paths fail with `RuntimeError: 3 : The selected folder does not exist.`
- Protected paths fail with `RuntimeError: 3 : The selected folder is not accessible.`
- Forward slashes work on Windows. Backslashes also work but forward is cleanest.

## 15. API documentation lookup (before guessing signatures)

Always set `apiCategory`. Null or omitted returns success with empty data (silent failure).

```json
{ "queryType": "apiDocumentation",
  "searchPattern": "createSTLExportOptions",
  "apiCategory": "member" }
```

- `member`: best for one named function. Returns signature and docstring.
- `class`: best for exploring an unknown class. Returns properties and functions.
- `all`: best when unsure. Returns everything matching.

Searches accept regex but plain substrings work. Multi-class hits (e.g. `setAllExtent` exists on 4 classes) help locate ownership.

## 16. Screenshot defaults

```json
{ "queryType": "screenshot",
  "width": 800, "height": 600,
  "direction": "current",
  "transparentBackground": true }
```

Run the fit-view block from section 20 BEFORE every screenshot. Without it, results are unreliable:

- `direction: "current"` auto-fits in the common case but stops auto-fitting on Untitled docs and right after script-driven feature additions.
- Named directions (`iso-top-right`, `top`, `right`, `front`) only set orientation; they NEVER auto-fit. If the camera was moved by a prior script, you get an empty frame with a navy background.

Background: `transparentBackground: true` gives a true transparent PNG for compositing. `transparentBackground: false` gives the Fusion workspace background (medium gray-blue with content, dark navy if empty).

## 17. Document search

Use when looking for a specific design (fuzzy, cross-project):

```json
{ "queryType": "document",
  "operation": "search",
  "name": "my-part-name" }
```

Matches case-insensitively. Treats `-` and `_` as equivalent. No project param needed.

Use `document/recent` for "what was I working on?" workflows. Use `document/open` to confirm active-doc state before destructive operations.

## 18. Partial-height interior features via top-shave cut

When dividers (or any interior wall) should stop short of the rim, leaving a common open bay across the top, add a second cut that shaves the top of the existing walls. Additive feature, foundation stays intact.

```python
# Compute divider rectangles from existing param values (read at script time)
cl    = p('compartment_length')
dt    = p('divider_thickness')
cw    = p('cavity_width')
cav_l = p('cavity_length')

# 2 dividers between 3 compartments
d1_x0 = -cav_l/2 + cl
d1_x1 =  d1_x0 + dt
d2_x0 =  cav_l/2 - cl - dt
d2_x1 =  d2_x0 + dt
y0, y1 = -cw/2, cw/2

sk = root.sketches.add(tray_top_plane)
sk.name = 'divider_tops'
sk.isComputeDeferred = True
for x0, x1 in [(d1_x0, d1_x1), (d2_x0, d2_x1)]:
    pts = [P(x0,y0,0), P(x1,y0,0), P(x1,y1,0), P(x0,y1,0)]
    for i in range(4):
        sk.sketchCurves.sketchLines.addByTwoPoints(pts[i], pts[(i+1)%4])
sk.isComputeDeferred = False
assert sk.profiles.count == 2

profs = adsk.core.ObjectCollection.create()
for i in range(sk.profiles.count): profs.add(sk.profiles.item(i))

cut_in = extrudes.createInput(profs,
    adsk.fusion.FeatureOperations.CutFeatureOperation)
ext_def = adsk.fusion.DistanceExtentDefinition.create(
    adsk.core.ValueInput.createByString('divider_top_offset'))
cut_in.setOneSideExtent(ext_def,
    adsk.fusion.ExtentDirections.NegativeExtentDirection)
cut_in.participantBodies = [body]
feat = extrudes.add(cut_in)
feat.name = 'divider_top_shave'
```

Verification math: volume change = (n_dividers) x (divider_thickness) x (cavity_width) x (offset_amount). For 2 x 3mm x 182mm x 20mm = 21.84 cm^3.

Caveat: the X positions in the sketch are computed from current param values at script time, not driven by sketch dimensions. Changing `divider_thickness`, `compartment_count`, or `cavity_length` later will not auto-update; re-run the script.

## 19. Targeted delete-and-rebuild

When changing layout fundamentally but keeping the body+lip foundation, delete only the cavity-shaping features by name, then rebuild. Faster than the section 13 "delete everything" approach when only the layout changes.

```python
# Delete in dependency order: cut features first, then sketches
to_delete_features = ['divider_top_shave', 'compartment_cavities']
to_delete_sketches = ['divider_tops', 'compartments']

for fname in to_delete_features:
    for i in range(root.features.count - 1, -1, -1):
        f = root.features.item(i)
        try:
            if f.name == fname:
                f.deleteMe()
                break
        except Exception:
            pass

for sname in to_delete_sketches:
    for sk in list(root.sketches):
        if sk.name == sname:
            sk.deleteMe()
            break

# Verify body returned to its pre-cut solid state
body = root.bRepBodies.itemByName('tray')
print(f"naked body volume: {body.volume:.2f} cm^3 (should match the solid math)")
```

Verification gate: print body volume after deletion. Should match the math for the body without any cavities (body_lower + lip_flange volumes). If off, something didn't delete cleanly.

Naming requirement: every feature MUST have a unique meaningful name. Without names, delete-by-name fails and you fall back to walking by index, which is fragile.

## 20. Force camera fit-view from script

Reliably frames the body for a screenshot. Closes the camera-control gap previously documented in gotchas.md.

```python
vp = app.activeViewport
vp.fit()
cam = vp.camera
cam.viewOrientation = adsk.core.ViewOrientations.IsoTopRightViewOrientation
cam.isFitView = True
vp.camera = cam
vp.refresh()
```

After this, `read` screenshot with `direction: "current"` produces a properly framed isometric. Named directions on their own do NOT reliably auto-fit; the camera position is whatever the last operation left behind.

Available orientations:

- `IsoTopRightViewOrientation`, `IsoTopLeftViewOrientation`, `IsoBottomLeftViewOrientation`, `IsoBottomRightViewOrientation`
- `FrontViewOrientation`, `BackViewOrientation`, `LeftViewOrientation`, `RightViewOrientation`
- `TopViewOrientation`, `BottomViewOrientation`

Run this block before every screenshot for consistent framing across design iterations.

## 21. Save viewport snapshot to disk

Save the current viewport as a PNG directly to a project path. Bypasses the base64 round-trip of `read` screenshot. Useful for capturing preview images straight into a project's docs folder.

```python
import os

docs_dir = 'C:/path/to/your/docs'
os.makedirs(docs_dir, exist_ok=True)

vp = app.activeViewport
preview_path = f'{docs_dir}/preview-isometric.png'
ok = vp.saveAsImageFile(preview_path, 1600, 1200)
print(f"viewport saved: {ok} -> {preview_path}")
```

Combines well with section 20 (force fit-view) before the save call. Returns `True` on success, `False` on failure. Overwrites silently like other Fusion exports.

## 22. Doc state sanity-check (read-only opener)

Run this BEFORE any modification on an unfamiliar doc. Fastest way to know what you're walking into.

```python
print(f"bodies={root.bRepBodies.count} sketches={root.sketches.count} "
      f"features={root.features.count} units={design.unitsManager.defaultLengthUnits}")
print(f"params={design.userParameters.count} "
      f"planes={root.constructionPlanes.count}")
print(f"doc.isSaved={app.activeDocument.isSaved} "
      f"doc.isModified={app.activeDocument.isModified}")
```

Tells you: is the doc empty, what state are bodies and sketches in, do existing params match what your script expects, is the doc saved (so MCP `save` will work) or Untitled (must SaveAs via UI first).

Pair with `print('PARAMS:', [p.name for p in design.userParameters])` when you suspect a prior script left params you should re-use rather than re-add.

## 23. Stepped solid via Join extrude (body + lip flange)

For tray/insert geometry that has a body section dropping into an opening plus a wider lip flange resting on top. Both sketches use the constrained centered-rectangle approach from section 2 (no raw vertices), via a small helper so body and lip get the identical parametric treatment.

```python
P = adsk.core.Point3D.create
DO = adsk.fusion.DimensionOrientations

def constrained_centered_rect(sk, w_expr, d_expr):
    """Fully constrained rectangle centered on the sketch origin, driven by two param expressions."""
    sk.isComputeDeferred = True
    rect = sk.sketchCurves.sketchLines.addCenterPointRectangle(
        P(0, 0, 0), P(2.5, 2.5, 0))
    L_top, L_left, L_bot, L_right = rect.item(0), rect.item(1), rect.item(2), rect.item(3)
    sk.isComputeDeferred = False

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
        DO.HorizontalDimensionOrientation, P(0, 3, 0))
    dim_d = sd.addDistanceDimension(L_left.startSketchPoint, L_left.endSketchPoint,
        DO.VerticalDimensionOrientation, P(-4, 0, 0))
    dim_x = sd.addDistanceDimension(sk.originPoint, sw_pt,
        DO.HorizontalDimensionOrientation, P(-2, -3, 0))
    dim_y = sd.addDistanceDimension(sk.originPoint, sw_pt,
        DO.VerticalDimensionOrientation, P(-4.5, -1, 0))
    dim_x.parameter.expression = '(%s) / 2' % w_expr   # math expr, binds first try
    dim_y.parameter.expression = '(%s) / 2' % d_expr
    dim_w.parameter.expression = '(%s) + 0 mm' % w_expr   # compound expr binds first try (avoids G6 bare-name freeze)
    dim_d.parameter.expression = '(%s) + 0 mm' % d_expr
    assert sk.isFullyConstrained
    assert sk.profiles.count == 1, f"expected 1 profile, got {sk.profiles.count}"
    return sk

extrudes = root.features.extrudeFeatures

# 1. Body lower: constrained rect driven by body_length/body_width, extrude up by body_height
sk_body = root.sketches.add(root.xYConstructionPlane)
sk_body.name = 'body_outer'
constrained_centered_rect(sk_body, 'body_length', 'body_width')

ext_in = extrudes.createInput(sk_body.profiles.item(0),
    adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
ext_in.setDistanceExtent(False, adsk.core.ValueInput.createByString('body_height'))
feat = extrudes.add(ext_in)
feat.name = 'body_lower'
body = feat.bodies.item(0)
body.name = 'tray'

# 2. Construction plane at body top
pi = root.constructionPlanes.createInput()
pi.setByOffset(root.xYConstructionPlane,
               adsk.core.ValueInput.createByString('body_height'))
body_top_plane = root.constructionPlanes.add(pi)
body_top_plane.name = 'body_top_plane'

# 3. Lip flange: wider constrained rect on the body-top plane, JOIN extrude up by lip_height
sk_lip = root.sketches.add(body_top_plane)
sk_lip.name = 'lip_outer'
constrained_centered_rect(sk_lip, 'tray_outer_length', 'tray_outer_width')

lip_in = extrudes.createInput(sk_lip.profiles.item(0),
    adsk.fusion.FeatureOperations.JoinFeatureOperation)
lip_in.setDistanceExtent(False, adsk.core.ValueInput.createByString('lip_height'))
lip_in.participantBodies = [body]   # join target, required (see gotchas.md)
feat = extrudes.add(lip_in)
feat.name = 'lip_flange'
```

Result: single combined body, footprint `body_length x body_width` from z=0 to z=body_height, transitioning to `tray_outer_length x tray_outer_width` at the top for the lip portion. Both sketches are fully constrained, so changing any of the four footprint params cascades correctly. Bambu Studio prints this floor-down with no supports for the 90-degree lip step (short overhangs bridge cleanly).

## 24. Volume sanity check (spec vs actual)

Body volume is in cm^3. Compare against the spec's solid-envelope-minus-cavities math to catch missing cuts or doubled extrusions.

```python
body = root.bRepBodies.itemByName('tray')
print(f"body.volume = {body.volume:.2f} cm^3")
# Cross-check: solid envelope - removed cavities
# e.g. tray = body_box + lip_box - 3 * compartment_box
```

If actual is significantly off from spec math, something is wrong (missed cut, doubled body, wrong participantBody). Catches problems screenshots miss.

CAVEAT: `body.volume` is the geometric solid volume. It is NOT a filament weight estimate. Slicer infill, walls, top/bottom layers, and supports all change the actual print weight by a factor of 2-5x downward from solid volume. Always slice and read grams from the slicer before pricing.

## 25. Countersink hole via HoleFeatureInput

Companion to section 8a (simple) and 8b (counterbore). For screws with conical heads (M3 flat-head, M4 wood screws):

```python
holes = root.features.holeFeatures
hi = holes.createCountersinkInput(
    adsk.core.ValueInput.createByString('4.2 mm'),   # hole dia (shank clearance)
    adsk.core.ValueInput.createByString('8.4 mm'),   # csink dia (head width)
    adsk.core.ValueInput.createByString('82 deg'))   # csink angle (82 deg = #6 wood, 90 deg = metric flat)
center = adsk.core.Point3D.create(2.5, 2.5, 1.0)
hi.setPositionByPoint(top_face, center)
hi.setDistanceExtent(adsk.core.ValueInput.createByString('10 mm'))
holes.add(hi)
```

Common cone angles: 82 degrees (US wood/sheet metal #4-#12), 90 degrees (metric flat head ISO 10642), 100 degrees (US aircraft/military). When in doubt, look up the screw spec.

## 26. Print all resolved parameter values

After idempotent param add (section 1), confirm computed expressions resolve to what you expect:

```python
for pname in [d[0] for d in param_defs]:
    pv = design.userParameters.itemByName(pname)
    print(f"  {pname:30s} expr={pv.expression:40s} value={pv.value*10:.3f} mm")
```

Catches:
- Expression typos that silently default a computed value to 0
- Missing prerequisite params (referencing an undefined name returns a Fusion eval error, but only when triggered)
- Order-of-add issues (a computed param that references a param not yet added)

Run this once after param add, before any body-adding feature that uses a computed value.

## 27. Constrained parametric ellipse

For dispense holes, slots, or any ellipse aligned to world axes. 5 DOFs, 5 constraints.

```python
sk = root.sketches.add(root.xYConstructionPlane)
sk.name = 'dispense_oval'

el = sk.sketchCurves.sketchEllipses.add(
    P(0, 0, 0),       # center (initial position)
    P(1.0, 0, 0),     # major axis end (10mm along X)
    P(0, 0.75, 0))    # any point on the ellipse (defines minor radius)

gc = sk.geometricConstraints
gc.addCoincident(el.centerSketchPoint, sk.originPoint)    # lock center
gc.addHorizontal(el.majorAxisLine)                        # lock major axis direction

sd = sk.sketchDimensions
dim_maj = sd.addEllipseMajorRadiusDimension(el, P(2, 0, 0))
dim_maj.parameter.expression = 'oval_x / 2'    # math expr, binds first try
dim_min = sd.addEllipseMinorRadiusDimension(el, P(0, 1.5, 0))
dim_min.parameter.expression = 'oval_y / 2'

assert sk.isFullyConstrained
```

API points worth knowing: `SketchEllipse.centerSketchPoint` (locatable center), `SketchEllipse.majorAxisLine` (constrain it horizontal/vertical to lock orientation), and `addEllipseMajorRadiusDimension` / `addEllipseMinorRadiusDimension(ellipse, textPoint)`. The third arg to `SketchEllipses.add()` is any point on the curve; it sets the minor radius implicitly.

## 28. Off-center constrained rectangle (edge-anchored)

Same as section 2 but anchor a meaningful corner to a derived expression instead of centering on origin, so the rectangle TRACKS a reference (e.g. a body edge) when params change.

```python
rect = sk.sketchCurves.sketchLines.addCenterPointRectangle(
    P(0.5, 0, 0), P(3.5, 1.1, 0))
L_top, L_left, L_bot, L_right = rect.item(0), rect.item(1), rect.item(2), rect.item(3)

ne_pt = ...  # NE corner: same shared-point lookup as section 2, on top + right edges

gc.addHorizontal(L_top); gc.addHorizontal(L_bot)
gc.addVertical(L_left);  gc.addVertical(L_right)

dim_w = sd.addDistanceDimension(..., DO.HorizontalDimensionOrientation, ...)
dim_w.parameter.expression = 'slot_len'
dim_d = sd.addDistanceDimension(..., DO.VerticalDimensionOrientation, ...)
dim_d.parameter.expression = 'slot_narrow_w'

# Anchor the NE corner to the body edge: if body_width changes, the slot follows the edge
dim_ne_x = sd.addDistanceDimension(sk.originPoint, ne_pt,
    DO.HorizontalDimensionOrientation, ...)
dim_ne_x.parameter.expression = 'body_width / 2'
dim_ne_y = sd.addDistanceDimension(sk.originPoint, ne_pt,
    DO.VerticalDimensionOrientation, ...)
dim_ne_y.parameter.expression = 'slot_narrow_w / 2'

dim_w.parameter.expression = 'slot_len'         # G6 re-assign
dim_d.parameter.expression = 'slot_narrow_w'
```

Choose the anchor corner by what the part should track. Verified cascade: `body_width` 70 to 90 mm moved the slot NE corner 35 to 45 mm (followed the edge) while the slot length stayed 60 mm.

## 29. Constrained parametric circle

Simplest closed curve. 3 DOFs (center X, center Y, radius).

```python
c = sk.sketchCurves.sketchCircles.addByCenterRadius(P(0, 0, 0), 0.25)  # radius in cm
gc.addCoincident(c.centerSketchPoint, sk.originPoint)         # locks center (2 DOFs)
dim_d = sd.addDiameterDimension(c, P(0.5, 0.5, 0))            # locks radius (1 DOF)
dim_d.parameter.expression = 'knuckle_dia'
assert sk.isFullyConstrained
```

For an off-center circle, replace `addCoincident` with two distance dimensions (Horizontal + Vertical) from origin to `c.centerSketchPoint`, like section 2's corner anchor. Note: `addDiameterDimension` bound a bare param name on FIRST try, so the section-2 G6 re-assign is not needed here (the binding bug appears specific to `addDistanceDimension`). `addRadialDimension(circle, textPoint)` is the radius-based alternative.

## 30. Constrained rectangle on yZ plane (G5-aware)

Same constraint pattern as section 28, on `yZConstructionPlane`. The only thing that changes is the G5 axis mapping: sketch X = NEGATIVE world Z, sketch Y = POSITIVE world Y (see gotchas.md). Account for that when choosing sketch coordinates.

```python
sk = root.sketches.add(root.yZConstructionPlane)
sk.name = 'clip_outer'

# Target: clip top at world Z=80, body face at world Y=20, clip out to Y=32
# Per G5: sketch X = -world Z (so X = -80..-30), sketch Y = +world Y (so Y = 20..32)
rect = sk.sketchCurves.sketchLines.addCenterPointRectangle(
    P(-5.5, 2.7, 0), P(-3.0, 3.4, 0))   # initial geometry, cm
L0, L1, L2, L3 = rect.item(0), rect.item(1), rect.item(2), rect.item(3)

# Classify lines by which sketch axis they run along
h_lines = [L for L in [L0,L1,L2,L3]
           if abs(L.startSketchPoint.geometry.y - L.endSketchPoint.geometry.y) < 0.01]
v_lines = [L for L in [L0,L1,L2,L3]
           if abs(L.startSketchPoint.geometry.x - L.endSketchPoint.geometry.x) < 0.01]

gc = sk.geometricConstraints
for L in h_lines: gc.addHorizontal(L)
for L in v_lines: gc.addVertical(L)

# ... size dims (math exprs bind first try) + an anchor dim on the known corner ...
# anchor example: top-of-body at body face = world Z=80, Y=20
dim_ax.parameter.expression = 'body_height'        # sketch X anchor
dim_ay.parameter.expression = 'body_depth / 2'     # sketch Y anchor
dim_ax.parameter.expression = 'body_height'        # G6 re-assign on the bare-name dim
assert sk.isFullyConstrained
```

Reminder: before committing yZ coordinates, drop one test `sketchPoint` and read `worldGeometry` to confirm the G5 mapping (gotchas.md "Habit for any non-XY plane"). Saves a re-orientation debug cycle.

## 31. Re-anchor an origin-pinned sketch entity

To move a sketch entity that is coincident-locked to the origin (e.g. relocate an oval from centered to a parametric offset) without rebuilding the sketch:

1. Delete the center-to-origin coincident constraint (match it by the origin's `entityToken`; the origin usually participates in only that one coincident).
2. `sk.move(collection, translationMatrix)` to pre-position the entity on the correct side.
3. Re-lock: `addHorizontalPoints(origin, center)` locks Y, then a horizontal distance dimension to a param locks X. (`addVerticalPoints` is the X-lock analogue.)

Distance dimensions are MAGNITUDE / unsigned, so pre-move the entity to the correct side first (step 2); the solver keeps it where you put it. Not assembly-specific, but it pairs with the components workflow (section 38) when relocating features after componentizing.

## 32. Edge finding by geometry (fillet/chamfer without UI selection)

To fillet or chamfer specific edges programmatically (e.g. internal corners after a cut), find edges by geometric properties instead of UI selection.

```python
target_edges = []
for edge in body.edges:
    g = edge.geometry
    if not isinstance(g, adsk.core.Line3D):
        continue
    sp, ep = g.startPoint, g.endPoint
    if abs(sp.y - ep.y) > 0.01 or abs(sp.z - ep.z) > 0.01:   # edge runs along X = const Y, const Z
        continue
    y, z = sp.y, sp.z   # cm
    if abs(z - 7.7) < 0.01 and (abs(y - 2.3) < 0.01 or abs(y - 2.9) < 0.01):
        target_edges.append(edge)

fillets = root.features.filletFeatures
fi = fillets.createInput()
fi.isRollingBall = True
ec = adsk.core.ObjectCollection.create()
for e in target_edges: ec.add(e)
fi.addConstantRadiusEdgeSet(ec,
    adsk.core.ValueInput.createByString('clip_fillet_r'), False)   # False = no tangent chain
fillets.add(fi)
```

Chamfer is the same selection feeding `chamfers.createInput(ec, False)` then `ci.setToEqualDistance(ValueInput.createByString('clip_chamfer'))`. A 0.01 cm (0.1mm) tolerance is reliable; keep geometric checks in cm to match internal units.

Volume math (use it as the post-selection sanity check, see section 24):
- Fillet on a CONCAVE edge ADDS material; on a CONVEX edge REMOVES material. Chamfer on a convex edge removes.
- Concave fillet, edge length L, radius r: volume added approx L * r^2 * (1 - pi/4) = L * r^2 * 0.2146.
- Convex chamfer, edge length L, equal distance d: volume removed = L * d^2 / 2.
- Verified on the clip build: 2 fillets (r=2mm, L=40mm) + 2 chamfers (d=1.5mm, L=40mm) gave net -21.3 mm^3 (added 68.7, removed 90); observed -21.33 mm^3.

## 33. Replicate features via Mirror (vs Rectangular Pattern)

To replicate a Join or Cut extrude across a symmetry plane, prefer `MirrorFeatures` over `RectangularPatternFeatures`. Mirror preserves the source operation type (Join stays Join, Cut stays Cut) and targets the same `participantBodies` correctly. `RectangularPattern` with symmetric direction misbehaves on Join extrudes (gotchas.md).

```python
mirror_feats = root.features.mirrorFeatures
input_entities = adsk.core.ObjectCollection.create()
input_entities.add(some_join_extrude_feature)
mi = mirror_feats.createInput(input_entities, root.yZConstructionPlane)
mfeat = mirror_feats.add(mi)
mfeat.name = 'my_mirror_feature'
```

Verified (V7): mirrored a body knuckle Join (X=+20 to X=-20) and lid knuckle pair (X=+10 to X=-10), plus CUT mirrors (body notch, lid notch) preserving each cut's `participantBodies`. No new disconnected bodies; volume deltas matched the source exactly.

Notes:
- The source feature is not moved; Mirror creates a new feature alongside it.
- For non-symmetric layouts (e.g. knuckles at -20, 0, +20): place the +20 instance manually on an offset construction plane, Mirror it to -20, and build the center (X=0) as its own feature.

## 34. Parametric clearance via composed expressions

When a clearance value appears in many cut/feature dimensions (print-in-place mechanisms, snap fits, sliding parts), define it ONCE as a param and reference it through composed expressions everywhere. The whole assembly then tunes from one knob.

```python
params.add('knuckle_clearance', ValueInput.createByString('0.2 mm'), 'mm', '')
params.add('pin_dia',           ValueInput.createByString('3 mm'),   'mm', '')
params.add('knuckle_dia',       ValueInput.createByString('5 mm'),   'mm', '')
params.add('knuckle_len',       ValueInput.createByString('8 mm'),   'mm', '')

# Body notch dia (clears lid knuckle + clearance on each side):
dim.parameter.expression = 'knuckle_dia + 2 * knuckle_clearance'   # = 5.4mm
# Body notch length (clears knuckle X extent + clearance each side):
ext_in.setSymmetricExtent(
    ValueInput.createByString('knuckle_len + 2 * knuckle_clearance'), True)   # = 8.4mm
# Pin bore dia:
dim.parameter.expression = 'pin_dia + 2 * knuckle_clearance'   # = 3.4mm
```

Why it matters: single source of truth (bump `knuckle_clearance` 0.2 to 0.3mm and every clearance cut plus the pin bore expand in one pass); the `+ 2 *` makes DIAMETRAL vs radial clearance explicit; the intent is self-documenting in the expression. Name clearance categories when motion types differ:

```python
params.add('rotational_clearance', ValueInput.createByString('0.2 mm'), 'mm', 'between rotating parts')
params.add('sliding_clearance',    ValueInput.createByString('0.15 mm'), 'mm', 'between sliding parts')
params.add('z_layer_clearance',    ValueInput.createByString('0.2 mm'), 'mm', 'between stacked parts at the layer interface')
```

The single-knob technique is sound and geometry-verified (6 clearance cuts + 1 pin bore on one test part all driven from one param, volume math to <1mm^3).

PRINTER-VERIFIED CORRECTION (captured slider+hinge test part, Bambu): the values 0.2mm rotational / 0.3mm Z / 1mm Y were TOO TIGHT for a captured print-in-place mechanism. The moving parts FUSED to the housing on the first layer (welded, never freed). The expression technique is unchanged; the takeaway is the values and the approach for this class of part:

- For print-in-place capture on FDM, these clearances are not enough. Expect to go looser, and validate at the printer before committing (geometry-verified is not print-verified).
- Preferred path for this class: print SEPARATE parts on one plate with ASSEMBLY clearances (looser, tunable, sandable after the fact) rather than captured-in-place. Componentize first (section 38), then export the parts laid out separately. Use a separate printed or filament/steel-rod pin through the hinge knuckles rather than a print-in-place knuckle.

## 35. OffsetStartDefinition for extrude start offset

When an extrude must start OFFSET from its sketch plane, use `startExtent = OffsetStartDefinition.create(ValueInput)`. Avoids an extra construction plane and keeps the sketch on a clean principal plane.

```python
sk = root.sketches.add(root.yZConstructionPlane)   # sketch at world X=0
# ... build profile ...

ext_in = extrudes.createInput(sk.profiles.item(0),
    adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
ext_in.startExtent = adsk.fusion.OffsetStartDefinition.create(
    adsk.core.ValueInput.createByString('body_width/2 - slot_len'))   # start at X=-25, parametric
ext_def = adsk.fusion.DistanceExtentDefinition.create(
    adsk.core.ValueInput.createByString('slider_len'))
ext_in.setOneSideExtent(ext_def, adsk.fusion.ExtentDirections.PositiveExtentDirection)
feat = extrudes.add(ext_in)
```

Sign convention: the offset is interpreted along the sketch normal. For yZ the normal is +X, so positive offset moves +X. Verified: `body_width/2 - slot_len` = -25 (body_width=70, slot_len=60) placed the start at X=-25, profile at X=0 unchanged, extrude landing at X=-25..+45. Double-verified across two sessions (slider head, plus stop_fin/stop_relief in section 41). Open question: works on principal planes; not yet confirmed on offset construction planes (should, since it operates in the sketch normal).

## 36. Multi-body cut via participantBodies

A single Cut extrude can cut several bodies at once by listing them in `participantBodies`. One timeline feature, guaranteed coaxial across all listed bodies.

```python
ext_in = extrudes.createInput(sk.profiles.item(0),
    adsk.fusion.FeatureOperations.CutFeatureOperation)
ext_in.setSymmetricExtent(adsk.core.ValueInput.createByString('pin_bore_len'), True)
ext_in.participantBodies = [body, lid]   # cuts both in one feature
feat = extrudes.add(ext_in)
feat.name = 'pin_bore'
```

Each body's volume drops by the portion of the cut intersecting it, independently, no double-counting. Verified on the test part's pin bore: a ~472 mm^3 cylinder removed 231 mm^3 from body and 170 mm^3 from lid, each matching per-body geometry. Use for pin bores through interleaved parts, dowel holes, slots crossing both halves of a hinge. NOT for joins: `JoinFeatureOperation` is one-to-one (one participant body); multi-body joins are not meaningful.

## 37. Composite body via NewBody + Join with overlap

To build one body from simple primitives (T-section, L-section, stepped profile), extrude each shape and Join them rather than constructing a complex closed polygon. Avoids the closed-polygon constraint issues in section 43.

```python
# 1. First primitive as NewBody, named
ext_in_1 = extrudes.createInput(sketch_a.profiles.item(0),
    adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
# ... set extent ...
feat_1 = extrudes.add(ext_in_1)
composite_body = feat_1.bodies.item(0)
composite_body.name = 'slider'

# 2. Second primitive as Join, targeting the body from step 1
ext_in_2 = extrudes.createInput(sketch_b.profiles.item(0),
    adsk.fusion.FeatureOperations.JoinFeatureOperation)
# ... set extent, overlapping step 1's range by 0.05-0.1mm ...
ext_in_2.participantBodies = [composite_body]
extrudes.add(ext_in_2)
```

Critical: Join needs NON-ZERO volume overlap. Bodies that merely touch at a planar face are not guaranteed to join. Design a tiny intentional overlap (0.05-0.1mm). Verified on the test part's slider: head (Z=1.0..2.4) + neck (Z=0..1.1, 0.1mm into the head) joined to one 4183.2 mm^3 body, exact. Each primitive is independently constrained via section 2/30, far easier than an 8-vertex T-polygon. Variant: a symmetric Join across a body face (half inside for the join, half outside) adds an external protrusion, e.g. a finger grab.

## 38. Bodies to components, preserving world position

Fusion joints operate on COMPONENTS (occurrences), not bodies. A body-only design must be restructured into components before any joint or motion work. This is the enabler for sections 39-42 AND for exporting separate parts laid out on one plate.

```python
occ = root.occurrences.addNewComponent(adsk.core.Matrix3D.create())  # identity transform
occ.component.name = 'holder_body'
root.bRepBodies.itemByName('body').moveToComponent(occ)   # fetch FRESH each time; moving mutates the collection
root.bRepBodies.itemByName('pin').moveToComponent(occ)    # several bodies can share a component
occ.isGrounded = True                                     # anchor the fixed base
```

- Identity occurrence transform preserves world position EXACTLY (bboxes to 0.01mm).
- After conversion `root.bRepBodies.count == 0`; reach bodies as proxies via `occ.bRepBodies.itemByName(...)`.
- Historical features stay in the root timeline and remain parametrically editable.
- NEW features can be added to sub-components after conversion (sketch on the sub-component's own `xYConstructionPlane`, which equals world for an identity occurrence). Verified: built a fin in one component and a relief cut in another, both parametric, after componentizing.

Verified: 4 bodies became 3 components (holder_body [body + pin, grounded], lid, slider).

## 39. As-built joints (revolute hinge + slider)

As-built joints define motion between two occurrences IN THEIR CURRENT POSITIONS (no repositioning needed).

```python
ji = root.asBuiltJoints.createInput(occurrenceOne, occurrenceTwo, jointGeometry)
ji.setAsRevoluteJointMotion(direction)   # or setAsSliderJointMotion(direction)
j = root.asBuiltJoints.add(ji); j.name = 'hinge_revolute'
```

Geometry and direction:
- `JointGeometry.createByNonPlanarFace(cylFace, adsk.fusion.JointKeyPointTypes.MiddleKeyPoint)` -> frame Z = cylinder axis (use for a hinge).
- `JointGeometry.createByPlanarFace(face, None, adsk.fusion.JointKeyPointTypes.CenterKeyPoint)` -> frame Z = face normal (use for a slider).
- `JointDirections` (X/Y/Z/Custom) are relative to the JOINT GEOMETRY frame, not world. Build the geometry so the desired axis is frame Z, then use `ZAxisJointDirection`.
- occurrenceTwo's geometry (e.g. a pin inside the grounded body) is valid for positioning.

Driving the joint (V11): `jointMotion` value props are read/write. `RevoluteJointMotion.rotationValue` is radians; `SliderJointMotion.slideValue` is cm. Re-fetch the body proxy after driving to read the updated bbox. CAUTION: a recompute resets driven joints (gotchas.md "Driving a joint does not persist through a recompute"); always re-drive as the final step before screenshot or export.

## 40. Joint limits

Constrain motion to the physically valid range.

```python
lim = js.jointMotion.slideLimits          # rotationLimits for a revolute joint
lim.isMinimumValueEnabled = True; lim.minimumValue = 0.0
lim.isMaximumValueEnabled = True; lim.maximumValue = 2.0   # cm (sliders); radians (revolute)
lim.isRestValueEnabled  = True; lim.restValue  = 0.0
```

Units are joint units: cm for sliders, radians for revolute. Setting `restValue` is the mitigation for the joint-drive-reset gotcha. IMPORTANT: a joint limit is a MOTION-STUDY constraint only, NOT a physical stop in the printed part. For a real mechanical stop, see section 41.

## 41. Internal travel-stop for a print-in-place slider

UNPROVEN technique (geometry-verified only; never exercised on a working print, because the test print fused, see section 34). Recorded for the captured-slider case it was designed for; it is NOT the recommended approach if you print separate parts.

A hard mechanical stop so a printed slider physically cannot pass its open position (the joint limit in section 40 is CAD-only). A fin on the MOVING part (Join) rides in a relief pocket cut into the FIXED part (Cut); the pocket's far wall is the stop. Solves "the head covers the stop location when closed" by putting the fin in a sliver above the head that the head never occupies.

Test-part values: stop_fin (Join to slider) box X=10..13, Y=+/-4, Z=2.3..3.0 (0.1mm into head top for the join); stop_relief (Cut from body) X=9..33, Y=+/-5, Z=2.5..3.3, with the +X wall at 33 = fin leading edge (13) + travel (20). Clearances Y +/-1, Z 0.3, -X 1mm. Both are offset-start extrudes (section 35); both sketches fully constrained. Hard stop at 20mm verified in CAD; volumes exact (+14.40, -192.00 mm^3).

Two caveats before reusing this:
- Designed for print-in-place CAPTURE. The whole fin-in-relief design only makes sense for a captured slider. If you print separate parts (the recommended path for this class, section 34), retention changes to a snap-in detent, an end cap, or a removable stop.
- Unproven on a real print. The fused first print never let the catch be tested. Treat the geometry as a starting point, not a validated mechanism.

## 42. Reorient a grounded jointed assembly for print/export

To rotate a whole assembly for print orientation without breaking joints or grounding, transform the GROUNDED BASE occurrence and let the joints carry the children.

```python
import math
# 1. Neutralize driven joints first (so transforms are known).
jh.jointMotion.rotationValue = 0.0; js.jointMotion.slideValue = 0.0
# 2. Unground the base, set its transform2 to the desired world transform.
occ_body.isGrounded = False
M = adsk.core.Matrix3D.create()
M.setToRotation(math.pi, adsk.core.Vector3D.create(1,0,0), adsk.core.Point3D.create(0,0,0))  # 180 deg about world X
M.translation = adsk.core.Vector3D.create(0, 0, dz_cm)   # seat: dz = -(assembly minZ after rotation)
occ_body.transform2 = M
occ_body.isGrounded = True                               # re-anchor in the new orientation
# 3. Re-drive joints LAST (grounding triggers a recompute that resets driven values).
jh.jointMotion.rotationValue = math.radians(180); js.jointMotion.slideValue = 0.0
```

Verified on the test part (V12):
- Moving the grounded base via `transform2` drags the jointed children rigidly; relative poses are preserved so the as-built joints stay satisfied. Lid and slider followed a 180-degree X flip with no child transforms set.
- Setting an ABSOLUTE `transform2` cleanly REPLACES the orientation; reassign a fresh matrix to switch poses.
- `Matrix3D.setToRotation(angle, axis, origin)`; `Matrix3D.translation` is a read/write `Vector3D` applied AFTER the rotation, giving a clean flip + seat.
- Seat on plate: after rotation, assembly minZ = -(extent that rotated to the bottom). Set `M.translation.z = -minZ`, measuring minZ across proxy bboxes WITH joints in their final driven pose (an open lid changes the lowest point).
- Re-ground, then drive joints AFTER grounding. Always re-drive joints as the LAST step before screenshot or export.

Print-orientation note: keep a print-in-place hinge axis HORIZONTAL on the bed (gotchas.md). On the test part the hinge axis is world X; the 180-degree X flip keeps it horizontal, while a side-lay via -90 deg about Y would make it vertical and was rejected.

## 43. Sketch constraint anti-patterns

Avoid these; each one produces a sketch the API reports as constrained while the UI disagrees (or otherwise wastes effort). Use the section 2 corner-anchor approach instead.

- **Diagonal construction line + midpoint centering.** Draw a SW-to-NE diagonal, `addMidPoint(origin, diagonal)`, then H/V + size dims. `isFullyConstrained` reads True, but the UI shows blue lines / no lock badge. Root cause: sign ambiguity, the corners can satisfy all constraints in any of 4 quadrant assignments. Section 2's positive distance dimensions to a SPECIFIC corner remove the ambiguity.
- **Assuming `addCenterPointRectangle` adds constraints.** The UI Center Rectangle tool adds H/V/symmetric automatically; the API call creates ONLY the four lines. Add constraints manually (section 2).
- **`sketch.project()` on an in-plane construction axis.** `sketch.project(root.xConstructionAxis)` on an XY sketch returns an empty collection (the axis is already in-plane). Cannot use it to import a model axis as a symmetry reference; draw and dimension a construction line, or use the corner-anchor pattern.
- **Symmetric constraints with manually-drawn axis lines.** Anchoring axis-line start points to origin and using `addSymmetry` leaves the axis lines' END points (their lengths) as free DOFs, so `isFullyConstrained` stays False. Adding length dims fixes it but buys nothing. Use section 2 instead.
- **RectangularPattern with `isSymmetricInDirectionOne` for Join extrudes.** Produces wrong/disconnected bodies (gotchas.md). Use Mirror (section 33) or place instances manually.
