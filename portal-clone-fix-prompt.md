# AI Prompt: Fix Rendering Issues in Portal Clone Framework

## Context
You are working on a 15KB browser-based Portal clone using raw WebGL. The framework code is in a single `index.html` file. The basic structure (shaders, math, camera, input, physics, game loop) is in place, but the **rendering is broken or visually wrong**. Below is a diagnosis of the specific issues found in the code that need to be fixed.

---

## Bug #1: Matrix Multiplication Has Transposed Indexing (CRITICAL)

**Location:** `M.mul()` function (~line 50)

**Problem:** The matrix multiply function uses row-major indexing (`a[i*4+j]`) but WebGL/OpenGL expects **column-major** matrices. The `Float32Array` layout for a mat4 in WebGL is column-major:

```
Index:  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]  [8]  [9] [10] [11] [12] [13] [14] [15]
        col0             col1             col2             col3
```

But the current multiply treats index `[i*4+j]` as row `i`, column `j` — which is row-major. This means every matrix multiplication in the entire pipeline (MVP, View-Projection, model transforms) produces a transposed result, causing geometry to render in the wrong positions and orientations, or not at all.

**Fix:** Rewrite `M.mul` to use column-major indexing. The correct formula for column-major `out = a * b` is:

```javascript
mul: (a, b) => {
  const out = new Float32Array(16);
  for (let i = 0; i < 4; i++) {
    for (let j = 0; j < 4; j++) {
      out[j*4+i] = a[0*4+i]*b[j*4+0] + a[1*4+i]*b[j*4+1] + a[2*4+i]*b[j*4+2] + a[3*4+i]*b[j*4+3];
    }
  }
  return out;
}
```

**Verify:** After fixing, also audit that `M.persp`, `M.lookAt`, `M.trans`, `M.rotX/Y/Z` are all consistently column-major. Currently they do appear to use column-major layout individually (e.g., `M.trans` puts x,y,z at indices 12,13,14 which is correct column-major). The issue is specifically that `M.mul` misinterprets the layout.

---

## Bug #2: Room Geometry Isn't Properly Positioned (MODERATE)

**Location:** `buildLevel()` function (~line 349)

**Problem:** The level data format uses `[x, y, z, ...]` to specify room center positions, and rooms are built as inverted boxes centered at origin, then translated via a model matrix. However:

1. Rooms define `y` as 0, meaning the room is centered at y=0 with the floor at y=-h/2. But the player's ground collision is hardcoded to `y = player.h/2` (0.9), which assumes the floor is at y=0. So either the room needs to be shifted up by h/2 so its floor is at y=0, OR the ground collision needs to match the room's actual floor.

2. The wall collision is a crude bounding box (`±20`), not per-room AABB collision. The player can walk through room walls.

**Fix:**
- Option A (preferred): In `buildLevel`, offset the model matrix Y by `+h/2` so the room floor sits at the specified Y coordinate. Change the translation to `M.trans(x, y + h/2, z)`.
- Option B: Change the level data so `y` values account for this offset.
- For wall collision: implement per-room AABB collision. Each room defines a walkable interior; the player's AABB should be tested against each room's walls.

---

## Bug #3: The `flags` Bitmask for Open Walls Is Ignored (MODERATE)

**Location:** `buildLevel()` (~line 349)

**Problem:** The level data includes a `flags` bitmask intended to control which walls are present (so rooms can have openings/doorways connecting them). The code completely ignores this — it always generates a full inverted box for every room. Rooms that should be connected have overlapping solid walls.

**Fix:** Instead of generating one inverted box per room, generate each face (front, back, top, bottom, right, left) separately based on the flag bits. For example:
- Bit 0 (0b000001) = front face (+Z)
- Bit 1 (0b000010) = back face (-Z)
- Bit 2 (0b000100) = top face (+Y)
- Bit 3 (0b001000) = bottom face (-Y)
- Bit 4 (0b010000) = right face (+X)
- Bit 5 (0b100000) = left face (-X)

Generate and buffer only the faces where the corresponding bit is 1. This means changing `createBox` to return individual face quads, or writing a new `buildRoom(w, h, d, flags)` function.

---

## Bug #4: Connected Rooms Don't Actually Connect (DESIGN)

**Location:** Level data (~line 340)

**Problem:** Room positions and sizes don't create flush connections:
- Room 0: center (0,0,0), size 10×4×10 → extends X from -5 to +5
- Room 1: center (10,0,0), size 8×4×12 → extends X from +6 to +14

There's a **1-unit gap** between room 0's right wall (X=+5) and room 1's left wall (X=+6). The rooms should share an edge for doorways to work.

**Fix:** Adjust room positions so adjacent rooms share a wall plane:
```javascript
const LEVEL_1 = [
  [0, 0, 0, 10, 4, 10, [0.9, 0.9, 0.9], 0b101111],    // right wall open
  [9, 0, 0,  8, 4, 12, [0.85, 0.85, 0.88], 0b011111],  // left wall open
  [-8, 0, 0, 6, 4, 8, [0.88, 0.88, 0.9], 0b101111]     // right wall open (toward room 0)
];
```
Compute positions so `room0.x + room0.w/2 == room1.x - room1.w/2`.

---

## Bug #5: Lighting Is Flat/Wrong on Inverted Geometry (MINOR)

**Problem:** For an inverted box (room interior), normals point inward. The directional light `vec3(0.5, 1.0, 0.3)` shines "downward into" the room, but the floor normal is (0, +1, 0) (pointing up into the room after inversion) while the ceiling normal is (0, -1, 0). This means the floor is brightly lit and the ceiling is dark, which is correct. However, all walls facing away from the light direction appear completely dark (only ambient 0.3), making the scene look muddy.

**Fix:** Consider one of:
- Add a second fill light from a different direction.
- Increase ambient from 0.3 to 0.4-0.45.
- Add a simple hemispherical ambient: `ambient = 0.3 + 0.15 * (normal.y * 0.5 + 0.5)` to softly brighten surfaces facing up.
- Use abs(dot) for a non-physical but more visually readable wrap lighting.

---

## Summary of Required Changes

| Priority | Issue | Fix |
|----------|-------|-----|
| 🔴 Critical | `M.mul` uses row-major indexing | Rewrite with column-major indexing |
| 🟡 Moderate | Room floor at wrong Y | Offset room model matrix by +h/2 |
| 🟡 Moderate | Wall flags ignored | Generate faces individually per flag bit |
| 🟡 Moderate | Room positions don't connect | Recalculate room X/Z so walls are flush |
| 🟢 Minor | Lighting too dark on walls | Increase ambient or add fill light |

## Instructions

1. Fix all bugs listed above in priority order.
2. After fixing, the scene should show a multi-room environment where you can walk between connected rooms through doorways.
3. Keep the code compact — this needs to stay well under 15KB gzipped.
4. Do not add any new features; only fix rendering and collision issues.
5. Test by verifying: you can see room interiors with visible walls/floor/ceiling, walk between rooms, and the lighting looks reasonable on all surfaces.
