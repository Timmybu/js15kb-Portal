# AI Prompt: 15KB Portal Clone — Framework Generation

## Project Overview

Build the foundational framework for a first-person 3D Portal clone that runs in-browser, with a hard constraint of **15KB total** (post-gzip). The game uses **raw WebGL**, **procedural generation**, and **mathematical models** — no external libraries, no asset files, no textures. Everything must be a single HTML file.

---

## Constraints & Ground Rules

- **Single HTML file** containing all HTML, CSS, JavaScript, and GLSL shaders inline.
- **No external dependencies** — no Three.js, no Babylon, no libraries of any kind.
- **No image/audio/model assets** — all visuals are procedural via shaders or generated geometry; all sound (if any) is WebAudio oscillator-based.
- **Target budget: ~30-40KB unminified source** (which should gzip to ≤15KB).
- **WebGL 1.0** for maximum compatibility (WebGL2 optional features noted where useful).
- Use `requestAnimationFrame` game loop.
- Pointer Lock API for mouse-look FPS controls.

---

## Phase 1: Core Framework (BUILD THIS NOW)

Generate the following foundational systems as clean, compact, well-commented code. Prioritize correctness and compactness. Use short but readable variable names (e.g., `gl`, `cam`, `mv`, `proj`).

### 1. Renderer Initialization
- Get a fullscreen WebGL canvas.
- Compile and link a **single versatile shader program** that supports:
  - Uniform-driven flat color per face (vec3 `uColor`).
  - Model-View-Projection matrix transforms (`uMVP` as mat4).
  - Basic directional lighting via vertex normals (single hardcoded light direction).
  - A `uPortalGlow` float uniform that adds an emissive orange or blue rim effect when > 0.
- Use a **attribute layout**: `aPosition` (vec3), `aNormal` (vec3).
- Enable depth testing, backface culling.

### 2. Math Utilities (No Libraries)
Write compact pure functions for:
- `mat4` operations: identity, multiply, perspective projection, lookAt, translate, rotateX/Y/Z, inverse, transpose.
- `vec3` operations: add, subtract, scale, dot, cross, normalize, length.
- Quaternion operations (if needed for portal orientation): multiply, from axis-angle, to mat4.
- All functions should operate on `Float32Array` for direct WebGL upload.

### 3. Camera & FPS Controls
- First-person camera with pitch (clamped ±89°) and yaw, stored as angles.
- Pointer Lock on canvas click; mouse movement drives look.
- WASD movement relative to camera yaw (ignore pitch for movement).
- Space for jump, Shift for crouch (optional).
- Derive view matrix from camera position + angles each frame.
- Mouse button 1 = fire blue portal, Mouse button 2 = fire orange portal (stub the raycast for now, just log the event).

### 4. Procedural Geometry Generator
Create functions that return `{ vertices: Float32Array, normals: Float32Array, indexCount: int }` for:
- **Box(w, h, d)** — axis-aligned box centered at origin with outward normals. This is the fundamental building block.
- **InvertedBox(w, h, d)** — same but with inward-facing normals (for rooms you stand inside).
- **Plane(w, h, nx, ny, nz)** — a flat quad with a given normal, for floors/walls/portal surfaces.
- All geometry uses indexed drawing (`drawElements`) with `Uint16Array` index buffers.

### 5. Game Loop Structure
```
function frame(t) {
    dt = (t - lastT) / 1000; lastT = t;
    handleInput(dt);       // movement, look
    updatePhysics(dt);     // gravity, collision
    updatePortals();       // portal logic stub
    render();              // draw scene
    requestAnimationFrame(frame);
}
```
- `handleInput`: reads key state map, applies movement.
- `updatePhysics`: applies gravity, does AABB collision against level geometry.
- `updatePortals`: placeholder — will later handle teleportation and view transforms.
- `render`: clears, sets projection + view matrices, iterates over level geometry and draws each piece with its color.

### 6. Level Data Structure
Define a level as a compact array of room descriptors:
```javascript
// Each room: [x, y, z, w, h, d, color, flags]
// flags: bitmask for which walls exist, door openings, portal-eligible surfaces
const LEVEL_1 = [
  [0, 0, 0,  10, 4, 10,  [0.9, 0.9, 0.9], 0b111111],  // starting room
  [10, 0, 0,  8, 4, 12,  [0.85, 0.85, 0.88], 0b111011], // connected room, one wall open
];
```
- Write a function `buildLevel(data)` that generates all room geometry from this compact format.
- Rooms are axis-aligned inverted boxes. Open walls (0 in bitmask) are omitted to create doorways.
- Store the generated geometry in VBOs ready to draw.

### 7. Simple AABB Physics
- Player is an AABB (0.6 wide, 1.8 tall, 0.6 deep).
- Gravity: 9.8 m/s² downward.
- Ground collision: clamp Y so player doesn't fall through floors.
- Wall collision: slide along walls using simple axis-aligned resolution.
- No need for swept collision — discrete step with small dt is fine.

### 8. Rendering Pass
For this initial framework, render the scene with a **single pass**:
1. Clear color (dark grey) and depth buffer.
2. Set projection matrix (perspective, ~70° FOV, near=0.1, far=100).
3. Compute view matrix from camera.
4. For each room/object: compute model matrix → upload MVP → set uColor → draw.
5. Draw a simple **crosshair** using a 2D overlay (a small + in the center via CSS or a tiny canvas2D overlay).

---

## Code Style Guidelines

- **Compact but readable**: Use 2-space indent. Short names for hot-path variables, descriptive names for setup functions.
- **No classes** — use plain objects and functions. Smaller output.
- **Inline shaders** as template literals.
- **Group related code** with clear `// === SECTION NAME ===` comments.
- **No semicolons** in optional positions (saves bytes, but this is style preference — either way is fine, just be consistent).

---

## What NOT To Build Yet

Do NOT implement these yet (they come in later phases):
- Portal rendering (stencil buffer / render-to-texture)
- Portal traversal (teleportation)
- Raycasting for portal placement
- Momentum/fling mechanics
- Procedural sound
- UI / text overlays beyond crosshair
- Level transitions

---

## Expected Output

A single `index.html` file that, when opened in a browser:
1. Shows a fullscreen 3D first-person view of a simple multi-room level.
2. Lets you look around with the mouse (after clicking to lock pointer).
3. Lets you walk with WASD, jump with Space.
4. Has basic gravity and wall collision so you can't walk through walls or fall through floors.
5. Rooms are flat-shaded with directional lighting, in Portal-esque muted grey/white tones.
6. A crosshair is visible at screen center.
7. Left/right click logs "fire blue/orange portal" to console (no actual portal yet).

This is the skeleton that all future systems (portals, puzzles, objects) will plug into.
