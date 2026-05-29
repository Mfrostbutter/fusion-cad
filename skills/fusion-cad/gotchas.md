# Fusion MCP Gotchas

Failure modes from production sessions plus stress-test findings. Each entry: what happens, why, the fix.

## Sketch plane orientation surprises

**XY plane (safe default).** Sketch X maps to world X, sketch Y maps to world Y. Extrude in `+Z` direction goes up. No surprises.

**XZ plane.** Sketch X maps to world X. Sketch Y maps to NEGATIVE world Z. For Z-up geometry, negate all Y values in your vertex list. A typical symptom: a side-profile bracket drawn Y-flipped on first attempt, only caught by the bounding box because the iso-top-right screenshot still looked plausible.

**YZ plane (`yZConstructionPlane`).** Sketch X maps to NEGATIVE world Z (inverted). Sketch Y maps to POSITIVE world Y. Normal/extrude direction is world X. So "up" in the sketch is sketch X with an inverted sign relative to world Z, and the sketch's vertical axis is sketch X, not Y. To place a point at world Z=80, use sketch X = -80; for world Y=20, use sketch Y = +20.

Verified universal in current Fusion (build `4bca736941837d3e42bba21bb36b9891e34b2fce`, May 2026) across two independent documents with a `worldGeometry` repro: a sketch point at (0.5, 1.0) cm reads world (0, 10mm, -5mm). Re-confirm if plane behavior changes in a future build. (The original guess for this entry was wrong, caught by the Dell-bracket session.)

**Habit for any non-XY plane.** Before committing to a coordinate scheme, drop one test `sketchPoint` and read its `worldGeometry` to confirm the mapping. Five lines, saves a full re-orientation debug cycle. See patterns.md N8 for a constrained yZ-plane worked example.

**Rule.** Default to XY for top-down designs (trays, panels, anything that lays flat). XZ for side-profile parts (brackets, arms), remembering to negate Y. Avoid YZ unless extrusion along world X is specifically wanted.

## Self-intersecting polygon returns `profiles.count == 2`

A polygon where two edges cross internally (e.g. an 8-vertex Z-profile where edges V8 to V1 cross V5 to V6, producing a figure-8) is treated by Fusion as TWO closed profiles, not an error. The extrude succeeds and produces wrong geometry.

**Rule.** Always `assert sk.profiles.count == <expected>` immediately after `sk.isComputeDeferred = False`. Print the count if uncertain. Bail before extruding.

Profile count of 0 means the polygon did not close (vertex typo or float mismatch).

## Undo wipes whole-script transactions silently

`update(undo)` reverses the last `execute` as ONE atomic transaction. If that script added params AND created geometry, undo wipes both. Common symptom: "wait, why are these params missing?" several scripts later.

**Same rollback on exception-driven failure.** It is not only user undo. If a script throws partway (say, on `params.add` call 6 of 25), the 5 adds that already succeeded are silently wiped too. The tool error shows the traceback for the failing call but does NOT report which prior changes rolled back. Re-running an idempotent script after the fix then picks up from the last committed state, not the failed transaction's intermediate state.

**Rule.** Make every script idempotent (params, bodies, sketches all checked-and-added). When rollback is plausible, split into two `execute` calls: params first, geometry second. Prefer the delete-loop cleanup over undo when resetting geometry.

## `apiDocumentation` with null apiCategory returns empty silently

Omitting `apiCategory` returns `{"message": "API documentation query completed", "success": true}` with NO data. Looks like success.

**Rule.** Always set `apiCategory` to `"member"`, `"class"`, `"description"`, or `"all"`.

## Internal units are cm, not mm

`userParameters.itemByName(name).value` returns CM. `Point3D.create(x, y, z)` takes CM. A 30mm parameter has `.value == 3.0`.

`ValueInput.createByString('30 mm')` accepts unit suffixes and Fusion converts. Use this for human-readable expressions.

**Trap.** Treating `.value` as mm and multiplying sketch coords by 10 doubles every dimension.

**Rule.** Only multiply by 10 when PRINTING dimensions for human display. Math inside scripts stays in cm.

## Screenshot `direction: "current"` auto-fits in most cases, but not all

`direction: "current"` auto-fits the camera to show the body in the common case. It does NOT preserve the current viewport camera; it re-fits.

Two known failure modes:

- Untitled docs: auto-fit is unreliable, often returns an empty frame.
- Immediately after script-driven feature additions: the camera may not have settled, fit can miss.

**Rule.** Always run the explicit fit-view block (patterns.md section 20) before screenshotting. Treat `direction: "current"` auto-fit as a happy-path bonus, not a guarantee.

## Camera reset has no MCP primitive (CLOSED 2026-05-22)

Resolved via the Python API from within an `execute` script. The MCP itself doesn't expose a camera primitive directly, but Fusion's `activeViewport.fit()` does what's needed:

```python
vp = app.activeViewport
vp.fit()
cam = vp.camera
cam.viewOrientation = adsk.core.ViewOrientations.IsoTopRightViewOrientation
cam.isFitView = True
vp.camera = cam
vp.refresh()
```

See patterns.md section 20. Verified across multiple successive design iterations; produces consistent framing every time.

Previous workarounds (asking the user to press Home, or hoping `direction: "current"` happened to do the right thing) are no longer needed.

## Named-direction screenshots still need an explicit fit-view call

Closing the camera-reset gap (above) does NOT change the fact that named directions like `iso-top-right` don't auto-fit on their own. If you call `read` screenshot with a named direction WITHOUT first running the `vp.fit()` pattern from patterns.md section 20, the result is whatever zoom the viewport happened to be at, often a tight crop or empty frame.

**Rule.** For deterministic screenshots, ALWAYS run the patterns.md section 20 fit-view block first, then take the screenshot. Even for `direction: "current"`. Controlling the camera explicitly is more reliable than depending on screenshot-side auto-fit behavior.

## `setOneSideExtent` with fixed distance stops at obstructions

A counterbore cut using `setDistanceExtent(floor_thickness)` to cut up from a gusset bottom can stop short when the gusset itself sits below the sketch plane. The fixed distance only clears the upper feature, leaving the cut blocked.

**Rule.** When geometry below the sketch plane is variable or unknown depth, use `setAllExtent(direction)` for cut and intersect operations. Cuts through everything in that direction.

## Do not catch exceptions in `run()`

```python
# BAD
def run(_context: str):
    try:
        # ... work ...
    except Exception as e:
        print(f"failed: {e}")  # loses the traceback
```

The MCP returns Python exceptions with full traceback as the tool error. Catching them turns a 30-second fix into a 10-minute debug because the line number and stack frame are gone.

**Rule.** Let exceptions propagate. Trust the tool error path.

## Counterbore vs countersink direction

For parts where screws drive upward into the part (e.g. mounting brackets installed from below), the counterbore must be cut UP from the screw-tab bottom, not DOWN from the top. Easy to get wrong on the first attempt by defaulting to "screw enters from the top".

**Rule.** Trace the actual screw direction in the install before placing the counterbore. Place the counterbore sketch on a plane positioned at the bottom-facing side, then `setAllExtent(NegativeExtentDirection)` to cut up.

## Save on Untitled documents fails via MCP

Calling `execute document save` on a never-saved doc returns `{"error": "Document 'Untitled' is new and must be saved by the user first.", "success": false}`. No dialog, no UI prompt. Clean refusal.

**Rule.** Initial SaveAs must happen in the Fusion UI. After that, MCP `save` works for revisions. Surface this to the user when they ask Claude to save a brand-new design.

## Close on dirty documents requires confirmation flag

`execute document close` on a dirty doc returns: `"has unsaved changes. Confirm with the user before closing. Specify userConfirmedSaveAndClose or userConfirmedCloseWithoutSave in the request."`

Also: `userConfirmedSaveAndClose: true` on an Untitled doc still fails with the same "must be saved by the user first" error, because the save step inside save-and-close hits the same wall.

**Rule.** Never auto-pick which confirmation flag to send. Surface the choice to the user. The `saved: false` field in the close response is the audit trail for "did the agent discard changes?"

## Export path requirements

- Relative paths fail with `RuntimeError: 3 : The selected folder does not exist.`
- Protected paths (e.g. `C:/Windows/System32/`) fail with `RuntimeError: 3 : The selected folder is not accessible.`
- Missing parent folder likely produces the same "does not exist" error.
- Overwrite is SILENT. Existing file clobbered without prompt.

**Rule.** Always absolute paths. Always `os.makedirs(dirname, exist_ok=True)` before export. If preserving prior versions matters, add a suffix to the path.

## `createC3MFExportOptions` capital C

The 3MF export function is `em.createC3MFExportOptions`, not `em.create3MFExportOptions`. Common signature-guess miss. The C prefix is consistent with how Fusion versions its color-capable formats.

## `isRollingBall = True` matters for fillets

Default `FilletFeatureInput` geometry is NOT the spherical-corner blend most CAD users expect. Set `isRollingBall = True` for the standard blend.

## `isComputeDeferred` matters for >10 vertex sketches

`sk.isComputeDeferred = True` before bulk `addByTwoPoints` calls, `= False` after. Without it, every line add re-solves the sketch. Below 10 vertices the difference is negligible; at 50+ Fusion freezes.

## HoleFeatureInput is worth using over sketch+cut

It is tempting to bypass `HoleFeatureInput` because the signature looks awkward, but it works on first attempt with `createCounterboreInput(holeDia, cboreDia, cboreDepth)` and `setPositionByPoint(face, point3d)`.

**Rule.** Use `HoleFeatures` for standard holes (simple, counterbore, countersink). Parametric, named, editable in timeline, cleaner than two extruded cuts. Reserve sketch+extrude-cut for non-standard hole geometry.

## Project ID format mismatch

`document/open` returns `parentProjectId` in base64 form (e.g. `a.YnVzaW5lc3M6Z21haWw...`). `document/recent`, `document/search`, and `projects` return the same project as a decimal ID (e.g. `202602051047590776`). Same project, two formats. Use whatever the consuming call accepts; not yet confirmed if `document/open` for fileId accepts either form.

## Flange overhangs need slicer tree supports

CAD-side note: 10mm+ horizontal flange overhangs require tree supports in the slicer. No CAD-side fix short of redesigning as a chamfered skirt. Worth flagging when proposing flanged designs.

## `defaultLengthUnits` is read-only via script

Display units (mm vs cm) must be set via the Fusion UI or document template, not via script. Internal geometry is always cm regardless of display units; this is cosmetic only.

## `setDistanceExtent` direction is implicit on HoleFeature

For `HoleFeatureInput.setDistanceExtent`, the through-direction comes from the face normal passed to `setPositionByPoint`. No explicit `ExtentDirections` argument needed. Different from `ExtrudeFeatureInput`, which requires explicit direction.

## Redo stack is consumed by new features

After `undo`, any new feature added via `execute` consumes the redo stack. Subsequent `redo` returns `{"error": "Nothing to redo", "canUndo": true, "canRedo": false, "success": false}`. Fails gracefully.

**Rule.** The skill can rely on `canUndo` and `canRedo` flags in the returned JSON for branching logic. Document that any new feature kills the redo stack.

## Untitled docs have no fileId

`document/open` (read) lists Untitled docs without an `id` field. Cannot pass them to `document/open` (execute) since fileId is required. Untitled doc workflows must start from creating fresh in the Fusion UI or saving the doc first.

## Document search treats `-` and `_` as equivalent

Query `My-Part` matches docs named `My_Part...`. Useful for fuzzy matching across naming conventions.

## `participantBodies` is required for cuts that target a specific body

When multiple bodies exist in a component and a cut should only affect one:

```python
cut_in.participantBodies = [body]
```

Without this, Fusion may pick the wrong body or apply the cut to all candidate bodies. Always set when more than one body is present. Also required for the Join extrude pattern (`patterns.md` section 23) so the lip flange joins onto the body rather than creating a new disconnected solid.

## CAD volume is NOT a filament-weight estimate

`body.volume` (returns cm^3) is the geometric solid volume. Real print weight is dramatically lower because slicers use sparse infill, fixed wall counts, and don't fill cavities the way a solid would.

Symptom: a 306 cm^3 tray modeled in Fusion estimates "~310g of ABS at solid density" but actually prints at ~90g (30% infill, 3 perimeters, 5 top/bottom layers). Pricing off the solid number kills margin; pricing off a guess is just as bad.

**Rule.** Treat any pre-slice filament weight as provisional. Slice the actual STL in Bambu Studio (or whichever slicer) and read grams + print time from there. Lock COGS only after the first verified print confirms the slicer numbers.

## `document/open` (execute) requires fileId in urn form

The `fileId` parameter for `execute document open` is the `urn:adsk.wipprod:dm.lineage:...` form returned in the `id` field of `document/search` or `document/recent` results.

**Workflow.** `read document search` (or `recent`) -> copy the `id` field -> pass as `fileId` to `execute document open`. Project ID is NOT a substitute and isn't required for the open call.

## Tapered cut needs the third arg form of `setOneSideExtent`

`setOneSideExtent(extentDef, direction)` is the standard signature. To add taper (countersink-style cone), pass a third `ValueInput` for the taper angle:

```python
ext_def = adsk.fusion.DistanceExtentDefinition.create(
    adsk.core.ValueInput.createByString('cs_depth'))
taper_vi = adsk.core.ValueInput.createByString('-cs_angle / 2')  # negative = inward
cut_in.setOneSideExtent(ext_def,
    adsk.fusion.ExtentDirections.NegativeExtentDirection,
    taper_vi)
```

Easy to miss the third positional arg. For standard countersinks, prefer `HoleFeatures.createCountersinkInput` (`patterns.md` section 25); it's parametric and editable in the timeline.

## STL refinement is a no-op for flat geometry, real for curves

`MeshRefinementLow`, `MeshRefinementMedium`, `MeshRefinementHigh` produce identical files for any flat-faceted geometry. The difference only shows up on curved surfaces (fillets, revolves, lofts), where High gives smoother facets at the cost of file size.

**Rule.** Default to `MeshRefinementHigh` for production exports. Cost on flat-dominated geometry is zero (identical output); benefit on curve-heavy parts is significant.

## Export artifact cleanup is your responsibility

The MCP writes export files wherever you point them, never warns on overwrite, and never cleans up. Test runs that wrote to `C:/temp/test_body.stl` etc. accumulate over sessions.

**Rule.** Use a scoped sub-folder for throwaway exports (e.g. `C:/temp/fusion-tests/`) that you can delete in bulk. Production exports should always go to a project-pathed destination, never `C:/temp`.

## Math-function names are reserved as user-parameter names

`floor` is not a valid user-parameter name; Fusion's expression language reserves it for the math function `floor(x)`. `params.add('floor', ...)` raises `RuntimeError: 3 : param name is not valid` with no indication of which name failed.

Likely also reserved (not exhaustively tested): `ceil`, `abs`, `sqrt`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `log`, `ln`, `exp`, `min`, `max`, `pi`, `e`. Plain English words like `wall` are fine, so the rule is specifically Fusion expression-language identifiers, not arbitrary words.

**Rule.** Suffix any potentially-colliding name: `floor_thk`, `min_clearance`. Adopt a `<noun>_<adjective>` convention for thickness, depth, and length params (`wall_thk`, `body_height`).

## Stdout buffering can hide which loop iteration failed

When a script crashes mid-loop, prints from before the crash may not flush in time. Symptom: the log shows iterations 1-5 succeeding, iteration 6 has no log line, yet the traceback fires from inside iteration 6.

**Rule.** Use `print(name, flush=True)` for diagnostic prints when narrowing a failing iteration, or print AFTER each successful op so the last printed name is the last success and the failure is whatever came next in the input list.

## `dim.parameter.expression = bare_param_name` may not bind on first assignment

Setting `dim.parameter.expression = 'body_width'` immediately after `addDistanceDimension` can silently fail to bind. After the script: `dim.parameter.expression == '70 mm'` (numeric), `dependencyParameters == []`, and `sketch.isFullyConstrained == True` (the API thinks it is fine) while the Fusion UI shows the sketch under-constrained (no lock badge). The API does not error, the constraint count looks right, but the parametric link is broken. The most insidious gotcha in this batch.

Verified behaviors:
- Bare param name on first assignment: often fails silently, stored as the numeric value at evaluation time.
- Math expression with the param (`'body_width / 2'`, `'body_width + 0 mm'`): binds reliably on first try.
- Re-assigning the bare name after the first attempt: the second assignment sticks.
- Reproduced across 4+ separate sketches. `addDiameterDimension` (patterns.md N7) bound a bare name on first try, so this is NOT 100 percent deterministic across dimension types; it appears specific to `addDistanceDimension`.
- Extrude extents and construction-plane offsets do NOT have this issue (see patterns.md V2). This affects sketch dimensions only.

**Workarounds (any one works):**
```python
# A: use a math expression
dim.parameter.expression = 'body_width + 0 mm'
# B: re-assign at the end of sketch construction (cleanest)
dim.parameter.expression = 'body_width'   # may not stick
# ... continue building ...
dim.parameter.expression = 'body_width'   # this one sticks
# C: verify and retry
if dim.parameter.dependencyParameters.count == 0:
    dim.parameter.expression = 'body_width'
```
Always check `dependencyParameters` to confirm the expression took parametrically. Open question: whether `design.computeManager.compute()` between creation and assignment fixes it (untested).

## Lock badge is ground truth, not `isFullyConstrained`

A tiny lock overlay on the sketch icon in the browser tree means fully constrained; absence means under-constrained. `Sketch.isFullyConstrained` can return True under conditions where the UI still considers the sketch incomplete (see the bare-name binding gotcha above, and the diagonal-midpoint anti-pattern in patterns.md). During constraint development, the lock badge is ground truth, not the API flag.

In-canvas line colors are the other tell: black = driving + fully constrained, blue = driving + under-constrained, red = over-constrained / conflict. The small red "pencil tip" in the default sketch icon is NOT a constraint indicator; the lock OVERLAY is. When in doubt, look at the line colors in edit mode.

## RectangularPattern with `isSymmetricInDirectionOne=True` makes wrong or disconnected bodies

Patterning a `JoinFeatureOperation` extrude (single knuckle at X=0) with `quantityOne=3`, `distanceOne='40 mm'`, `ExtentPatternDistanceType`, and `isSymmetricInDirectionOne=True` does NOT produce 3 instances at X=-20, 0, +20. It produces 6 new disconnected bRepBodies at X plus or minus 40 (body count went 2 to 8), and the Join did not merge. The result matches no standard interpretation of the inputs.

**Workaround.** Use `MirrorFeatures` (patterns.md N10, verified V7) for symmetric replication, or place instances manually on offset construction planes. Root cause unresolved but reproducible. Until investigated, prefer Mirror.

## SketchPoint is not hashable

`adsk.fusion.SketchPoint` cannot go in a `set()`. To find a corner, collect into a LIST and use `min(pts, key=lambda sp: (sp.geometry.x, sp.geometry.y))`. Applies to most Fusion API objects; prefer lists plus key functions over sets.

## Driving a joint does not persist through a recompute

Setting `jointMotion.rotationValue` / `slideValue` repositions the model, but adding a feature or any recompute (including setting `transform2` or `isGrounded`) resets driven joints to their rest position. Observed repeatedly: a lid driven to 180 degrees snapped back to 0 after new features were added and on every reorientation. Mitigations: set `restValue` on the joint limits (patterns.md N15); always re-drive joints as the final step before screenshot or export. Treat joint drive state as transient.

## Print-in-place at tight clearances can fuse on FDM (CAD-side print note)

PRINTER-VERIFIED on a captured slider+hinge test part: a print-in-place slider plus hinge at 0.2mm rotational / 0.3mm Z / 1mm Y clearances FUSED on the first Bambu print (moving parts welded to the housing). For this class of part, prefer SEPARATE parts laid out on one plate with assembly clearances (looser, tunable, sandable) over a captured print-in-place mechanism, and use a separate printed or filament/rod pin through the hinge knuckles rather than a print-in-place knuckle. See patterns.md N11 and N17 for the parametric clearance technique and the separate-parts recommendation.

IF you do commit to a print-in-place hinge, keep its axis HORIZONTAL / parallel to the build plate so the knuckle-separation gaps print as vertical seams (reliably free). Laying the part so the hinge axis is VERTICAL turns those gaps into horizontal layer planes that fuse easily, giving a stiff or welded pivot. Same family as the "flange overhangs need slicer tree supports" note above.
