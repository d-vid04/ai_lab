# instructions.md — Cannon vs. Building (spec for the coding agent)

This is the implementation specification for `cannon.html`. Follow `AGENTS.md` for workflow
rules (git check, stop after each milestone, version bumps). Build in the milestone order in §10.
If anything here is ambiguous, ask before guessing.

---

## 1. Goal and scope

A single-file HTML + JavaScript game rendered on a `<canvas>`:

- A **cannon** sits on the ground at the left and can be aimed and fired.
- A **brick building** (a dome-shaped stack of small bricks) stands on flat ground at the right.
- Between them the **ground is a rolling curve with a hill**. The ground is **deformable**:
  every explosion removes a circular chunk (a crater), and the ground is drawn and
  collision-tested with those holes.
- Cannonballs follow a ballistic arc, collide with the ground and bricks, blow out craters,
  destroy bricks, and cause unsupported bricks to fall.

Constraints:

- One file: `cannon.html`. Inline `<style>` and `<script>`. No external assets or libraries
  (no images, fonts, or CDN scripts) unless approved per `AGENTS.md`.
- Plain Canvas 2D API. Works when opened directly from disk (`file://`).
- Runs at 60 fps on a normal laptop with ~150 bricks and up to ~200 craters.

Non-goals: multiplayer, levels, menus, saving, sound (sound is a stretch goal, §12).

---

## 2. Look and feel (match the reference screenshot)

The reference is a calm, flat, pastel scene in a rounded rectangle:

- **Page**: background `#0f1b2d`. The game canvas is centered, keeps a **16:9** aspect ratio,
  fits the window with a 16px margin, has `border-radius: 18px` and a soft shadow.
- **Sky**: vertical gradient `#cfeaec` (top) → `#eef6f2` (horizon).
- **Sun**: a soft pale circle `#f4faf7`, radius ~38, at about (43%, 18%) of the world.
- **Clouds**: 1–2 small white blobs (overlapping circles, 70% opacity) upper left, drifting
  right very slowly (~6 units/s) and wrapping around.
- **Far hills**: a pale band `#d5e8e1` just above the horizon, a very gentle curve. Static,
  decorative, never collides.
- **Ground**: teal, vertical gradient `#72b8a3` (surface) → `#4d8c7d` (bottom), with a
  slightly lighter 2px edge along the original surface. Crater edges get a darker rim
  (`#3f7a6c`, ~4px) so holes read clearly.
- **Cannon**: dark navy `#1f2d4a`, a short rounded barrel on a small wheeled base, on the far
  left sitting on the ground.
- **Aim preview**: a short dotted line from the muzzle (small dots, dark navy at ~35%
  opacity, fading out along the arc).
- **Bricks**: small rounded rectangles in warm tones picked at random from
  `#e76f51`, `#ee8959`, `#f4a261`, `#f6bd60`, `#e9c46a`, each with a 1px darker outline
  (`rgba(120,40,20,0.35)`), 1px gaps between bricks.
- **Status pill**: top-right, white rounded pill with a small colored dot and small, bold,
  uppercase, letter-spaced text: `READY TO FIRE` (teal dot `#2a9d8f`).
- **Font**: system UI stack (`system-ui, -apple-system, "Segoe UI", Roboto, sans-serif`).

---

## 3. File layout (section order inside `cannon.html`)

```
<!-- required header comment (see AGENTS.md) -->
<!doctype html> <html> <head> <meta charset> <meta viewport> <title> <style>…</style> </head>
<body> <canvas id="game"></canvas> <div id="test-panel" hidden></div>
<script>
"use strict";
// ===== CONFIG =====          VERSION, world size, physics & tuning constants (§11), palette
// ===== MATH UTILS =====      clamp, lerp, deg/rad, rng with seed, circle-rect overlap
// ===== TERRAIN =====         createTerrain, surfaceY, isSolid, addCrater, groundBelow, renderTerrainLayer
// ===== BUILDING =====        createBuilding, brick support check, falling bricks
// ===== CANNON & AIM =====    cannon state, muzzle position, input → aim
// ===== PROJECTILE =====      stepBody (shared by projectile AND preview), collisions, explode
// ===== PARTICLES =====       debris
// ===== GAME STATE =====      createGame, resetGame, state machine (§6.4)
// ===== RENDER =====          draw scene layers in order (§4.4)
// ===== INPUT =====           mouse + keyboard handlers
// ===== LOOP =====            fixed-timestep loop
// ===== SELF-TESTS =====      runSelfTests (only when location.hash === "#test")
// ===== BOOT =====
</script>
</body></html>
```

Design rule: **all logic functions take the state they operate on as a parameter**
(`isSolid(terrain, x, y)`, not a global). This is what lets the self-tests build their own
small test worlds without touching the live game.

---

## 4. World, coordinates, and the game loop

### 4.1 World units
- Fixed logical world: **W = 1600, H = 900** units. Origin top-left, **y increases downward**.
- All game logic uses world units. Only rendering and input convert to screen pixels.

### 4.2 Canvas sizing
- On load and on `resize`: compute the largest 16:9 rectangle that fits the window minus
  margins; set the canvas CSS size to that; set the backing size to CSS size ×
  `devicePixelRatio` (capped at 2). Each frame, `ctx.setTransform(scale, 0, 0, scale, 0, 0)`
  where `scale = canvas.width / W`, then draw in world units.
- Mouse → world: `worldX = (e.clientX - rect.left) * (W / rect.width)`, same for y.

### 4.3 Fixed-timestep loop
- `requestAnimationFrame` loop with an accumulator; simulation step `DT = 1/120` s.
- Clamp the frame delta to 0.1 s (tab switches shouldn't explode the sim).
- `update(game, DT)` as many times as needed, then `render(game)` once.
- Deterministic: the same aim must produce the same shot (important for §9 T10).

### 4.4 Render order (back to front)
sky → sun → clouds → far hills → terrain layer → bricks → cannon → projectile →
debris → aim preview → status pill → HUD text.

---

## 5. Terrain (the core of the lab)

### 5.1 Model
The ground is **an original height function minus a set of circles**:

```
terrain = {
  surface: Float32Array(W + 1),   // precomputed surfaceY for x = 0..W
  craters: [],                    // { cx, cy, r }
  buckets: Map<int, crater[]>,    // spatial index by x (see 5.4)
  layer: OffscreenCanvas|canvas,  // cached drawing of the ground with holes
}
```

A point is **solid ground** when it is below the original surface **and** not inside any
crater, **or** it is in the bedrock strip at the very bottom (so the world never gets a hole
through the floor).

### 5.2 Original surface function
Horizon/base level `GROUND_Y = 620`. Smaller y = higher ground.

```
baseSurfaceY(x) =
    GROUND_Y
  - 110 * exp(-((x - 400) / 75)^2)    // main hill, left of centre (the "wave" in the reference)
  + 28  * exp(-((x - 300) / 32)^2)    // small dip just right of the cannon
  - 10  * sin(x / 140) * flatten(x)   // gentle roll
flatten(x) = 1 for x < 1150, fades linearly to 0 by x = 1250, 0 beyond
```

The region x ≥ 1250 must be perfectly flat at `GROUND_Y` (the building sits there).
Precompute `surface[x]` for integer x at creation; `surfaceY(terrain, x)` linearly
interpolates between samples and clamps x to [0, W].

These numbers are a starting point; the agent may tune them to match the reference, but
must keep the flat building area and keep the cannon at x = 120 on reasonably level ground.

### 5.3 Hit test — `isSolid(terrain, x, y)`
```
if x < 0 or x > W:                 return false   // open sides, projectiles fly off
if y >= H - BEDROCK:               return true    // bedrock, 20 units, indestructible
if y < surfaceY(terrain, x):       return false   // above the original ground → air
for each crater in bucket(x):
    if (x-cx)^2 + (y-cy)^2 < r^2:  return false   // inside a hole → air
return true
```
- Circles entirely above the surface have no effect (harmless).
- Overlapping craters naturally form a union of holes.
- Holes can create overhangs and caves; that's allowed and must render and collide correctly.

### 5.4 Fast lookup: bucket index
- Bucket width `BUCKET = 64` units. `bucketIndex = floor(x / BUCKET)`.
- `addCrater` inserts the crater into **every** bucket its span `[cx - r, cx + r]` overlaps.
- `isSolid` checks only the bucket for x. This keeps each test to a handful of circles even
  with hundreds of craters. (Test T6/T7 cover this.)

### 5.5 `addCrater(terrain, cx, cy, r)`
- Push to `craters`, insert into buckets, and **punch the hole into the cached layer
  incrementally** (see 5.6). No full redraw per crater.

### 5.6 Rendering with holes
Keep an offscreen canvas at world size × `TERRAIN_RES` (2) for crisp edges.

Full build (on create/reset only):
1. Clear. Build a path: move along `surface[x]` every 2 units from x = 0 to W, then down to
   (W, H), across to (0, H), close. Fill with the ground gradient.
2. Stroke the surface line with the lighter edge colour.
3. Fill the bedrock strip in a slightly darker teal.
4. Apply all existing craters with step 5.6b.

Per crater (step 5.6b):
1. `globalCompositeOperation = "destination-out"`, fill the circle → removes pixels (the hole).
2. `globalCompositeOperation = "source-atop"`, stroke the same circle with the rim colour,
   `lineWidth = 8` (half of it lands on remaining ground, giving a 4px rim only where ground
   still exists).
3. Re-fill the bedrock strip with `source-over` so bedrock is never visually removed.
4. Reset to `source-over`.

Each frame, the main render just `drawImage`s the layer scaled to world size.

### 5.7 Helper — `groundBelow(terrain, x, yStart)`
Returns the first y ≥ yStart where `isSolid` is true, scanning down in 1-unit steps (bedrock
guarantees termination). Used to place the cannon and to land falling things.

---

## 6. Cannon, aiming, controls, HUD

### 6.1 Cannon
- Fixed x = 120. Its base rests on the ground: on creation, `y = groundBelow(terrain, 120, 0)`.
- If a crater removes ground beneath it, the cannon **falls** with gravity until
  `isSolid(terrain, x, y + 1)` is true again (it can end up in a pit; that's fine).
- Drawing: base = navy half-disc radius 16 with two small wheels; barrel = navy rounded rect
  48 × 14 pivoting at the top of the base, rotated to the aim angle.
- `muzzle(cannon)` = pivot + (cos a, −sin a) × 48. Shots spawn at the muzzle.

### 6.2 Aim state
- `angle` in degrees, clamped to **[5°, 85°]**, 0° = pointing right, positive = up.
- `power` in **[0.15, 1.0]**. Launch speed = `power × MAX_SPEED`.
- Defaults: angle 35°, power 0.65.

### 6.3 Controls
| Input | Effect |
|---|---|
| Mouse move over canvas | Angle points from pivot to the cursor; power = distance(pivot, cursor) / `AIM_RANGE` (450), clamped |
| Left click | Fire (only when READY) |
| ← / → or A / D (hold) | Angle −/+ at 60°/s |
| ↑ / ↓ or W / S (hold) | Power +/− at 0.6/s |
| Space | Fire |
| R | Reset the whole game (new terrain, new building, shots = 0) |
| T | Toggle the aim preview |

Mouse and keyboard edit the same aim state; the last input wins. Prevent default on Space
and the arrow keys so the page doesn't scroll.

### 6.4 State machine and status pill
| State | Pill text | Dot colour | Leaves when |
|---|---|---|---|
| `READY` | READY TO FIRE | `#2a9d8f` | player fires → `IN_FLIGHT` |
| `IN_FLIGHT` | IN FLIGHT | `#e9a23b` | projectile explodes or leaves the world → `RELOADING` |
| `RELOADING` | RELOADING | `#9aa5a9` | `RELOAD_TIME` (0.8 s) elapsed → `READY` (or `WON`) |
| `WON` | BUILDING DOWN | `#e76f51` | R → reset |

Only one projectile exists at a time. Aim can still be adjusted during `IN_FLIGHT` and
`RELOADING`, but the preview is only drawn in `READY`.

### 6.5 HUD
Bottom-left, small (14px), navy at 70% opacity:
`Angle 35° · Power 65% · Shots 0 · Bricks 148/148 · v0.x.y`

On `WON`, a centered white rounded card: **"Building down in N shots"** and a smaller line
"Press R to play again".

### 6.6 Aim preview
- Simulate a virtual projectile from the muzzle using **the exact same `stepBody` function**
  as the real projectile (no duplicated physics), for up to `PREVIEW_TIME` (0.9 s) or until it
  would hit ground or a brick.
- Draw every 6th simulated point as a dot, radius 2.2, opacity fading from 0.45 to 0.
- Recompute when the aim or the world changes, not necessarily every frame.

---

## 7. Projectile, collisions, explosions

### 7.1 Motion
- Body: `{ x, y, vx, vy, r: 7 }`, gravity `GRAVITY = 900` units/s² downward, no drag.
- `stepBody(body, dt)` integrates with semi-implicit Euler (update v, then position).

### 7.2 Sub-stepping (no tunnelling)
Within each `DT`, split the move into `n = ceil(speed × DT / MAX_STEP)` sub-steps with
`MAX_STEP = 3` units, and check collisions after each sub-step. A fast ball must never pass
through a thin ledge or a single brick (test T9).

### 7.3 Collision checks per sub-step (in this order)
1. **Bricks**: circle–rectangle overlap against every non-destroyed brick
   (static or falling). First hit → explode at the ball's position.
2. **Terrain**: test the ball centre plus 4 points on its rim (±r in x and y). Any solid →
   explode at the ball centre.
3. **Out of world**: x < −50, x > W + 50, or y > H + 50 → remove, no explosion. There is **no
   top boundary**; a high shot comes back down.

### 7.4 `explode(game, x, y)`
1. `addCrater(terrain, x, y, CRATER_R)` (45). Applied for every explosion; it only matters
   where it overlaps ground.
2. Destroy every brick whose **centre** is within `BLAST_R` (42) of (x, y). Each destroyed
   brick spawns 3 debris particles (§8.4).
3. Run the brick support pass (§8.3).
4. Re-check the cannon's footing (§6.1).
5. Invalidate the aim preview; switch state to `RELOADING`.

---

## 8. Building (bricks)

### 8.1 Layout
- Brick size `BRICK_W = 22`, `BRICK_H = 13`, 1-unit gap drawn (bricks tile at 22 × 13 pitch,
  drawn inset by 0.5 on each side).
- 12 columns, centred at x ≈ 1420, bottom row resting exactly on `GROUND_Y`.
- **Dome profile**: column i (0..11) has height
  `rows(i) = max(5, round(14 × sqrt(1 − ((i − 5.5) / 6.5)^2)))`. That gives a rounded top
  like the reference (~148 bricks). Colours random from the palette using a seeded RNG so
  reset produces a new but valid building.

### 8.2 Brick record
`{ x, y, w, h, color, state: "static" | "falling", vy }`. Destroyed bricks are removed from the
list (and become debris particles). Do not keep a grid; after bricks fall they are no longer
grid-aligned, so support is geometric.

### 8.3 Support pass — who stays up
A static brick is **supported** if either:
- another **static** brick's top is within 1 unit of this brick's bottom **and** they overlap
  horizontally by at least 40% of `BRICK_W`; or
- the terrain is solid at at least 2 of 3 sample points 1 unit below the brick's bottom edge
  (left quarter, centre, right quarter).

Algorithm: sort static bricks bottom-to-top (largest y first); repeat passes marking
unsupported bricks as `falling` until a pass changes nothing (cascading collapse).

### 8.4 Falling bricks and debris
- **Falling bricks** drop straight down (no rotation, no sideways motion) under `GRAVITY`.
  They land when their bottom touches a static brick (same overlap rule) or solid terrain;
  then snap to that contact, become `static`, and trigger another support pass. They can be
  hit by projectiles while falling.
- **Debris**: small squares (4–6 units) in the brick's colour, velocity = random 80–260
  units/s directed away from the blast (plus a random upward kick), gravity applied. On
  hitting terrain: `vy *= −0.3`, `vx *= 0.6`. Fade out over 1.5 s, then remove. Debris is
  purely visual: no collision with bricks, projectiles, or each other.

### 8.5 Win condition
When remaining bricks ≤ 10% of the starting count, enter `WON` after the current
`RELOADING` finishes.

---

## 9. Self-tests (run with `#test`)

When `location.hash === "#test"`, run `runSelfTests()` after boot. Each test builds its **own**
small terrain/game objects (never the live game). Show results in `#test-panel` (fixed,
top-left, monospace, green ✓ / red ✗ per test, plus a total) and `console.table` them. A
failing test prints expected vs. actual. The game still runs underneath.

| ID | Test |
|---|---|
| T1 | Point 30 units below the original surface, no craters → solid |
| T2 | Point 30 units above the surface → not solid |
| T3 | Crater r = 40 centred on the surface; point 20 below surface at crater centre → not solid |
| T4 | Same crater; point 45 units from the centre (outside), below surface → solid |
| T5 | Two overlapping craters; a point inside only the second → not solid; inside neither → solid |
| T6 | Crater at x = 100; point at x = 1500 below surface → solid (bucket isolation) |
| T7 | Crater centred exactly on a bucket boundary (cx = 64 × 10), r = 50; points inside it on both sides of the boundary → not solid |
| T8 | Crater that reaches the bottom; point in the bedrock strip inside it → solid |
| T9 | Ball dropped straight down at 3000 units/s from above the flat area → first reported impact within 3 units of the surface (no tunnelling) |
| T10 | Aim preview's collision point equals the real projectile's explosion point for the same aim (within 1 unit) |
| T11 | Brick sitting on ground; add a crater under it → support pass marks it `falling` |
| T12 | Stack of 3 bricks; destroy the bottom one → the two above become `falling` |
| T13 | `isSolid` with x < 0 or x > W → false |
| T14 | Cannon placed on terrain, then crater under it → after enough updates it rests on solid ground again |

---

## 10. Milestones (build in this order, stop after each)

Each milestone: git-check → plan → implement → run self-tests that exist so far → bump
version + CHANGELOG → stop and report (per `AGENTS.md`).

The lab suggests seeing what you can get in about 10 minutes. **M0–M4 is the minimum playable
game**; M5–M6 complete the reference behaviour. If time is short, stop cleanly at the end of
any milestone.

| # | Version | Deliverable | Accept when |
|---|---|---|---|
| M0 | 0.1.0 | Header comment, section banners, `VERSION`, CONFIG constants, canvas sizing (16:9, DPR), fixed-timestep loop, sky gradient, version text in the corner, `CHANGELOG.md` | Opens with no console errors; resizing keeps 16:9 and stays sharp |
| M1 | 0.2.0 | Static scene: sun, drifting clouds, far hills, terrain layer from the surface function (no craters), status pill showing READY TO FIRE | Side by side with the reference it reads as the same scene layout |
| M2 | 0.3.0 | Craters: model, buckets, `isSolid`, `addCrater`, incremental punch rendering with rims, `groundBelow`. Self-test harness + T1–T8, T13. Debug: **Shift + click digs a crater at the cursor** (remove in M6) | Shift-click holes render cleanly incl. overlaps and overhangs; tests pass |
| M3 | 0.4.0 | Cannon drawn on terrain, aim via mouse + keys, aim preview using `stepBody` | Barrel tracks the mouse; angle/power clamp; preview arc stops at the ground |
| M4 | 0.5.0 | Real projectile, sub-stepping, terrain collision, `explode` with craters, cannon falling, state machine + pill states, HUD. Tests T9, T10, T14 | Shots arc, crater the hill, can tunnel through it over several shots; pill cycles correctly |
| M5 | 0.6.0 | Building: dome layout, brick collisions, debris, support pass, falling bricks, win state. Tests T11, T12 | Hitting the base collapses the bricks above; hitting the top chips it; win card appears |
| M6 | 0.7.0 | Reset (R), preview toggle (T), remove Shift-click debug dig, final tuning to match the reference, header filled in, `project_setup.md` updated | Full game plays start to win to reset with zero console errors; all tests pass |

---

## 11. Tuning constants (all in CONFIG)

| Name | Value | Notes |
|---|---|---|
| `W`, `H` | 1600, 900 | World size |
| `GROUND_Y` | 620 | Flat ground level |
| `BEDROCK` | 20 | Indestructible bottom strip |
| `TERRAIN_RES` | 2 | Offscreen terrain layer resolution multiplier |
| `BUCKET` | 64 | Crater index bucket width |
| `DT` | 1/120 | Simulation step (s) |
| `MAX_STEP` | 3 | Max sub-step distance (units) |
| `GRAVITY` | 900 | units/s² |
| `MAX_SPEED` | 1150 | Launch speed at power 1.0 (should reach just past the building) |
| `BALL_R` | 7 | Projectile radius |
| `CRATER_R` | 45 | Crater radius per explosion |
| `BLAST_R` | 42 | Brick destruction radius |
| `RELOAD_TIME` | 0.8 | s |
| `AIM_RANGE` | 450 | Mouse distance for full power |
| `PREVIEW_TIME` | 0.9 | s of preview arc |
| `BRICK_W`, `BRICK_H` | 22, 13 | Brick size |
| `WIN_FRACTION` | 0.10 | Win when ≤ 10% bricks remain |

Tuning targets: at power 1.0 and ~40°, a shot should land around or slightly past the
building; the hill should block low, weak shots so the player has to arc over it or dig
through it.

---

## 12. Stretch goals (only after M6, and only if the human says go)

- Wind (random per shot, shown in the HUD) applied in `stepBody`.
- Screen shake and a short expanding flash ring on explosions.
- Sound effects generated with the Web Audio API (no audio files).
- Rotating, tumbling bricks via a physics library such as Matter.js (would require a pinned CDN
  script tag; ask first per `AGENTS.md`, and keep the terrain collision custom).
- Crater size scaling with impact speed.
