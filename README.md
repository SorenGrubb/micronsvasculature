# MICrONS vasculature outline data

Traced vessel outlines (in minnie65 voxel coordinates) for the [MICrONS → Neuroglancer cell identification tool](https://github.com/SorenGrubb/microns-neuroglancer-tool), split into 10 categories by vessel type and capillary order. Capillary categories are packed as line segments; the four major-vessel categories (pial/penetrating arterioles, ascending/pial venules) are packed as ellipsoids — see "Line-count reduction" below for why.

| File | Category |
|---|---|
| `pial_arterioles.json` | Pial arterioles |
| `penetrating_arterioles.json` | Penetrating arterioles |
| `arterial_cap_order1.json` | Arterial capillaries, 1st order |
| `arterial_cap_order2.json` | Arterial capillaries, 2nd order |
| `arterial_cap_order3.json` | Arterial capillaries, 3rd order |
| `venous_cap_order1.json` | Venous capillaries, 1st order |
| `venous_cap_order2.json` | Venous capillaries, 2nd order |
| `venous_cap_order3.json` | Venous capillaries, 3rd order |
| `ascending_venules.json` | Ascending venules |
| `pial_venules.json` | Pial venules |

Each file has the shape:

```json
{
  "name": "Pial arterioles",
  "color": "#c92a2a",
  "nPoints": 22,
  "nLines": 0,
  "pointsB64": "<base64-encoded little-endian Float32 array, 3 floats per point: x,y,z>",
  "linesB64": "<base64-encoded little-endian Float32 array, 6 floats per line: pointA.x,pointA.y,pointA.z,pointB.x,pointB.y,pointB.z>",
  "nEllipsoids": 1118,
  "ellipsoidsB64": "<base64-encoded little-endian Float32 array, 6 floats per ellipsoid: center.x,center.y,center.z,radii.x,radii.y,radii.z>"
}
```

`nEllipsoids`/`ellipsoidsB64` are only present on the four major-vessel files; the six capillary files only ever populate `linesB64`. Consumers should treat `pointsB64`/`linesB64`/`ellipsoidsB64` as independently optional — check each field before decoding it.

Coordinates are voxel indices, exactly as they appear in the original Neuroglancer annotation-layer tracings (their own `outputDimensions` transform declares `x`/`y` at 4 nm/voxel and `z` at 40 nm/voxel) — the same convention the main tool uses everywhere else, so no unit conversion is needed when consuming these files. Do not divide by 4/40 again. Ellipsoid radii are likewise in voxel units, per-axis, following the same 4/40 nm/voxel anisotropy.

This repo exists purely as a lightweight fetch target for the main tool (via `raw.githubusercontent.com`, chosen over jsDelivr's GitHub CDN for its short, predictable ~5-minute cache — see the main tool's comments above `VASC_TRACE_BASE`), which lazy-fetches only the categories a user toggles on rather than embedding ~52 MB of raw tracing data in the tool itself. Source tracings by Søren Grubb.

## Line-count reduction

Neuroglancer local annotations are embedded directly in the viewer's state JSON, which then goes in the URL — so unlike a normal download, size here is bounded by URL length rather than file size. The raw hand tracings (up to ~57,000 line segments for one category) exceeded that limit and silently failed to open in Neuroglancer.

**Capillaries** (`arterial_cap_order1/2/3`, `venous_cap_order1/2/3`) are still line-based: each traced vessel is decomposed into its connected "chain" between branch points and free ends, then simplified with Ramer–Douglas–Peucker at a 300 nm tolerance to drop interior points that don't meaningfully change the path. `venous_cap_order2` and `venous_cap_order3` additionally have whole chains subsampled (after sorting into Z-order/Morton code, so the retained subset stays evenly spread rather than clustering), since most of their segment count comes from capillary *count* rather than per-capillary detail.

**Major vessels** (`pial_arterioles`, `penetrating_arterioles`, `ascending_venules`, `pial_venules`) are packed as **ellipsoids**, not lines. Two earlier approaches were tried and rejected first: dropping whole chains (e.g. keeping only every 4th penetrating arteriole) visibly thinned out the vessel pattern — missing whole arterioles/venules reads very differently from a coarser line through the same ones, so it was reverted; a looser RDP tolerance (1000 nm) with no chain-dropping kept every vessel but still wasn't small enough for two categories to reliably open.

The fix came from a property of the source data: each traced "chain" is not a path running along a vessel's length, it's a near-planar **cross-sectional trace** through the vessel at one point along it. But not every chain traces the same amount of the vessel wall. Measuring how far each chain's own points sweep around its centroid (in its own best-fit plane) splits chains into two populations: most `pial_arterioles`/`pial_venules` chains sweep close to 360° — a **complete traced ring**. Most `penetrating_arterioles`/`ascending_venules` chains sweep only ~180-200° — **half a ring**, apparently because the annotator could often only trace the near-visible wall of these thinner, more numerous vessels within a single EM tile.

The first version of this packing (chains_to_ellipsoids in an earlier `extract_pack_v3.py`) converted every chain into one ellipsoid regardless of which population it came from. For full rings that's fine — centroid and mean point-to-centroid distance genuinely describe the ring. For half-rings it isn't: the centroid is biased toward just the visible half, and the mean-distance radius under-estimates the true lumen — visually indistinguishable from having simply swapped each line for a small ellipsoid, which is exactly what was flagged as wrong.

`extract_pack_v4.py` fixes this by reconstructing full rings from half-ring chains before computing each ellipsoid: two "arc" chains (sweep < 250°) are merged into one ellipsoid only if they are **mutual nearest neighbours at a free (degree-1) endpoint within 200 nm** — evidence of a genuine traced seam, not a coincidentally nearby *different* ring — and each chain is used in at most one such pairing (no transitive chaining). A first attempt allowed transitive chaining (any chain within range of an already-matched chain could join the same group) and it was worse than the original bug: because rings are often densely spaced along a vessel's length, nearby-but-different rings kept fusing into one giant multi-ring blob (radii over 100 µm, physically nonsensical). Capping every merge at exactly 2 chains is what prevents that. Depending on category, 55-90% of half-ring chains find a genuine partner this way; the rest fall back to the single-chain estimate, same limitation as before, since that's still the best available representation of what was actually traced for that one vessel.

Once chains (or matched pairs of chains) are resolved, each becomes one ring — center = the (possibly merged) point set's centroid, radius = an isotropic sphere sized to the mean point-to-centroid distance (converted from physical nm to this app's per-axis voxel units). This re-encoding is what actually made these categories small enough to reliably open: a chain of N points was costing N-1 line segments to encode what is really one measurement, "the vessel has roughly this centre and this radius, here." The one real trade-off: Neuroglancer's ellipsoid annotation has no rotation/orientation field, so an arbitrarily-tilted or non-circular cross-section can only be rendered as a sphere, not a tilted disc.

**Vessel-graph gap-filling and radius smoothing** (`extract_pack_v5.py`). Even with pairing fixed, real Neuroglancer renders still showed two problems: visible gaps between consecutive ellipsoids along a vessel (traced rings are sparse, discrete samples along its length — nothing fills the space between them), and individual ellipsoid sizes that looked inconsistent next to their neighbours (single-ring radius estimates are noisy). Both need per-vessel context that a single ring or pair doesn't have on its own.

Vessel topology is reconstructed from ring centroids alone (there's no other connectivity info once arcs are resolved): a k-nearest-neighbour graph (k=8) over one category's centroids, with any edge over 20,000 nm dropped, reduced to a minimum spanning tree. The 20,000 nm cap is what keeps the tree from bridging two unrelated vessels — same-vessel consecutive-ring spacing is empirically a few thousand nm at the 10th-99th percentile in every major-vessel category, while separate parallel vessels in these tracings are typically tens to hundreds of microns apart. Checked against penetrating_arterioles: the resulting tree leaves only 1 of 5097 rings fully isolated, and node degree is overwhelmingly 1-2 (a simple run along the vessel) with a smaller population of degree≥3 nodes at what look like real anatomical branch points.

Each ring's radius is then replaced by the median of itself and its direct tree-neighbours — a 1-hop median filter that damps single-ring noise while still letting genuine tapering along a vessel's length show through, since it's local rather than global.

**Radius caps and axis-aligned elongation** (`extract_pack_v6.py`). Two rounds of feedback from Søren after actually looking at the render: some ellipsoids were still physically implausible (bigger than any real vessel), and — separately — he asked for elongated ellipsoids along each vessel's run instead of chains of touching spheres, both to look smoother and to cut annotation count.

*Hard radius cap*: after smoothing, every radius is clamped to an anatomical bound Søren provided directly — no traced vessel exceeds 50 micron diameter (25,000 nm radius) anywhere in this dataset, and penetrating arterioles / ascending venules specifically cap at 25 micron diameter (12,500 nm radius). *Axis-aligned stretching*: where an edge in the vessel graph runs close to one global axis, the two rings' own ellipsoids stretch along that axis until they overlap, with no new annotation needed — this cut counts for the less-branchy categories but not much for ascending_venules, where the freed-up size budget went into tighter (better-looking) gap interpolation instead.

**Depth-aware connectivity and capsule merging** (`extract_pack_v7.py`, current version). Søren's next observation, after looking again: elongated ellipsoids only ever reached within one z-layer, never across depth — the vessel's actual 3D structure wasn't being respected — and he still wanted fewer, longer ellipsoids overall, not just touching neighbours.

The root cause, confirmed against real data: the vessel graph up to this point connected each ring to its raw nearest-3D-distance neighbour. That sounds reasonable but is wrong here, because these tracings were made by working through the EM volume one 2D section at a time — tracing every visible vessel cross-section in that section before moving to the next. A given ring's nearest neighbour by raw distance is therefore very often a DIFFERENT, unrelated vessel traced in the same z-slice, not the true next cross-section of the SAME vessel one or more z-slices away. Checked on penetrating_arterioles: the old graph's edges were dominant-axis Y more often than Z (1927 vs. 1657 out of 5042), with a median z-fraction of only 0.46 — even though these vessels by definition run close to perpendicular to the cortical surface. Ellipsoids only elongated sideways because the graph itself was mostly connecting sideways.

The fix uses information that was already being computed and then discarded: each ring's own best-fit-plane normal vector (from the same SVD used to classify it as a ring vs. an arc). A ring is a cross-section *perpendicular* to the vessel's local direction, so that normal *is* the vessel's local direction at that point — the true next ring along the same vessel should lie close to along it, while a same-z different-vessel "neighbour" lies roughly perpendicular to it. Candidate neighbours are still found spatially (over-fetching more than needed via a k-d tree), then re-ranked by alignment with each ring's own normal rather than taken as the raw nearest, with a hard floor (candidates more than ~78° off the ring's normal are never accepted, even if nothing better is available — an unconnected ring falls back to an isotropic sphere, which is preferable to a confidently wrong connection). This raised z-dominant edges to a clear plurality of the total and the median z-fraction to 0.66.

On the corrected graph, consecutive same-vessel rings are no longer just pairwise-stretched to touch their immediate neighbour. The graph is decomposed into simple paths between branch points (the same connected-component walk `build_chains()` uses on raw line segments, applied one level up — here nodes are rings, edges are graph connections), and each path is greedily walked, merging consecutive rings into ONE elongated ellipsoid for as long as the run stays at least 70% aligned to a single global axis and the vessel's width doesn't vary by more than 2×. A branch or leaf point is always kept as its own separate ellipsoid, since it's shared by multiple paths — merging it into any one of them would silently double-count it (an early version of this script did exactly that and produced *more* ellipsoids than input rings; there's now an assertion guarding against a regression). This directly reduces annotation count rather than just closing gaps: pial_arterioles 1232→543, penetrating_arterioles 6408→4546, ascending_venules 9739→7163, pial_venules 2855→1124. Final encoded sizes: pial_arterioles 0.08 MB, penetrating_arterioles 0.63 MB, ascending_venules 1.00 MB, pial_venules 0.16 MB — all well under the tool's 1.95 MB warning threshold, with more headroom than any earlier version.

One consequence of this design: an ellipsoid's elongated (length) axis is deliberately left *uncapped* by the anatomical diameter bound — only its two width axes are clamped to it. Length describes how far a capsule reaches along the vessel's own course, not the vessel's physical diameter, so applying the same 25/50 micron cap to it would have artificially limited how much merging could ever achieve.

**Guaranteed gap closure** (`extract_pack_v8.py`, current version). Søren's next look: the largest ellipsoids were still rendering as isolated spheres with gaps between them. Measured on the output that prompted this: 44% of ellipsoids were effectively spherical and 8–27% of neighbouring pairs had a genuine gap. Three separate defects, each verified rather than assumed:

1. *A single-ring group recorded no elongation axis*, so the width cap clamped all three of its axes — every stretch applied to it was silently clamped back off. This hit hardest on the largest ellipsoids, the ones already sitting at the cap, which is exactly the population Søren flagged.
2. *Closure reused the merge alignment threshold.* Wrong test: two rings can sit obliquely and still be reachable by stretching, even where merging them into one straight capsule would misrepresent the vessel's path. Closure now has its own looser threshold, and it checks whether the two ellipsoids *actually* fail to reach (computing each one's true extent along the connecting direction) rather than only asking whether the edge looks well-aligned.
3. *A merged capsule is axis-aligned, so it only truly contains its own run if that run's perpendicular excursion stays inside the capsule's width.* The old fixed alignment-fraction test is scale-blind — 30% off-axis is harmless over a short hop and a large uncovered bow over a long one. This is why the pial categories, which run along the surface at angles rather than straight down, kept showing gaps despite passing the alignment test. Merging is now bounded by the width itself.

Whatever stretching still cannot close is bridged by **bridge capsules**: elongated ellipsoids laid along the connecting segment, each sized so it provably contains its own piece of that segment. Earlier rounds used isotropic filler *spheres* here, which is what made those boundaries read as a string of separate beads.

Verified by sampling the traced vessel path every 500 nm and testing each sample against every output ellipsoid: **0.6–3.3% of path points remain uncovered** per category (pial_arterioles 3.3%, penetrating_arterioles 1.0%, ascending_venules 0.6%, pial_venules 2.3%), down from 8–27% of neighbouring pairs having outright gaps. Roughly half of all ellipsoids are now genuinely elongated; the remainder are rings whose neighbours already overlap them, where stretching would add nothing. Exactly one axis per ellipsoid may exceed the anatomical diameter bound (its run direction) — permitting two produced visibly fat slab-shaped blobs at junctions, so the largest axis wins and the rest are clamped back to the width bound. Counts and encoded sizes: pial_arterioles 1144 / 0.16 MB, penetrating_arterioles 6254 / 0.87 MB, ascending_venules 9895 / 1.37 MB, pial_venules 2725 / 0.38 MB.

The remaining uncovered percentage is concentrated in the pial categories, whose vessels run diagonally along the cortical surface — an axis-aligned ellipsoid fundamentally cannot follow a diagonal run without either leaving a gap or being fattened past the vessel's real diameter, and fattening was judged the worse error. Closing that last few percent would require an annotation type Neuroglancer does not offer.

**Anti-spike limit and short-run merging** (`extract_pack_v10.py`, current version). An intermediate version stretched each ellipsoid to whatever length closed its gap, which produced long needles standing perpendicular to the vessels — because an axis-aligned ellipsoid elongated along axis A, on a vessel running at angle θ off A, protrudes sideways by roughly L·sin(θ). Elongation is now bounded by **L ≤ W/sin(θ)** (W = the vessel's own half-width): the length at which the ellipsoid would just leave its lumen. Worst-case elongation fell from ~217× to ~11×, with 3–5× at the 99th percentile.

A final **short-run merge** then brings the count down, following Søren's two observations: comparable radius is used as the same-vessel signal for which neighbours may combine ("a large ellipsoid near a large ellipsoid belongs to the same vessel"), and each run of up to 3 neighbours collapses into a single ellipsoid *elongated* to span them at the members' own widest width plus a small margin — not fattened. Short runs are what make this safe under the anti-spike rule: spanning 2–3 neighbours keeps the half-length small enough to stay inside the vessel, and any merge that would violate the limit is rejected with its originals kept, so the pass can only reduce count, never reintroduce needles.

Two failed variants are recorded because they bracket the working rule: containing merely the members' *centres* let their flanks stick out (uncovered path points jumped from ~1% to ~8%), while demanding full containment of each member's width is geometrically impossible — a merged ellipsoid tapers, so at any axial offset its cross-section is narrower than the member sitting there, and 0% of merges qualified. The working rule tests coverage of the vessel *path* the members spanned and rejects runs that bow off a straight line.

**Validation against the original traced perimeter** (`validate_vs_outline.py`, added at Søren's suggestion) reports two metrics, since each approach fails in a different direction: *coverage* (what fraction of traced perimeter points fall inside some output ellipsoid — low means missing vessel) and *spill* (how far points sampled from inside the output ellipsoids sit from the nearest traced point — high means occupying space where no vessel was traced, which is what a needle does). For `penetrating_arterioles`:

| version | n | coverage | spill p95 | spill max |
|---|---|---|---|---|
| depth-aware + merge (`v7`) | 4546 | 66.1% | 4.6 µm | 16.6 µm |
| unbounded stretching (`v8`) | 6254 | 76.5% | 13.2 µm | 57.9 µm |
| anti-spike (`v9`) | 7926 | 70.4% | 4.2 µm | 15.6 µm |
| **anti-spike + run merge (`v10`)** | **3448** | **78.6%** | **4.4 µm** | 23.0 µm |

The current version has both the fewest annotations and the best coverage of any version tried. Encoded sizes: pial_arterioles 0.11 MB, penetrating_arterioles 0.48 MB, ascending_venules 0.78 MB, pial_venules 0.31 MB — well under the tool's 1.95 MB warning threshold.

If a category's outline still looks unacceptably coarse, sized wrong, gappy, or the file is unusably large, tell the tool's maintainer which category and what's wrong — the RDP tolerance (capillaries); the endpoint-gap threshold or minimum-radius floor (ring/arc pairing, 200 nm / 50 nm); the k-d tree candidate count, normal-alignment floor, edge-length cap (vessel graph, currently 20 candidates / cos(78°) / 20,000 nm); or the merge alignment threshold, width-ratio limit, or max merge span (capsule merging, currently 70% / 2.0× / 60,000 nm) are the parameters to revisit.

`arterial_cap_order3.json` uses the `"3. order1"` (unarchived / current) layer from the July 2026 export of `arterial 3 order cap.txt`, not the earlier `"3. order"` layer, which was an unfinished pass.
