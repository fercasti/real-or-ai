# AI Race - Technical Design Document (TDD)

> Companion to [PRD.md](./PRD.md). This document specifies **how** to build the game so an implementer can produce the single-file deliverable with minimal ambiguity.

---

## 0. Reference Inputs

| Input | Location | Purpose |
|-------|----------|---------|
| Game Design Document | `realorai/PRD.md` | Requirements, rules, feel targets |
| Asset Index | `assets/assets.json` | Paths and metadata for all models |
| Reference Mockup | `assets/game_reference.png` | Target visual direction |
| GLTF Characters (x4) | `assets/Characters/glTF/*.gltf` | Crowd + optional player avatar |

---

## 1. Deliverable & Constraints

| Property | Value |
|----------|-------|
| Output | Single `index.html` file with inline `<style>` and `<script type="module">` |
| Runtime dependencies | Three.js r160 via CDN + addons (GLTFLoader, SkeletonUtils, EffectComposer, UnrealBloomPass, RenderPass) |
| Target resolution | 960x540 logical (16:9), responsive letterbox to fill viewport |
| Platforms | Desktop (keyboard) + Mobile (touch) |
| Asset loading | GLTF files from `/assets/Characters/glTF/` — fallback capsules on failure |
| Persistence | `localStorage` key `"AI_Race_v1_highscore"` |

### Import Map

All Three.js modules must be resolved through a single import map to avoid version drift:

```html
<script type="importmap">
{
    "imports": {
        "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
        "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
    }
}
</script>
```

Imports used:

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import * as SkeletonUtils from 'three/addons/utils/SkeletonUtils.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
```

---

## 2. Visual Direction — Cyberpunk Neon

The reference mockup overrides the PRD's daylight palette. The game targets a **dark cyberpunk / synthwave** aesthetic with neon glow.

### Color Palette

| Token | Hex | Usage |
|-------|-----|-------|
| `NEON_CYAN` | `#00FFFF` | Grid lines, AI lane, positive accents |
| `NEON_MAGENTA` | `#FF00FF` | Real lane, billboard frame, UI highlights |
| `NEON_PINK` | `#FF69B4` | Secondary accents, correct-answer burst |
| `DARK_BG` | `#0A0A1A` | Scene background / fog color |
| `ROAD_DARK` | `#111122` | Road surface base |
| `GRID_LINE` | `#00FFFF` | Road grid overlay (emissive) |
| `CORRECT_GREEN` | `#06D6A0` | Correct answer feedback |
| `ERROR_RED` | `#EF476F` | Wrong answer / HP loss |
| `TEXT_WHITE` | `#FFFFFF` | UI text |
| `BILLBOARD_BG` | `#1A0030` | Billboard panel background |

### Bloom (Post-Processing)

UnrealBloomPass with:
- `strength`: 1.2
- `radius`: 0.4
- `threshold`: 0.2

Emissive materials on neon objects drive bloom. Non-glowing objects use `emissive: 0x000000` so bloom ignores them.

### Tone Mapping

```js
renderer.toneMapping = THREE.ReinhardToneMapping;
renderer.toneMappingExposure = 1.0;
```

### Fog

Exponential fog colored `DARK_BG` with density 0.015 to fade distant objects and reinforce depth.

```js
scene.fog = new THREE.FogExp2(0x0A0A1A, 0.015);
scene.background = new THREE.Color(0x0A0A1A);
```

---

## 3. Coordinate System & World Layout

Three.js is right-handed: +X right, +Y up, +Z toward camera.

The PRD specifies player moves **+Z forward**. Since GLTF models face **-Z by default**, the player character model needs `rotation.y = Math.PI` to face the direction of travel.

### World Axes

| Axis | Game Meaning |
|------|-------------|
| +Z | Forward (player travel direction) |
| -Z | Behind player |
| +X | Right lane |
| -X | Left lane |
| +Y | Up |

### Lane Layout

| Lane | Center X | Label |
|------|----------|-------|
| Left | -1.2 | "AI" |
| Right | +1.2 | "REAL" |
| Center (start) | 0.0 | Between lanes |

Lane membership tolerance: player center within **+/-0.6 units** of lane center.

---

## 4. Scene Graph Hierarchy

```
scene
├── ambientLight
├── directionalLight (sun stand-in, casts no shadow in V1 for perf)
├── fog (exponential)
├── roadGroup                         ← recycled road segments
│   ├── roadSegment[0..N]            ← PlaneGeometry tiles
│   └── gridOverlay[0..N]           ← line meshes for neon grid
├── environmentGroup
│   ├── sideDecorations[]            ← wireframe buildings (left + right)
│   └── distantBackdrop              ← optional far-plane geometry
├── gatePool                          ← object pool of Image Gates
│   └── gate[i]
│       ├── billboardFrame           ← BoxGeometry frame
│       ├── imagePanel               ← PlaneGeometry with texture
│       ├── leftLabel                ← sprite or plane ("AI")
│       └── rightLabel               ← sprite or plane ("REAL")
├── playerGroup
│   ├── vehicleMesh                  ← BoxGeometry hover-car
│   ├── vehicleLights[]              ← small emissive accents
│   └── avatarModel (optional)       ← GLTF character on car
├── crowdGroup (optional)
│   └── crowdNPC[i]                  ← SkeletonUtils-cloned GLTF characters
├── particleSystem                    ← reusable particle sprites
└── cameraRig                         ← empty Object3D for camera follow
    └── camera
```

---

## 5. Renderer & Camera Setup

### Renderer

```js
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.toneMapping = THREE.ReinhardToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

Canvas is appended inside a container `<div>`. The container uses CSS to center with letterbox:

```css
#game-container {
    width: 100vw;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #000;
    overflow: hidden;
}
```

Resize logic preserves 16:9:

```js
function resize() {
    const aspect = 16 / 9;
    let w = window.innerWidth;
    let h = window.innerHeight;
    if (w / h > aspect) {
        w = h * aspect;
    } else {
        h = w / aspect;
    }
    renderer.setSize(w, h);
    camera.aspect = aspect;
    camera.updateProjectionMatrix();
    composer.setSize(w, h);  // bloom composer also resized
}
```

### Camera

Third-person chase, PerspectiveCamera:

```js
const camera = new THREE.PerspectiveCamera(60, 16 / 9, 0.1, 200);
```

**Camera rig**: An invisible `Object3D` that moves with the player Z. Camera is offset from rig:

| Property | Value |
|----------|-------|
| Rig Y | 0 (moves along ground plane) |
| Camera offset from rig | `(0, 2.5, -5)` |
| Look-at target offset | `(0, 1.0, +10)` (ahead of player) |
| Follow damping | `lerp` with factor 0.08 per frame |
| Tilt on lane change | +/-0.03 radians Z rotation, 0.18s ease-out |

The camera smoothly follows the player Z position. It does **not** follow X laterally to keep the road centered and both lanes visible.

---

## 6. Asset Loading Pipeline

### Initialization Sequence

```
1. Show HTML loading screen (pure CSS, no Three.js)
2. Initialize Three.js renderer, scene, camera, lights
3. Build procedural geometry (road, vehicle, environment)
4. Load GLTF models in parallel via Promise.all
5. On success: clone models for crowd, attach avatar to vehicle
6. On failure: create capsule fallbacks
7. Initialize game state to MENU
8. Hide loading screen, show menu UI
9. Start animation loop
```

### Model Cache (from threejs skill Pattern 5)

Use a `ModelCache` class with `SkeletonUtils.clone()` for animated models. Load one model for the avatar, one for crowd variety. All four GLTF characters share the same skeleton and animations, so load 2 variants max to save bandwidth.

**Primary load**: `Character_Male_1.gltf` (player avatar)  
**Secondary load**: `Character_Female_1.gltf` (crowd variety)

```js
const MODELS_TO_LOAD = [
    { id: 'avatar', path: '/assets/Characters/glTF/Character_Male_1.gltf' },
    { id: 'crowd',  path: '/assets/Characters/glTF/Character_Female_1.gltf' }
];
```

### Fallback Geometry

If any GLTF fails, create a capsule (CylinderGeometry + SphereGeometry) scaled to 1.6 units, colored `#8E8E8E`:

```js
function createFallbackCharacter() {
    const group = new THREE.Group();
    const bodyGeo = new THREE.CylinderGeometry(0.2, 0.2, 1.0, 8);
    const headGeo = new THREE.SphereGeometry(0.22, 8, 6);
    const mat = new THREE.MeshStandardMaterial({ color: 0x8E8E8E });
    const body = new THREE.Mesh(bodyGeo, mat);
    body.position.y = 0.5;
    const head = new THREE.Mesh(headGeo, mat);
    head.position.y = 1.12;
    group.add(body, head);
    return group;
}
```

### Animation Mixer Management

All active `THREE.AnimationMixer` instances are tracked in a global `mixers[]` array. The game loop calls `mixer.update(scaledDt)` for each.

**Animation mapping for characters** (from `assets.json`):

| Game State | Animation Clip Name | Loop |
|------------|-------------------|------|
| Standing idle | `Idle` | Yes |
| Vehicle ride idle | `Idle_Hold` | Yes |
| Correct answer cheer | `Wave` | No (one-shot) |
| Wrong answer react | `HitReact` | No (one-shot) |
| Death | `Death` | No (clamp last frame) |
| Crowd idle | `Idle` | Yes |
| Crowd celebrate | `Wave` | No |

Use the **crossfade pattern** from the game-patterns skill reference. The `switchAnimation()` function must check `currentAction === newAction` before calling `.reset()` to avoid frame-freeze bugs.

### Model Normalization

Use the **mesh-only bounding box** approach (from GLTF loading guide Pattern 6). Do NOT use `Box3.setFromObject()` on animated models — it includes invisible armature bones and causes models to float. Traverse only `child.isMesh` children.

Target sizes:
- Player avatar on vehicle: scale to 1.8 units tall
- Crowd NPCs: scale to 1.6 units tall

GLTF models face -Z by default. Rotate `model.rotation.y = Math.PI` so they face +Z (forward direction of travel).

---

## 7. Game State Machine

```
┌──────────┐    Enter/Click     ┌──────────┐
│          │ ──────────────────> │          │
│   MENU   │                    │ PLAYING  │
│          │ <────────────────  │          │
└──────────┘   (retry from GO)  └────┬─────┘
                                     │
                              Esc/P  │  HP<=0 or Score>=3
                                     │
                                ┌────▼─────┐
                                │          │
                           ┌──> │  PAUSED  │ ──┐
                           │    │          │   │
                           │    └──────────┘   │
                           │    Resume (Esc/P) │
                           └───────────────────┘
                                     │
                                     │ (from PLAYING)
                                     ▼
                                ┌──────────┐
                                │ GAME_OVER│
                                │ (win/lose)│
                                └──────────┘
```

### State Data

```js
const state = {
    current: 'LOADING',   // LOADING | MENU | PLAYING | PAUSED | GAME_OVER
    score: 0,             // 0..3 (win at 3)
    hp: 3,                // starts 3, lose at 0
    highScore: 0,         // from localStorage
    playerZ: 0,           // current forward position
    playerLane: 0,        // -1 = left, +1 = right, 0 = center (start only)
    targetLaneX: 0,       // interpolation target X
    currentLaneX: 0,      // current interpolated X
    forwardSpeed: 8.0,    // units/sec
    timeScale: 1.0,       // for slow-mo effects
    laneChangeLocked: false,
    nextGateZ: 30,        // Z position of next gate to spawn
    gatesAnswered: 0,
    gameResult: null       // 'win' | 'lose'
};
```

---

## 8. Road System

### Concept

An infinite forward-scrolling road built from recycled tiles. Each tile is a rectangular `PlaneGeometry` with a neon grid overlay.

### Road Segment

| Property | Value |
|----------|-------|
| Tile length (Z) | 20 units |
| Tile width (X) | 6 units (3 units per lane side, plus margins) |
| Active tiles | 6 (covers 120 units of visible road) |
| Surface material | `MeshStandardMaterial` color `ROAD_DARK`, roughness 0.9, metalness 0.1 |
| Grid overlay | `LineSegments` using `EdgesGeometry` or `BufferGeometry` lines, with `MeshBasicMaterial` emissive cyan for bloom |

### Grid Overlay Construction

The neon grid visible in the mockup is a regular rectangular grid drawn on the road surface. Build it as `LineSegments` with `LineBasicMaterial({ color: 0x00FFFF })`:

```js
function createRoadGrid(tileLength, tileWidth, spacingX, spacingZ) {
    const points = [];
    // Longitudinal lines (along Z)
    for (let x = -tileWidth / 2; x <= tileWidth / 2; x += spacingX) {
        points.push(new THREE.Vector3(x, 0.01, 0));
        points.push(new THREE.Vector3(x, 0.01, tileLength));
    }
    // Lateral lines (along X)
    for (let z = 0; z <= tileLength; z += spacingZ) {
        points.push(new THREE.Vector3(-tileWidth / 2, 0.01, z));
        points.push(new THREE.Vector3(tileWidth / 2, 0.01, z));
    }
    const geo = new THREE.BufferGeometry().setFromPoints(points);
    const mat = new THREE.LineBasicMaterial({ color: 0x00FFFF });
    return new THREE.LineSegments(geo, mat);
}
```

Grid spacing: 1.0 unit in both X and Z for the cyan grid. Use 0.01 Y offset above road surface to prevent z-fighting.

### Lane Split Visual

At gate positions, the road grid transitions to two distinct color zones:
- Left half: cyan tint (AI)  
- Right half: magenta tint (REAL)

This is achieved by placing two small overlay planes (3 x 5 units each) with transparent `MeshBasicMaterial` and emissive coloring, positioned at the gate Z and fading out after.

### Tile Recycling

```
Each frame:
  for each tile:
    if tile.endZ < playerZ - 10:
      tile.position.z += tileCount * tileLength   // wrap to front
      regenerate side decorations for this tile
```

---

## 9. Side Environment (Wireframe Buildings)

The mockup shows low-poly **wireframe structures** flanking the road in neon cyan (left) and neon magenta/pink (right).

### Construction

Use `EdgesGeometry` wrapped around `BoxGeometry` of varying sizes to create wireframe building outlines. Material is `LineBasicMaterial` with emissive neon color for bloom.

```js
function createWireframeBuilding(width, height, depth, color) {
    const geo = new THREE.BoxGeometry(width, height, depth);
    const edges = new THREE.EdgesGeometry(geo);
    const mat = new THREE.LineBasicMaterial({ color });
    const line = new THREE.LineSegments(edges, mat);
    line.position.y = height / 2;
    return line;
}
```

Place 3-5 buildings per road tile on each side:
- Left side buildings: X range [-5, -3.5], color `NEON_CYAN`
- Right side buildings: X range [3.5, 5], color `NEON_MAGENTA`
- Randomized heights: 2-8 units
- Randomized widths: 1-3 units
- Z positions staggered along the tile length

Buildings are children of road tiles, so they recycle automatically.

---

## 10. Image Gate System

### Gate Structure (Three.js Group)

Each Image Gate is a `THREE.Group` containing:

```
gate (Group)
├── arch (Group) — the frame structure
│   ├── topBar  (BoxGeometry 4.8 x 0.3 x 0.15) — neon magenta wireframe or solid emissive
│   ├── leftPillar (BoxGeometry 0.15 x 3 x 0.15)
│   └── rightPillar (BoxGeometry 0.15 x 3 x 0.15)
├── billboard (Group) — "CHOOSE YOUR PATH" text + image
│   ├── panelBg   (PlaneGeometry 4 x 2.5) — dark background
│   ├── imageQuad (PlaneGeometry 2 x 2)   — texture with the AI/Real image
│   └── titleText (Sprite or CanvasTexture plane) — "CHOOSE YOUR PATH"
├── leftLaneLabel (Group)
│   ├── labelBg   (PlaneGeometry 1.5 x 0.6) — cyan background
│   └── labelText (CanvasTexture) — "AI" or "DIGITAL REALITY"
├── rightLaneLabel (Group)
│   ├── labelBg   (PlaneGeometry 1.5 x 0.6) — magenta background
│   └── labelText (CanvasTexture) — "REAL" or "AUTHENTIC EXISTENCE"
└── laneMarkers (Group)
    ├── leftFloorText  (PlaneGeometry on road) — large "AI" text
    └── rightFloorText (PlaneGeometry on road) — large "REAL" text
```

### Dimensions & Positioning

| Part | Size | Position (relative to gate center) |
|------|------|-----------------------------------|
| Gate group center | — | (0, 0, gateZ) |
| Top bar | 4.8 x 0.3 x 0.15 | (0, 3.5, 0) |
| Left pillar | 0.15 x 3.0 x 0.15 | (-2.4, 1.5, 0) |
| Right pillar | 0.15 x 3.0 x 0.15 | (2.4, 1.5, 0) |
| Image panel | 2.0 x 2.0 | (0, 2.5, -0.1) |
| Left label | at lane center | (-1.2, 0.8, +2) |
| Right label | at lane center | (1.2, 0.8, +2) |
| Floor "AI" text | 2.0 x 1.0 | (-1.2, 0.02, +4) rotated flat |
| Floor "REAL" text | 2.0 x 1.0 | (1.2, 0.02, +4) rotated flat |

Gate arch material: `MeshBasicMaterial` with emissive neon magenta for bloom glow.  
Label materials: `MeshBasicMaterial` with CanvasTexture-drawn text.

### Gate Object Pool

Pre-allocate **4 gates** in a ring buffer. Only 1-2 are visible at a time.

```js
class GatePool {
    constructor(scene, poolSize = 4) { ... }
    spawn(z, imageData) { ... }     // activate a pooled gate at position Z with image
    despawn(gate) { ... }            // return to pool
    updateAll(playerZ) { ... }       // check for despawn behind player
}
```

### Image Loading for Gates

Each gate displays one image. The image set is embedded as an array of objects:

```js
const IMAGE_SET = [
    { url: 'data:image/...', answer: 'ai' },      // base64 or external URL
    { url: 'https://example.com/photo.jpg', answer: 'real' },
    // ...
];
```

For V1, use a small curated set of 10-20 images (can be placeholder colored squares with "AI" or "REAL" text for initial implementation). Images are loaded as `THREE.TextureLoader` textures and applied to the imageQuad plane.

**Texture loading strategy**: Pre-load the next 2-3 images while the current gate is active. Use `TextureLoader.load()` with a completion callback that assigns the texture to the pooled gate's image panel material.

### CanvasTexture for Text Labels

Create text labels by drawing to an offscreen `<canvas>` and using it as a `THREE.CanvasTexture`:

```js
function createTextTexture(text, fontSize, color, bgColor, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    if (bgColor) {
        ctx.fillStyle = bgColor;
        ctx.fillRect(0, 0, width, height);
    }
    ctx.fillStyle = color;
    ctx.font = `bold ${fontSize}px Arial, sans-serif`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(text, width / 2, height / 2);
    return new THREE.CanvasTexture(canvas);
}
```

---

## 11. Player System

### Vehicle (Procedural)

The hover-car is built from Three.js primitives:

```
playerGroup (Group)
├── vehicleBody (BoxGeometry 1.2 x 0.35 x 0.8) — rounded via bevel or kept sharp
│   material: MeshStandardMaterial color #06A3FF, metalness 0.3, roughness 0.4
├── vehicleTop (BoxGeometry 0.8 x 0.2 x 0.5) — windshield area
│   material: MeshStandardMaterial color #0488CC, metalness 0.5, roughness 0.2, transparent, opacity 0.7
├── lightBarFront (BoxGeometry 0.6 x 0.05 x 0.05)
│   material: MeshBasicMaterial color #FFD166 (emissive for bloom)
├── lightBarRear (BoxGeometry 0.6 x 0.05 x 0.05)
│   material: MeshBasicMaterial color #EF476F (emissive for bloom)
├── hoverGlow (PlaneGeometry 1.0 x 0.6, facing down)
│   material: MeshBasicMaterial color NEON_CYAN, transparent, opacity 0.3
└── avatar (optional GLTF model, positioned standing on vehicle)
```

Vehicle position: Y = 0.25 (floating slightly above road).

### Movement

Lane switching is **discrete target + interpolation**:

```js
function setTargetLane(direction) {
    if (state.laneChangeLocked) return;
    const newLane = direction; // -1 or +1
    if (state.playerLane === newLane) {
        // Already in this lane — trigger denied feedback
        triggerDeniedFeedback(direction);
        return;
    }
    state.playerLane = newLane;
    state.targetLaneX = newLane * 1.2;
    state.laneChangeLocked = true;
    // Unlock after interpolation completes (~0.18s)
    setTimeout(() => { state.laneChangeLocked = false; }, 200);
}
```

Lateral interpolation each frame:

```js
const laneSpeed = 6.0; // units/sec
const dx = state.targetLaneX - state.currentLaneX;
const step = Math.sign(dx) * Math.min(Math.abs(dx), laneSpeed * dt);
state.currentLaneX += step;
// Or use lerp for smoother feel:
state.currentLaneX += (state.targetLaneX - state.currentLaneX) * (1 - Math.exp(-25 * dt));
```

Forward motion each frame:

```js
state.playerZ += state.forwardSpeed * state.timeScale * dt;
```

### Idle Bob

When in PLAYING or MENU state, the vehicle bobs vertically:

```js
vehicleGroup.position.y = 0.25 + Math.sin(elapsed * 3.9) * 0.04;
```

---

## 12. Camera System

### Chase Camera

The camera rig is an `Object3D` that tracks `playerZ`:

```js
function updateCamera(dt) {
    // Smooth Z follow
    cameraRig.position.z += (state.playerZ - cameraRig.position.z) * 0.08;

    // Camera offset from rig
    camera.position.set(0, 2.5, cameraRig.position.z - 5);
    camera.lookAt(0, 1.0, cameraRig.position.z + 10);

    // Lane-change tilt
    const targetTilt = -state.playerLane * 0.03;
    camera.rotation.z += (targetTilt - camera.rotation.z) * (1 - Math.exp(-12 * dt));
}
```

### Zoom Pulse

On correct answer, temporarily modify camera FOV:

```js
function zoomPulse() {
    const baseFOV = 60;
    camera.fov = baseFOV * 0.94; // 6% zoom in
    camera.updateProjectionMatrix();
    // Animate back over 0.18s
    tweenValue(camera, 'fov', baseFOV, 0.18, () => {
        camera.updateProjectionMatrix();
    });
}
```

---

## 13. Input System

### Keyboard

```js
const keys = {};
window.addEventListener('keydown', (e) => {
    keys[e.code] = true;
    handleInput(e.code, 'down');
});
window.addEventListener('keyup', (e) => {
    keys[e.code] = false;
});

function handleInput(code, type) {
    if (state.current === 'PLAYING' && type === 'down') {
        if (code === 'ArrowLeft' || code === 'KeyA') setTargetLane(-1);
        if (code === 'ArrowRight' || code === 'KeyD') setTargetLane(+1);
        if (code === 'Escape' || code === 'KeyP') togglePause();
    }
    if (state.current === 'MENU' && type === 'down') {
        if (code === 'Enter' || code === 'Space') startGame();
    }
    if (state.current === 'GAME_OVER' && type === 'down') {
        if (code === 'Enter' || code === 'Space') restartGame();
    }
}
```

### Touch

Split screen into zones:

| Zone | X Range | Action |
|------|---------|--------|
| Left | 0% - 40% | Left lane |
| Center | 40% - 60% | Pause (during play) / Start (menu) |
| Right | 60% - 100% | Right lane |

```js
renderer.domElement.addEventListener('touchstart', (e) => {
    e.preventDefault();
    const touch = e.touches[0];
    const xRatio = touch.clientX / renderer.domElement.clientWidth;
    if (state.current === 'PLAYING') {
        if (xRatio < 0.4) setTargetLane(-1);
        else if (xRatio > 0.6) setTargetLane(+1);
        else togglePause();
    }
    // ...menu/gameover handlers
}, { passive: false });
```

### Input Feedback

Every input gets a same-frame visual acknowledgment:
- **Valid lane switch**: brief flash on the corresponding control hint (CSS opacity pulse 50ms)
- **Denied (already in lane)**: small CSS shake on control hint (0.12s), red tint flash `#FFDDDD`

---

## 14. Collision / Decision System

### Gate Decision Check

Each frame during PLAYING, check if the player has passed the decision threshold of the nearest active gate:

```js
function checkGateDecisions(playerZ, playerLaneX) {
    for (const gate of gatePool.active) {
        if (gate.decided) continue;

        const decisionZ = gate.position.z + 2; // slightly past gate
        if (playerZ >= decisionZ) {
            gate.decided = true;

            // Determine which lane the player is in
            const inLeftLane = playerLaneX < 0;
            const inRightLane = playerLaneX > 0;

            // Check correctness
            const correct = (inLeftLane && gate.correctLane === 'left') ||
                            (inRightLane && gate.correctLane === 'right');

            // Near-miss detection
            const nearMiss = detectNearMiss(gate);

            if (correct) {
                onCorrectAnswer(nearMiss);
            } else {
                onWrongAnswer();
            }
        }
    }
}
```

### Correct Answer

```js
function onCorrectAnswer(isNearMiss) {
    state.score++;
    state.forwardSpeed = 10.0; // brief sprint
    setTimeout(() => { state.forwardSpeed = 8.0; }, 300);

    // Vehicle squash-stretch anticipation
    squashStretch(vehicleGroup, 0.95, 1.0, 0.08, 0.22);

    // Zoom pulse
    zoomPulse();

    // Floating text
    showFloatingText('+1', CORRECT_GREEN);

    // Progressive intensity
    updateProgressiveIntensity(state.score);

    if (isNearMiss) {
        triggerNearMissReward();
    }

    if (state.score >= 3) {
        triggerWin();
    }
}
```

### Wrong Answer

```js
function onWrongAnswer() {
    state.hp--;

    // Screen shake
    shakeScreen(0.035, 0.28);

    // Red flash overlay
    flashScreen('#EF476F', 0.22);

    // Avatar HitReact animation
    switchAnimation(avatarMixer, 'HitReact', { loop: false });

    // Update HP UI
    updateHPDisplay();

    if (state.hp <= 0) {
        triggerGameOver();
    }
}
```

### Near-Miss Detection

```js
function detectNearMiss(gate) {
    // Was the player in the wrong lane within 0.25s before the decision?
    // Track lane-switch timestamps
    const timeSinceSwitch = performance.now() - state.lastLaneSwitchTime;
    return timeSinceSwitch < 250; // switched within 250ms of decision
}

function triggerNearMissReward() {
    triggerSlowMoSmooth(0.6, 0.14, 0.3);
    zoomPulse();
    spawnParticleBurst(vehicleGroup.position, NEON_CYAN, 20);
    showFloatingText('+0.5 CLOSE!', '#FFD166');
}
```

---

## 15. UI / HUD System

All UI is HTML/CSS overlaid on the canvas. Use `position: absolute` elements inside the game container, styled with the neon palette.

### UI Layer Structure

```html
<div id="ui">
    <!-- HUD (visible during PLAYING) -->
    <div id="hud">
        <div id="hp">♥ ♥ ♥</div>             <!-- top-left -->
        <div id="score">0 / 3</div>            <!-- top-right -->
        <div id="controls-hint-left">← A</div> <!-- bottom-left -->
        <div id="controls-hint-right">D →</div><!-- bottom-right -->
    </div>

    <!-- Menu Screen -->
    <div id="menu-screen">
        <h1>AI RACE</h1>
        <p>Drive left for AI, right for Real</p>
        <div id="menu-buttons">
            <button id="btn-ai">← AI</button>
            <button id="btn-play">PLAY</button>
            <button id="btn-real">REAL →</button>
        </div>
        <p id="best-score">Best: 0</p>
    </div>

    <!-- Pause Overlay -->
    <div id="pause-screen" class="hidden">
        <h2>PAUSED</h2>
        <p>Controls: ← A / D → or touch sides</p>
        <button id="btn-resume">RESUME</button>
    </div>

    <!-- Game Over Screen -->
    <div id="gameover-screen" class="hidden">
        <h2 id="result-title"></h2>
        <p id="result-summary"></p>
        <p id="result-best"></p>
        <button id="btn-retry">RETRY</button>
    </div>

    <!-- Flash overlay -->
    <div id="flash-overlay"></div>

    <!-- Floating text container -->
    <div id="floating-texts"></div>
</div>
```

### UI Styling

- Font: system sans-serif, bold, with `text-shadow` glow matching neon colors
- HP hearts: red `#EF476F`, pulsing slightly
- Score: white with cyan glow
- Menu title: large, neon magenta text shadow
- Buttons: bordered with neon colors, transparent bg, glow on hover
- Play button: gentle scale pulse animation (±3% over 1.2s via CSS `@keyframes`)

### Screen Transitions

| Transition | Effect |
|------------|--------|
| Menu → Playing | Menu fades out 0.3s, HUD fades in 0.2s |
| Playing → Paused | Overlay fades in 0.15s, dim background |
| Playing → Game Over (lose) | Red flash, desaturate, overlay slides in 0.35s |
| Playing → Game Over (win) | Confetti particles, banner slides in 0.35s |
| Game Over → Playing | Quick fade 0.2s, full reset |

---

## 16. Effects / Juice System

### Particle System

A simple sprite-based particle system using a pool of `THREE.Sprite` objects:

```js
class ParticlePool {
    constructor(scene, maxParticles = 50) { ... }
    emit(position, color, count, spread, lifetime) { ... }
    update(dt) { ... } // move particles, fade, recycle
}
```

Particle sprite material: `SpriteMaterial` with a small circular texture (draw a radial gradient on a canvas) and `blending: THREE.AdditiveBlending` for glow.

### Screen Shake

From game-patterns skill. Store camera base position, add random offset:

```js
let shakeIntensity = 0;
let shakeDuration = 0;

function shakeScreen(intensity, duration) {
    shakeIntensity = intensity;
    shakeDuration = duration;
}

function updateShake(dt) {
    if (shakeDuration > 0) {
        shakeDuration -= dt;
        const decay = Math.max(shakeDuration / 0.28, 0);
        camera.position.x += (Math.random() - 0.5) * shakeIntensity * decay;
        camera.position.y += (Math.random() - 0.5) * shakeIntensity * decay;
    }
}
```

### Time Dilation

From game-patterns skill. Modulate `state.timeScale`:

```js
function triggerSlowMoSmooth(factor, holdTime, rampTime) {
    state.timeScale = factor;
    setTimeout(() => {
        const start = performance.now();
        const rampMs = rampTime * 1000;
        function ramp() {
            const t = Math.min((performance.now() - start) / rampMs, 1);
            state.timeScale = factor + (1 - factor) * t;
            if (t < 1) requestAnimationFrame(ramp);
        }
        ramp();
    }, holdTime * 1000);
}
```

### Squash & Stretch

```js
function squashStretch(obj, squashY, restoreY, squashDur, restoreDur) {
    obj.scale.y = squashY;
    obj.scale.x = obj.scale.z = 1 + (1 - squashY) * 0.5; // compensate volume
    setTimeout(() => {
        // Animate back with ease-out
        const start = performance.now();
        function restore() {
            const t = Math.min((performance.now() - start) / (restoreDur * 1000), 1);
            const eased = 1 - Math.pow(1 - t, 3); // cubic ease-out
            obj.scale.y = squashY + (restoreY - squashY) * eased;
            obj.scale.x = obj.scale.z = 1;
            if (t < 1) requestAnimationFrame(restore);
        }
        restore();
    }, squashDur * 1000);
}
```

### Progressive Intensity

| Score | Visual Change |
|-------|--------------|
| 0 | Baseline: subtle bloom, calm colors |
| 1 | Bloom strength +0.2, add vignette overlay (CSS), road grid slightly brighter |
| 2 | Bloom strength +0.4, particle effects larger, wireframe buildings pulse |
| 3 (win) | Full confetti burst, screen-wide particle explosion, bright flash |

### Death Sequence

1. Freeze frame: 0.12s (set `state.timeScale = 0`)
2. Vehicle collapse: scale Y to 0.6 over 0.14s
3. White flash: 0.1s
4. Desaturate: apply CSS `filter: saturate(0.3)` on canvas over 0.5s
5. Vehicle fade out: opacity 0 over 0.6s
6. Show Game Over screen

Total: ~0.8s before Game Over UI.

### Win Celebration

1. Speed boost: `forwardSpeed = 12` for 0.5s
2. Camera hop: Y += 0.5 over 0.18s then settle
3. Confetti particles: 100 multi-colored sprites
4. Banner: "YOU WIN!" slides in from top (CSS animation)
5. Crowd NPCs play `Wave` animation

---

## 17. Game Loop

```js
const clock = new THREE.Clock();

function gameLoop() {
    const rawDt = Math.min(clock.getDelta(), 0.1); // cap for tab-away
    const dt = rawDt * state.timeScale;

    // Always update
    updateAnimationMixers(dt);
    updateParticles(rawDt);
    updateShake(rawDt);
    updateScreenEffects(rawDt);
    updateIdleAnimations(rawDt); // vehicle bob, flag sway

    switch (state.current) {
        case 'PLAYING':
            // Forward motion
            state.playerZ += state.forwardSpeed * dt;

            // Lateral interpolation
            updatePlayerLateralPosition(rawDt);

            // Update vehicle position
            playerGroup.position.set(state.currentLaneX, 0.25, state.playerZ);

            // Camera follow
            updateCamera(rawDt);

            // Road recycling
            recycleRoadTiles(state.playerZ);

            // Gate spawning
            spawnGatesIfNeeded(state.playerZ);

            // Decision checks
            checkGateDecisions(state.playerZ, state.currentLaneX);

            // Despawn passed gates
            gatePool.updateAll(state.playerZ);

            // Update environment (wireframe buildings)
            // (handled by road recycling)
            break;

        case 'MENU':
            // Gentle camera drift or idle scene animation
            updateMenuScene(rawDt);
            break;

        case 'PAUSED':
            // Render but no physics
            break;

        case 'GAME_OVER':
            // Continue rendering death sequence / celebration
            updateGameOverScene(rawDt);
            break;
    }

    // Render with post-processing
    composer.render();
}

renderer.setAnimationLoop(gameLoop);
```

---

## 18. Performance Budget

| Metric | Target |
|--------|--------|
| Draw calls | < 80 per frame |
| Triangles | < 50,000 per frame |
| Texture memory | < 30 MB |
| Frame rate | 60 FPS on mid-range desktop, 30+ FPS on mobile |
| GLTF load time | < 3s on broadband |
| Time to interactive | < 5s total (show menu) |

### Optimization Strategies

1. **Object pooling** for gates and particles (no runtime `new Mesh()` in game loop)
2. **Geometry reuse**: create road tile geometry once, reuse for all tiles
3. **Material reuse**: shared materials for wireframe buildings, road segments
4. **InstancedMesh** for crowd NPCs if > 6 are visible (future optimization)
5. **Texture atlas**: crowd characters already use single Atlas texture
6. **Pixel ratio cap**: `Math.min(devicePixelRatio, 2)`
7. **Fog culling**: objects beyond fog distance are naturally invisible; set frustum far plane to 200
8. **Conditional shadow**: no shadow maps in V1 (significant perf win)

---

## 19. Implementation Phases

### Phase 1: Scaffold & Renderer (Est. ~1 hour)

- [ ] HTML boilerplate with import map
- [ ] Three.js scene, camera, renderer with letterbox resize
- [ ] Post-processing pipeline (EffectComposer + UnrealBloomPass)
- [ ] Dark background, fog, basic lighting
- [ ] Game state machine skeleton (LOADING → MENU → PLAYING → etc.)

### Phase 2: Road & Environment (Est. ~1 hour)

- [ ] Procedural road tiles with neon grid overlay
- [ ] Road tile recycling system
- [ ] Wireframe building decorations on both sides
- [ ] Basic forward scrolling (camera moves +Z)

### Phase 3: Player Vehicle (Est. ~30 min)

- [ ] Procedural hover-car geometry with neon accents
- [ ] Lane switching input (keyboard + touch)
- [ ] Lateral interpolation with cubic ease-out
- [ ] Idle bob animation
- [ ] Vehicle emissive lights for bloom

### Phase 4: Image Gates (Est. ~1.5 hours)

- [ ] Gate group construction (arch + billboard + labels)
- [ ] Gate object pool
- [ ] Gate spawning at intervals ahead of player
- [ ] CanvasTexture text labels ("AI", "REAL", "CHOOSE YOUR PATH")
- [ ] Image loading and display on billboard panel
- [ ] Lane color split at gate positions
- [ ] Floor text markers ("AI" / "REAL" on road surface)

### Phase 5: Collision & Scoring (Est. ~45 min)

- [ ] Decision zone detection (player Z passes gate Z + threshold)
- [ ] Lane membership check with ±0.6 tolerance
- [ ] Correct/incorrect logic
- [ ] Score and HP state updates
- [ ] Near-miss detection (lane switch within 0.25s)
- [ ] Win condition (score >= 3) and lose condition (hp <= 0)
- [ ] localStorage high score persistence

### Phase 6: UI / HUD (Est. ~1 hour)

- [ ] HP hearts display (top-left)
- [ ] Score counter (top-right)
- [ ] Controls hints (bottom corners)
- [ ] Menu screen with title, instructions, play button
- [ ] Pause overlay
- [ ] Game Over screen (win and lose variants)
- [ ] Floating text popups (+1, +0.5, CLOSE!)
- [ ] Flash overlay element

### Phase 7: Juice & Effects (Est. ~1.5 hours)

- [ ] Screen shake on wrong answer
- [ ] Red flash on HP loss
- [ ] Zoom pulse on correct answer
- [ ] Squash-stretch on vehicle
- [ ] Time dilation for near-miss
- [ ] Particle system (sprite pool)
- [ ] Particle burst on near-miss and win
- [ ] Progressive intensity (bloom increase per score)
- [ ] Death sequence (freeze, collapse, desaturate)
- [ ] Win celebration (confetti, banner, camera hop)
- [ ] Input feedback (press flash, denied shake)
- [ ] Lane-change camera tilt

### Phase 8: GLTF Characters (Est. ~45 min)

- [ ] Model loading with fallback capsules
- [ ] Avatar attachment to vehicle (optional)
- [ ] Animation mixer setup and crossfade
- [ ] Crowd NPC placement along road sides
- [ ] Crowd idle + wave on correct answer
- [ ] Model normalization (mesh-only bounds, rotation fix)

### Phase 9: Polish & Testing (Est. ~1 hour)

- [ ] Responsive letterbox testing at various viewport sizes
- [ ] Mobile touch testing
- [ ] Performance profiling (draw calls, frame time)
- [ ] Edge cases: rapid input spam, tab-away and return, very slow devices
- [ ] Placeholder image set (10-20 AI/real images or colored placeholders)
- [ ] Final bloom/color tuning to match mockup aesthetic

---

## 20. Risk Mitigation

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| GLTF files are large (~850KB each), slow to load | Medium | Load only 2 of 4 models. Show loading bar. Characters are optional — game is playable without them. |
| Bloom post-processing tanks mobile FPS | High | Detect mobile via `navigator.userAgent` or low `devicePixelRatio` — reduce bloom strength or disable. Provide fallback emissive materials without composer. |
| Image loading for gates fails or is slow | Medium | Pre-load next 2-3 images. Use placeholder colored panels if load fails. Keep images small (< 200KB each). |
| CanvasTexture text looks blurry | Low | Use 2x canvas resolution relative to display size. Set `texture.minFilter = THREE.LinearFilter`. |
| Lane-change feels laggy | Low | Input target is set same-frame. Interpolation is purely visual. Lock period of 200ms prevents double-tap issues. |
| Import map not supported (older browsers) | Low | Three.js r160 with import maps requires modern browsers. This is acceptable for V1 — document in README. |
| Single HTML file is very large | Medium | Inline JS will be substantial. Consider a build step or external JS file if file size exceeds 500KB. For V1, accept large file. |

---

## 21. File Reference Quick-Look

```
projects/game/public/
├── index.html                          ← THE DELIVERABLE
├── assets/
│   ├── assets.json                     ← asset index (runtime reference)
│   ├── game_reference.png              ← visual target (design reference only)
│   └── Characters/
│       └── glTF/
│           ├── Character_Female_1.gltf
│           ├── Character_Female_2.gltf
│           ├── Character_Male_1.gltf
│           └── Character_Male_2.gltf
└── realorai/
    ├── PRD.md                          ← game design requirements
    └── TDD.md                          ← this document
```
