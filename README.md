# SunSync 360

**Solar geometry and true-north registration for 360° outdoor imagery.**
Part of the Vis-O-Matic suite. Single-file HTML, runs entirely in the browser, nothing is uploaded.

Version 1.0 — 14 August 2026 · `sunsync-360_index_2026-08-14.html`

---

## What problem it solves

An equirectangular 360° frame has a horizontal axis that looks like a compass and isn't one. Consumer 360 cameras do not always write their heading: `GPano:PoseHeadingDegrees` into the EXIF. Without a heading, a column profile, a grid statistic, or any azimuthal claim is measured against an arbitrary reference that can differ between frames of the same capture.

SunSync recovers the heading from the sun, using the GPS position and timestamp already in the file.

**Why it can be trusted:** solar *elevation* does not depend on heading. The gap between the sun's measured elevation in the frame and its computed elevation is therefore a free, independent confidence score on every frame — a small residual means the detection found the sun, a large one means it found a bright cloud and the frame is gated out rather than silently mis-registered.

**Second opinion:** cross-correlating the static terrain band below the horizon gives a *relative* heading with no astronomy involved. One sun-anchored frame makes it absolute. Where the two methods agree, both are right; where they diverge, the sun detection has locked onto something that is not the sun.

---

## Validation

Everything below was measured, not assumed.

### Solar position
NOAA algorithm in plain arithmetic, no library. Cross-checked against `pvlib`'s NREL SPA on 2,000 random times and locations spanning five years and ±70° latitude:

| | p99 error | max error |
|---|---|---|
| Apparent elevation (sun > 5°) | 0.012° | **0.018°** |
| Azimuth (sun > 5°) | 0.033° | **0.233°** |

One pixel of a 2880-wide equirectangular frame is 0.125°.

### Sun detection
The browser implementation reproduces the Python reference **exactly** — worst yaw difference **0.000°** across four test frames covering clear midday sun, low evening sun, and a thin-cloud case.

### Terrain cross-correlation
- Synthetically rolled frames, six shifts from −50° to +90°: recovered to **0.0000°**.
- Against the sun on nine real frames spanning a **71° camera rotation** (Anderson Hill, 6 Aug 2026): agreement **0.00–0.43°**.
- Across all 19 test frames from four captures: median **0.14°**, worst **0.43°**.

Astronomy and scene content are entirely independent measurements. Agreement at this level is strong evidence that both are right.

### Internal consistency
With registration applied, the computed-sun marker lands on the detected-sun marker to **0.0000°** in the viewer — a direct check that the sign convention holds end to end.

---

## The sign convention

Fixed once and used everywhere. Getting it wrong flips headings by twice the yaw, which is why it is stated explicitly:

```
yaw = frame azimuth of the sun − true azimuth of the sun
true bearing   = frame azimuth − yaw
frame azimuth  = bearing + yaw
```

Terrain: if content is displaced by `shiftDeg` relative to the anchor, this frame's yaw is the anchor's **plus** that shift.

---

## Workflow

1. **Load** — drop in images or a folder. EXIF, GPS, and GPano geometry are read automatically. Partial equirectangular strips and fisheye frames are supported if you declare their geometry. Missing position or time can be typed in and is flagged as manual in the export.
2. **Geometry** — confirm the timezone, review computed solar positions and air mass.
3. **Register** — solve from the sun, cross-check against terrain, inspect the yaw-versus-time trace, choose per-frame or consensus.
4. **Viewer** — equirectangular, polar, or drag-to-look rectilinear, with sun markers, true-north ticks, scattering-angle contours and mask preview.
5. **Analyse** — scattering-angle annuli, solar almucantar, principal plane, or elevation bands; phase curves.
6. **Match** — solar index across the whole library, with a pairwise dataset-overlap matrix.
7. **Export** — geometry sidecar CSV/JSON, and optionally north-aligned images.

---

## Read the timezone warning

This is the one input the tool cannot verify for you.

These files carry no `OffsetTimeOriginal` and no GPS clock, so the offset is inferred — from this computer's timezone rules evaluated on the capture date, which does apply daylight saving. **An hour of error moves the sun about 15° in azimuth, and near midday it barely changes the elevation, so the residual check will not catch it.**

A constant clock error and a constant heading error are mathematically indistinguishable at high sun. Low-sun frames break the tie: near the horizon the sun descends fast enough that a clock error shows up in the elevation residual, roughly ±1 minute per ±0.25°. **Register an evening or morning set first, and carry its clock correction to your midday sets.**

---

## Sidecar, not resampling

The geometry CSV is one row per frame, joinable on filename stem — the key the Batch Image Analyzer's compare tool already uses. Nothing is resampled; the imagery is untouched.

Re-exporting rotated images is offered but is the second-best path: it resamples every pixel and moves the stitch seam to a new part of the sky. With integer-pixel snapping on (the default) the roll is an exact lossless shift in 0.125° steps, which removes almost all of that objection. Note that the browser's canvas encoder does not carry EXIF through — keep the originals.

### Sidecar columns

`filename`, `filename_stem`, `dataset`, `datetime_local`, `tz_assumed_hours`, `clock_offset_s`, `lat`, `lon`, `altitude_m`, `position_source`, `time_source`, `solar_elevation_deg`, `solar_azimuth_deg`, `solar_zenith_deg`, `air_mass`, `solar_declination_deg`, `equation_of_time_min`, `sun_detected`, `sun_frame_az_deg`, `sun_frame_el_deg`, `elev_residual_deg`, `detect_prominence`, `detect_fwhm_deg`, `saturated`, `yaw_offset_deg`, `yaw_source`, `yaw_confidence`, `yaw_consensus_deg`, `yaw_deviation_from_consensus_deg`, `terrain_yaw_deg`, `terrain_correlation`, `method_disagreement_deg`, `heading_recorded_in_file_deg`, `registration_status`, `iso`, `exposure_time_s`, `f_number`, `ev`, `canvas_w`, `canvas_h`, `projection`, `elev_top_deg`, `elev_bottom_deg`, `az_span_deg`, `geometry_source`

Every row carries its own `yaw_source` and `registration_status`, so any figure built from this can state how its frames were registered.

---

## What the Batch Image Analyzer needs to consume it

1. **Load geometry sidecar** — join on filename stem, report the match rate the way `runCompare()` already reports `onlyA`/`onlyB`.
2. **Azimuth offset** — one additive term in the x→bearing mapping, applied in the column-profile, grid and Moran's-I sampling paths. The existing 360° wrap logic handles the rest.
3. **Scattering-angle spatial mode** — alongside Cartesian and polar, with Θ annuli as the aggregation unit.

---

## Design decisions worth knowing

- **Stability is measured per dataset, never pooled.** Pooling folders would report a "rotating camera" whenever two captures simply point different ways.
- **Terrain anchors are per dataset.** Correlating frames from different sites is meaningless, so it is not attempted.
- **Fixed vs roving is decided from the data**, not declared by the user: GPS spread under ~10 m with yaw spread under 2° is a stable mount; GPS fixed but yaw spread large is *a fixed position with a rotating camera*, the case no consensus heading can describe.
- **Exposure normalization is reported, not applied by default.** EV and air mass are written as metadata columns. Applying them means multiplying gamma-encoded 8-bit JPEG values, which is not the linear operation the arithmetic assumes.
- **Solid-angle weighting matters twice.** Scattering-angle annuli have their own solid angles — a bin at Θ=90° covers far more sky than one at Θ=10° — so the `solid_angle_weight` column, not the pixel count, is the honest measure of how much sky each bin represents. Phase-curve markers are sized by it.

---

## Limits

- **No sun, no terrain, no reference frame → no heading.** Uniform overcast with no ground content is unregisterable, and the tool says so rather than guessing.
- **Clock error and heading error are degenerate at high sun** (see above).
- **Auto white balance cannot be undone**, and clipped highlights near the sun carry no colour information — exclude them by angle, not by brightness threshold.
- **Sub-degree registration under heavy haze is not reliable** from the sun alone. Gate hard and expect to discard.
- **Mapillary import needs an access token** (free, from mapillary.com → Dashboard → Developers). Mapillary's `computed_compass_angle` is applied as a provisional heading marked `manual` — the sun solver should still be run to check it against the sky.
