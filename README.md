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

The fix came from a property of the source data: each traced "chain" is not a path running along a vessel's length, it's a single **cross-sectional trace** through the vessel at one point along it (confirmed by fitting a best-fit plane to each chain's points — over 99% sit within 1 nm of that plane, across all four major-vessel categories). A chain of N points was costing N-1 line segments to encode what is really one measurement: "the vessel has roughly this centre and this radius, here." Each chain is therefore packed as **one ellipsoid annotation** instead — center = the chain's centroid, radii = an isotropic sphere sized to the mean point-to-centroid distance (converted from physical nm to this app's per-axis voxel units). Every traced cross-section is kept, none dropped, and nothing is approximated away — this is a re-encoding of the same measurements, not a lossy simplification, unlike RDP. Annotation count drops to roughly 1/(average chain length) of the line-based encoding. The one real trade-off: Neuroglancer's ellipsoid annotation has no rotation/orientation field, so an arbitrarily-tilted cross-section can only be rendered as a sphere, not a tilted disc — adequate for "fill the lumen" at that point, not a precise reconstruction of vessel-wall geometry.

If a category's outline still looks unacceptably coarse, sized wrong, or the file is unusably large, tell the tool's maintainer which category and what's wrong — the tolerance (capillaries) or the minimum-radius floor (major vessels, currently 50 nm, guarding against near-coincident points producing a degenerate near-zero sphere) are the parameters to revisit.

`arterial_cap_order3.json` uses the `"3. order1"` (unarchived / current) layer from the July 2026 export of `arterial 3 order cap.txt`, not the earlier `"3. order"` layer, which was an unfinished pass.
