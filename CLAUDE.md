# orbital sandbox

Single-file n-body gravity playground. Vanilla JS + Canvas 2D, no dependencies, mobile-first (Android-targeted). This is an ongoing exploration project, not a shipped product - lean drafts over polish.

## files

- `index.html` - the app. Everything lives in one file: markup, styles, physics, rendering, input handling. Named `index.html` directly so GitHub Pages serves it at root with no redirect needed.
- `physics_notes.md` - companion doc, your context for the physics math. Canonical record of engine behavior, scenario configs, stability math, and gotchas already paid for once.

No filename versioning - that convention existed to track iterations across chat sessions with no repo behind them. Git history is now the version record. Edit both files in place; commit is the checkpoint.

## commit discipline

If a commit touches physics, stability, or scenarios, `physics_notes.md` gets updated in the *same* commit, not a follow-up. This is the one convention that survives from the pre-repo days - the notes file drifting out of sync with the engine is worse than no notes file.

Commit messages should say what changed and why in plain language, e.g. `fix plunger drag velocity on trackpad input` - no need to reference a version number, `git log` already gives that.

## engine facts (read physics_notes.md for the full picture)

- Velocity Verlet (leapfrog KDK) integrator. `G` is a live-adjustable slider (default 0.5), `SOFT = 6`, `STEP_DT = 0.5`.
- `CULL_RADIUS = 5000` auto-removes escapees past this COM-frame distance.
- Roche shatter is live by default (`ROCHE_MASS_RATIO = 20`) - any new scenario validator that skips it will report false survival. This has bitten the project before (trisolaris, v32).
- Merge-only collisions. Fixed bodies always win merges; two fixed bodies can't merge.
- `bodies[]` is the single source of truth for simulation state.

## don't break

- Touch guard (`lastTouchTime`) - prevents double-spawn on mobile.
- Plunger velocity is computed from screen-space displacement, not world-space (v9 fix - world-space caused phantom velocity when camera was tracking).
- Pin mode and delete mode are mutually exclusive; Escape clears both.
- `orbitalVelocity()` inherits the parent body's velocity so moons of moving planets stay in orbit.
- COM-correct any central body that absorbs orbiting-body momentum, or the system drifts (see physics_notes.md "COM drift" gotcha).
- Headless validation before shipping any new scenario: Node.js clone of computeAccelerations + KDK Verlet + merge + cull + **Roche shatter**, run across multiple seeds. Skipping Roche in the validator is the most common way a "validated" scenario turns out broken live.

## git workflow

- Commit on demand only - don't auto-commit after every edit.
- Never push without being asked explicitly, even after a commit.
- When asked to commit: stage, write a real commit message (not "wip" or "fix"), fold in the physics_notes.md update per commit discipline above if applicable.
- Deploy is GitHub Pages watching this repo - a push to the default branch is live in under a minute. No build step; don't add one.

## style

- Terse commit messages and code comments. Sentence case, hyphens not em dashes.
- Short, reactable increments over large speculative changes - this project moves chunk by chunk.
- Use the `ponytail` skill for edits - lean drafts over polish means the smallest change that works, not the most thorough one.
