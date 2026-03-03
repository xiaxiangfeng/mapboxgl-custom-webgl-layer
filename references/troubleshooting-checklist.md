# Troubleshooting Checklist

## Nothing Renders
1. Verify layer exists: `map.getLayer(layerId)`.
2. Verify source data normalization produced features — log feature count.
3. Log vertex/index counts after geometry build — if zero, fix data pipeline.
4. Print shader compile/link logs (`gl.getShaderInfoLog`, `gl.getProgramInfoLog`).
5. Confirm attribute locations are not `-1` (`gl.getAttribLocation` returns -1 if name not found or optimized away).
6. Confirm `drawElements` / `drawArrays` count > 0.
7. Check `renderingMode` matches intent — `'3d'` layers may be clipped by depth buffer.

## Renders But Jitters On Pan/Zoom
1. Confirm RTC pipeline is active (origin subtracted from vertices).
2. Confirm vertex shader uses local offsets with `w=0`: `matrix * vec4(local, 0, 0)`.
3. Confirm `originClip` is computed on CPU each frame with `mulMat4Vec4(matrix, [ox, oy, 0, 1])`.
4. Confirm no hidden path still uses `world + local` or `local + origin` in GPU math.
5. Check that `precision highp float` is declared in vertex shader.

## Colors Look Wrong (Too Bright, Washed Out, Semi-Transparent When Opaque)
1. Confirm blend function is premultiplied alpha: `gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA)`.
2. Confirm fragment shader outputs premultiplied color: `gl_FragColor = vec4(rgb * a, a)`.
3. If using `SRC_ALPHA, ONE_MINUS_SRC_ALPHA` (non-premultiplied), colors will composite incorrectly with Mapbox's rendering pipeline.
4. Check if uniform color values are already premultiplied — avoid double premultiplication.

## Spikes Or Self-Intersections (Lines)
1. Reduce miter limit (try `miterLimit ≈ 2.0`).
2. Add bevel fallback on sharp turns (angle > threshold).
3. Remove duplicate consecutive points before join math (`dist(p1, p2) < epsilon`).
4. Check centerline sample order is stable — do not reorder points.
5. Check that normals are not flipped (consistent winding).

## Polygon Triangulation Issues
1. Confirm rings follow GeoJSON winding convention (outer CCW, holes CW) or that earcut handles both.
2. Check for self-intersecting rings — earcut may produce incorrect results.
3. Verify `holeIndices` array is correct when passing to earcut.
4. Log triangle count: should be `(vertexCount - 2 - 2*holeCount)` triangles for simple polygons.
5. Visualize wireframe (`gl.LINE_STRIP` or `gl.LINES`) to inspect triangulation quality.

## Flicker Or Z-Fighting
1. Disable depth test for 2D overlay layers: `gl.disable(gl.DEPTH_TEST)`.
2. Keep blend state explicit — don't rely on Mapbox's state.
3. Ensure draw order (`beforeId` in `addLayer`) is deterministic.
4. For 3D layers using depth, apply small polygon offset: `gl.polygonOffset(1, 1)`.

## Looks Different Across Devices / Browsers
1. Add fragment precision fallback guard (see Precision Playbook).
2. Avoid undefined shader behavior: uninitialized variables, division by zero, out-of-range values.
3. Clamp configurable uniforms to safe ranges.
4. Test on both WebGL 1 and WebGL 2 contexts — some extensions differ.

## Layer Disappears After Tab Switch / Context Loss
1. Listen for `webglcontextlost` on the map canvas — set `indexCount = 0` to prevent drawing.
2. Listen for `webglcontextrestored` — recreate program, buffers, textures, re-upload data.
3. Call `e.preventDefault()` in the `webglcontextlost` handler to allow restore.
4. Remove listeners in `onRemove`.

## Performance Issues
1. Check if geometry is being rebuilt every frame (should only rebuild on data change).
2. Check if `triggerRepaint()` is called when not needed (causes unnecessary redraws).
3. Profile draw call count — batch features into fewer draw calls.
4. Check buffer usage hint: use `STATIC_DRAW` for static geometry.
5. Consider disabling unused attributes to avoid unnecessary GPU work.
