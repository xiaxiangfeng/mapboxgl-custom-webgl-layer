---
name: mapboxgl-custom-webgl-layer
description: Build and troubleshoot mapboxgl `type: 'custom'` layers using raw WebGL (no threejs), including shader setup, buffer pipelines, matrix usage, animation uniforms, and precision-safe coordinate handling (RTC/local origin). Use for custom GIS overlays (line ribbons, polygons, animated arrows/symbols) and for issues like jitter during pan/zoom, geometry tearing, flicker, or empty rendering.
---

# Mapboxgl Custom Webgl Layer

Implement custom mapbox layers with deterministic WebGL lifecycle, stable geometry generation, and precision-safe rendering. Follow the workflow and checklists below before changing style-level settings.

## Workflow
1. Normalize input data into explicit `LineString`/`MultiLineString`/polygon primitives.
2. Construct a mapbox layer object with `id`, `type: 'custom'`, `renderingMode`, `onAdd`, `render`, and `onRemove`.
3. Compile/link shaders in `onAdd`, cache attribute/uniform locations, and allocate buffers.
4. Build geometry on CPU and upload typed arrays with `gl.bufferData`.
5. In `render`, set uniforms (`u_matrix`, time, colors, style params), bind buffers, and draw.
6. Trigger repaint only when animation is active.
7. Delete buffers/program in `onRemove`.

## Precision Rules
1. Keep geographic coordinates on CPU.
2. Convert to `MercatorCoordinate` on CPU only.
3. Pick one local origin (`originMercator`) per geometry batch.
4. Store vertex positions as local offsets: `local = world - origin`.
5. Compute `originClip = matrix * vec4(origin, 0, 1)` on CPU each frame.
6. In vertex shader, transform only local offsets with `w=0`, then add `originClip`.
7. Avoid adding large world coordinates and small offsets in GPU vertex math.
8. Use `highp` in vertex shader and fragment fallback guard:

```glsl
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif
```

Reference: [precision-playbook.md](references/precision-playbook.md)

## Geometry Rules For Path Ribbons
1. Sample centerline in input order (do not reorder points).
2. Compute direction from neighboring samples and derive normals.
3. Use miter join with clamp (`miterLimit`) and bevel fallback on sharp turns.
4. Build triangle strip indices deterministically.
5. For animated patterns, compute cumulative line distance (`lineDistance`) on CPU.
6. Rebuild buffers only when geometry-affecting props change (data/width/offset/join style).

Reference: [implementation-template.md](references/implementation-template.md)

## Render-State Rules
1. Set blend state explicitly for alpha layers (`SRC_ALPHA`, `ONE_MINUS_SRC_ALPHA`).
2. Disable depth test for pure overlay layers unless 3D ordering is required.
3. Keep attribute bindings explicit every frame.
4. Avoid hidden global state assumptions between custom layers.
5. Keep `renderingMode` aligned with intent (`2d` for overlay, `3d` if depth interaction is needed).

## Debug Sequence
1. Check layer is actually added: `map.getLayer(id)`.
2. Check shader compile/link logs before geometry debugging.
3. Log `indexCount`/`vertexCount`; if zero, fix data normalization or geometry build.
4. If geometry exists but flickers/jitters, verify RTC pipeline and `w=0` local transform.
5. If spikes or self-intersections appear, tune join strategy (`miterLimit`, bevel fallback).
6. If visuals disappear on some GPUs, check precision declarations and fragment fallback.

Reference: [troubleshooting-checklist.md](references/troubleshooting-checklist.md)

## Output Contract
1. Return implementation changes with concrete file paths and line-level rationale.
2. Explain precision decisions explicitly when touching vertex transforms.
3. Add a short validation checklist (render visible, zoom/pan stability, no shader errors).
