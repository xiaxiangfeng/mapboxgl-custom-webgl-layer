# Precision Playbook

## Goal
Keep large geographic coordinates out of GPU math paths that mix large and small values. This prevents jitter / shimmer during pan and zoom.

## Coordinate System Overview
- `lng/lat` → geographic degrees. Never send to GPU directly.
- `MercatorCoordinate` → range [0, 1] on both axes. Computed on CPU via `mapboxgl.MercatorCoordinate.fromLngLat(lngLat)`.
- Local offset → `mercator - originMercator`. These are small values safe for `float32` on GPU.

## MercatorCoordinate API Reference

```js
// Convert geographic to Mercator
const mc = mapboxgl.MercatorCoordinate.fromLngLat([lng, lat], altitudeMeters);
// mc.x, mc.y, mc.z are in [0, 1] range

// Convert metric units to Mercator scale at a specific latitude
const metersPerUnit = mc.meterInMercatorCoordinateUnits();
// Use: mercatorWidth = widthInMeters * metersPerUnit
```

Key: `meterInMercatorCoordinateUnits()` varies by latitude. Always compute at the data's latitude, not at the equator.

## Recommended Pipeline (RTC)
1. Convert `lng/lat` to Mercator on CPU.
2. Select one `originMercator` for the batch (centroid or first point).
3. Store vertex attributes as local offsets:
   `local = mercator - originMercator`.
4. Per frame on CPU:
   `originClip = matrix * vec4(originMercator.x, originMercator.y, 0, 1)`.
5. In vertex shader:
   `offsetClip = matrix * vec4(a_pos, 0, 0)` — note `w=0`.
6. Output:
   `gl_Position = originClip + offsetClip`.

## Why This Works
- CPU uses **double precision** (64-bit) for the large origin transform.
- GPU handles only **local offsets near zero** — safe in float32.
- The dangerous operation (`big + small`) is moved out of GPU world space.

## Numerical Safety
- Mercator coordinates range [0, 1]. Origin values are ~0.5 for typical locations.
- Local offsets for city-scale data: ~0.0001 range → well within float32 precision.
- For country-scale data spanning many degrees, split into multiple batches with separate origins.

## Anti-Patterns
- ❌ `gl_Position = matrix * vec4(local + origin, 0, 1)` — large + small in GPU.
- ❌ Recomputing origin per-vertex on GPU.
- ❌ Feeding `lng/lat` directly as vertex attributes.
- ❌ Using `mediump` vertex precision for map transforms.
- ❌ Passing Mercator world coordinates directly as attributes without subtracting origin.

## Fallback For Fragment Precision
Use:

```glsl
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif
```

## Extreme Cases
- If very large scenes still jitter, split coordinates into high/low float parts (fp64-like emulation technique from deck.gl).
- Rebase origin when camera moves far from current batch center.
- For 3D terrain layers, apply the same RTC approach to the altitude component.
