# Precision Playbook

## Goal
Keep large geographic coordinates out of GPU math paths that mix large and small values.

## Recommended Pipeline (RTC)
1. Convert `lng/lat` to Mercator on CPU.
2. Select one `originMercator` for the batch.
3. Store vertex attributes as local offsets:
   `local = world - originMercator`.
4. Per frame on CPU:
   `originClip = matrix * vec4(originMercator, 0, 1)`.
5. In vertex shader:
   `offsetClip = matrix * vec4(local, 0, 0)`.
6. Output:
   `gl_Position = originClip + offsetClip`.

## Why This Works
- CPU uses double precision for the big transform.
- GPU handles mostly local values near zero.
- The dangerous operation (`big + small`) is avoided in world space.

## Anti-Patterns
- `gl_Position = matrix * vec4(local + origin, 0, 1)`.
- Recomputing origin per-vertex.
- Feeding `lng/lat` directly as vertex attributes.
- Using `mediump` vertex precision for map transforms.

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
- If very large scenes still jitter, split coordinates into high/low parts (fp64-like).
- Rebase origin when camera moves far from current batch center.
