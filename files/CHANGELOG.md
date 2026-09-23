# CHANGELOG

## 0.15.2 — 2026-09-23
- **The win registers much sooner.** 0.15.0 waited until every brick in the world had stopped
  moving, and non-free modes also waited for the reload, so a round could end about 3 s after
  the building had visibly fallen. There is no longer a settle wait: the building-down check
  runs in every state, and a win is declared once it has held for 0.5 s (`DOWN_HOLD`). After a
  flattening volley, rounds now end in about 1.6 s instead of about 3.2 s.
- To make that safe without waiting for the rubble, the stack test got stricter about what
  counts as the *building*:
  - A brick only extends a stack if it sits on the brick that was originally built under it,
    so rubble that lands flat on other rubble no longer looks like a wall.
  - Bricks still moving faster than 30 u/s (`MOVING_SPEED`) count as possibly standing, so a
    toppling piece can't trigger a win in mid-fall.
  - The upright tolerance widened from about 11° to about 27° (`UPRIGHT_SIN`), so a wall that
    is merely leaning still counts.
  Checked over 16 simulated games: no false wins where a standing stack reappeared (at most one
  brick over the limit shifting afterwards), and the tower, shoved-wall, two-rows and rubble
  cases in T42 still come out right.
- Tests: T45 (the win registers in under 2 s while rubble is still moving) and T46 (a near
  miss that rattles the building doesn't count as down).

## 0.15.1 — 2026-09-23
- **Rapid fire is much faster.** It used to halve the reload but still wait for each ball to
  land, which worked out to about one shot every 1.7 s. It now has a 0.3 s cooldown
  (`RAPID_COOLDOWN`) and can fire again while earlier balls are still flying: 17 shots in
  5 s against Single's 3. The bar caption shows "RAPID · EVERY 0.3 S".
- T34 now checks that the second Rapid shot is allowed after about 0.3 s with the first ball
  still airborne, and that a too-early click is ignored.

## 0.15.0 — 2026-09-23
- Two extra bricks cap the dome on its centre columns: 142 bricks, 15 tall.
- **Ground-breaking animation**: before a crater is cut, points inside it are sampled, and each
  one that is still ground becomes a dirt clod sprayed up and out of the hole. The clods reuse
  the brick-debris particles (gravity, bounce, fade). Bombs throw about twice as many. Blasts in
  mid-air throw none.
- **Building-down check rewritten** (replaces spec §8.5's "≤ 10 % of bricks still in place").
  The old rule was measurably wrong both ways: one untouched 14-brick column (exactly 10 %)
  counted as a win, a whole building shoved 16 units sideways counted as a win, and two flat
  rows left did not. The building now counts as down once:
  - every brick in the world has settled (below 30 u/s; bricks flung off the sides are
    ignored), and
  - no **intact stack** (upright bricks sitting squarely on each other) is taller than **4**
    bricks (`DOWN_STACK`). Tilted rubble heaps don't form stacks, so they count as down.
  The HUD shows "Tallest stack N (down at 4)", and the Controls page states the goal.
- Brick shards and dirt clods share one `createChunk` helper (it replaces `createDebris`).
- Tests: T42 (tower no / shoved wall no / two rows yes / rubble yes / still moving no), T43
  (142 bricks, stack of 15, standing), T44 (ground hits throw clods, air blasts don't). T18 now
  leaves a 3-brick stack for its win.

## 0.14.0 — 2026-09-23
- The win card moved up, and a **🏆 TOP TIMES** card sits under it: the session's best 5 rounds,
  fastest first (rank, player BOZO, round with ✋ if tampered, time, shots). The round just won
  is highlighted. If it missed the top 5, it is added below a divider with its real rank.
- **"Uncle Sam" detector**: every bomb launch is timed. If 7 or more bombs (`BOMB_LIMIT`) are
  fired inside any 5-second window (`BOMB_WINDOW`) during a round, the win taunt becomes "UNCLE
  SAM IS CALLING YOUR NAME!" whatever the shot count. Only bombs count, not cannonballs, so in
  practice it comes from Spam clicking or Full auto.
- Long taunts shrink to fit the win card.
- Tests T40 (the detector triggers at 7-in-5 s, not at 6 or at 1 per second, and overrides the
  taunt) and T41 (leaderboard order, top 5, the current round shown with its rank).

## 0.13.0 — 2026-09-22
- **Fire modes**, unlocked by buildings knocked down this session, chosen with keys **1–6** or
  the new bar at the top centre. Each slot shows its key number and an icon; locked slots show
  a padlock, and hovering one says how many wins it needs. A toast under the bar announces each
  unlock, and the chosen mode survives resets.
  1. SINGLE (•): as before.
  2. RAPID (• with speed lines), 1 win: half the reload time.
  3. BURST (•••), 2 wins: three balls 0.12 s apart, then the normal reload.
  4. BOMB (bomb icon), 4 wins: black bombs with a lit fuse. Crater and brick-destruction radius
     ×2, a wider and stronger fling, a flash and shock ring, and a deeper, longer boom.
  5. SPAM (three bombs), 10 wins: every click fires a bomb, with no reload.
  6. FULL AUTO (bomb »), 10 wins: holding the mouse or Space streams bombs every 0.1 s.
- Several balls can now be in the air at once (`game.projectiles`). The state machine waits
  for all of them to land and then reloads for the current mode. Free-fire modes can win
  mid-barrage. Every ball counts as a shot.
- Craters that can't change the ground (inside an existing crater, or entirely above the
  surface) are skipped, so sustained full auto stays around 1 ms per physics step.
- The Controls page lists the fire modes, unlocks and full-auto hold.
- Tests T33–T39: unlock thresholds and locked selection, rapid reload, burst, bomb crater
  size, spam, full auto hold/release, the unlock toast, and the mode surviving a reset.

## 0.12.1 — 2026-09-22
- Fix: the in-flight whoosh kept playing after switching tabs while the ball was in the air
  (a hidden tab stops the frame loop that normally silences it). It now stops when the tab is
  hidden and resumes on return if the ball is still flying.

## 0.12.0 — 2026-09-22
- On load, a prompt asks for the player's name (the game is paused and keys go to the text
  box). Submitting shows **WELCOME BOZO!** for 1.8 s (click to skip). Whatever the name, the
  game calls the player BOZO (`PLAYER_NAME`).
- Past records has a new **Player** column (always BOZO) before Round.
- The win card shows a taunt above "Building down in N shots", based on N: ≤1 "DID YOU CHEAT
  BOZO?", 2 "YOU FOUND THE STRAT BOZO!", ≤4 "GOOD JOB BOZO!", ≤6 "OK WORK BOZO!", ≤8 "COULD BE
  BETTER BOZO!", otherwise "UNINSTALL ALREADY BOZO!". The card is taller to fit it.
- The Esc menu no longer freezes audio: the song ducks to 20 % (`SONG_MENU_DUCK`) and fades back
  when the menu closes. The in-flight whoosh goes quiet while the ball is frozen.
- The test panel now sits above the overlays. Test T32 covers the taunt thresholds.

## 0.11.1 — 2026-09-22
- The song toggle moves from S to **F** (pill key cap and menu row updated). **S lowers power
  again**, as in 0.10.
- Song volume up a little: `SONG_VOLUME` 0.22 → 0.32 (about +3 dB), still under the effects.
- New **⌨ Controls** page in the Esc menu listing every control: aim, angle, power, fire,
  Tamper (grab, throw, dig), aim preview, song, reset, pause.
- Test T31: the key bindings (S lowers power, F isn't an aim key).

## 0.11.0 — 2026-09-22
- Background song, synthesised in the file and looping: a bouncy 8-bar tune in C at 132 BPM,
  with a marimba melody over an oom-pah bass, offbeat chord stabs, a soft kick and a woodblock
  tick. Every other pass the melody drops an octave. It plays quietly (`SONG_VOLUME` 0.22,
  under the effects), is scheduled on the audio clock, fades in and out, and pauses with the
  Esc menu.
- **♪ SONG · ON/OFF** pill with an **S** key cap under ESC MENU, plus a "♪ Song" row (On/Off, S)
  in the Esc menu. S toggles the song everywhere, including while the menu is open.
- The song is on by default (it starts on the first click or key press, since browsers block
  audio until then). Its state lives in the session, so a reset never changes it.
- **S no longer lowers power**; use ↓. The menu's "Sound" row is renamed "All sound" (it mutes
  effects and song together).
- Tests T29 (default on, button placement, state survives resets) and T30 (song data shape).

## 0.10.1 — 2026-09-22
- Redid the second half of the dig sound. The bright high-pass "soil pattering" bursts
  sounded like crumpling paper. After the opening scrape and thump, the loosened clump now
  lands with a dull low thud and settles in four fading, muffled bumps (low-passed below
  ~700 Hz).

## 0.10.0 — 2026-09-22
- Renamed "Tamper-Tantrum Mode" to **Tamper** everywhere.
- **T** now toggles Tamper. The aim-preview toggle moves from T to **P**; this departs from
  spec §6.3 because the human asked for T.
- The Tamper, Reset and Menu pills now show key caps (**T**, **R**, **ESC**). The Esc menu has
  a "✋ Tamper mode" row with its On/Off state and the T key; T also works while the menu is open.
- Resetting a round (R, the button, the menu or the win card) always turns Tamper off.
- **The cannon collides with bricks.** It is now a rigid body in the brick physics: a 40×26 box
  that never rotates and weighs about six bricks. It shoves bricks when carried, can't pass
  through them, bricks can land on it, and it lands on them. Carrying it uses the same spring
  pull as bricks; the old hand-rolled cannon motion (`updateCannon`, `liftOutOfGround`) was
  removed.
- Digging in Tamper plays its own sound (dirt scrapes, a soft thump, soil pattering down) in
  place of the cannonball-impact blast. The crater and brick physics are unchanged: `blastAt`
  no longer plays sound, and each caller picks its own.
- Tests: T14/T20/T21 updated for the physics cannon, T17 checks Tamper is off after a reset,
  T27 (the carried cannon shoves bricks without passing through), T28 (a brick lands on the
  cannon).

## 0.9.0 — 2026-09-22
- The win card is clickable: clicking it starts a new round (R still works).
- New **ESC MENU** pill in the top-right, under Reset, styled like the other pills. It shows that
  Esc opens the pause menu, and clicking it opens the menu too.
- **Tamper-Tantrum Mode**: a gold toggle in the top-left. While it's on:
  - The cannon stops following the mouse. Keyboard aiming and Space still work.
  - The cursor becomes a cartoon hand with a dashed reach ring. Bricks in reach are outlined.
  - Drag the cannon, or up to 3 bricks within reach, anywhere. Held bricks are pulled toward the
    hand through the rigid-body solver, so they still shove other bricks. The cannon is never
    placed inside the ground.
  - On release, gravity takes over. A quick drag throws: the hand's velocity over the last
    100 ms (capped at 1800 u/s) carries over to the cannon or bricks.
  - Clicking bare ground digs a hole with exactly the same blast as a cannonball (`blastAt`,
    now shared by both).
  - Rounds won after tampering are marked ✋ in Past records.
- The cannon is now a free ballistic body (x/vx as well as y/vy): it can be thrown, lands on
  ground, stops at walls and ceilings, and stays inside the world. `updateCannonFall` became
  `updateCannon`.
- Self-tests T20–T26: carrying and dropping the cannon, throwing it, grabbing ≤ 3 bricks,
  throw velocity, digging, the drag-velocity maths, and the canvas button hit areas.

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
