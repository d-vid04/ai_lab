# CHANGELOG

## 0.8.1 — 2026-09-22
- Softer in-flight whoosh: about 10 dB quieter (new `WHIZZ_VOLUME`), a broad low filter
  instead of a narrow high one (a rush of air, not a hiss), a lower pitch range
  (250–1100 Hz, was 300–2200 Hz), a fainter whistle, and a gentle fade in and out.

## 0.8.0 — 2026-09-22
- Sound effects, all synthesised with the Web Audio API in the file (no audio files): a cannon
  boom; a whoosh + whistle that follows the ball (the pitch drops as it falls); a deep blast for
  ground hits; a shorter, brighter blast with clattering shrapnel for brick hits; a ceramic
  clack for brick-on-brick collisions and a duller thock for brick-on-ground. Collision sounds
  are driven by new physics contacts, scale with impact speed, are panned by x, and are
  throttled so a collapse doesn't stack hundreds of voices.
- Esc opens a pause menu: the simulation and sound freeze, and the cannon ignores the mouse and
  keys. It offers Resume, Reset game, Past records (round, time, shots, best marked ★) and a
  Sound on/off toggle.
- Records are kept in memory for the session only (AGENTS.md: no storage). Time counts from the
  first shot to the building coming down. The HUD and win card now show the time.
- `advanceBody` now reports `"brick"` / `"ground"` instead of `"hit"`, so the right sound plays.
- Self-tests T18 (win makes a session record, reset keeps it) and T19 (collision events fire for
  real impacts and stay silent for the resting building). T9 now expects `"ground"`.

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
