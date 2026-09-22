# CHANGELOG

## 0.7.1 — 2026-09-22
- Added an on-canvas **↻ RESET** button below the status pill. It does the same as R: new
  terrain, a new building, shots back to 0. The preview toggle and held keys carry over.
  The button highlights on hover, and aiming pauses while the pointer is over it.
- Self-test T17: reset after a shot and a crater gives a fresh READY game.

## 0.7.0 — 2026-09-22
First build of `cannon.html`. Milestones M0–M6 were built in one pass at the human's request,
instead of stopping after each one.
- M0–M1: canvas sizing (16:9, DPR ≤ 2), fixed-timestep loop, pastel scene (sky, sun, drifting
  clouds, far hills), status pill, HUD with the version number.
- M2: deformable terrain (surface function minus craters), bucket index, incremental crater
  punching with rims, `groundBelow`. No Shift-click debug dig (it was due to be removed in M6).
- M3–M4: cannon aimed by mouse and keys, aim preview that shares `advanceBody`/`stepBody`
  with the real shot, sub-stepped projectile, explosions, a cannon that falls into craters,
  the state machine.
- M5: dome building of 140 bricks. **Deviation from §8.3/8.4 (per human request):** each brick
  is a rigid body with rotation, friction and stacking, like Angry Birds game objects. It uses
  a sequential-impulse solver written in the file; no library is loaded. The blast flings
  nearby bricks. The win condition counts *standing* bricks (not moved or tipped over).
- M6: R resets, T toggles the preview, cartoon-style cannon (wooden carriage, spoked wheels,
  banded barrel) and cartoon bricks (bevels and ink outlines), per human request.
- Self-tests T1–T14. T11/T12 were rewritten for rigid bodies. Added T15 (the untouched
  building stays still) and T16 (a blast at the base knocks bricks down).
