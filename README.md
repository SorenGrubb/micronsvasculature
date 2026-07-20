# MICrONS vasculature outline data

Traced vessel centerlines (points + line segments, in minnie65 voxel coordinates) for the [MICrONS → Neuroglancer cell identification tool](https://github.com/SorenGrubb/microns-neuroglancer-tool), split into 10 categories by vessel type and capillary order:

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
  "nLines": 4705,
  "pointsB64": "<base64-encoded little-endian Float32 array, 3 floats per point: x,y,z>",
  "linesB64": "<base64-encoded little-endian Float32 array, 6 floats per line: pointA.x,pointA.y,pointA.z,pointB.x,pointB.y,pointB.z>"
}
```

Coordinates are voxel indices, exactly as they appear in the original Neuroglancer annotation-layer tracings (their own `outputDimensions` transform declares `x`/`y` at 4 nm/voxel and `z` at 40 nm/voxel) — the same convention the main tool uses everywhere else, so no unit conversion is needed when consuming these files. Do not divide by 4/40 again.

This repo exists purely as a lightweight fetch target for the main tool (via `raw.githubusercontent.com`, chosen over jsDelivr's GitHub CDN for its short, predictable ~5-minute cache — see the main tool's comments above `VASC_TRACE_BASE`), which lazy-fetches only the categories a user toggles on rather than embedding ~52 MB of raw tracing data in the tool itself. Source tracings by Søren Grubb.

## Line-count reduction

Neuroglancer local annotations are embedded directly in the viewer's state JSON, which then goes in the URL — so unlike a normal download, size here is bounded by URL length rather than file size. The raw hand tracings (up to ~57,000 line segments for one category) exceeded that limit and silently failed to open in Neuroglancer. Each file here has therefore already been simplified from the raw tracing: every traced vessel is decomposed into its connected "chain" between branch points and free ends, then simplified with Ramer–Douglas–Peucker to drop interior points that don't meaningfully change the path, at a tolerance well under any vessel's own radius.

Two tolerances are used, by category:

- **Capillaries** (`arterial_cap_order1/2/3`, `venous_cap_order1/2/3`) — 300 nm tolerance. `venous_cap_order2` and `venous_cap_order3` additionally have whole chains subsampled (after sorting into Z-order/Morton code, so the retained subset stays evenly spread rather than clustering) since most of their segment count comes from capillary *count* rather than per-capillary detail.
- **Major vessels** (`pial_arterioles`, `penetrating_arterioles`, `ascending_venules`, `pial_venules`) — 1000 nm tolerance, but **no chain-dropping**: every single traced vessel is kept. An earlier version chain-dropped these too (e.g. keeping only every 4th penetrating arteriole) for a tighter file size, but that visibly thinned out the vessel pattern rather than just coarsening each vessel's line — missing whole arterioles/venules reads very differently from a slightly straighter line through the same ones. Consequence: `ascending_venules.json` and `penetrating_arterioles.json` are the two largest files here (and, individually, larger than the main tool's own URL-size warning threshold) — that warning exists specifically to let a user decide whether to proceed rather than to force this file to shrink further at the cost of completeness.

If a category's outline still looks unacceptably coarse or the file is unusably large, the tolerance for that category is the parameter to revisit — tell the tool's maintainer which category and which direction (finer detail vs. smaller file).

`arterial_cap_order3.json` uses the `"3. order1"` (unarchived / current) layer from the July 2026 export of `arterial 3 order cap.txt`, not the earlier `"3. order"` layer, which was an unfinished pass.
