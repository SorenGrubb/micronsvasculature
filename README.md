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

**Vessel-graph gap-filling and radius smoothing** (`extract_pack_v5.py`, current version). Even with pairing fixed, real Neuroglancer renders still showed two problems: visible gaps between consecutive ellipsoids along a vessel (traced rings are sparse, discrete samples along its length — nothing fills the space between them), and individual ellipsoid sizes that looked inconsistent next to their neighbours (single-ring radius estimates are noisy). Both need per-vessel context that a single ring or pair doesn't have on its own.

Vessel topology is reconstructed from ring centroids alone (there's no other connectivity info once arcs are resolved): a k-nearest-neighbour graph (k=8) over one category's centroids, with any edge over 20,000 nm dropped, reduced to a minimum spanning tree. The 20,000 nm cap is what keeps the tree from bridging two unrelated vessels — same-vessel consecutive-ring spacing is empirically a few thousand nm at the 10th-99th percentile in every major-vessel category, while separate parallel vessels in these tracings are typically tens to hundreds of microns apart. Checked against penetrating_arterioles: the resulting tree leaves only 1 of 5097 rings fully isolated, and node degree is overwhelmingly 1-2 (a simple run along the vessel) with a smaller population of degree≥3 nodes at what look like real anatomical branch points.

Each ring's radius is then replaced by the median of itself and its direct tree-neighbours — a 1-hop median filter that damps single-ring noise while still letting genuine tapering along a vessel's length show through, since it's local rather than global. It isn't a complete fix for every outlier: a ring whose only neighbour is also unusually large stays large, which may be real anatomy (e.g. a wide trunk segment near a vessel's root) rather than noise, and this script has no independent way to tell the two apart.

Extra ellipsoids are linearly interpolated (centre and radius) along any tree edge whose two ends are farther apart than a spacing multiplier times the smaller of their two (smoothed) radii — closing the gap so consecutive spheres touch rather than float apart. The multiplier starts at 2.0 (spheres just touch) and is loosened in steps, per category, only as far as needed to keep that category's own encoded size under a 1.7 MB budget (deliberately under the tool's 1.95 MB warning threshold, since the size estimate used during packing is approximate, not measured against the final JSON). In practice: pial_arterioles, penetrating_arterioles, and pial_venules stayed at 2.0×; ascending_venules (the busiest category) needed ~3.1× to fit. Final real encoded sizes, measured against the actual packed JSON: pial_arterioles 0.20 MB, penetrating_arterioles 1.11 MB, ascending_venules 1.25 MB, pial_venules 0.51 MB.

If a category's outline still looks unacceptably coarse, sized wrong, gappy, or the file is unusably large, tell the tool's maintainer which category and what's wrong — the RDP tolerance (capillaries), the endpoint-gap threshold or minimum-radius floor (ring/arc pairing, currently 200 nm and 50 nm), or the MST edge cap / spacing multiplier / size budget (vessel-graph step, currently 20,000 nm / 2.0× / 1.7 MB) are the parameters to revisit.

`arterial_cap_order3.json` uses the `"3. order1"` (unarchived / current) layer from the July 2026 export of `arterial 3 order cap.txt`, not the earlier `"3. order"` layer, which was an unfinished pass.
