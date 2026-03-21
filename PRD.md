# PRD: Option 2 — InstancedMesh Leaf Quads with Alpha Cutout

## Overview

Replace the current **`THREE.Points`**-based foliage system with a single **`THREE.InstancedMesh`** sharing one `PlaneGeometry` and one `MeshLambertMaterial` (with `alphaTest`). Each leaf becomes an **instance** with its own **4×4 transform matrix** and per-instance **color**. This gives the same visual result as alpha-mapped sprite quads but handles **hundreds to thousands of leaves** with a **single draw call**.

## File

All changes are in **`index.html`** (single-file app, no modules).

---

## Current implementation

### Leaf generation pipeline

- **`buildLeaves(root, params, rng)`** (≈602–658) walks the tree via inner `visit(node)`. A node gets leaves when it is **terminal** (`living.length === 0`) or at **`node.depth >= params.maxDepth - 1`**.
- For each eligible node, **`getLeafPlacements(node, params, rng)`** (≈524–599) returns an array of `{ t, angle, radius }` objects — one per leaf. Placement count is driven by **`params.leafDen`** and arrangement by **`params.leafArr`** (`'alternate'`, `'opposite'`, `'whorled'`, `'fascicled'`, `'radial'`).
- World position is computed by **`bentPointAt(node, t, angle, radius)`** (≈511–519), which follows the node's **spline** (from `buildNodeSpline`) and offsets radially.

### Leaf attributes collected today

| Array | Purpose | Used by |
|-------|---------|---------|
| `positions` (xyz) | World placement | `BufferGeometry` `position` attribute |
| `colors` (rgb) | Per-leaf tint; oak uses autumn palette | `color` attribute → `vertexColors` |
| `growEnds` | Growth reveal threshold per leaf | `aGrowEnd` attribute → `leafVert` |
| `sizes` | `rng.range(params.leafSz*8, params.leafSz*18)` — screen-space point size | `aSize` attribute → `gl_PointSize` formula |
| `rots` | `0 … 2π` — rotates `gl_PointCoord` in fragment | `aRot` attribute → `leafFrag` UV rotation |
| `owners` | Node reference per leaf | `geo.userData.owners` (used only by prune-mode leaf selection) |

### Current rendering objects

- **`leafPoints`** — `new THREE.Points(geo, leafMat)`, added to **`treeGroup`**.
- **`leafMat`** — from **`makeLeafMat()`** (≈740–746): custom `ShaderMaterial` with `leafVert`/`leafFrag`, uniforms `{ uReveal, uMap }`, flags `vertexColors: true`, `transparent: true`, `depthWrite: false`.
- **`leafTex`** — from **`makeLeafTex(shape)`** (≈163–350): procedural **64×64** `CanvasTexture`, white shape on transparent background, with vein detail via `destination-out` compositing.

### Growth animation

- **`updateGrowth(dt)`** (≈823–845): drives `leafMat.uniforms.uReveal.value` from `state.growthTime`.
- **`leafVert`**: `smoothstep(aGrowEnd - 0.08, aGrowEnd + 0.05, uReveal)` → scales `gl_PointSize` to zero and hides via `gl_Position = vec4(9999.)` when `t < 0.01`.

### Prune-mode leaf interaction

- **`dropSelectedLeaf()`** (≈961–991): zeroes `aSize` at index, spawns falling `Sprite` with `leafTex`.
- **Picking**: `raycaster.intersectObject(leafPoints)` with `raycaster.params.Points.threshold = 0.15`.
- **`clearLeafSel()`** (≈942–950): restores original color at `_selLeaf` index.

### Rebuild paths

- **`rebuildLeaves()`** (≈929–937): disposes old `leafPoints.geometry`, rebuilds, re-adds to `treeGroup`, sets `uReveal = 1.0`.
- **`rebuild()`** (≈1191–1221): full tree regen; creates new `leafTex`, `leafMat`, calls `buildBranches` then `buildLeaves`.

---

## Target behavior

### Geometry

- **One shared `PlaneGeometry`**: `new THREE.PlaneGeometry(1, 1)` (unit quad, scaled per instance). Two triangles per leaf.
- **One `InstancedMesh`**: `new THREE.InstancedMesh(planeGeo, leafMaterial, maxCount)` where `maxCount` is the total leaf count from `buildLeaves`.
- Each instance gets a **4×4 matrix** encoding **position** (from `bentPointAt`), **rotation** (random yaw + pitch so quads aren't coplanar), and **scale** (from `leafSz`).

### Material

- **`MeshLambertMaterial`** (or `MeshStandardMaterial` if roughness control is wanted):
  - `map`: species texture (from `makeLeafTex` or loaded PNG).
  - `alphaTest: 0.5` — binary cutout, no transparency sorting.
  - `transparent: false` — because `alphaTest` handles edges.
  - `side: THREE.DoubleSide` — leaves visible from both sides when orbiting.
  - `vertexColors: false` — per-instance color via `InstancedMesh.setColorAt`.

### Per-instance data

| Data | How stored | Derived from |
|------|-----------|--------------|
| Position | Instance matrix (translation column) | `bentPointAt(node, pl.t, pl.angle, pl.radius)` |
| Orientation | Instance matrix (rotation component) | Two random angles: **yaw** (full circle) + **pitch** (±25–40° from branch-local up) into a `Quaternion` for each instance matrix |
| Scale | Instance matrix (scale component) | Map `rng.range(params.leafSz*8, params.leafSz*18)` from old screen-space to **world units** — target range **~0.04–0.15** world units (comparable to tip `node.rad` values) |
| Color | `instancedMesh.setColorAt(i, color)` + `instanceColor` attribute | Same per-leaf color logic from `buildLeaves` (autumn palette for oak, `leafCol` tint for others) |
| Growth threshold | Custom `InstancedBufferAttribute` (`float`, one per instance) | Same `growEnds` calculation: `(node.growStart + 0.8) / (maxGrow * 1.1) + rng.range(0, 0.08)` |
| Owner node | Parallel JS array (not GPU) | Same `owners` array for prune-mode lookup |

### Scale mapping

Scene scale reference:
- Trunk length: **1.6** (Bonsai) to **4.0** (Pine).
- Terminal branch radius: roughly **0.02–0.08** depending on depth and taper.
- Old `sizes` range: `leafSz * 8` to `leafSz * 18` where `leafSz` defaults to **0.1** → screen-space **0.8–1.8 px multiplied by `250 / -mv.z`**.

Target quad dimensions in world units: **`leafSz * 0.6`** to **`leafSz * 1.4`** (i.e. **~0.06–0.14** at default `leafSz = 0.1`). Exact mapping should be tuned visually to match canopy density of the old `Points` rendering at the default camera distance (~14 units from origin).

### Growth animation

Two viable approaches:

**A. Custom shader chunk injection (`onBeforeCompile`) — recommended:**
Add a `float aGrowEnd` `InstancedBufferAttribute`. In `material.onBeforeCompile`, inject a uniform `uReveal` and vertex shader logic equivalent to the old `leafVert`: scale to zero and discard when `uReveal < aGrowEnd`. This preserves the existing smooth reveal.

**B. JS-side matrix update per frame:**
Each frame, iterate instances and set scale to zero when `growthTime < growEnd[i]`. Requires `instanceMatrix.needsUpdate = true` every frame during growth. Simple but **O(n) JS per frame** and burns CPU during the growth phase.

### Texture

Keep **`makeLeafTex(shape)`** as the texture source. The existing canvas art produces white-on-transparent shapes with vein detail — this works with `alphaTest` as long as the alpha channel has **clean edges** (current bezier paths produce smooth edges; `alphaTest: 0.5` will produce slightly jagged silhouettes at 64×64). Consider:
- Bumping canvas resolution to **128×128** for cleaner `alphaTest` edges.
- Or adding a slight SDF-style feather in the `onBeforeCompile` fragment if softness is needed (out of scope for v1).

---

## What must change

### Remove

| Symbol | Location | Why |
|--------|----------|-----|
| `leafVert` | ≈111–128 | Replaced by `MeshLambertMaterial` + `onBeforeCompile` |
| `leafFrag` | ≈130–144 | Same |
| `makeLeafMat()` | ≈740–746 | Replaced by new material factory |
| `leafPoints` usage | ≈714, 929–935, 1216–1219 | Replaced by `InstancedMesh` |
| `gl_PointSize` / `gl_PointCoord` logic | shaders | N/A for mesh rendering |

### Rewrite

| Symbol | Change |
|--------|--------|
| **`buildLeaves(root, params, rng)`** | Instead of flat arrays → `BufferGeometry`, compute **instance matrices** (via `THREE.Matrix4` composed from position/rotation/scale) and **colors** (`THREE.Color`). Return an object `{ matrices: Float32Array, colors: Float32Array, growEnds: Float32Array, owners: Array, count: number }`. |
| **`rebuildLeaves()`** | Dispose old `InstancedMesh`. Create `new THREE.InstancedMesh(sharedPlaneGeo, leafMaterial, count)`. Call `setMatrixAt` / `setColorAt` in a loop. Attach `growEnds` as `InstancedBufferAttribute`. Add to `treeGroup`. |
| **`rebuild()`** | Replace `leafMat = makeLeafMat()` with new material creation. Replace `leafPoints = new THREE.Points(...)` with instanced mesh setup. |
| **`updateGrowth(dt)`** leaf section (≈841–844) | Drive `uReveal` uniform on the new material (if using `onBeforeCompile` approach). |
| **`dropSelectedLeaf()`** | Set instance scale to zero via `setMatrixAt(idx, zeroMatrix)` + `instanceMatrix.needsUpdate = true`. Spawn falling sprite as before. |
| **Leaf picking** | Replace `raycaster.intersectObject(leafPoints)` with `raycaster.intersectObject(instancedMesh)`. `InstancedMesh` raycasting returns `instanceId` on the intersection — use that instead of `hit.index`. |
| **`clearLeafSel()`** | Use `setColorAt(idx, originalColor)` + `instanceColor.needsUpdate = true` instead of buffer attribute manipulation. |

### Add

| New code | Purpose |
|----------|---------|
| Shared `PlaneGeometry` (module-level `const`) | One quad reused by all species |
| `InstancedBufferAttribute` for `aGrowEnd` | Per-instance growth threshold, read by injected vertex shader chunk |
| `material.onBeforeCompile` callback | Inject `uReveal` uniform + vertex scale/discard logic into `MeshLambertMaterial` |
| Orientation helper (e.g. `leafQuaternion(rng)`) | Compose random **yaw** (full circle) + **pitch** (±25–40° from branch-local up) into a `Quaternion` for each instance matrix |

---

## What not to touch

- **Branching / trunk / bark**: `generateTree`, `makeNode`, `makeSplineBranchGeo`, `buildBranches`, `rebuildBranchMesh`, `branchVert`/`branchFrag`, `makeBranchMat`, `cascadePositions`, `updateGrowth` branch scaling section (≈825–839).
- **Spline / placement utilities**: `buildNodeSpline`, `bentPoint`, `bentPointAt`, `getLeafPlacements`, `GOLDEN_ANGLE`, all arrangement logic — these remain the source of truth for leaf positions.
- **Lighting / scene**: `AmbientLight`, `DirectionalLight`, `scene.background`, `ground` mesh, `camera`, `OrbitControls`, `EffectComposer` / `postShader`.
- **Branch pruning**: `pruneSelected`, `getSubtree`, `fallingDebris` animation loop, `scheduleRegrowth` — only touch the shared `rebuildLeaves()` call at the end of these functions, which the new code must still honor.
- **GUI structure / species**: `PRESETS` object, `fStruct` folder, `fGrow` folder. In `fStyle`, the `leafCol`, `leafDen`, `leafSz`, `leafArr` controls and their `.onChange(() => rebuildLeaves())` wiring should continue to work — only the internal implementation of `rebuildLeaves` changes.
- **`treeGroup`** parenting and **`state.showLeaves`** visibility toggle (≈1268–1270).

---

## Complications and risks

1. **`InstancedMesh` raycasting performance.** Three.js tests every instance bounding box on raycast. With thousands of leaves this can be slow on click. Mitigation: set `instancedMesh.frustumCulled = false` and use a coarse pre-filter (e.g. check distance to `treeGroup` center before raycasting), or accept the cost since it only fires on click.

2. **`onBeforeCompile` fragility.** Shader injection depends on Three.js internal shader chunks (`#include <begin_vertex>`, `#include <project_vertex>`). Pin the Three.js version (currently **0.170.0** via CDN import map) and test after any upgrade. Alternative: use a full custom `ShaderMaterial` that reimplements Lambert lighting — more stable but more code.

3. **Instance count changes on rebuild.** `InstancedMesh.count` can be set lower than `maxCount` to hide instances, but **cannot grow** past the allocated `maxCount`. `rebuildLeaves` must **dispose and recreate** the `InstancedMesh` when leaf count changes (density slider, prune, species switch). This matches the current `leafPoints` dispose/recreate pattern.

4. **Per-instance color + growth attribute.** `InstancedMesh` natively supports `instanceColor` (via `setColorAt`). The custom `aGrowEnd` `InstancedBufferAttribute` must be manually attached to `instancedMesh.geometry` — this works but is less documented. Verify attribute divisor is set correctly (`meshPerAttribute: 1`).

5. **Orientation vs. current UV rotation.** The old system rotates **texture UVs** in the fragment shader (`vRot`); the new system rotates **geometry in world space**. This means leaf silhouettes will look different at glancing angles — leaves will have visible **thin edges** when viewed nearly edge-on. This is desirable for realism but changes the visual character. Tuning the pitch range controls how much edge is visible.

6. **Falling leaf on drop.** `dropSelectedLeaf` currently spawns a `Sprite` from `leafTex`. With `InstancedMesh`, the "drop" should either: (a) extract the instance matrix, spawn a **standalone `Mesh`** (same `PlaneGeometry` + material clone) at that world position, animate it falling, then dispose; or (b) animate the instance matrix directly (but this complicates the debris system). Option (a) is simpler and consistent with the existing `fallingDebris` pattern.

7. **Texture per species.** All instances of one `InstancedMesh` share **one material** and therefore **one `map`**. Species that need different leaf shapes (oak vs. needle vs. maple) require **separate `InstancedMesh` instances per texture**, or a **texture atlas** with per-instance UV offset. For v1, since `rebuild()` recreates everything on species change and only one species is active at a time, **one `InstancedMesh` per rebuild** is sufficient.

---

## Success criteria

- Dense canopy (Oak at `leafDen: 12`, `maxDepth: 6`) renders in **one draw call** with no transparency sorting artifacts.
- Leaves respond to directional light — visible shading difference between sun-facing and shadow-facing quads.
- Growth animation reveals leaves progressively, matching the timing feel of the old `uReveal` system.
- Species switch (`PRESETS`) changes leaf texture shape and color palette.
- Prune-mode leaf selection (click) and drop (Delete/Backspace) still work.
- `rebuildLeaves()` disposes cleanly with no WebGL resource leaks.
- FPS stays above **50** on a mid-range laptop at default settings.

---

## Opening prompt for Claude Code

In `index.html`, replace the `THREE.Points`-based foliage system (`leafPoints`, `buildLeaves`, `makeLeafMat`, `leafVert`/`leafFrag`) with a `THREE.InstancedMesh` approach: one shared `PlaneGeometry(1,1)`, one `MeshLambertMaterial` with `map` from the existing `makeLeafTex(shape)`, `alphaTest: 0.5`, `side: DoubleSide`, and per-instance color via `setColorAt`. Rewrite `buildLeaves` to compute per-instance `Matrix4` (position from `bentPointAt`, random yaw + pitch orientation, world-unit scale mapped from `leafSz`) and per-instance color (same autumn/tint logic). Attach a custom `InstancedBufferAttribute` for `aGrowEnd` and inject `uReveal`-based vertex scaling via `material.onBeforeCompile` to preserve the growth animation. Update `rebuildLeaves`, `rebuild`, `updateGrowth` (leaf section), `dropSelectedLeaf`, leaf picking, and `clearLeafSel` to work with `InstancedMesh` instead of `Points`. Keep `getLeafPlacements`, `bentPointAt`, `PRESETS` fields, GUI wiring, and all branch/trunk/bark/lighting code untouched. Target leaf quad scale ~0.06–0.14 world units at default `leafSz: 0.1`. Dispose and recreate the `InstancedMesh` on every `rebuildLeaves` call.
