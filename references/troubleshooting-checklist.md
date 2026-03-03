# Troubleshooting Checklist

## Nothing Renders
1. Verify layer exists: `map.getLayer(layerId)`.
2. Verify source data normalization produced features.
3. Log vertex/index counts after geometry build.
4. Print shader compile/link logs.
5. Confirm attribute locations are not `-1`.
6. Confirm `drawElements` count > 0.

## Renders But Jitters On Pan/Zoom
1. Confirm RTC pipeline is active.
2. Confirm shader uses local offsets with `w=0`.
3. Confirm `originClip` is computed on CPU each frame.
4. Confirm no hidden path still uses `world + local` in GPU.

## Spikes Or Self-Intersections
1. Reduce miter limit.
2. Add bevel fallback on sharp turns.
3. Remove duplicate consecutive points before join math.
4. Check centerline sample order is stable.

## Flicker Or Z-Fighting
1. Disable depth test for overlay layers.
2. Keep blending explicit.
3. Ensure draw order (`beforeId`) is deterministic.

## Looks Different Across Devices
1. Add fragment precision fallback guard.
2. Avoid undefined shader behavior (uninitialized vars, divide by zero).
3. Clamp configurable uniforms to safe ranges.
