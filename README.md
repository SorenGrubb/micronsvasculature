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
  "nLines": 6229,
  "pointsB64": "<base64-encoded little-endian Float32 array, 3 floats per point: x,y,z>",
  "linesB64": "<base64-encoded little-endian Float32 array, 6 floats per line: pointA.x,pointA.y,pointA.z,pointB.x,pointB.y,pointB.z>"
}
```

Coordinates are voxel indices, exactly as they appear in the original Neuroglancer annotation-layer tracings (their own `outputDimensions` transform declares `x`/`y` at 4 nm/voxel and `z` at 40 nm/voxel) — the same convention the main tool uses everywhere else, so no unit conversion is needed when consuming these files. Do not divide by 4/40 again.

This repo exists purely as a lightweight CDN target (via [jsDelivr](https://www.jsdelivr.com/)) for the main tool, which lazy-fetches only the categories a user toggles on rather than embedding ~52 MB of raw tracing data in the tool itself. Source tracings by Søren Grubb.

## Line-count reduction

Neuroglancer local annotations are embedded directly in the viewer's state JSON, which then goes in the URL — so unlike a normal download, size here is bounded by URL length rather than file size. The raw hand tracings (up to ~57,000 line segments for one category) exceeded that limit and silently failed to open in Neuroglancer. Each file here has therefore already been simplified from the raw tracing, in two steps:

1. **Collinear-point removal.** Each traced vessel was decomposed into its connected "chain" between branch points and free ends, then simplified with Ramer–Douglas–Peucker at a 300 nm tolerance — well under any vessel's radius, so no visible change in shape — to drop interior points that don't meaningfully change the path.
2. **Chain subsampling.** For the categories built from many independent, short vessels (penetrating arterioles, ascending venules, and the busiest capillary orders) rather than one long connected structure, most of the segment count comes from vessel *count*, not per-vessel detail. For those, whole chains were additionally subsampled after sorting into Z-order (Morton code) — e.g. every 4th penetrating arteriole kept — so the retained subset stays evenly spread through the volume and still conveys the overall density/pattern, rather than exhaustively including every last traced vessel.

Every category was brought below the size of the smallest category already known to open reliably, for a safety margin. If a category still fails to open for you, it likely needs a tighter subsampling stride — tell the tool's maintainer which one, with how many vessel types you had checked at once.

`arterial_cap_order3.json` uses the `"3. order1"` (unarchived / current) layer from the July 2026 export of `arterial 3 order cap.txt`, not the earlier `"3. order"` layer, which was an unfinished pass.
