---
name: rhino-nurbs-drawings
description: Use when drawing any 2D line drawing directly in Rhino by script, such as elevations, sections and plans, objects and products, diagrams, illustration, botanical or anatomical studies, patterns, ornament, lettering and other curve-heavy linework, with NURBS curves, line weights, hatches and vector export. Researches the subject (sources, dimensions, reference images) before drawing. Not for 3D modelling or sheet layout.
---

# 2D drawing in Rhino by script

This is a general drawing tool. It draws flat linework directly, whatever the subject: a building elevation, a chair in profile, a flower, a machine part, a pattern, a diagram. It does not build 3D models and does not use Make2D or any hidden-line projection, because projected linework comes out fragmented and needs cleaning. For 3D modelling use the `rhino-nurbs-modelling` skill.

It makes the drawing itself, not a presentation sheet: no border, title block, labels, scale bar or legend unless the user asks for them. It adds no shadows (cast, reveal or shading) unless the user asks; line weight and clipping carry the depth. Architectural drawings are one use among many; the building examples below are examples, not defaults.

## Core rules (read first)

0. **Research before you draw.** Before any setting-out or code, research the subject: search the web for authoritative texts (with dimensions), the most frontal reference image, close-ups of every detail, and any existing drawings, and save what you find in the job folder. Do this even when the user supplies a reference image: one image is a starting point, not a source. Run the research in a background subagent if one is available, set up the document meanwhile, and wait for the findings before fixing the setting-out. Research again whenever a check, a capture or a reviewer shows that something is uncertain. For invented or stylistic drawings, research the style: the designer's, artist's or period's real examples, not a generic idea of it. Only a purely abstract drawing built from rules the user gives (a pattern, a diagram of their own) needs no research; say so and skip. Details in "Sources first".
1. **Write scripts, never use primitive tools.** Create geometry only through Python scripts against RhinoCommon. Do not call MCP tools such as `create_object` or any "add box/circle" tool: they are the reason output collapses into circles and rectangles.
2. **Draw in 2D, on the World XY plane, at z = 0.** No 3D model, no Make2D, no `HiddenLineDrawing`. A view is drawn as that view: closed outlines for each element, curves for what is seen.
3. **Depth is built, not computed.** Every element belongs to a depth band (back to front). Draw back to front, clip whatever passes behind a nearer closed outline, and let the band decide the line weight.
4. **Curves come from rules, not typed coordinates.** Use setting-out numbers, loops and helper functions (stems, ribbons, tweens, offsets, mirrors). Hand-typed point lists are where coordinates go wrong.
5. **One pass per script, run it, check it, then continue.** Keep each script under about 80 lines of logic.
6. **Numbers gate, pictures advise.** A pass is accepted only when its numeric checks pass. A capture is a second check for gross errors. Never read a dimension off a capture of your own drawing; dimensions come from sources, or are scaled from a frontal reference photograph against one known dimension and labelled as estimates. Never report "done" because a script ran without errors.
7. **Look up before calling.** Use the calls in this file as written. For anything else, check inside Rhino first with a three-line script that prints `hasattr(rg.Curve, "Name")`, or read `https://developer.rhino3d.com/api/rhinocommon/`. Do not guess a function name.
8. **Stop refining after three rounds per pass.** Keep the best version and report what still differs.
9. **Checks verify geometry, not fact.** A drawing can pass every numeric check and still be wrong about what it shows. Accuracy needs its own check: the element table from the research (see "Checks") and an independent review against the references.
10. **Keep every version.** A rebuild goes on its own layer tree (`JOB_V2`, `JOB_V3`) and its own script folder. Never delete or overwrite the previous version; place the new one beside it for comparison.

## Environment

Checked on Rhino 9.0.26300 for Windows with Python 3.13.13 on 23 Nov 2026. Every helper and call in this file ran there, including viewport capture and PDF, SVG and DWG export (first used on a full facade drawing the same day). The export block now sizes the page from the drawing extents instead of a fixed sheet; that framing has not been run yet, so check the first PDF. The same scripts are expected to run on Rhino 8 (Python 3.9).

Rules for the user's document:
- Read `doc.ModelUnitSystem` and `doc.ModelAbsoluteTolerance` first and write them in the intent file. If the smallest feature is under about 20 times the tolerance, joins, booleans and hatches will fail: tell the user and suggest a new document in millimetres with a tolerance of 0.01 or finer. Do not change their document's tolerance yourself.
- The open document may hold the user's own work. Create everything under one parent layer named for the job, touch only objects on that layer tree, and never loop over `doc.Objects` as a whole.
- Offer to work in a new empty document when the open one is large.

## Step 0: choose the route

| Route | When | How scripts run | How results come back |
|---|---|---|---|
| B. McNeel Rhino MCP (built into Rhino 9; `Rhino-MCP-Platform` package on Rhino 8; started with `MCPConnect`) | Its tools are present: `list_slots`, `run_python`, `get_viewport_image` | Send each pass through `run_python` with the slot id | `run_python` returns what the script prints in `payload.stdout`, so the template prints the JSON report |
| A. `rhinocode` CLI | Rhino open on the same Windows machine, user has run `StartScriptServer`. Not yet tried on Rhino 9 | `RhinoCode.exe script C:\abs\path\pass.py`; the executable is in `C:\Program Files\Rhino 9\System` | The script writes a JSON report and PNG to disk; read them |
| C. Community `rhinomcp` | Its tools are present: `execute_rhinoscript_python_code`, `capture_viewport` | Use the script tool only; its timeout defaults to 15 s, so keep passes small | Tool output plus the JSON report |
| D. No Rhino | No live Rhino is reachable | ezdxf in a Python venv. See "Without Rhino" | DXF and SVG files rendered to PNG |

Prefer B on Rhino 9 and tell the user which route is in use. On route B:
- Call `list_slots` first and use the slot whose `version` matches the user's Rhino. Pass its `slotId` as `slot` on every call. If there is none, ask the user to open Rhino and run `MCPConnect`. Do not let a call start a new Rhino: the router starts Rhino 8 by default.
- If Rhino 8 keeps opening when Rhino 9 is wanted, the `rhino` entry under `mcpServers` in `~/.claude.json` needs `"args": ["-v", "9"]`. Tell the user; do not edit their config unasked.
- `run_python` injects `__rhino_doc__`. Use it as the document for everything. rhinoscriptsyntax acts on whichever document is in front, so this skill uses RhinoCommon on `doc` only.

## Workflow

0. **Sources first** (always; rule 0). If subagents are available, send one to research while you set up; otherwise search yourself. Search in the subject's own languages as well as English. Collect:
   - an authoritative text with known dimensions, at least one overall dimension. Which kind depends on the subject: a survey, monograph or heritage listing for a building; a manufacturer's datasheet, patent or catalogue for an object; a flora, herbarium sheet or anatomy text for a natural form; a standard or specification for a part;
   - the most frontal, highest-resolution photograph, plate or drawing you can find, to scale proportions from (scale it by the known dimension; state that other dimensions are scaled estimates);
   - close-up images of every detail element, saved into the job folder (`_refs\`), not a temporary folder.
   Then write an **element table** in the intent file: element, count, position (axis or offset), shape, material or surface, source. Check each count and position against two references. Where sources conflict, say which one you followed and why. One small photo is never enough: in testing, a facade read from one 245-pixel photo had a missing oriel, a missing mansard, wrong floor heights and the wrong number of columns.
1. **Intent file.** Before any code, write `intent.md` in the working folder:
   - drawing: subject, view (elevation, section, plan, profile, detail, diagram, free illustration), units, and scale if the drawing has one;
   - setting-out as named numbers: overall extents, axes, grids, repeat spacings and key sizes (for a building, bay widths, floor levels and opening sizes; for a chair, seat height, depth and leg spacing; for a flower, petal count and lengths);
   - depth bands from back to front, for example `BACK` (the furthest plane), `MID`, `FRONT` (the nearest projecting parts), `CUT` (anything sliced by the drawing plane);
   - every element with its band, whether it is a closed outline, and the rule that generates its curves;
   - the curved features named explicitly (for example "whiplash stems rising from each pier, tapering, branching twice", or "five petals, each a lens bent along its midrib").
   If the user gives master curves in Rhino or a reference image, describe them in words with dimensions first and confirm. Ask once if scale, extents or the element list are unknown.
2. **Expected values.** For each pass write the expected counts, extents and closed or open status before running it.
3. **Passes**, one script each: (a) layers and setting-out lines on a non-printing layer, (b) closed outlines per band, back to front, (c) secondary elements, (d) ornament and curve families, (e) clipping of everything that passes behind a nearer outline, (f) fills and hatches, (g) export.
4. **After each pass:** read the JSON report, fix execution errors (up to three retries), compare checks with expected values, then look at a capture.
5. **Final review.** If subagents are available, give a fresh one the intent file, the reports, the exported drawing, the reference images, read-only access to the live Rhino slot (camera moves only) and permission to search for more references. It must not see the build reasoning or the scripts. Ask it to score the drawing 1 to 10 on: accuracy of elements, proportions, silhouette, detail fidelity, line-weight hierarchy, depth, and material or surface detail; then list SHOULD fixes (errors) and COULD refinements, each concrete. When comparing versions, have it score both on the same rubric.
6. **Hand over** the files, the route used, check results, review scores, and what still differs.

## Script template and helpers

```python
#! python 3
import json, math, os, traceback
import Rhino, System
import scriptcontext as sc
import Rhino.Geometry as rg
from Rhino.Geometry.Intersect import Intersection
from System.Collections.Generic import List

OUT = r"C:\abs\path\to\job\_out"                 # absolute path, created beforehand
JOB, PASS = "JOB01", "p03_ornament"              # JOB is the parent layer name
doc = globals().get("__rhino_doc__") or sc.doc   # McNeel MCP injects __rhino_doc__
tol = doc.ModelAbsoluteTolerance
XY = rg.Plane.WorldXY
BLACK = System.Drawing.Color.Black
report = {"pass": PASS, "ok": False, "checks": {}, "ids": {}}

def pts(seq):                                    # [(x, y), ...] -> typed point list on z = 0
    l = List[rg.Point3d]()
    for x, y in seq: l.Add(rg.Point3d(x, y, 0))
    return l

def layer(path, plot_mm=0.0, color=BLACK):       # "JOB01::FRONT"; creates parents; returns index
    i = doc.Layers.FindByFullPath(path, -1)
    if i >= 0: return i
    parts = path.split("::")
    L = Rhino.DocObjects.Layer()
    L.Name, L.Color, L.PlotColor, L.PlotWeight = parts[-1], color, BLACK, plot_mm
    if len(parts) > 1:
        L.ParentLayerId = doc.Layers[layer("::".join(parts[:-1]))].Id
    return doc.Layers.Add(L)

def add(geo, band, name):                        # curve or hatch -> document on JOB::band
    a = doc.CreateDefaultAttributes()
    a.LayerIndex, a.Name = layer(JOB + "::" + band), name
    return doc.Objects.Add(geo, a)

def job_objects():                               # only what this job created
    objs = []
    for L in doc.Layers:
        if not L.IsDeleted and (L.FullPath == JOB or L.FullPath.startswith(JOB + "::")):
            objs.extend(doc.Objects.FindByLayer(L) or [])
    return objs

def clip(curve, region, keep_inside=False):      # parts of curve outside (or inside) a closed outline
    ev = Intersection.CurveCurve(curve, region, tol, tol)
    ts = List[System.Double]()
    for t in sorted(e.ParameterA for e in ev if e.IsPoint): ts.Add(t)
    pieces = list(curve.Split(ts)) if ts.Count else [curve]
    want = rg.PointContainment.Inside if keep_inside else rg.PointContainment.Outside
    return [p for p in pieces if region.Contains(p.PointAt(p.Domain.Mid), XY, tol) == want]

def ribbon(centre, w0, w1, n=24):                # closed tapering band around a centre curve
    left, right = [], []
    ts = list(centre.DivideByCount(n, True))
    for k, t in enumerate(ts):
        w = (w0 + (w1 - w0) * k / (len(ts) - 1.0)) / 2.0
        p, tan = centre.PointAt(t), centre.TangentAt(t)
        nrm = rg.Vector3d.CrossProduct(rg.Vector3d.ZAxis, tan); nrm.Unitize()
        left.append(p + nrm * w); right.append(p - nrm * w)
    a = rg.Curve.CreateInterpolatedCurve(pts([(q.X, q.Y) for q in left]), 3)
    b = rg.Curve.CreateInterpolatedCurve(pts([(q.X, q.Y) for q in right]), 3)
    caps = [rg.LineCurve(a.PointAtStart, b.PointAtStart), rg.LineCurve(a.PointAtEnd, b.PointAtEnd)]
    j = list(rg.Curve.JoinCurves(System.Array[rg.Curve]([a, b] + caps), 2.1 * tol))
    return j[0] if len(j) == 1 and j[0].IsClosed else None

def fill(regions, pattern="Solid"):              # nested closed outlines in one call become holes
    pat = doc.HatchPatterns.FindName(pattern)
    if pat is None and pattern == "Solid":       # a new document may not hold it yet
        pat = doc.HatchPatterns[doc.HatchPatterns.Add(Rhino.DocObjects.HatchPattern.Defaults.Solid)]
    return list(rg.Hatch.Create(System.Array[rg.Curve](regions), pat.Index, 0.0, 1.0, tol))

def check(curves):                               # numbers for the report
    boxes = [c.GetBoundingBox(True) for c in curves]
    prim = sum(1 for c in curves if c.IsLinear() or c.IsArc())
    return {"count": len(curves),
            "flat": all(abs(b.Min.Z) < tol and abs(b.Max.Z) < tol for b in boxes),
            "closed": sum(1 for c in curves if c.IsClosed),
            "short": sum(1 for c in curves if c.IsShort(tol)),
            "self_crossing": sum(1 for c in curves if Intersection.CurveSelf(c, tol).Count > 0),
            "duplicates": sum(1 for i, a in enumerate(curves) for b in curves[i + 1:] if rg.GeometryBase.GeometryEquals(a, b)),
            "primitive_share": round(prim / max(len(curves), 1), 2)}

def capture(name, w=2000, bb=None):              # ran on Rhino 9; pass bb to capture a close-up
    view = doc.Views.Find("Top", False) or doc.Views.ActiveView
    pen = Rhino.Display.DisplayModeDescription.FindByName("Pen")   # light background, line work only
    if pen is not None: view.ActiveViewport.DisplayMode = pen
    if bb is None:
        bb = rg.BoundingBox.Empty
        for o in job_objects(): bb.Union(o.Geometry.GetBoundingBox(True))
    sz = view.ActiveViewport.Size                # can be tiny (about 100 x 400) when Rhino runs behind the MCP
    h = int(w * sz.Height / float(max(sz.Width, 1)))
    view.ActiveViewport.ZoomBoundingBox(bb)
    vc = Rhino.Display.ViewCapture()
    vc.Width, vc.Height = w, h
    vc.ScaleScreenItems = False
    vc.DrawAxes = vc.DrawGrid = vc.DrawGridAxes = False
    vc.TransparentBackground = False
    bmp = vc.CaptureToBitmap(view)
    vp, k = view.ActiveViewport, w / float(max(sz.Width, 1))       # crop to the extents, or the job is a speck
    a, b = vp.WorldToClient(bb.Min), vp.WorldToClient(bb.Max)
    x0, x1 = sorted((a.X * k, b.X * k)); y0, y1 = sorted((a.Y * k, b.Y * k))
    m = 0.03 * max(x1 - x0, y1 - y0)
    x0, y0 = max(0, int(x0 - m)), max(0, int(y0 - m))
    x1, y1 = min(bmp.Width, int(x1 + m)), min(bmp.Height, int(y1 + m))
    if x1 - x0 > 50 and y1 - y0 > 50:
        bmp = bmp.Clone(System.Drawing.Rectangle(x0, y0, x1 - x0, y1 - y0), bmp.PixelFormat)
    path = os.path.join(OUT, name + ".png")
    bmp.Save(path, System.Drawing.Imaging.ImageFormat.Png)
    return path

try:
    doc.Views.RedrawEnabled = False
    # ... draw, then fill report["checks"] with check([...]) and expected values ...
    report["ok"] = True
except Exception:
    report["error"] = traceback.format_exc()
finally:
    doc.Views.RedrawEnabled = True
    doc.Views.Redraw()
    try: report["capture"] = capture(PASS)
    except Exception: report["capture_error"] = traceback.format_exc()
    with open(os.path.join(OUT, PASS + ".json"), "w") as f:
        json.dump(report, f, indent=1, default=str)
    print(json.dumps(report, default=str))
```

Keep the helpers in one file in the working folder and paste or import them into each pass, so every pass uses the same tested code.

Python in Rhino, traps that cost a round trip:
- `curve.Split(...)` with several parameters needs a `List[System.Double]`. A `System.Array[System.Double]` raises a TypeError.
- .NET `out` parameters come back in the return tuple.
- Pass typed collections where a method takes `IEnumerable<T>`: `System.Array[rg.Curve](list_of_curves)` or `List[rg.Point3d]`.
- .NET enumerables are iterators: wrap in `list(...)` before `len` or indexing.
- Enum members named `None` are spelled `NONE` on Rhino 9.
- Do not branch on `isinstance` for .NET types; pass the type you know or use `doc.Objects.Add(geometry, attributes)`.
- `region.Contains(point)` returns `Coincident`, not `Inside`, for a point on the boundary. To ask whether a small detail lies on a host's face, test the detail's bounding-box centre, not its start point.
- A polyline with two equal consecutive points makes `Offset` fail and `flatbar` return `None`. Build stepped meanders so no corner repeats.
- A closed outline whose wavy edge touches another of its own edges (a scalloped skirt whose crests reach the top line) self-intersects. Keep amplitudes clear of the other edges.
- Run a pass that edits earlier geometry (clipping) only once per build. To rerun, delete the job's objects and rebuild from pass 1; keep a `rebuild` cell that runs every pass in order and prints one line of check results per pass.
- Long passes (clipping hundreds of curves) can exceed the MCP's 120 s and move to the background; wait for the notification rather than starting another pass.

## Curve toolkit (all ran on Rhino 9)

| Need | Call | Note |
|---|---|---|
| Smooth curve through computed points | `rg.Curve.CreateInterpolatedCurve(pts(seq), 3)` | With end tangents: `(points, 3, rg.CurveKnotStyle.Chord, start_tangent, rg.Vector3d.Unset)` |
| Closed seamless curve | `rg.NurbsCurve.Create(True, 3, points)` | Periodic; points act as control points, so the curve never overshoots |
| Join two curves smoothly | `rg.Curve.CreateBlendCurve(a, b, rg.BlendContinuity.Curvature, bulge_a, bulge_b)` | Joins the end of `a` to the start of `b`. Lower bulges (for example 0.4) give a tighter blend than the default |
| Family between two curves | `rg.Curve.CreateTweenCurves(a, b, count, tol)` | Use for hair, reeds, drapery, graded mouldings |
| Parallel curve | `curve.Offset(XY, distance, tol, rg.CurveOffsetCornerStyle.Smooth)` | Returns an array; sign of distance sets the side |
| Thickness that tapers | `ribbon(centre, w0, w1)` | Closed outline, ready to fill or clip against |
| Branch leaving a stem tangentially | Start the child at `stem.PointAt(t)` with `stem.TangentAt(t)` as its start tangent | Checked: start angle to the stem was 0 degrees |
| Symmetry and repetition | `c.Transform(rg.Transform.Mirror(rg.Plane(axis_point, rg.Vector3d.XAxis)))`, `rg.Transform.Translation(dx, dy, 0)`, `Rotation`, `Scale` | Duplicate first: `c.DuplicateCurve()` |
| Merge or cut closed outlines | `rg.Curve.CreateBooleanUnion(array, tol)`, `CreateBooleanDifference(a, b, tol)`, `CreateBooleanIntersection(a, b, tol)` | Closed, planar, coplanar curves only |
| All regions made by crossing curves | `rg.Curve.CreateBooleanRegions(array, XY, False, tol)` | `.RegionCount`, `.RegionCurves(i)` |
| Fill | `fill([outer, inner, ...])` then `add(hatch, "HATCH", name)` | One call with nested outlines gives holes |
| Free whiplash that grows from a bar | `whip(x, y, angle, length, bend, curl)` below, then `ribbon(c, 24, 8)` | Asymmetric by construction; vary the four numbers per element |
| Flattened opening heads (rounded corners, flat crown) | `superhead(x0, x1, sill, spring, rise)` below | Many Art Nouveau and Victorian heads are neither semicircles nor plain segments |
| Petal, leaf, lens | `lens(x0, y0, x1, y1, half_width)` below | Closed; fans of petals, iris leaves, fleurs-de-lis |
| Stepped flat bar, meander | `flatbar([(x, y), ...], w)` below | No repeated consecutive points |
| 45-degree cast shadow (only when asked) | `cast(closed_outline, depth)` below | Shadow length equals the projection depth |
| Reveal shadow inside an opening (only when asked) | `reveal(glass_outline, depth)` below | Top and left strips |

```python
def whip(x, y, ang, L, bend=0.6, curl=4.0, n=60):   # heading = ang + bend*sin(pi*s) + curl*s^3 (radians), s = 0..1
    th, px, py, out = math.radians(ang), x, y, [(x, y)]
    for i in range(1, n):
        s = i / (n - 1.0); h = th + bend * math.sin(math.pi * s) + curl * s ** 3
        px += L / (n - 1.0) * math.cos(h); py += L / (n - 1.0) * math.sin(h); out.append((px, py))
    return rg.Curve.CreateInterpolatedCurve(pts(out), 3)

def superhead(x0, x1, y0, ys, rise, inset=0.0, p=3.2, n=24):
    x0, x1, y0 = x0 + inset, x1 - inset, y0 + inset; r = max(rise - inset, 1.0); w = x1 - x0
    top = [(x1 - w * k / (n - 1.0), ys + r * (1 - abs(2 * (1 - k / (n - 1.0)) - 1) ** p) ** (1.0 / p)) for k in range(n)]
    top[0], top[-1] = (x1, ys), (x0, ys)
    segs = [rg.LineCurve(rg.Point3d(x0, ys, 0), rg.Point3d(x0, y0, 0)), rg.LineCurve(rg.Point3d(x0, y0, 0), rg.Point3d(x1, y0, 0)),
            rg.LineCurve(rg.Point3d(x1, y0, 0), rg.Point3d(x1, ys, 0)), rg.Curve.CreateInterpolatedCurve(pts(top), 3)]
    return list(rg.Curve.JoinCurves(System.Array[rg.Curve](segs), 2.1 * tol))[0]

def lens(x0, y0, x1, y1, w):
    mx, my, dx, dy = (x0 + x1) / 2.0, (y0 + y1) / 2.0, x1 - x0, y1 - y0
    L = math.hypot(dx, dy) or 1.0; nx, ny = -dy / L * w, dx / L * w
    P = lambda a, b: rg.Point3d(a, b, 0)
    arcs = [rg.ArcCurve(rg.Arc(P(x0, y0), P(mx + nx, my + ny), P(x1, y1))), rg.ArcCurve(rg.Arc(P(x1, y1), P(mx - nx, my - ny), P(x0, y0)))]
    return list(rg.Curve.JoinCurves(System.Array[rg.Curve](arcs), 2.1 * tol))[0]

def flatbar(seq, w):
    c = rg.PolylineCurve(pts(seq))
    a = c.Offset(XY, w / 2.0, tol, rg.CurveOffsetCornerStyle.Sharp)[0]; b = c.Offset(XY, -w / 2.0, tol, rg.CurveOffsetCornerStyle.Sharp)[0]
    j = list(rg.Curve.JoinCurves(System.Array[rg.Curve]([a, b, rg.LineCurve(a.PointAtStart, b.PointAtStart),
                                                        rg.LineCurve(a.PointAtEnd, b.PointAtEnd)]), 2.1 * tol))
    return j[0] if len(j) == 1 and j[0].IsClosed else None

def cast(c, d):                                   # light from top left; returns shadow regions
    m = c.DuplicateCurve(); m.Transform(rg.Transform.Translation(d, -d, 0))
    return list(rg.Curve.CreateBooleanDifference(m, c, tol) or [])

def reveal(c, d):
    m = c.DuplicateCurve(); m.Transform(rg.Transform.Translation(d, -d, 0))
    return list(rg.Curve.CreateBooleanDifference(c, m, tol) or [])
```

## Depth bands, clipping and line weights

- Line weight follows depth and importance, not material. Create the bands once with their print widths, in millimetres, unless the user has a standard or the drawing style asks for something else (a single-weight diagram, a fine-line botanical study): `CUT` 0.50, `OUTLINE` 0.50 (the overall silhouette), `FRONT` 0.35 (the nearest projecting parts), `MID` 0.18, `BACK` 0.13 (set-back parts), `FINE` 0.07 (joints, texture, veins, carved detail), `GHOST` 0.09 (context, grey), `HATCH` 0.0 (plus `SHADOW` and `REVEAL` 0.0 only when shadows are asked for), and `SETOUT` with -1.0 (does not print).
- Add bands the subject needs. A building elevation, for example, adds `GROUND` 0.70 with a grey poché below, `IRON` 0.18 for metalwork as twin thin outlines and `IRONFILL` 0.0; there `FRONT` holds balconies, oriels and cornices, `MID` frames and hoods, `BACK` glazing.
- Check the hierarchy on a print preview: if the finest real element (ironwork, glazing, ornament, hairs, veins) carries the most ink, it is inverted. Never fill thin members solid dark; give them a light fill. Keep fills in separate values (with shadows, if asked for: shadows lightest, glass light, reveal shadows darker). Screen colours vanish in print, so judge on the PDF.
- Draw the backmost band first. Before adding a curve to a band, clip it against every closed outline in nearer bands: `for region in nearer: pieces = [q for p in pieces for q in clip(p, region)]`.
- Clipping closed outlines with `CreateBooleanDifference` has three traps, all met in testing:
  - an outline that fully contains a hider comes back with the hider as a hole, which duplicates the hider's line: drop pieces that lie inside another piece;
  - pockets enclosed only by hider edges (between rails and bars) come back as pieces that share no line with the original: keep a piece only if some of its points lie on the original outline (`ClosestPoint` distance under 2 x tol);
  - when the boolean fails on coincident edges, fall back to `clip` as open curves and report the element.
- Detail drawn on a hider's face (dentils on a strip, rivets on a beam, carving on a console) is not behind it: exempt a curve from a hider that contains its bounding-box centre. Windows on a projecting oriel are hidden only by things in front of the oriel, not by the oriel itself.
- On a building, metalwork hides stone behind it: clip `FRONT` against `IRON` too, except elements that sit in front of the iron (handrails, caps).
- No shadows unless the user asks for them. Do not add cast shadows, reveal shadows, shading or tonal fills to suggest depth on your own: depth comes from line weight and clipping. When shadows are asked for: `cast(outline, depth)` for each projecting closed outline the user names (or all of them if they say so), with the depth from a table per element, filled in one light tint under everything else; `reveal(glass, depth)` inside openings, deeper for set-back glazing.
- An element that has thickness should be a closed outline (use `ribbon` or offsets and caps), so that it can clip what is behind it and take a fill.
- `clip` is for open curves. A closed outline that passes behind a nearer one stays closed, and fillable, with `rg.Curve.CreateBooleanDifference(outline, nearer, tol)`.
- Section drawings: everything sliced by the drawing plane is a closed outline on `CUT`, filled with `fill`. Everything beyond is drawn in bands as in an elevation.
- To mask ornament inside a panel, keep the inside: `clip(curve, panel, keep_inside=True)`.

## Recipes for curve-heavy work

These are construction patterns, not verified styles. The user judges the result.
- **Design each piece from its own reference.** Do not stamp one motif on every instance: in testing, one symmetric scroll repeated on five railings read as generic Victorian, not as the architect's work. Look at the close-up, name what is there (counts of bays, fans, bars, loops, petals, teeth; where each curve starts and ends; what it grows from; symmetric or not), then compose it from `whip`, `lens`, `flatbar` and offsets. Repeat a design only where the subject repeats it.
- **Attach every curl to a member.** Start each `whip` on a bar, stem, edge or rail point. Floating fragments read as noise.
- **True sizes.** Take member sizes from the sources and check thin parts especially: oversized members change what the thing reads as (in building ironwork, flat bar is 10 to 25 mm in elevation; bars drawn at 50 mm turn a railing into furniture). Check that a near element does not hide a feature of the silhouette it should not.
- **Setting out first.** Derive every start point, axis and limit from the named numbers in the intent file, then grow curves from them.
- **Whiplash stem.** Points that alternate side to side along an axis with growing amplitude, interpolated: `[(x0 + sway * (i / (n - 1.0)) ** 1.5 * (-1) ** i, h * i / (n - 1.0)) for i in range(n)]`. Give it body with `ribbon(stem, w0, w1)`.
- **Branching.** Children leave the parent tangentially at chosen parameters and end in a curl (a few points on a shrinking spiral). Shorten and thin each generation.
- **Fields of lines.** Tween between two master curves and clip the family to a closed outline.
- **Frames and mouldings.** Successive offsets of an opening outline, each on a lighter band.
- **Repeats.** Build one unit (a bay, a tile, a petal, a tooth), mirror about its axis, then translate or rotate by the spacing. Vary named parameters per unit if the brief asks for irregularity.
- Offer two or three variants by changing named parameters, not by rewriting scripts.

## Checks

Record each as expected versus actual in the JSON report. Any failure blocks the pass.
- `flat` is true: everything lies at z = 0.
- Counts per band match the intent file; every named element exists (`o.Attributes.Name` on `job_objects()`).
- Every outline meant to be closed is closed; `short`, `duplicates` and unintended `self_crossing` are zero. The duplicate test compares every pair, so run it per band on large drawings.
- Extents of `job_objects()` match the overall dimensions in the intent file.
- Symmetric or repeated units hold equal counts.
- `primitive_share` stays below the limit set from the brief (a heuristic guard against line-and-arc output in a curve-heavy drawing).
- Fill area is plausible: `rg.AreaMassProperties.Compute(hatch).Area`. If an expected range fails but the fill equals the area of the outlines it fills, the estimate was wrong, not the drawing: correct the expected value and say so.
- **Accuracy table.** For each row of the element table (count, position, head shape), compare what was drawn by name with what the sources give. A mismatch blocks the pass like a geometry failure.
- **Nothing unexplained.** Every object's name maps to an element in the intent file; remove strips, stubs and fragments no source accounts for.

## Visual review

Look at a capture of the Top view framed on the job (the template does this), and close-up captures (`capture(name, w, bb)`) of each dense area. Compare each close-up side by side with its reference photograph. Ask three to five yes or no questions taken from the intent file, each allowing "unclear", for example: "Does any line pass through an element that should hide it?", "Does the railing have the same number of bays as the photo?", "Does every curl grow from a bar?". Act only on the items that fail. Agents judge their own drawings poorly by eye, so the numeric checks, the independent review and the user's review carry the weight.

## Export (ran on Rhino 9, 23 Nov 2026)

The export is the drawing alone, on a page just larger than its extents (or a paper size the user names). Laying it out on a sheet is the user's job.

```python
view = doc.Views.Find("Top", False)
dpi, scale, margin_mm = 300, 50.0, 10.0                    # 1:50; use 1.0 for full size
bb = rg.BoundingBox.Empty
for o in job_objects(): bb.Union(o.Geometry.GetBoundingBox(True))
w_mm = (bb.Max.X - bb.Min.X) / scale + 2 * margin_mm       # page size in mm, from the drawing extents
h_mm = (bb.Max.Y - bb.Min.Y) / scale + 2 * margin_mm
m = margin_mm * scale
page = rg.BoundingBox(rg.Point3d(bb.Min.X - m, bb.Min.Y - m, 0), rg.Point3d(bb.Max.X + m, bb.Max.Y + m, 0))
view.ActiveViewport.ZoomBoundingBox(page)                  # the page is centred on the view: frame the page, not the bare extents
size = System.Drawing.Size(int(w_mm / 25.4 * dpi), int(h_mm / 25.4 * dpi))
s = Rhino.Display.ViewCaptureSettings(view, size, dpi)
s.RasterMode = False            # vector output that honours print widths
s.UsePrintWidths = True
s.OutputColor = Rhino.Display.ViewCaptureSettings.ColorMode.PrintColor
s.DrawGrid = False; s.DrawAxis = False; s.DrawBackground = False
s.SetModelScaleToValue(scale)   # verified: 50.0 gives 1:50 (a 5 m bar measured 100.0 mm in the PDF)
pdf = Rhino.FileIO.FilePdf.Create(); pdf.AddPage(s); pdf.Write(pdf_path)
Rhino.Display.ViewCapture.CaptureToSvg(s).Save(svg_path)
doc.Objects.UnselectAll()
for o in job_objects(): doc.Objects.Select(o.Id)
doc.ExportSelected(dwg_path, Rhino.FileIO.FileDwgWriteOptions().ToDictionary())
doc.Objects.UnselectAll()
```

- DWG keeps model units, so its scale is inherent: prefer it when scale matters.
- For PDF and SVG, confirm the scale before stating it: measure a known dimension of the drawing in the PDF itself. `pymupdf` lists drawn paths with `page.get_drawings()`; convert points to mm with `/ 72 * 25.4`. Render a preview with `page.get_pixmap(dpi=100)` and look at it.
- Do not add a border, title block, labels, datums, scale bar or legend unless the user asks for them.
- The PDF and SVG capture everything visible in the view. In a shared document, ask before hiding the user's layers, or export from a separate document.
- Rhino 9 has no `FileSvgWriteOptions`, `FilePdfWriteOptions` or `FileDxfWriteOptions`.

## Without Rhino (route D)

Ran with ezdxf 1.4.5 on Linux: `pip install ezdxf cairosvg`.

```python
import ezdxf
from ezdxf.addons.drawing import Frontend, RenderContext, layout, config, svg
d = ezdxf.new("R2018", setup=True); msp = d.modelspace()
d.layers.add("FRONT", color=7, lineweight=35)        # hundredths of a millimetre
msp.add_spline(fit_points=[(x, y, 0) for x, y in points], dxfattribs={"layer": "FRONT"})   # repeat the first point to close
h = msp.add_hatch(color=8, dxfattribs={"layer": "HATCH"})
h.paths.add_edge_path().add_spline(control_points=ring, degree=3, periodic=1)
backend = svg.SVGBackend()
cfg = config.Configuration(background_policy=config.BackgroundPolicy.WHITE, color_policy=config.ColorPolicy.BLACK)
Frontend(RenderContext(d), backend, config=cfg).draw_layout(msp)
open("drawing.svg", "w").write(backend.get_string(layout.Page(420, 297, layout.Units.mm)))
d.saveas("drawing.dxf")
```

Review image: `cairosvg.svg2png(url="drawing.svg", write_to="drawing.png", output_width=2400, background_color="white")`. ezdxf stores true splines but has no curve booleans or clipping, so compute geometry with the same rules and keep overlaps simple. The DXF opens in Rhino for finishing.

## Limits to state in the hand-over

- No published evaluation covers agents producing detailed 2D line drawings; this method transfers evidence from 3D CAD studies.
- The agent executes curves well and judges composition poorly. Results are strongest when the user supplies master curves, a setting-out or a reference, and when the agent researches more references itself.
- What research changed in one test (a Horta facade, 23 Nov 2026): v1 from one small photo scored about 3.5 of 10 with an independent reviewer; v3, rebuilt from a near-frontal historic photo, close-ups and the critique's advice, scored about 6. The largest gains came from proportions and element counts (the sheet also improved, but sheet layout has since been taken out of this skill); ornament fidelity remained the weakest part.
- Dimensions scaled from photographs are estimates (about 10%); say so in the hand-over.
- Clipping handles closed outlines. Open or self-crossing elements cannot hide what is behind them: report any that were left unclipped.
- Aesthetic decisions stay with the user.
