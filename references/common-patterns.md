# Common Patterns

Reusable implementation patterns for common custom layer use cases. All patterns follow the RTC precision pipeline and premultiplied alpha blending described in the main skill document.

---

## Pattern 1: Solid Line Ribbon

**Use case**: Roads, routes, boundaries with constant or per-segment width.

### Data Flow
1. Input: array of `[lng, lat]` coordinate arrays (one per line feature).
2. Convert each point to Mercator. Pick batch origin.
3. For each segment, compute direction `d = normalize(p[i+1] - p[i])` and normal `n = [-d.y, d.x]`.
4. At each vertex, emit two points: `pos ± normal * halfWidth`.
5. Handle joins:
   - **Miter**: average adjacent normals, extend by `1/cos(halfAngle)`, clamp to `miterLimit`.
   - **Bevel**: when miter exceeds limit, insert extra triangle at the join.
6. Build triangle indices: for N centerline points → `2*(N-1)` triangles.

### Vertex Attributes
| Attribute | Type | Description |
|---|---|---|
| `a_pos` | vec2 | Local offset from origin (Mercator) |
| `a_normal` | vec2 | Perpendicular normal direction |
| `a_distance` | float | Cumulative distance along line (for dashes/animation) |

### Key Shader Uniforms
`u_matrix`, `u_originClip`, `u_width`, `u_color`, `u_time` (if animated).

---

## Pattern 2: Dashed / Animated Line

**Use case**: Dashed borders, flowing traffic, animated routes.

Extends Pattern 1 by using `a_distance` (cumulative line length) in the fragment shader:

```glsl
float dashPhase = fract(v_distance * u_dashScale - u_time * u_speed);
float alpha = step(u_gapRatio, dashPhase);
gl_FragColor = u_color * alpha;
```

Key: compute `a_distance` on CPU as cumulative Euclidean distance along the centerline in Mercator space. Adjust `u_dashScale` for dash density.

---

## Pattern 3: Filled Polygon

**Use case**: Building footprints, zones, regions, heatmap cells.

### Data Flow
1. Input: GeoJSON Polygon with outer ring and optional holes.
2. Flatten coordinates to `[x0, y0, x1, y1, ...]` array in Mercator local offsets.
3. Build hole indices array: `[outerRingLength, outerRingLength + hole1Length, ...]`.
4. Triangulate: `const indices = earcut(flatCoords, holeIndices, 2)`.
5. Upload positions and indices as buffers.

### Notes
- For extrusion (3D buildings), add `a_height` attribute and set `renderingMode: '3d'`.
- For per-feature coloring, add `a_color` attribute or use a texture lookup.
- earcut is available as an npm package: `earcut`.

---

## Pattern 4: Point Circles (Billboard Quads)

**Use case**: Sensor locations, event markers, scatter plots.

### Data Flow
1. Input: array of `[lng, lat]` points.
2. For each point, emit 4 vertices (quad corners) with shared `a_pos` and different `a_quadOffset`.
3. Indices per quad: `[0,1,2, 0,2,3]` (two triangles).

### Fragment Shader (Circle SDF)
```glsl
float dist = length(v_uv);
float edge = fwidth(dist);
float alpha = 1.0 - smoothstep(1.0 - edge, 1.0, dist);
gl_FragColor = u_color * alpha; // premultiplied
```

### Pixel-Size Points
To make points a fixed pixel size regardless of zoom, compute size in the vertex shader using the projection matrix diagonal:

```glsl
float pixelScale = u_size / u_matrix[0][0]; // approximate
vec2 offset = a_pos + a_quadOffset * pixelScale;
```

Or pass `u_pixelRatio` and screen dimensions as uniforms.

---

## Pattern 5: Textured Icons

**Use case**: Custom map markers, POI icons, directional arrows.

### Data Flow
1. Prepare a texture atlas (spritesheet) with all icons.
2. For each icon, emit a billboard quad (same as Pattern 4).
3. Pass texture coordinates (`a_texCoord`) pointing to the icon's position in the atlas.
4. In `onAdd`, load the texture with `gl.texImage2D`.

### Key Details
- Use `gl.NEAREST` or `gl.LINEAR` filtering depending on icon style.
- For rotated icons (e.g., arrow heads on a route), pass `a_rotation` attribute and rotate UV or quad offset in shader.
- Use premultiplied alpha textures or adjust blend function accordingly.

---

## Pattern 6: Pulsing / Ripple Animation

**Use case**: Alert indicators, expanding search radius, hover effects.

Extend Pattern 4 with time-based scale and fade:

```glsl
float phase = fract(u_time * u_speed);
float radius = mix(u_minRadius, u_maxRadius, phase);
float alpha = 1.0 - phase; // fade out as expanding
float circle = 1.0 - smoothstep(radius - edge, radius, dist);
gl_FragColor = u_color * circle * alpha;
```

Use multiple phase offsets for concurrent ripples.

---

## Pattern 7: Data-Driven Heatmap

**Use case**: Density visualization, weighted point clouds.

### Approach
1. Render points as large additive quads into an offscreen FBO (use `prerender`).
2. In `prerender`: use `gl.blendFunc(gl.ONE, gl.ONE)` for additive blending.
3. In `render`: sample the FBO texture and apply a color ramp.

This requires the `prerender` method of the CustomLayerInterface.

---

## General Tips
- **Batch features** into one draw call per visual style whenever possible.
- **Cache geometry** — rebuild only when data or geometry-affecting style changes.
- **Interleave attributes** into a single buffer when attributes are always used together.
- For > 65535 vertices, use `Uint32Array` indices with the `OES_element_index_uint` extension.
- Always test with the **globe** projection in addition to Mercator if the layer needs to work globally.
