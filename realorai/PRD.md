# AI Race - Game Design Document

## 1. Summary
AI Race is a short 3D decision-racing game where the player drives forward on a bifurcated road and must choose whether a presented image is "AI" or "Real" by steering into the left or right lane; 3 correct answers win the run, 3 incorrect answers deplete HP until loss. The core mechanic is rapid visual classification mapped to an immediate steering choice; correct choices reward progress, incorrect choices reduce HP and trigger clear failure feedback.

- Assumptions:
  - Rendering will use Three.js (r160) because GLTF character models are provided.
  - Single HTML file (index.html) will load models from the provided /assets path.
  - Units: world units where 1 unit ≈ 1 meter; an average human model is ~1.8 units tall.
  - Platform: desktop + mobile (keyboard + touch); default camera is third-person behind the vehicle.
  - The game uses the provided GLTF character models for optional NPCs / crowd; primary gameplay uses simple vehicle/camera primitives.
  - No external sound/music in V1 (out of scope).

## 2. Technical Requirements
- Rendering: Three.js r160 (must be referenced by a single remote script tag; however the deliverable is a single HTML file that includes this CDN reference).
- Single HTML file with inline CSS and JS (index.html).
- Unit system: world units (1 unit ≈ 1 meter). Use precise heights/sizes below.
- Three.js materials to use when creating surfaces and characters: MeshStandardMaterial, MeshBasicMaterial, MeshPhongMaterial (choose depending on desired shading).
- Lights to use: AmbientLight, DirectionalLight, PointLight for localized effects.
- Valid geometries: BoxGeometry, PlaneGeometry, CylinderGeometry, SphereGeometry for props and UI affordances.
- GLTF models: use provided character GLTF files in /assets/Characters/glTF/*.gltf. Fallbacks: simple capsules/boxes if a model fails to load.

## 3. Canvas & Viewport
- Dimensions: default canvas 960×540 (16:9) on page load. Canvas resizes to fill container while letterboxing to keep aspect ratio.
- Background: gradient sky (top #7CC7FF → mid #CDEBFF → horizon #FFF7E0). Road flat dark asphalt (#2C2F33) with slightly reflective specular highlights.
- Aspect ratio behavior: responsive with letterboxing. Camera viewport preserves 16:9; UI overlays scale with safe margins.

## 4. Visual Style & Art Direction
- Color palette (8+ hex colors with purpose):
  - #7CC7FF (sky top - daylight tone)
  - #CDEBFF (sky mid)
  - #FFF7E0 (horizon warmth)
  - #2C2F33 (road main)
  - #50555A (road edging)
  - #FFD166 (UI highlights / correct)
  - #EF476F (UI error / wrong)
  - #06D6A0 (positive feedback / success)
  - #FFFFFF (text)
  - #000000 (accents / silhouettes)
- Art style: stylized low-to-medium poly with clean, readable shapes; characters are realistic-ish via supplied GLTFs but treated as simple silhouettes at a distance. UI is flat with bold, high-contrast icons.
- Mood/atmosphere: brisk, focused, slightly playful. Emphasis on clarity of the presented image and immediate readability of choices.

For 3D specifics:
- Camera style: third-person chase camera slightly above and behind the vehicle, tilted downward to show the upcoming road and the two images panel.
- Camera position: default at (0, 1.6, -6) relative to player vehicle (player at z=0), looking at (0, 1.2, 0). Smooth damp follow.
- Depth feel: primarily forward-scrolling with depth; minor parallax: roadside props and crowds provide layered motion.
- Lighting mood: bright daylight with soft shadows (DirectionalLight for sun, AmbientLight for fill). Use warm rim light on characters/images for readability.

## 5. Player Specifications
- Appearance: the player "vehicle" is a simple low-profile hover-car primitive: box body with rounded edges and small light bars. The player's chosen character avatar (optional) can ride on the car; use one of the provided GLTF models positioned on the car.
- Size:
  - Vehicle bounding box: 1.2 units long × 0.8 units wide × 0.5 units tall.
  - Player character (if attached): ~1.8 units tall (use model scale to match).
- Colors:
  - Primary: #06A3FF (vehicle body)
  - Accent: #FFD166 (lights)
- Starting position: centered between the two lanes on z = 0, world X = 0, Y = 0.25 (vehicle rests slightly above road).
- Movement constraints:
  - Lane-based steering only: two discrete lanes (left and right). Player can be centered at either lane X = -1.2 (left) or X = +1.2 (right). The player can transition lane with a quick lateral slide (interpolated).
  - Forward motion is automatic (vehicle constantly moves forward at fixed forward speed).
  - No free rotation or braking in V1; choice is binary: left lane = "AI", right lane = "Real" (or vice versa; clearly labeled per image).
- Animated states (for the visible avatar on the vehicle, optional):
  - Idle: subtle bob and head look right/left when steering.
  - Run/Drive: subtle lean in direction of lane change.
  - Hit/Death: brief collapse or fade.

## 6. Physics & Movement
Direction: +Z is forward (player moves toward increasing Z). Up = +Y.

| Property | Value | Unit |
|----------|-------|------|
| Gravity | 0 | units/sec² (vehicle is grounded; gravity affects optional loose props only) |
| Lateral lane switch speed | 6.0 | units/sec (interpolated movement between lanes) |
| Forward speed | 8.0 | units/sec (base forward motion) |
| Sprint speed (momentary) | 10.0 | units/sec (brief speed boost when correct answer, 0.3s) |
| Max lateral snap | instantaneous target ± interpolation | units |
| Ground position (road Y) | 0.0 | units |

Notes:
- Forward speed is constant; correct answers may briefly increase speed for satisfying feedback.
- No jumping in V1.

## 7. Obstacles/Enemies
There are no traditional enemies; gameplay entities that present images and hazards are symmetric with player constraints.

- "Image Gates" (primary interactive objects):
  - Appearance: a floating billboard spanning both lanes slightly ahead of the fork that displays a single image (square panel 2×2 units) and a left/right sign above the corresponding lane indicating which lane maps to "AI" and "Real".
  - Size: billboard panel 2 units wide × 2 units tall, mounted on a 0.3 unit thick frame.
  - Colors: frame #50555A, panel background #FFFFFF while image covers panel; left label background #FFD166 (AI) or right label #06D6A0 (Real), text #000000.
  - Spawn position: at Z distances ahead of player on the central road immediately before the lane split (spawn at Z = playerZ + 30 to 40).
  - Pattern: sequential spawn every 3.0 seconds initially; spacing increases slightly with player progress.
  - Movement: static world objects; player moves toward them.
  - Despawn: when behind the player by 5 units or after passing beyond Z = playerZ + 2 after lane commit.
  - Symmetry: Because the player is constrained to two lanes, the Image Gate occupies both lanes equally and requires a left/right discrete choice—enemy/hazard behavior is symmetric (both lanes can be safe or hazardous depending on the correct answer).
- "Wrong-Path Pit" (visual hazard on incorrect lane):
  - Appearance: painted danger stripe texture on road in the lane that corresponds to the wrong choice just past the gate.
  - Effect: if player enters lane that is wrong, an immediate HP deduction is triggered (no persistent damage-over-time).
- Optional crowd props (use GLTF characters):
  - Appearance and size: provided character models scaled to ~1.6 units tall and placed at sidewalk positions; they are decorative and optionally wave when player makes correct answers.
  - Movement: subtle idle animations only (no collision).

Spawn timing:
- Initial spawn interval: 3.0 sec.
- Minimum spawn interval: 1.8 sec as difficulty ramps (not implemented in V1; table for future).

## 8. World & Environment
- Scroll direction: player moves +Z; environment appears to scroll backward.
- Road layout: straight central road that divides into two side-by-side lanes at the decision point. Lanes merge back to center after each gate for visual clarity.
- Loop/regeneration:
  - Procedural repetition of road segments: segments are recycled to maintain constant forward motion; a repeating tile length of 20 units.
  - Image Gates are queued ahead using a ring buffer to avoid GC spikes.
- Layers/back-to-front:
  - Sky and distant horizon (parallax ratio 0.1)
  - Far buildings/landmarks (parallax 0.25)
  - Trees/props and crowds (parallax 0.5)
  - Road and barriers (parallax 1.0)
  - Foreground UI layer (screen-space)
- Asset usage:
  - GLTF characters for crowds and optional avatar. Use their Idle, Wave, Run animations but restricted to simple loops.
  - Road, vehicle, billboard, and UI are procedurally generated with Three.js primitives and textured materials.
- Fallbacks:
  - If a GLTF fails to load, spawn a capsule primitive (Cylinder + Sphere) scaled to 1.6 units tall and colored #8E8E8E in the same location.

## 9. Collision & Scoring
- Collision detection approach: simple lane-based overlap + rectangular proximity for Image Gate panel (no per-poly collision). When player X lane equals answer-lane at the decision Z threshold, treat as collision/choice.
- Hitbox adjustments: lane membership checks are forgiving — player is considered in lane if center X is within ±0.6 units of lane center (lane centers at ±1.2 units). This is a shrink of 0.2 units from physical vehicle half width.
- What triggers game over:
  - HP reaches 0 (after 3 incorrect answers in V1).
  - Optional: a time limit isn't used in V1.
- Score:
  - Correct answer increments "score" by +1 (progress toward the 3-win goal).
  - Incorrect answer increments "misses" and subtracts 1 HP or reduces HP by 1 per miss; HP starts at 3.
- Near-miss threshold:
  - If player switches lane within 0.25 seconds after the threshold (i.e., nearly missed the valid lane) and ends up correct, reward a near-miss bonus: +0.5 visual score ping (display as "+0.5") and small visual flourish.
- High score storage:
  - localStorage key: "AI_Race_v1_highscore" (deterministic based on game name "AI_Race_v1").

## 10. Controls
| Input | Action | Condition |
|-------|--------|-----------|
| Left Arrow / A / Tap left side of screen | Move to left lane | When not already in left lane and not during locked lane-change animation |
| Right Arrow / D / Tap right side of screen | Move to right lane | When not already in right lane and not during locked lane-change animation |
| Space / Tap top area | Small visual confirm (optional) | Not required to answer; used for accessibility to confirm selection (configurable) |
| Escape / P | Pause | During gameplay |
| Click / Tap center (menu) | Start game | From menu state |

Controls visible:
- On menu: show two large lane buttons labeled "AI ←" and "REAL →" and keyboard hints.
- During gameplay: show small persistent overlay at bottom-left with "Left: ← / A / Tap" and bottom-right "Right: → / D / Tap", plus text "Choose AI or REAL by driving".

## 11. Game States

Menu:
- Display: title "AI Race", short instructions "Drive left for AI, right for Real", an example image panel showing one sample image, two large lane buttons (Left = AI, Right = Real), best score displayed, and "Play" button.
- Controls MUST be visible and interactive (keyboard hints and touch buttons).
- How to start: tap/click Play or press Enter.

Playing:
- Active systems: forward movement, spawning Image Gates, UI score & HP.
- UI shown: top-left HP (3 hearts), top-right Score (0/3), bottom-left/right controls hint, center small timer/progress bar showing upcoming gate distance.
- Visuals: current Image Gate panel visible above the fork when within 40 units.

Paused:
- Trigger: Escape or P.
- Shown: semi-transparent overlay "Paused" with visible controls reminder; everything frozen (animations excluding UI pulse freeze).
- Resume: press P/Escape or click Resume.

Game Over:
- Trigger: HP ≤ 0 OR Score ≥ 3 (win condition).
- What's shown:
  - Lose: "Game Over" with red flash, score summary, best score, Retry button.
  - Win: "You Win!" with celebratory banner and best score comparison.
- How to retry: click Retry or press Enter — resets HP and score and respawns.

## 12. Game Feel & Juice (REQUIRED)

12.1 Input Response
- Lane switch input responds same-frame with a small visual acknowledgment: a brief input flash on the pressed control icon (50ms opacity pulse) and a subtle camera tilt toward the chosen lane.
- Lane movement: immediate target set same-frame, interpolation for lateral movement. While the lateral interpolation occurs, show a "lean" animation on the vehicle/driver (0.12s ease-out lean).
- Denied input (double-tap same lane while already in it): small shake of the pressed control icon (0.12s) and a soft tint flash (#FFDDDD).

12.2 Animation Timing
- Lane-change interpolation: 0.18s duration, easing cubic-out for quick snap and soft settle.
- Anticipation on correct-choice reward:
  - On registering correct answer: 0.08s "anticipation" squash on the vehicle (scale Y 0.95) followed by 0.22s stretch/breathe back to 1.0 with easing back-out.
- UI transitions:
  - Score increment float text: rise 0.6s, fade out 0.6s, easing quint-out.
  - HP loss flash: quick red overlay at 0.18s fade out.
- Near-miss slow-mo: 0.14s slow to 0.6x then recover 0.3s.

12.3 Near-Miss Rewards
- Detection: if player switches into correct lane within 0.25s before or after the decision Z threshold and was previously in the other lane, classify as near-miss.
- Visual: brief slow-motion (0.6x for 0.14s), camera zoom +5% scale pulse, and a particle burst of small yellow pings along the vehicle.
- Audio cue: out of scope; visual equivalent is required (particles + floating "+0.5").
- Score: add a small bonus indicator and attractive floating text.

12.4 Screen Effects
| Effect | Trigger | Feel |
|--------|---------|------|
| Shake | Wrong answer impact | Intensity 0.035 world units, duration 0.28s, quick decay |
| Flash | Wrong answer / HP loss | Red overlay opacity 0.22 → 0 over 0.22s |
| Zoom pulse | Correct answer | Scale +6% over 0.12s then settle over 0.18s |
| Time dilation | Near-miss | Slowdown factor 0.6 for 0.14s, linear recover 0.3s |

12.5 Progressive Intensity
- Score thresholds (visual changes):
  - 0 correct: baseline visuals.
  - 1 correct: UI accent color shifts slightly warmer, small vignette introduced.
  - 2 correct: road contrast increases, particle effects for correct answers more pronounced.
  - 3 correct (win): celebratory particle burst, full-screen confetti and bright banner.
- Visual intensity increases primarily via brightness, particle count, and camera shake on mistakes.

12.6 Idle Life
- Player: subtle up/down bob (±0.04 units over 1.6s), blink animation on character head when idle.
- Environment: drifting clouds, roadside flags sway with small sine motion (amplitude 0.08 units).
- UI: Play button gently pulses (scale ±3% over 1.2s).

12.7 Milestone Celebrations
- Milestones: every correct answer triggers a mini celebration; final milestone (3 correct) triggers large celebration.
- Effects: banner slide-in (0.35s), UI color sweep, and a camera brief hop (0.18s) on final milestone.

12.8 Death Sequence
- Failure visual:
  - Player vehicle performs a dramatic collapse: scale Y compress to 0.6 over 0.14s, then fade-out to 0 over 0.6s.
  - Screen effect: quick white flash then desaturate to 30% over 0.5s.
  - Freeze-frame: 0.12s freeze at impact before starting fade.
- Timing: total death animation ~0.8s including overlays before Game Over screen.

## 13. UX Requirements
- Controls must be visible on menu screen (two large lane buttons plus keyboard hints).
- Controls hint during gameplay as persistent small overlays bottom-left and bottom-right.
- Forgiving collision: lane membership is checked with ±0.6 units tolerance; narrow hitboxes are avoided.
- Mobile/touch support: full-screen touch zones (left 40% for left, right 40% for right); center 20% reserved for pause/menu.
- Image readability: the image panel is large and centered on the gate; images are scaled to fit a 1024×1024 display area and presented with a subtle border and drop-shadow to maximize legibility.

## 14. Out of Scope (V1)
- Sound / music / voiceover
- Complex particle physics systems (only simple particle sprites)
- Persistent difficulty progression across runs
- Power-ups, in-game currency, or unlockables
- Multiplayer / online sharing
- Settings menu beyond basic restart/pause
- Achievements and multiple levels
- Complex animation blending beyond simple loops for characters

## 15. Success Criteria
- [ ] Runs from a single HTML file without runtime errors and loads Three.js r160 and provided GLTFs (with fallbacks).
- [ ] Controls visible on menu AND during gameplay (touch and keyboard).
- [ ] Input feels instant (same-frame response) and lane change interpolation completes in ~0.18s.
- [ ] Correct answer triggers anticipation/zoom/stretch feedback and small speed boost.
- [ ] Near-misses trigger a recognizable reward (slow-mo, particles, floating text).
- [ ] Score updates with visible floating text and top-right counter.
- [ ] High score persists across sessions using localStorage key "AI_Race_v1_highscore".
- [ ] Pause/resume freezes game and UI overlay displays controls.
- [ ] Death (3 incorrect) feels impactful with screen shake/flash and freeze-frame.
- [ ] Collision is forgiving: lane hit detection uses ±0.6 units tolerance.
- [ ] Something moves during idle (player bob, flags, clouds).
- [ ] Winning after 3 correct answers triggers celebratory banner and particle burst.
- [ ] All inputs have defined visual feedback (press flash, denied shake, selection animations).