# AI Prompt: Fix Portal See-Through Rendering (Phase 2 Bug Fix)

## Symptom
Both portals can be placed (blue/orange ovals appear on walls), but **nothing renders inside the portals** — you only see solid colored ovals. The stencil-based see-through effect is not working. The scene as viewed from the linked portal's perspective should be visible inside each oval.

## Root Cause Analysis

After auditing the code, there are **multiple interacting bugs** that collectively break the portal view rendering. They must all be fixed together.

---

### Bug #1: Main Scene Overwrites Portal Views (CRITICAL)

**Location:** `render()` function (~line 924)

**Problem:** The render order is:
1. Render portal stencil + virtual camera scene ✓
2. `gl.disable(gl.STENCIL_TEST)` then `drawScene(proj, view)` ✗

Step 2 draws the **entire scene** with no stencil test, which completely overwrites the portal view that was just rendered in step 1. The main scene paints over the pixels where the portal view was drawn.

**Fix:** When drawing the main scene after portals, use the stencil buffer to **exclude** the portal regions:

```javascript
// After rendering both portals:
// Draw main scene everywhere EXCEPT where portals were drawn
gl.enable(gl.STENCIL_TEST);
gl.stencilFunc(gl.EQUAL, 0, 0xFF);  // Only draw where stencil is 0 (no portal)
gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP);
drawScene(proj, view);
gl.disable(gl.STENCIL_TEST);
```

This preserves the portal view pixels while filling in the rest of the scene normally.

---

### Bug #2: `gl.clear(gl.DEPTH_BUFFER_BIT)` Clears Entire Screen (CRITICAL)

**Location:** `renderPortal()` function (~line 908)

**Problem:** Between the stencil marking pass and the virtual camera scene pass, the code calls:
```javascript
gl.clear(gl.DEPTH_BUFFER_BIT);
```

This clears the depth buffer for the **entire screen**, not just the stencil region. Since the first portal's scene has already been drawn, clearing the depth buffer globally destroys the depth information for portal 1 when rendering portal 2. It also means the main scene pass will have no depth buffer data from the portal regions, causing depth-fighting artifacts.

**Fix:** Instead of a global depth clear, use a depth-range trick or simply draw the portal quad with depth writes first to "reset" depth only in the portal region. The simplest correct approach:

```javascript
// After stencil mark, before drawing virtual scene:
// Write a far-plane depth value into just the portal region using the stencil mask
gl.stencilFunc(gl.EQUAL, stencilVal, 0xFF);
gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP);
gl.colorMask(false, false, false, false);
gl.depthMask(true);
gl.depthFunc(gl.ALWAYS);  // Force write regardless of current depth
drawPortalQuad(portalA, proj, mainView);  // Writes portal-surface depth
gl.depthFunc(gl.LESS);    // Restore normal depth test
gl.colorMask(true, true, true, true);
```

Or, if simpler: accept the global depth clear but render portals in a way that the second portal's stencil pass re-marks correctly. The key insight is that `gl.clear(gl.DEPTH_BUFFER_BIT)` ignores the stencil mask — it always clears everything. The correct approach is to **not call gl.clear** at all and instead rely on the portal quad's depth write to act as a "reset" for the region.

---

### Bug #3: Portal Oval Drawn with `gl.disable(gl.DEPTH_TEST)` Covers Everything (MODERATE)

**Location:** `drawPortalOval()` function (~line 861)

**Problem:** The portal oval frame (the colored ring you see) is drawn with depth testing disabled:
```javascript
gl.disable(gl.DEPTH_TEST);
```

This means the oval always renders on top of everything, including the portal view that was rendered via stencil. Since the oval is a **solid filled ellipse** (not just a rim), it covers the entire portal area with the solid color, hiding whatever was rendered through the stencil.

**Fix — two options:**

**Option A (simple):** Don't draw the oval at all over the portal interior. Instead, draw it **before** the portal stencil pass as a background, or draw only a ring (not a filled oval). A ring can be generated as two concentric ellipses:

```javascript
const createPortalRing = () => {
  const segments = 32;
  const outerW = 1.3, outerH = 2.1;  // Slightly larger than portal
  const innerW = 1.2, innerH = 2.0;  // Match portal size
  const positions = [];
  const normals = [];
  const indices = [];

  for (let i = 0; i <= segments; i++) {
    const angle = (i / segments) * Math.PI * 2;
    const cos = Math.cos(angle), sin = Math.sin(angle);
    // Outer vertex
    positions.push(cos * outerW/2, sin * outerH/2, 0);
    normals.push(0, 0, 1);
    // Inner vertex
    positions.push(cos * innerW/2, sin * innerH/2, 0);
    normals.push(0, 0, 1);
  }

  for (let i = 0; i < segments; i++) {
    const o = i * 2;
    indices.push(o, o+1, o+2, o+1, o+3, o+2);
  }

  return { positions: new Float32Array(positions), normals: new Float32Array(normals), indices: new Uint16Array(indices) };
};
```

**Option B (quick):** Keep the filled oval but draw it with depth test **enabled** and draw it **before** the stencil scene pass. The virtual camera scene (drawn into the stencil region) will then correctly write depth values in front of the oval, making the scene visible through the portal while the oval edge remains visible as a frame.

**Recommendation:** Option A (ring mesh) is cleaner and more Portal-like. The original game has a glowing rim, not a filled oval.

---

### Bug #4: Portal Model Matrix Columns May Not Be Orthonormal (MINOR)

**Location:** `drawPortalQuad()` (~line 834), `drawPortalOval()` (~line 864)

**Problem:** The portal model matrix is built directly from `portal.right`, `portal.up`, and `portal.normal`:
```javascript
const model = new Float32Array([
  portal.right[0], portal.right[1], portal.right[2], 0,
  portal.up[0], portal.up[1], portal.up[2], 0,
  portal.normal[0], portal.normal[1], portal.normal[2], 0,
  portal.pos[0], portal.pos[1], portal.pos[2], 1
]);
```

This is correct **if** the three vectors are orthonormal. Check that portal placement always normalizes these vectors and that `right = normalize(cross(up, normal))` and `up` is recomputed for orthogonality after `right` is derived. Currently the wall-portal case does:
```javascript
up = [0, 1, 0];
right = V.norm(V.cross(up, face.normal));
```
But `up` is not recomputed from `cross(normal, right)`. For axis-aligned walls this works out, but it's fragile. Add:
```javascript
up = V.cross(face.normal, right); // Ensure orthogonal
```

---

### Bug #5: Stencil Buffer Not Cleared Between Portals (MINOR)

**Location:** `renderPortal()` (~line 920)

**Problem:** At the end of `renderPortal()`, the stencil func is reset to ALWAYS:
```javascript
gl.stencilFunc(gl.ALWAYS, 0, 0xFF);
```
But the stencil buffer values (1 for blue, 2 for orange) are **never cleared**. When the second portal is rendered, the first portal's stencil values are still present. This doesn't break things if the portals don't overlap on screen, but if they do, stencil value 1 from portal A might conflict with stencil value 2 for portal B.

**Fix:** The current approach of using different stencil values (1 and 2) is fine for non-overlapping portals. But for correctness, after rendering each portal's scene, clear just the stencil region:
```javascript
// At end of renderPortal, after depth fill:
gl.stencilFunc(gl.EQUAL, stencilVal, 0xFF);
gl.stencilOp(gl.KEEP, gl.KEEP, gl.ZERO);  // Zero out the stencil we wrote
gl.colorMask(false, false, false, false);
gl.depthMask(false);
drawPortalQuad(portalA, proj, mainView);
gl.colorMask(true, true, true, true);
gl.depthMask(true);
```

Actually, a simpler and cleaner approach: **keep the stencil values** (don't zero them) and use them in the main scene pass to mask out both portals. This is what Bug #1's fix does with `gl.stencilFunc(gl.EQUAL, 0, 0xFF)`.

---

## Corrected Render Pipeline

Here is the correct render order with all fixes applied:

```javascript
const render = () => {
  gl.clearColor(0.15, 0.15, 0.17, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT | gl.STENCIL_BUFFER_BIT);

  const aspect = canvas.width / canvas.height;
  const proj = M.persp(cam.fov, aspect, 0.1, 100);
  const view = getViewMatrix();

  const bothActive = portals.blue?.active && portals.orange?.active;

  if (bothActive) {
    // --- Portal A (blue → see orange side) ---
    // Step 1a: Mark blue portal shape in stencil
    gl.enable(gl.STENCIL_TEST);
    gl.stencilFunc(gl.ALWAYS, 1, 0xFF);
    gl.stencilOp(gl.KEEP, gl.KEEP, gl.REPLACE);
    gl.colorMask(false, false, false, false);
    gl.depthMask(false);
    drawPortalQuad(portals.blue, proj, view);

    // Step 2a: Draw scene from orange side, masked to blue portal stencil
    gl.stencilFunc(gl.EQUAL, 1, 0xFF);
    gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP);
    gl.colorMask(true, true, true, true);
    gl.depthMask(true);
    // Reset depth ONLY in portal region by drawing portal quad with ALWAYS depth
    gl.depthFunc(gl.ALWAYS);
    drawPortalQuad(portals.blue, proj, view);
    gl.depthFunc(gl.LESS);
    // Now draw the virtual scene (only pixels passing stencil == 1)
    const virtualViewA = computePortalView(portals.blue, portals.orange, cam.pos, getCamDir());
    drawScene(proj, virtualViewA);

    // --- Portal B (orange → see blue side) ---
    // Step 1b: Mark orange portal shape in stencil
    gl.stencilFunc(gl.ALWAYS, 2, 0xFF);
    gl.stencilOp(gl.KEEP, gl.KEEP, gl.REPLACE);
    gl.colorMask(false, false, false, false);
    gl.depthMask(false);
    drawPortalQuad(portals.orange, proj, view);

    // Step 2b: Draw scene from blue side, masked to orange portal stencil
    gl.stencilFunc(gl.EQUAL, 2, 0xFF);
    gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP);
    gl.colorMask(true, true, true, true);
    gl.depthMask(true);
    gl.depthFunc(gl.ALWAYS);
    drawPortalQuad(portals.orange, proj, view);
    gl.depthFunc(gl.LESS);
    const virtualViewB = computePortalView(portals.orange, portals.blue, cam.pos, getCamDir());
    drawScene(proj, virtualViewB);
  }

  // --- Main scene: draw everywhere EXCEPT portal regions ---
  if (bothActive) {
    gl.enable(gl.STENCIL_TEST);
    gl.stencilFunc(gl.EQUAL, 0, 0xFF);  // Only where no portal was drawn
    gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP);
  } else {
    gl.disable(gl.STENCIL_TEST);
  }
  drawScene(proj, view);

  // --- Portal frames (glowing rings, with depth test ON) ---
  gl.disable(gl.STENCIL_TEST);
  if (portals.blue?.active) drawPortalFrame(portals.blue, proj, view, [0.3, 0.6, 1.0]);
  if (portals.orange?.active) drawPortalFrame(portals.orange, proj, view, [1.0, 0.5, 0.1]);
};
```

---

## Summary of Required Changes

| # | Severity | Issue | Fix |
|---|----------|-------|-----|
| 1 | 🔴 Critical | Main scene overwrites portal views | Use stencil `EQUAL 0` mask when drawing main scene |
| 2 | 🔴 Critical | `gl.clear(DEPTH_BUFFER_BIT)` clears entire screen | Replace with portal-quad depth write using `gl.depthFunc(gl.ALWAYS)` |
| 3 | 🟡 Moderate | Filled oval covers portal interior | Replace filled oval with a ring mesh, draw with depth test enabled |
| 4 | 🟢 Minor | Portal `up` vector not reorthogonalized | Add `up = V.cross(normal, right)` after computing `right` |
| 5 | 🟢 Minor | Stencil values persist between portals | Handled by using different values (1, 2) and masking main scene with 0 |

## Instructions

1. Fix all bugs above.
2. Replace `drawPortalOval` with `drawPortalFrame` that uses a ring mesh (or at minimum, enable depth test when drawing it).
3. Restructure `render()` to follow the corrected pipeline above.
4. Remove the separate `renderPortal()` function and inline the logic into `render()` for clarity (or refactor `renderPortal` to not call `gl.clear`).
5. Keep code compact — stay within the 15KB gzip budget.
6. After fixing: when both portals are placed, looking at one portal should show the room/scene as viewed from the other portal's position and orientation.
