# Implementation Template

## Utility Functions

Every custom layer needs shader compilation and matrix math. Include these utilities:

### Shader Compilation

```js
function createShader(gl, type, source) {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, source);
  gl.compileShader(shader);
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    const log = gl.getShaderInfoLog(shader);
    gl.deleteShader(shader);
    throw new Error('Shader compile error: ' + log);
  }
  return shader;
}

function createProgram(gl, vsSource, fsSource) {
  const vs = createShader(gl, gl.VERTEX_SHADER, vsSource);
  const fs = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
  const program = gl.createProgram();
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    const log = gl.getProgramInfoLog(program);
    gl.deleteProgram(program);
    throw new Error('Program link error: ' + log);
  }
  // Shaders can be detached and deleted after linking
  gl.deleteShader(vs);
  gl.deleteShader(fs);
  return program;
}
```

### CPU Matrix Multiply (vec4)

```js
function mulMat4Vec4(m, v) {
  return [
    m[0] * v[0] + m[4] * v[1] + m[8]  * v[2] + m[12] * v[3],
    m[1] * v[0] + m[5] * v[1] + m[9]  * v[2] + m[13] * v[3],
    m[2] * v[0] + m[6] * v[1] + m[10] * v[2] + m[14] * v[3],
    m[3] * v[0] + m[7] * v[1] + m[11] * v[2] + m[15] * v[3],
  ];
}
```

### Mercator Conversion

```js
function lngLatToMercator(lng, lat) {
  const mc = mapboxgl.MercatorCoordinate.fromLngLat([lng, lat]);
  return [mc.x, mc.y];
}
```

---

## Line / Path Ribbon Layer Template

Full skeleton for rendering variable-width lines with RTC precision and animated patterns.

```js
const VS_LINE = `
  precision highp float;
  attribute vec2 a_pos;       // local offset from origin (Mercator)
  attribute vec2 a_normal;    // perpendicular normal
  attribute float a_distance; // cumulative line distance for animation
  uniform mat4 u_matrix;
  uniform vec4 u_originClip;
  uniform float u_width;      // half-width in Mercator units
  varying float v_distance;
  void main() {
    vec2 offset = a_pos + a_normal * u_width;
    vec4 clipOffset = u_matrix * vec4(offset, 0.0, 0.0);
    gl_Position = u_originClip + clipOffset;
    v_distance = a_distance;
  }
`;

const FS_LINE = `
  #ifdef GL_FRAGMENT_PRECISION_HIGH
  precision highp float;
  #else
  precision mediump float;
  #endif
  uniform vec4 u_color;  // premultiplied RGBA
  uniform float u_time;
  varying float v_distance;
  void main() {
    // Optional animated dash pattern
    float pattern = step(0.5, fract(v_distance * 0.1 - u_time * 0.5));
    gl_FragColor = u_color * pattern;
  }
`;

const lineLayer = {
  id: 'custom-line',
  type: 'custom',
  renderingMode: '2d',

  onAdd(map, gl) {
    this.map = map;
    this.program = createProgram(gl, VS_LINE, FS_LINE);
    this.loc = {
      aPos:       gl.getAttribLocation(this.program, 'a_pos'),
      aNormal:    gl.getAttribLocation(this.program, 'a_normal'),
      aDistance:   gl.getAttribLocation(this.program, 'a_distance'),
      uMatrix:    gl.getUniformLocation(this.program, 'u_matrix'),
      uOriginClip: gl.getUniformLocation(this.program, 'u_originClip'),
      uWidth:     gl.getUniformLocation(this.program, 'u_width'),
      uColor:     gl.getUniformLocation(this.program, 'u_color'),
      uTime:      gl.getUniformLocation(this.program, 'u_time'),
    };
    this.posBuf = gl.createBuffer();
    this.normalBuf = gl.createBuffer();
    this.distBuf = gl.createBuffer();
    this.idxBuf = gl.createBuffer();
    this.indexCount = 0;

    // Build and upload geometry
    this._buildGeometry(gl);

    // Handle WebGL context loss
    this._onLost = () => { this.indexCount = 0; };
    this._onRestored = () => {
      this.program = createProgram(gl, VS_LINE, FS_LINE);
      // Re-cache locations, recreate buffers, re-upload geometry
      this._buildGeometry(gl);
    };
    map.getCanvas().addEventListener('webglcontextlost', this._onLost);
    map.getCanvas().addEventListener('webglcontextrestored', this._onRestored);
  },

  _buildGeometry(gl) {
    // 1. Convert coordinates to Mercator
    // 2. Pick origin, compute local offsets
    // 3. Compute normals, miter joins, line distances
    // 4. Build index buffer
    // 5. Upload all buffers with gl.bufferData(gl.ARRAY_BUFFER, data, gl.STATIC_DRAW)
    // this.origin = [originX, originY];
    // this.indexCount = indices.length;
  },

  render(gl, matrix) {
    if (!this.indexCount) return;

    const originClip = mulMat4Vec4(matrix, [this.origin[0], this.origin[1], 0, 1]);

    gl.useProgram(this.program);
    gl.uniformMatrix4fv(this.loc.uMatrix, false, matrix);
    gl.uniform4fv(this.loc.uOriginClip, originClip);
    gl.uniform1f(this.loc.uWidth, this.halfWidth);
    gl.uniform4fv(this.loc.uColor, [0.0, 0.5, 1.0, 1.0]); // premultiplied
    gl.uniform1f(this.loc.uTime, performance.now() / 1000);

    // Bind position
    gl.bindBuffer(gl.ARRAY_BUFFER, this.posBuf);
    gl.enableVertexAttribArray(this.loc.aPos);
    gl.vertexAttribPointer(this.loc.aPos, 2, gl.FLOAT, false, 0, 0);

    // Bind normal
    gl.bindBuffer(gl.ARRAY_BUFFER, this.normalBuf);
    gl.enableVertexAttribArray(this.loc.aNormal);
    gl.vertexAttribPointer(this.loc.aNormal, 2, gl.FLOAT, false, 0, 0);

    // Bind distance
    gl.bindBuffer(gl.ARRAY_BUFFER, this.distBuf);
    gl.enableVertexAttribArray(this.loc.aDistance);
    gl.vertexAttribPointer(this.loc.aDistance, 1, gl.FLOAT, false, 0, 0);

    // Draw
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.idxBuf);
    gl.disable(gl.DEPTH_TEST);
    gl.enable(gl.BLEND);
    gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
    gl.drawElements(gl.TRIANGLES, this.indexCount, gl.UNSIGNED_SHORT, 0);

    if (this.animated) this.map.triggerRepaint();
  },

  onRemove(map, gl) {
    gl.deleteBuffer(this.posBuf);
    gl.deleteBuffer(this.normalBuf);
    gl.deleteBuffer(this.distBuf);
    gl.deleteBuffer(this.idxBuf);
    gl.deleteProgram(this.program);
    map.getCanvas().removeEventListener('webglcontextlost', this._onLost);
    map.getCanvas().removeEventListener('webglcontextrestored', this._onRestored);
  },
};
```

---

## Polygon Fill Layer Template

Skeleton for rendering filled polygons with earcut triangulation and RTC precision.

```js
const VS_POLYGON = `
  precision highp float;
  attribute vec2 a_pos;
  uniform mat4 u_matrix;
  uniform vec4 u_originClip;
  void main() {
    vec4 clipOffset = u_matrix * vec4(a_pos, 0.0, 0.0);
    gl_Position = u_originClip + clipOffset;
  }
`;

const FS_POLYGON = `
  #ifdef GL_FRAGMENT_PRECISION_HIGH
  precision highp float;
  #else
  precision mediump float;
  #endif
  uniform vec4 u_color;
  void main() {
    gl_FragColor = u_color; // premultiplied RGBA
  }
`;

const polygonLayer = {
  id: 'custom-polygon',
  type: 'custom',
  renderingMode: '2d',

  onAdd(map, gl) {
    this.map = map;
    this.program = createProgram(gl, VS_POLYGON, FS_POLYGON);
    this.loc = {
      aPos:       gl.getAttribLocation(this.program, 'a_pos'),
      uMatrix:    gl.getUniformLocation(this.program, 'u_matrix'),
      uOriginClip: gl.getUniformLocation(this.program, 'u_originClip'),
      uColor:     gl.getUniformLocation(this.program, 'u_color'),
    };
    this.posBuf = gl.createBuffer();
    this.idxBuf = gl.createBuffer();
    this.indexCount = 0;

    this._buildGeometry(gl);
  },

  _buildGeometry(gl) {
    // 1. Flatten polygon rings to coordinate array
    // 2. Convert to Mercator, compute local offsets from origin
    // 3. Triangulate: const indices = earcut(flatCoords, holeIndices, 2);
    // 4. Upload vertex and index buffers
  },

  render(gl, matrix) {
    if (!this.indexCount) return;

    const originClip = mulMat4Vec4(matrix, [this.origin[0], this.origin[1], 0, 1]);

    gl.useProgram(this.program);
    gl.uniformMatrix4fv(this.loc.uMatrix, false, matrix);
    gl.uniform4fv(this.loc.uOriginClip, originClip);
    gl.uniform4fv(this.loc.uColor, [0.2, 0.0, 0.0, 0.2]); // premultiplied: rgba(1,0,0,0.2)

    gl.bindBuffer(gl.ARRAY_BUFFER, this.posBuf);
    gl.enableVertexAttribArray(this.loc.aPos);
    gl.vertexAttribPointer(this.loc.aPos, 2, gl.FLOAT, false, 0, 0);

    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.idxBuf);
    gl.disable(gl.DEPTH_TEST);
    gl.enable(gl.BLEND);
    gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
    gl.drawElements(gl.TRIANGLES, this.indexCount, gl.UNSIGNED_SHORT, 0);
  },

  onRemove(map, gl) {
    gl.deleteBuffer(this.posBuf);
    gl.deleteBuffer(this.idxBuf);
    gl.deleteProgram(this.program);
  },
};
```

---

## Point / Symbol Layer Template

Skeleton for rendering points as billboard quads.

```js
const VS_POINT = `
  precision highp float;
  attribute vec2 a_pos;       // point center (local offset)
  attribute vec2 a_quadOffset; // corner: [-1,-1], [1,-1], [1,1], [-1,1]
  uniform mat4 u_matrix;
  uniform vec4 u_originClip;
  uniform float u_size;       // point radius in Mercator units
  varying vec2 v_uv;
  void main() {
    vec2 offset = a_pos + a_quadOffset * u_size;
    vec4 clipOffset = u_matrix * vec4(offset, 0.0, 0.0);
    gl_Position = u_originClip + clipOffset;
    v_uv = a_quadOffset; // [-1,1] range for circle SDF
  }
`;

const FS_POINT = `
  #ifdef GL_FRAGMENT_PRECISION_HIGH
  precision highp float;
  #else
  precision mediump float;
  #endif
  uniform vec4 u_color;
  varying vec2 v_uv;
  void main() {
    float dist = length(v_uv);
    float alpha = 1.0 - smoothstep(0.9, 1.0, dist);
    gl_FragColor = u_color * alpha; // premultiplied
  }
`;
```

Build quads: for each point `P`, emit 4 vertices with `a_pos = P` and `a_quadOffset` at each corner. Index buffer: `[0,1,2, 0,2,3]` per quad.

---

## CPU Geometry Notes
1. Convert coordinates to Mercator using `mapboxgl.MercatorCoordinate.fromLngLat()`.
2. Build local positions relative to one origin for RTC precision.
3. Build index buffer once, cache counts.
4. Re-upload buffers only after data/style changes, never per-frame.
5. For interleaved attributes, compute stride/offset carefully.

## Animation Notes
1. Add `lineDistance` or `progress` attribute for flowing/animated patterns.
2. Keep animation parameters as uniforms (`u_time`, `u_speed`, `u_phase`).
3. Avoid rebuilding geometry per frame — animate via shader uniforms only.
4. Call `map.triggerRepaint()` only while animation is active.

## Context Loss Recovery
Register handlers in `onAdd`, remove in `onRemove`:

```js
onAdd(map, gl) {
  // ... init code ...
  this._onLost = (e) => {
    e.preventDefault(); // Allow restore
    this.indexCount = 0; // Prevent drawing with deleted resources
  };
  this._onRestored = () => {
    // Recreate program, buffers, textures, re-upload data
    this._initGPUResources(gl);
  };
  map.getCanvas().addEventListener('webglcontextlost', this._onLost);
  map.getCanvas().addEventListener('webglcontextrestored', this._onRestored);
},
onRemove(map, gl) {
  // ... cleanup code ...
  map.getCanvas().removeEventListener('webglcontextlost', this._onLost);
  map.getCanvas().removeEventListener('webglcontextrestored', this._onRestored);
},
```
