# Implementation Template

## Custom Layer Skeleton

```js
const customLayer = {
  id: 'my-custom-layer',
  type: 'custom',
  renderingMode: '2d',
  onAdd(map, gl) {
    this.map = map;
    this.gl = gl;
    this.program = createProgram(gl, VS, FS);
    this.locations = {
      aPos: gl.getAttribLocation(this.program, 'a_pos'),
      uMatrix: gl.getUniformLocation(this.program, 'u_matrix'),
      uOriginClip: gl.getUniformLocation(this.program, 'u_originClip'),
      uTime: gl.getUniformLocation(this.program, 'u_time'),
    };
    this.buffers = {
      pos: gl.createBuffer(),
      idx: gl.createBuffer(),
    };
    this.uploadGeometry();
  },
  render(gl, matrix) {
    if (!this.indexCount) return;
    const originClip = multiplyMat4Vec4(matrix, [this.origin[0], this.origin[1], 0, 1]);

    gl.useProgram(this.program);
    gl.uniformMatrix4fv(this.locations.uMatrix, false, matrix);
    gl.uniform4fv(this.locations.uOriginClip, originClip);
    gl.uniform1f(this.locations.uTime, performance.now() / 1000);

    gl.bindBuffer(gl.ARRAY_BUFFER, this.buffers.pos);
    gl.enableVertexAttribArray(this.locations.aPos);
    gl.vertexAttribPointer(this.locations.aPos, 2, gl.FLOAT, false, 0, 0);

    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.buffers.idx);
    gl.disable(gl.DEPTH_TEST);
    gl.enable(gl.BLEND);
    gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
    gl.drawElements(gl.TRIANGLES, this.indexCount, gl.UNSIGNED_SHORT, 0);

    this.map.triggerRepaint();
  },
  onRemove(map, gl) {
    gl.deleteBuffer(this.buffers.pos);
    gl.deleteBuffer(this.buffers.idx);
    gl.deleteProgram(this.program);
  },
};
```

## CPU Geometry Notes
1. Convert line coordinates to Mercator.
2. Build local positions relative to one origin.
3. Build index buffer once, cache counts.
4. Re-upload only after data/width/join changes.

## Animation Notes
1. Add `lineDistance` attribute for flowing patterns.
2. Keep animation params as uniforms.
3. Avoid rebuilding geometry per frame.
