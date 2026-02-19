# xatlas Bug Fixes: Chart Orientation

This document describes two related bugs in xatlas that cause UV charts to be incorrectly flipped/mirrored, along with their fixes.

## Overview

Both bugs result in charts having incorrect orientation (negative signed area in UV space), which causes textures to appear mirrored when applied to the mesh. The issues are:

1. **`rotateCharts` reflects instead of rotates** — When the packing algorithm rotates a chart 90° to improve packing efficiency, it actually reflects the chart across the line y=x, flipping its orientation.

2. **`fixWinding` not applied to piecewise charts** — Charts that fail initial parameterization and are recomputed via the piecewise parameterization path never have their winding corrected, even when `ChartOptions::fixWinding` is enabled.

---

## Bug 1: `rotateCharts` Reflects Instead of Rotates

**Related issue**: [Charts are reflected instead of rotated during pack optimization #150](https://github.com/jpcy/xatlas/issues/150)

### Problem

When `PackOptions::rotateCharts` is enabled, the packing algorithm attempts to rotate charts 90° to find better placements. However, the implementation uses `swap(x, y)` for both the rasterized image and the texture coordinates, which is a **reflection across the line y=x**, not a rotation.

Mathematically:
- **Reflection** (current): `(x, y) → (y, x)` — reverses orientation
- **90° CW rotation** (correct): `(x, y) → (y, W-x)` — preserves orientation

### Affected Code

**1. `drawTriangleCallback`** (rasterizing the rotated chart image):
```cpp
// Before (reflection):
args->chartBitImageRotated->set(y, x);

// After (90° CW rotation):
args->chartBitImageRotated->set(y, args->chartWidth - 1 - x);
```

**2. Texture coordinate transformation** (in `packCharts`):
```cpp
// Before (reflection):
swap(t.x, t.y);

// After (90° CW rotation):
// Must use the same width reference as the bitmap rotation to ensure
// UV placement matches collision detection.
const float rotationWidth = (float)(chartImage.width() - 1);
const float temp = t.x;
t.x = t.y;
t.y = rotationWidth - temp;
```

**Critical**: The UV rotation must use `chartImage.width() - 1` (not `chartExtents[c].x`) to match the bitmap rotation. Using the UV-space extent instead of the pixel-space dimension causes the UVs to be offset from where collision detection thinks they are, resulting in overlapping charts.

### Fix Details

1. Add `chartWidth` to `DrawTriangleCallbackArgs`:
   ```cpp
   struct DrawTriangleCallbackArgs
   {
       BitImage *chartBitImage, *chartBitImageRotated;
       int chartWidth; // Width of chartBitImage, needed for 90° rotation
   };
   ```

2. Update `drawTriangleCallback` to use proper rotation:
   ```cpp
   if (args->chartBitImageRotated) {
       // 90° clockwise rotation: (x, y) -> (y, width - 1 - x)
       args->chartBitImageRotated->set(y, args->chartWidth - 1 - x);
   }
   ```

3. Set `chartWidth` when creating callback args:
   ```cpp
   args.chartWidth = (int)chartImage.width();
   ```

4. Update the texcoord transform to match (using same width reference as bitmap):
   ```cpp
   if (best_r) {
       // 90° clockwise rotation: (x, y) -> (y, width - x)
       // Must use chartImage.width() - 1 to match the bitmap rotation
       const float rotationWidth = (float)(chartImage.width() - 1);
       const float temp = t.x;
       t.x = t.y;
       t.y = rotationWidth - temp;
   }
   ```

---

## Bug 2: `fixWinding` Not Applied to Piecewise Charts

### Problem

When `ChartOptions::fixWinding` is enabled, xatlas checks each chart's orientation after parameterization and flips the U coordinates if the chart has negative signed area. This logic exists in `Chart::parameterize()`:

```cpp
if (options.fixWinding && m_unifiedMesh->computeFaceParametricArea(0) < 0.0f) {
    for (uint32_t i = 0; i < unifiedVertexCount; i++)
        m_unifiedMesh->texcoord(i).x *= -1.0f;
}
```

However, when a chart fails initial parameterization (due to self-intersection, flipped triangles, etc.), it gets recomputed using `PiecewiseParam`. These recomputed charts are created directly from `PiecewiseParam::texcoords()` without going through `Chart::parameterize()`, so the `fixWinding` logic is never applied to them.

### Affected Code

In `runCreateAndParameterizeChartTask`, piecewise charts are created in a loop:
```cpp
for (;;) {
    const bool facesRemaining = pp.computeChart();
    if (!facesRemaining)
        break;
    // Chart created directly - fixWinding never applied!
    Chart *chart = XA_NEW_ARGS(..., pp.texcoords(), ...);
    args->charts.push_back(chart);
}
```

### Fix Details

1. Add a `fixWinding()` method to `PiecewiseParam`:
   ```cpp
   void fixWinding()
   {
       if (m_patch.isEmpty())
           return;
       // Check the signed area of the first face in the patch
       const uint32_t face = m_patch[0];
       const Vector2 &v1 = m_texcoords[m_mesh->vertexAt(face * 3 + 0)];
       const Vector2 &v2 = m_texcoords[m_mesh->vertexAt(face * 3 + 1)];
       const Vector2 &v3 = m_texcoords[m_mesh->vertexAt(face * 3 + 2)];
       const float signedArea = ((v2.x - v1.x) * (v3.y - v1.y) - 
                                 (v3.x - v1.x) * (v2.y - v1.y)) * 0.5f;
       if (signedArea < 0.0f) {
           // Flip all U coordinates for vertices in this patch
           for (uint32_t f = 0; f < m_patch.size(); f++) {
               const uint32_t patchFace = m_patch[f];
               for (uint32_t i = 0; i < 3; i++) {
                   const uint32_t vertex = m_mesh->vertexAt(patchFace * 3 + i);
                   m_texcoords[vertex].x *= -1.0f;
               }
           }
       }
   }
   ```

2. Call `fixWinding()` after `computeChart()` returns:
   ```cpp
   for (;;) {
       const bool facesRemaining = pp.computeChart();
       if (!facesRemaining)
           break;
       // Fix winding for piecewise charts (same as Chart::parameterize does)
       if (groupArgs->options->fixWinding)
           pp.fixWinding();
       Chart *chart = XA_NEW_ARGS(..., pp.texcoords(), ...);
       args->charts.push_back(chart);
   }
   ```

---

## Testing

These fixes were tested using a mesh that generates many piecewise charts. Before the fix:
- 385 out of 726 charts had incorrect (flipped) orientation

After the fix:
- All 726 charts have correct orientation
- The `fixWinding` option now works correctly for all chart types

---

## Files Modified

- `source/xatlas/xatlas.cpp`
  - `struct DrawTriangleCallbackArgs` — added `chartWidth` field
  - `drawTriangleCallback()` — fixed rotation transform for rasterization
  - `packCharts()` — fixed rotation transform for texture coordinates, set `chartWidth`
  - `struct PiecewiseParam` — added `fixWinding()` method
  - `runCreateAndParameterizeChartTask()` — call `pp.fixWinding()` for piecewise charts
