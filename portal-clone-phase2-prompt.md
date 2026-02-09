# AI Prompt: 15KB Portal Clone — Phase 2: Portal Rendering & Placement

## Context

You are building a 15KB (post-gzip) browser-based Portal clone in a single `index.html` file using raw WebGL. Phase 1 established the framework: a first-person camera, WASD movement, pointer lock, procedural room geometry (inverted boxes with per-face generation via flag bitmasks), basic AABB physics, directional lighting, and a working game loop. The player can walk between connected rooms through doorways.

Phase 2 adds the **core Portal mechanic**: the ability to shoot two portals onto walls, see through them, and eventually walk through them.

---

## Phase 2 Deliverables

### 1. Raycasting System

Implement a ray-wall intersection system for portal placement.

**Ray origin:** Camera position (`cam.pos`).
**Ray direction:** Camera forward vector derived from `cam.yaw` and `cam.pitch`:
```javascript
const dir = [
  Math.cos(cam.pitch) * -Math.sin(cam.yaw),
  Math.sin(cam.pitch),
  Math.cos(cam.pitch) * -Math.cos(cam.yaw)
];
```

**What to intersect against:**
- Every wall/floor/ceiling quad in the level that is flagged as portal-eligible.
- Each face is an axis-aligned plane with known bounds.
- Use ray-plane intersection: for an axis-aligned plane (e.g., a wall at X=5 spanning Y=[0,4], Z=[-5,5]), solve `t = (planeX - ray.origin.x) / ray.dir.x`, then check if the hit point's Y and Z are within the face bounds.

**Output:** A hit result containing:
```javascript
{
  point: [x, y, z],      // world-space hit point
  normal: [nx, ny, nz],  // face normal (pointing into the room)
  faceIndex: int,         // which face was hit
  roomIndex: int,         // which room it belongs to
  t: float                // ray parameter (distance)
}
```

**Constraints:**
- Only allow portals on walls and floors (not ceilings, for simplicity).
- Max range: ~50 units.
- Pick the nearest hit (smallest positive `t`).

**Integration:**
- Left click fires blue portal → raycast → store hit as `portalBlue`.
- Right click fires orange portal → raycast → store hit as `portalOrange`.
- Each portal stores: `{ pos, normal, roomIndex, up, right }` where `up` and `right` define the portal's local orientation on the surface.

---

### 2. Portal Surface Data

Each placed portal needs a well-defined coordinate frame on the wall:

```javascript
const portal = {
  pos: [x, y, z],          // center point on the wall
  normal: [nx, ny, nz],    // outward normal (into the room)
  up: [ux, uy, uz],        // local up direction on the surface
  right: [rx, ry, rz],     // local right direction on the surface
  color: [r, g, b],        // blue [0.1, 0.4, 1.0] or orange [1.0, 0.5, 0.0]
  active: true
};
```

**Deriving the frame:**
- For wall portals (normal is horizontal): `up = [0, 1, 0]`, `right = cross(up, normal)`.
- For floor portals (normal is [0,1,0]): `up = [0, 0, -1]` (or based on camera yaw), `right = cross(up, normal)`.
- Portal dimensions: width = 1.2, height = 2.0 (enough for the player to walk through).

**Snap portal to surface:** The portal center must be offset slightly from the wall (e.g., 0.01 units along the normal) to avoid z-fighting. Also clamp the portal position so it doesn't extend beyond the face edges.

---

### 3. Portal Rendering (Stencil Buffer Approach)

This is the core technical challenge. Use the **stencil buffer** to render what's visible through each portal.

**High-level algorithm for each active portal pair:**

```
For each portal (A = blue, B = orange):
  1. STENCIL PASS — Mark portal shape in stencil buffer:
     - Disable color and depth writes.
     - Enable stencil test.
     - Set stencil op to REPLACE with value 1.
     - Draw the portal quad (a plane at portal A's position/orientation) using the normal MVP.
     - This writes 1 into the stencil buffer wherever portal A is visible.

  2. SCENE PASS THROUGH PORTAL — Draw the world as seen from portal B:
     - Enable color and depth writes.
     - Set stencil func to EQUAL 1 (only draw where the portal quad was).
     - Clear the depth buffer (only in the stencil region — or just clear it fully, performance isn't critical).
     - Compute the "virtual camera": the camera as if it were teleported from portal A to portal B.
     - Use this virtual camera's view matrix to render the full scene.
     - This fills the portal shape with the view from the other side.

  3. DEPTH FILL — Write portal quad depth:
     - Disable color writes, enable depth writes.
     - Draw the portal quad again with normal MVP.
     - This ensures objects in front of the portal correctly occlude it.

  4. Clean up stencil for next portal.
```

**Virtual camera computation:**

This is the key math. To compute what camera B sees through portal A:

```javascript
function getPortalView(portal, linked, camPos, camDir, camUp) {
  // Transform camera position relative to portal A into portal B's frame
  // 1. Get camera position in portal A's local space
  // 2. Mirror/rotate to portal B's local space
  // 3. Transform back to world space

  // Portal A's basis: right_A, up_A, normal_A
  // Portal B's basis: right_B, up_B, normal_B

  // The transform from A's frame to B's frame involves:
  // - A 180° rotation around the up axis (you look "into" A and "out of" B)
  // - Mapping A's local axes to B's local axes

  const offset = V.sub(camPos, portal.pos);

  // Project offset onto A's local axes
  const localX = V.dot(offset, portal.right);
  const localY = V.dot(offset, portal.up);
  const localZ = V.dot(offset, portal.normal);

  // In B's frame, flip X and Z (180° rotation around up axis)
  const virtualPos = V.add(linked.pos, 
    V.add(V.scale(linked.right, -localX),
    V.add(V.scale(linked.up, localY),
          V.scale(linked.normal, -localZ)))
  );

  // Similarly transform camera direction
  const dirLocalX = V.dot(camDir, portal.right);
  const dirLocalY = V.dot(camDir, portal.up);
  const dirLocalZ = V.dot(camDir, portal.normal);

  const virtualDir = V.add(
    V.scale(linked.right, -dirLocalX),
    V.add(V.scale(linked.up, dirLocalY),
          V.scale(linked.normal, -dirLocalZ))
  );

  const virtualTarget = V.add(virtualPos, virtualDir);
  return M.lookAt(virtualPos, virtualTarget, [0, 1, 0]);
}
```

**Important:** Do NOT implement recursive portals (portal seeing a portal seeing a portal). Cap at **one level** — each portal shows the scene from the other side, but portals visible through portals just render as solid colored ovals.

---

### 4. Portal Visual — Oval Quad with Glowing Rim

**Portal geometry:** Generate an oval (ellipse) mesh for the portal surface. Approximate it with a triangle fan of ~16 segments:

```javascript
function createPortalMesh(width, height, segments) {
  // Center vertex + ring vertices
  const verts = [0, 0, 0]; // center
  const norms = [0, 0, 1]; // all face same way
  const indices = [];

  for (let i = 0; i <= segments; i++) {
    const angle = (i / segments) * Math.PI * 2;
    verts.push(Math.cos(angle) * width/2, Math.sin(angle) * height/2, 0);
    norms.push(0, 0, 1);
    if (i > 0) {
      indices.push(0, i, i + 1);
    }
  }
  // ... return as Float32Arrays
}
```

**Rendering the oval:**
- The portal mesh is defined in local space (on XY plane, facing +Z).
- Build a model matrix that transforms it to the portal's world position and orientation using the portal's `pos`, `right`, `up`, and `normal` as basis vectors.

**Glow shader:** Use the existing `uPortalGlow` uniform. In the fragment shader, compute distance from the fragment to the portal edge and add a glow:

```glsl
// In fragment shader, add:
// vUV is a varying set to the local XY position of the portal mesh vertex
float edgeDist = 1.0 - length(vUV); // 0 at edge, 1 at center
float rim = smoothstep(0.0, 0.15, edgeDist) * (1.0 - smoothstep(0.15, 0.3, edgeDist));
vec3 glowColor = uPortalGlow > 0.5 ? vec3(1.0, 0.5, 0.0) : vec3(0.1, 0.4, 1.0);
color += glowColor * rim * 2.0;
```

Or keep it simpler: render the portal oval as a solid color with a brighter ring around its edge. The portal interior (what you see through it) comes from the stencil pass.

---

### 5. Oblique Near-Plane Clipping (IMPORTANT)

**Problem:** When rendering the scene through a portal, geometry between the virtual camera and the portal plane should be clipped — otherwise you see the back side of the wall the portal is on.

**Solution:** Modify the projection matrix to use an oblique near plane aligned with the portal surface. This is a well-known technique:

```javascript
function clipProjection(proj, clipPlane) {
  // clipPlane is [a, b, c, d] in view space (portal plane equation)
  // Modify the projection matrix so the near plane aligns with clipPlane
  const q = [
    (Math.sign(clipPlane[0]) + proj[8]) / proj[0],
    (Math.sign(clipPlane[1]) + proj[9]) / proj[5],
    -1,
    (1 + proj[10]) / proj[14]
  ];
  const c = V.scale(clipPlane, 2.0 / V.dot(clipPlane, q));
  proj[2] = c[0];
  proj[6] = c[1];
  proj[10] = c[2] + 1.0;
  proj[14] = c[3];
  return proj;
}
```

Apply this when rendering the scene through each portal. The clip plane is the linked portal's surface plane transformed into the virtual camera's view space.

If this is too complex to get right initially, **skip it** and accept minor visual artifacts (seeing through walls near portals). Mark it as a TODO.

---

### 6. Updated Render Pipeline

The new render order becomes:

```javascript
function render() {
  gl.clearColor(0.15, 0.15, 0.17, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT | gl.STENCIL_BUFFER_BIT);

  const proj = M.persp(cam.fov, aspect, 0.1, 100);
  const view = getViewMatrix(); // from cam.pos, cam.pitch, cam.yaw

  // 1. Render portals (stencil + virtual camera scene)
  if (portalBlue.active && portalOrange.active) {
    renderPortalView(portalBlue, portalOrange, proj, view, 1);
    renderPortalView(portalOrange, portalBlue, proj, view, 2);
  }

  // 2. Reset stencil, render main scene normally
  gl.disable(gl.STENCIL_TEST);
  drawScene(proj, view);

  // 3. Draw portal oval outlines / glow (always visible, on top of scene)
  if (portalBlue.active) drawPortalFrame(portalBlue, proj, view);
  if (portalOrange.active) drawPortalFrame(portalOrange, proj, view);
}
```

**Ensure stencil buffer is requested** in the WebGL context:
```javascript
const gl = canvas.getContext('webgl', { stencil: true });
```

---

### 7. Portal Eligible Face Tracking

Extend the level data to track which faces are portal-eligible. When building the level, store a reference for each generated face:

```javascript
// During buildLevel, for each face:
levelFaces.push({
  roomIndex: i,
  faceIndex: f,       // 0-5 for front/back/top/bottom/right/left
  plane: planeEq,     // [a, b, c, d] plane equation
  bounds: [minU, maxU, minV, maxV], // bounds in face-local 2D
  normal: [nx, ny, nz],
  portalEligible: true // false for ceilings, doorway openings
});
```

Raycasting iterates over `levelFaces` to find intersections.

---

## What NOT To Build Yet

- Portal traversal (walking through portals) — Phase 3
- Momentum preservation / fling — Phase 3
- Pickable objects (companion cube) — Phase 4
- Puzzle mechanics — Phase 5
- Sound — Phase 5

---

## Expected Result

After Phase 2, the game should:
1. Let you left-click to place a blue portal and right-click to place an orange portal on eligible wall/floor surfaces.
2. Show an oval portal shape on the wall with a colored glowing rim (blue or orange).
3. When both portals are placed, you can **see through** each portal to the view from the other portal's location. The perspective should be correct — moving your camera should change what you see through the portal, as if it were a real window.
4. Portals should not be placeable on doorway openings or ceilings.
5. Shooting a new portal of the same color replaces the old one.
6. The rest of the game (movement, physics, room rendering) continues to work exactly as before.

---

## Size Budget Reminder

The total file must stay under ~35-40KB unminified (≤15KB gzipped). The stencil-based portal rendering approach was chosen specifically because it's code-light. Reuse the existing shader program — add uniforms as needed but avoid creating additional shader programs if possible. The portal oval mesh and the raycasting system should each be under ~50 lines.
