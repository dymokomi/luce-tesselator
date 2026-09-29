# luce-tesselator

> **Archived (2026-09-28).** This package is merged into
> [luce-cad](https://github.com/dymokomi/luce-cad) 0.2.0: the same NURBS
> evaluation and trim/grid/recombine meshing live in `src/luce_cad/tessellation/`
> and are exported as `tessellation` (`from tessellation import NurbsSurface,
> TrimPolygon, TrimGrid, ...`), with these contract tests in luce-cad's runner.
> Depend on `dymokomi/luce-cad` instead; this repository is read-only and the
> package is no longer on the registry.

Original **Luce Base** rational tensor-product B-spline evaluation and mesh
tessellation. The requested package spelling is retained. Public export:
`tesselator.NurbsSurface.tessellate(points, weights, nu, nv, degree_u, degree_v,
knots_u, knots_v, segments=16)`.

Points/positive weights use u-major, v-minor order. Knot vectors are expanded,
finite and nondecreasing. The evaluator uses Cox–de Boor basis functions and a
rational weighted sum over only the active controls, with binary span lookup.
`sample_checked` is for immutable nets already validated by `check`; callers
must not change the net afterward. Returns an immutable indexed polygon mesh; its quads are
triangulated by luce-3d for rendering. No UI, File node or STEP syntax lives here.

`surface_normal_checked` evaluates exact rational first derivatives on an
already validated immutable net and returns their normalized cross product.
It returns zero at a singular parameterization; CAD owns limiting-pole policy.

`tessellate_region` also accepts `u0, u1, v0, v1` before `segments` for a
rectangular parameter subdomain; `check` and `check_region` validate input without
allocating a mesh. `luce-cad` owns the face's chosen domain.

`NurbsCurve.at` and `NurbsSurface.at` expose rational evaluation at actual
knot-domain parameters. `TrimPolygon.triangulate` accepts a planar outer loop
and holes, checks intersections/nesting, builds visible bridges, and clips ears
without dropping boundary vertices. `quadrangulate` first improves interior
diagonals using bounded Delaunay-style edge flips, then pairs adjacent
triangles only when convex, with all angles between 20 and 160 degrees and edge
length ratio at most 10. It does not move vertices or remove trim boundaries.
`quadrangulate(..., edge_size=0)` optionally inserts an interior regular lattice
before improving diagonals and recombining. Zero disables it. Shared boundary
sampling remains the caller's responsibility; this routine never splits a
boundary edge independently. Refinement is bounded to 8192 candidate cells.

Trim intersections, ear membership and hole containment use dominant-plane
orientation signs, with an ordinary floating-point filter and a bounded
floating-expansion fallback for ambiguous binary64 determinants. Close points
are not automatically touching or welded. These topology predicates are
separate from a CAD reader's geometric fitting tolerance. Ear selection still
requires locally conditioned display triangles; genuinely degenerate domains
remain errors. The fallback assumes finite, normal-range products in the
bounded mesh coordinate range, not arbitrary unbounded/subnormal inputs.

Ear clipping avoids leaving a remainder with no conditioned convex corner.
If it still stalls on a single loop, it may add one area-centroid interior seed,
but only after every original boundary segment forms a positive, conditioned
triangle to that point and the area is preserved. This kernel check rejects
invisible spokes; holes never take this fallback. Boundary positions and edges
remain exact. The seed is subsequently eligible for the ordinary lattice,
diagonal-improvement and pairing steps. Diagonal guards are relative to local
squared edge length, so ordinary model-unit changes do not disable flips.

`triangulate_seams(points, sizes, normal, boundary_ids)` and
`quadrangulate_seams(points, sizes, normal, boundary_ids, edge_size=0,
second_spacing=0)` accept explicit nonnegative canonical boundary identities.
A reversed coincident segment within one loop is legal only when both endpoints
have matching identities and exact chart positions. Distinct periodic chart
images remain separate. Mere proximity or equal positions with different IDs
never authorize overlapping constraints. A bounded visible-diagonal partition
can separate a retraced slit tip before ear clipping; it keeps every authored
segment and does not insert arbitrary spokes through the domain.

`TrimConstraints.build(points, ids, sizes)` records coincident seam constraints
by stable input point index. Its owner calls `close()`; meshing stages borrow the
table. `contains(a,b)` protects those segments from grid insertion, diagonal
flips, quad pairing and caller refinement. The table is empty for ordinary
boundaries, which are protected by one-face incidence. Callers must retain input
point indices. This is not a general intersecting-constraint arrangement solver.

Surface-grid sampling is uniform, not a guaranteed chord-error tolerance.

`TrimGrid.clip(points, sizes, us, vs)` intersects a supplied UV grid with one
outer polygon and holes. Whole cells stay quads; boundary cells keep their
polygon outlines. Holes within a cell and cells over the 256-corner polygon
limit are triangulated locally. **Input trim segments must already be split at
every grid crossing.** The first output points retain every input point and ID;
the clipper neither invents CAD seam samples nor welds distinct vertices. Output
validation requires every input trim segment exactly once and every interior
edge twice. Touching/ambiguous arrangements are rejected, not silently patched.
The implementation uses per-cell boundary buckets and per-row scanline parity.

Curved trim evaluation, cross-patch station planning, adaptive support checks,
periodic bands and shared CAD-edge ownership live in `luce-cad`. General
field-aligned all-quad remeshing is not implemented.

Limits: degree 1–8, 2–1,024 surface control points per direction (262,144 total), up to 4,096 curve
control points, divisions 1–64. Invalid
weights, knot counts and collapsed surface polygons are checked errors.
Planar trims are bounded to 64 loops and 16,384 vertices. Exact shared-vertex
junctions within one loop are supported; touching distinct loops and unidentified
overlapping segments remain invalid. Explicit retraced seams use the identity
API above. Regression tests include
a bilinear plane, rational quarter-cylinder/curve, multiple holes, invalid
boundaries, conservative quad recombination, grid-crossing holes, oblique trims,
oversized cut cells and rejection of unsplit trim crossings.
