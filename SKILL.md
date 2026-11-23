---
name: rhino-nurbs-drawings
description: Use when drawing 2D linework directly in Rhino by script, such as elevations, sections, plans, facades, ornament and other curve-heavy drawings with NURBS curves, line weights, hatches and vector export. Not for 3D modelling.
---

# 2D drawing in Rhino by script

This is a drawing tool. It draws flat linework directly: elevations, sections, plans, facades, ornament. It does not build 3D models and does not use Make2D or any hidden-line projection, because projected linework comes out fragmented and needs cleaning. For 3D modelling use the `rhino-nurbs-modelling` skill.

## Core rules (read first)

1. **Write scripts, never use primitive tools.** Create geometry only through Python scripts against RhinoCommon. Do not call MCP tools such as `create_object` or any "add box/circle" tool: they are the reason output collapses into circles and rectangles.
2. **Draw in 2D, on the World XY plane, at z = 0.** No 3D model, no Make2D, no `HiddenLineDrawing`. An elevation is drawn as an elevation: closed outlines for each element, curves for what is seen.
3. **Depth is built, not computed.** Every element belongs to a depth band (back to front). Draw back to front, clip whatever passes behind a nearer closed outline, and let the band decide the line weight.
4. **Curves come from rules, not typed coordinates.** Use setting-out numbers, loops and helper functions (stems, ribbons, tweens, offsets, mirrors). Hand-typed point lists are where coordinates go wrong.
5. **One pass per script, run it, check it, then continue.** Keep each script under about 80 lines of logic.
6. **Numbers gate, pictures advise.** A pass is accepted only when its numeric checks pass. A capture is a second check for gross errors. Never read a dimension off an image, and never report "done" because a script ran without errors.
7. **Look up before calling.** Use the calls in this file as written. For anything else, check inside Rhino first with a three-line script that prints `hasattr(rg.Curve, "Name")`, or read `https://developer.rhino3d.com/api/rhinocommon/`. Do not guess a function name.
8. **Stop refining after three rounds per pass.** Keep the best version and report what still differs.

## Environment

Checked on Rhino 9.0.26300 for Windows with Python 3.13.13 on 23 Nov 2026. Every helper and call in this file ran there except viewport capture and file export, which have not been run. The same scripts are expected to run on Rhino 8 (Python 3.9).

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

1. **Intent file.** Before any code, write `intent.md` in the working folder:
   - sheet: units, scale, paper size, drawing type (elevation, section, plan, detail);
   - setting-out as named numbers: overall width and height, axes, bay widths, floor levels, opening sizes;
   - depth bands from back to front, for example `BACK` (wall plane), `MID` (frames, mouldings), `FRONT` (balconies, ironwork), `CUT` (anything sliced by the drawing plane);
   - every element with its band, whether it is a closed outline, and the rule that generates its curves;
   - the curved features named explicitly (for example "whiplash stems rising from each pier, tapering, branching twice").
   If the user gives master curves in Rhino or a reference image, describe them in words with dimensions first and confirm. Ask once if scale, extents or the element list are unknown.
2. **Expected values.** For each pass write the expected counts, extents and closed or open status before running it.
3. **Passes**, one script each: (a) layers and setting-out lines on a non-printing layer, (b) closed outlines per band, back to front, (c) secondary elements, (d) ornament and curve families, (e) clipping of everything that passes behind a nearer outline, (f) fills and hatches, (g) export.
4. **After each pass:** read the JSON report, fix execution errors (up to three retries), compare checks with expected values, then look at a capture.
5. **Final review.** If subagents are available, give a fresh one only the intent file, the reports and the exported images, and have it answer the review questions. It must not see the build reasoning.
6. **Hand over** the files, the route used, check results, and what still differs.

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
JOB, PASS = "FACADE", "p03_ornament"             # JOB is the parent layer name
doc = globals().get("__rhino_doc__") or sc.doc   # McNeel MCP injects __rhino_doc__
tol = doc.ModelAbsoluteTolerance
XY = rg.Plane.WorldXY
BLACK = System.Drawing.Color.Black
report = {"pass": PASS, "ok": False, "checks": {}, "ids": {}}

def pts(seq):                                    # [(x, y), ...] -> typed point list on z = 0
    l = List[rg.Point3d]()
    for x, y in seq: l.Add(rg.Point3d(x, y, 0))
    return l

def layer(path, plot_mm=0.0, color=BLACK):       # "FACADE::FRONT"; creates parents; returns index
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

def capture(name, w=2400, h=1700):               # not yet run on Rhino 9
    view = doc.Views.Find("Top", False) or doc.Views.ActiveView
    bb = rg.BoundingBox.Empty
    for o in job_objects(): bb.Union(o.Geometry.GetBoundingBox(True))
    view.ActiveViewport.ZoomBoundingBox(bb)
    vc = Rhino.Display.ViewCapture()
    vc.Width, vc.Height = w, h
    vc.ScaleScreenItems = False
    vc.DrawAxes = vc.DrawGrid = vc.DrawGridAxes = False
    vc.TransparentBackground = False
    path = os.path.join(OUT, name + ".png")
    vc.CaptureToBitmap(view).Save(path, System.Drawing.Imaging.ImageFormat.Png)
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

## Depth bands, clipping and line weights

- Create the bands once with their print widths, in millimetres, unless the user has a standard: `layer(JOB + "::CUT", 0.50)`, `FRONT` 0.35, `MID` 0.25, `BACK` 0.13, `FINE` 0.09 (ornament interior lines, hatching lines), `HATCH` 0.0, and `SETOUT` with -1.0 (does not print).
- Draw the backmost band first. Before adding a curve to a band, clip it against every closed outline in nearer bands: `for region in nearer: pieces = [q for p in pieces for q in clip(p, region)]`.
- An element that has thickness should be a closed outline (use `ribbon` or offsets and caps), so that it can clip what is behind it and take a fill.
- `clip` is for open curves. A closed outline that passes behind a nearer one stays closed, and fillable, with `rg.Curve.CreateBooleanDifference(outline, nearer, tol)`.
- Section drawings: everything sliced by the drawing plane is a closed outline on `CUT`, filled with `fill`. Everything beyond is drawn in bands as in an elevation.
- To mask ornament inside a panel, keep the inside: `clip(curve, panel, keep_inside=True)`.

## Recipes for curve-heavy ornament

These are construction patterns, not verified styles. The user judges the result.
- **Setting out first.** Derive every start point, axis and limit from the named numbers in the intent file, then grow curves from them.
- **Whiplash stem.** Points that alternate side to side along an axis with growing amplitude, interpolated: `[(x0 + sway * (i / (n - 1.0)) ** 1.5 * (-1) ** i, h * i / (n - 1.0)) for i in range(n)]`. Give it body with `ribbon(stem, w0, w1)`.
- **Branching.** Children leave the parent tangentially at chosen parameters and end in a curl (a few points on a shrinking spiral). Shorten and thin each generation.
- **Fields of lines.** Tween between two master curves and clip the family to a closed outline.
- **Frames and mouldings.** Successive offsets of an opening outline, each on a lighter band.
- **Bays.** Build one bay, mirror about its axis, then translate by the bay width. Vary named parameters per bay if the brief asks for irregularity.
- Offer two or three variants by changing named parameters, not by rewriting scripts.

## Checks

Record each as expected versus actual in the JSON report. Any failure blocks the pass.
- `flat` is true: everything lies at z = 0.
- Counts per band match the intent file; every named element exists (`o.Attributes.Name` on `job_objects()`).
- Every outline meant to be closed is closed; `short`, `duplicates` and unintended `self_crossing` are zero. The duplicate test compares every pair, so run it per band on large drawings.
- Extents of `job_objects()` fit the sheet at the stated scale.
- Symmetric bays hold equal counts.
- `primitive_share` stays below the limit set from the brief (a heuristic guard against line-and-arc output in a curve-heavy drawing).
- Fill area is plausible: `rg.AreaMassProperties.Compute(hatch).Area`.

## Visual review

Look at a capture of the Top view framed on the job (the template does this), or `get_viewport_image` for a quick look. Ask three to five yes or no questions taken from the intent file, each allowing "unclear", for example: "Does any line pass through a balcony that should hide it?", "Do the stems rise from every pier?", "Is the left half a mirror of the right?". Act only on the items that fail. Agents judge their own drawings poorly by eye, so the numeric checks and the user's review carry the weight.

## Export (not yet run on Rhino 9; expect to correct a call)

```python
view = doc.Views.Find("Top", False)
w_mm, h_mm, dpi = 420.0, 297.0, 300
size = System.Drawing.Size(int(w_mm / 25.4 * dpi), int(h_mm / 25.4 * dpi))
s = Rhino.Display.ViewCaptureSettings(view, size, dpi)
s.RasterMode = False            # vector output that honours print widths
s.UsePrintWidths = True
s.OutputColor = Rhino.Display.ViewCaptureSettings.ColorMode.PrintColor
s.DrawGrid = False; s.DrawAxis = False; s.DrawBackground = False
s.SetModelScaleToValue(100.0)   # direction of this ratio is undocumented: verify
pdf = Rhino.FileIO.FilePdf.Create(); pdf.AddPage(s); pdf.Write(pdf_path)
Rhino.Display.ViewCapture.CaptureToSvg(s).Save(svg_path)
doc.Objects.UnselectAll()
for o in job_objects(): doc.Objects.Select(o.Id)
doc.ExportSelected(dwg_path, Rhino.FileIO.FileDwgWriteOptions().ToDictionary())
doc.Objects.UnselectAll()
```

- DWG keeps model units, so its scale is inherent: prefer it when scale matters.
- For PDF and SVG, draw a scale bar of known length, export, and confirm its length before stating a scale.
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
open("sheet.svg", "w").write(backend.get_string(layout.Page(420, 297, layout.Units.mm)))
d.saveas("sheet.dxf")
```

Review image: `cairosvg.svg2png(url="sheet.svg", write_to="sheet.png", output_width=2400, background_color="white")`. ezdxf stores true splines but has no curve booleans or clipping, so compute geometry with the same rules and keep overlaps simple. The DXF opens in Rhino for finishing.

## Limits to state in the hand-over

- No published evaluation covers agents producing detailed architectural drawings; this method transfers evidence from 3D CAD studies.
- The agent executes curves well and judges composition poorly. Results are strongest when the user supplies master curves, a setting-out or a reference.
- Clipping handles closed outlines. Open or self-crossing elements cannot hide what is behind them: report any that were left unclipped.
- Aesthetic decisions stay with the user.
