# AGENTS.md

Standing instructions for any AI coding agent working in this repository.
Claude Code does not read this file on its own; `CLAUDE.md` imports it with `@AGENTS.md`
(see `project_setup.md`). If you are an agent and you can read this, these rules apply to you.

## Project in one paragraph

This repo holds a single self-contained browser game, `cannon.html`: a cannon on the left
fires at a brick building on the right across a deformable 2D terrain. Everything (HTML, CSS,
JavaScript) lives in that one file and runs by double-clicking it. It is a graded lab submission
for CS 4500 / DS 4850 (AI Coding Lab, Fall 2026). The full specification is in `instructions.md`.
Read `instructions.md` before writing any code, and follow its milestone order.

## Files

| File | Purpose | Who edits it |
|---|---|---|
| `cannon.html` | The game. The only deliverable. | Agent |
| `instructions.md` | Feature spec, milestones, acceptance criteria | Human (agent may propose edits) |
| `AGENTS.md` | These rules | Human |
| `CLAUDE.md` | One-line import of this file, plus Claude-only notes | Human |
| `project_setup.md` | How to set the project up from scratch on a new machine | Agent keeps it current |
| `CHANGELOG.md` | One entry per version bump | Agent |

Do not create any other files (no `src/`, no build tooling, no `package.json`, no separate
`.js` or `.css` files) unless the human explicitly asks.

## Workflow rules

1. **Git safety check before every major update.** Before starting any milestone, refactor, or
   change touching more than ~30 lines, stop and ask: *"Have you committed your current work
   to git? Reply yes to continue."* Do not proceed until the human confirms. Do not run
   `git commit`, `git reset`, `git checkout -- .`, `git stash`, or any history-rewriting
   command yourself unless asked.
2. **Stop after each significant update for review.** Finish one milestone from
   `instructions.md`, then stop. Report: what changed, how to test it by hand (exact clicks and
   keys), the self-test results, and anything you were unsure about. Wait for approval before
   starting the next milestone.
3. **Plan before large changes.** For each milestone, first give a short plan (functions you
   will add or change, data structures, risks). Keep it under ~20 lines.
4. **Ask instead of guessing** when the spec is ambiguous or two parts of it conflict. Offer
   a recommended option so the human can answer with one word.
5. **Finish every session with commit-log bullets.** When the human says the session is
   done (or asks), give 3–7 high-level bullets suitable for a git commit message, starting
   with the version number, e.g. `v0.4.0: projectile physics and cratering`.

## Code rules

1. **Recommend libraries before building "common" functionality from scratch.** If a
   feature is normally done with a library (physics engine, tweening, noise), name the library,
   the trade-off, and how it would be loaded (a pinned CDN `<script>` tag keeps the one-file
   rule), then ask. The current default in `instructions.md` is plain Canvas 2D with no
   libraries; do not add one without a yes.
2. **Remove old and dead code.** When something is replaced, delete the old version. No
   commented-out blocks, no unused functions, no `_old` / `_v2` / `legacy` names.
3. **No compatibility shims.** Always call the updated code. When a function is replaced, update
   every call site in the file to use the new one in the same change.
4. **No fallback code.** If something is not really implemented, do not fake it with a
   placeholder, silent default, or `try { ... } catch {}` that hides the problem. Let it throw
   with a clear message (`throw new Error("Brick physics not implemented yet")`).
   Exception: input validation at the edges (clamping angle/power) is real behaviour, not a
   fallback.
5. **No persistence or migrations.** The game keeps no saved data. Do not use `localStorage`,
   cookies, or any storage, and do not write any data-migration logic.
6. **Bump the version with every fix or feature.** The file has a single
   `const VERSION = "0.x.y";` near the top, also shown in the page's bottom-left corner.
   - New milestone → bump minor (`0.3.0` → `0.4.0`)
   - Fix or tweak → bump patch (`0.4.0` → `0.4.1`)
   Add a matching line to `CHANGELOG.md` each time.
7. **Keep the file organized** in the section order given in `instructions.md` §3, with a
   banner comment per section (`// ===== TERRAIN =====`). Functions stay small; no function
   longer than ~60 lines.
8. **Modern, plain JavaScript**: `const`/`let`, strict mode (`"use strict";`), no globals
   besides a single `game` state object and the section-level constants. No TypeScript,
   no JSX, no build step.
9. **Comment the why, not the what.** Short comments where the math is non-obvious (crater
   hit test, sub-stepping, brick support). No comment on every line.

## Required file header

The very top of `cannon.html` must be this comment block (the human fills in the bracketed
values; never invent them, leave the placeholders if they are not yet filled in):

```html
<!--
  Cannon vs. Building — AI Coding Lab
  Name:   [Student Name]
  uID:    [u0000000]
  Date:   [YYYY-MM-DD]
  Class:  CS 4500 / DS 4850 — Fall 2026
  Agent:  Claude Code ([model name as shown by /model], [effort setting if any])
  Other AI used: [e.g. Claude in browser for the spec in instructions.md]
  How to run: open this file in any modern browser (Chrome, Firefox, Edge, Safari).
  Self-tests: open with #test at the end of the URL.
  Version: see VERSION constant below.
-->
```

## Testing rules

- The file contains built-in self-tests (spec in `instructions.md` §9) that run when the URL
  ends in `#test`. Add or update tests in the same change that adds or changes the behaviour.
- After every milestone, state which self-tests pass. If you cannot run a browser yourself,
  say so plainly and give the human the exact steps to run them; never claim a test passed
  that you did not see pass.
- If you can run commands, `node` may be used to syntax-check extracted script text, but
  the game itself must not depend on Node.

## Things never to do

- Split the game into multiple files.
- Load anything from the network without approval (fonts, images, scripts).
- Edit `AGENTS.md`, `CLAUDE.md`, or `instructions.md` without asking first.
- Touch files outside this repository folder.
- Fill in the student's name, uID, or date with made-up values.
