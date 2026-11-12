# physics notes

Reference for the orbital sandbox. Read this when working on physics, stability, or new scenarios. **Update it when you discover a new constraint, validate a new scenario, or hit a non-obvious bug worth remembering** - this file is supposed to grow with the project, not freeze.

## engine

Single-file HTML, vanilla JS + Canvas 2D, no dependencies. All state lives in a `bodies[]` array.

**v17: velocity Verlet (leapfrog KDK) integrator.** Replaced the v16 symplectic Euler with second-order velocity Verlet. Both are symplectic (the v16 notes incorrectly claimed the old integrator was "not symplectic" - symplectic Euler IS symplectic, just first-order). The Verlet upgrade gives 50-800x better energy conservation depending on the scenario, measured over 60-300s of sim time. This is the single biggest stability improvement in the project's history.

Constants:

- `G = 0.5` (default) - gravitational constant. **Now mutable via live slider** (range ~0.05 to 5.0, logarithmic). Changing G invalidates the Verlet acceleration cache.
- `SOFT = 6` - softening length, prevents close-encounter blowups. Slightly underestimates gravity, which causes a mild radial "breathing" in long-running circular orbits (~5-7% over 60s). Intrinsic to the engine, not a bug.
- `STEP_DT = 0.5` - integrator step. Comfortable for typical orbits; marginal for very tight ones. Below ~150 integrator steps per orbital period, the orbit visibly degrades.
- `CULL_RADIUS = 5000` - v17: bodies past this COM-frame distance are automatically removed. Prevents ejected bodies from silently perturbing the system from far away.
- Time speed compounds integrator error proportionally. 5x speed = ~5x error per second of wall time. This is why trojans look fine at 1x and spin out at 5x+.

**Verlet implementation notes.** The KDK (kick-drift-kick) form keeps positions and velocities synchronised, which matters for orbit prediction, circularize, and rendering. Accelerations are stored between frames and reused as the first half-kick of the next step. Cache is invalidated when bodies are added, removed, merged, culled, or when G changes. Cost: ~1 force evaluation per step in steady state (same as old Euler), 2 evals when cache is cold.

## stability rules of thumb

These come up over and over. Apply at scenario design time, not as runtime fixes.

**Hill radius** - how close a satellite can orbit a planet before the parent star strips it away:

```
R_H = R_orbit * cbrt(m_planet / (3 * M_star))
```

Stable satellite orbits sit inside ~0.3-0.4 of R_H. Past 0.5 they get stripped within minutes of sim time.

**Mutual Hill radius** - minimum separation between adjacent moons of the same planet:

```
R_H_mut = (a1 + a2)/2 * cbrt((m1 + m2) / (3 * M_planet))
required separation > 2*sqrt(3) * R_H_mut  ≈ 3.46 mutual Hill radii
```

This is what bit v12 moons - adjacent moons sat at 2.79 to 2.88 mutual Hill radii, well below the stability threshold, and merged within ~50s.

**Holman-Wiegert** - circumbinary orbit must sit at > 2.4x the binary separation. The binary scenario uses ~5.5x for safety.

**Routh L4/L5 stability** - mass ratio for stable Lagrange trojans:

```
µ = m_planet / (M_star + m_planet) < 0.0385
```

Necessary but not sufficient. L4/L5 are *neutrally* stable - linearly stable but with no restoring force. Real solar system trojans persist because reality is symplectic; ours aren't, fully. Even at safe µ, our trojans drift out eventually.

**Two-body proper circular velocity** - when the planet is heavy enough to actually move the sun:

```
v_planet = sqrt(G * M_sun^2 / (M_total * R))
```

Reduces to `sqrt(G*M/R)` when `m_planet << M_sun`. Use the proper form whenever `m_planet / M_sun > ~1%` or the orbit will be slightly elliptical instead of circular.

## gotchas

**COM drift (the big one).** A "fixed" sun at velocity 0 with heavy planets orbiting it violates momentum conservation. The COM should be stationary, but instead the sun sits still while planets carry momentum in one direction, so the entire COM slides. For a 14% mass ratio (v12 moons), the system drifted ~1000 units in 120 seconds. The fix is to give the sun a counter-velocity:

```js
sun.vx = -(sum of m_i * v_i_x for all planets) / M_sun
sun.vy = -(sum of m_i * v_i_y for all planets) / M_sun
```

Affects: solar (4% Jupiter), moons (14%, fixed in v14), trojans (1.4%, fixed in v16). Negligible for binary and three-body (already symmetric setups), lagrange/fixed pair (pinned), rings and dust (tiny perturbers). Even small ratios like trojans' 1.4% matter when the scenario relies on a clean rotating frame (see trojan instability below).

**Test contamination by COM drift.** When validating a scenario in node, escape detection in the origin frame mistakes COM drift for body escape. Use COM-frame distance instead, recomputing COM each check. The whole "8 escapes per seed" panic in v13 was almost entirely the system sliding sideways while bodies orbited normally.

**Trojans need co-rotation, not local Keplerian velocity.** An asteroid placed at `r = R + δ` with `v = sqrt(G*M/r)` has its own orbital period and falls out of phase with the planet within one orbit, breaking the libration. Use the two-body angular velocity around the COM:

```
omega = sqrt(G * M_total / R^3)          // NOT M_star alone
v_trojan = omega * r_from_COM             // tangent to COM, prograde
```

**Trojan initial spread controls stability more than mass ratio.** The v15 trojans used ±10° angular spread and R=260, which gave 160/200 trials unstable at 60 wall-seconds. The integrator can't preserve neutral stability at L4/L5 - it slowly pumps energy into the libration. Wider initial spread = larger libration amplitude = faster escape. v16 uses ±2.3° spread at R=500, giving 0/200 at 60s. The three levers in order of impact: (1) angular spread around the exact L4/L5 point, (2) orbit radius (larger R = slower orbits = fewer orbits per wall-minute), (3) mass ratio µ (marginal effect alone).

**Tight orbits degrade.** With STEP_DT fixed, an orbit at small radius gets few steps per period. d=13 around an 80-mass planet is ~93 steps/orbit and will visibly drift. Either push the orbit outward, increase the parent's mass, or accept the drift.

**Pinning a body that should be moving makes things worse.** Pinning the sun in trojans breaks the rotating frame the trojans need - tested in v13 and confirmed it increased escapes. Pinning is the right tool when a body is meant to be stationary in inertial space (lagrange/fixed pair), wrong when it "should be" orbiting.

## validated scenarios as of v24

- **solar** - sun=3500, planet masses 1 to 18, COM-corrected. Clean over 90s × 6 seeds.
- **binary** - 5.5x Holman-Wiegert margin on circumbinary planets. Stable.
- **three (figure-8)** - canonical Chenciner-Montgomery initial conditions, mass=1400. Periodic.
- **rings** - heavy central body + light particles, no mutual gravity issues.
- **dust** - light particles around a moderate proto-star, no mutual gravity issues at typical N.
- **trojans** - v16 rebuild: COM-corrected barycentric setup, R=500, µ=0.014, ±2.3° angular spread. 0/200 at 60s/1x, 13/200 at 90s/1x, 5/200 at 60s/1.5x. Degrades past 3x as expected from integrator drift.
- **moons** - sun=2500, M1=300 R1=220 with one moon, M2=300 R2=520 with three moons, COM-corrected. Mostly clean over 120s × 10 seeds; inner moon occasionally drifts after 2-4 minutes.
- **lagrange / fixed pair** - pinned stars, no drift on the heavy bodies.
- **trisolaris** - v31, planet fixed in v32: three near-equal suns (m≈650, rad=5) from a sub-rigidly-rotating equilateral triangle (s=380, spin=0.96) + one solid world (m=50, rad=6) on a wide orbit (r=680), auto-fit on load. Chaotic by design; no stability target. Validated headless **with Roche shatter on** (the v31 validator omitted it - see v32 notes): planet survives 13/16 over 150s, a sun merges in only 1/16, planet swings 5u (scorching) to ~3600u (frozen). The world is mass-50 so it's below the Roche ratio against a single sun (650/50=13 < 20) and sears instead of vaporizing on a close pass.
- **empty** - obviously fine.

## tooling and conventions

Single-file HTML. Bump version on every material change: copy `v<N>.html` → `v<N+1>.html`, edit, syntax-check (`new Function(scriptContent)` in node), test physics in node where relevant. The current good test pattern is in `/tmp/` scripts during build sessions: build a clone of the scenario in JS, run the same step function as the game, check merges and COM-frame escapes across multiple seeds.

Recent version history:

- v9 - SOFT raised 4 → 6 to dampen close encounters
- v10 - merge flashes, deletion pops, orbit prediction overlay
- v11 - pin system (fixed flag on bodies)
- v12 - trojans, figure-8, moons scenarios added
- v13 - trojan co-rotation fix (was using local Keplerian)
- v14 - solar and moons rebuilt with COM correction; mass ratios reduced
- v15 - circularize mode (snap any body to circular orbit around its dominant gravitational parent)
- v16 - trojan scenario rebuilt: COM-corrected barycentric setup, R=500, pM=35, ±2.3° spread (was R=260, pM=60, ±10°). Fixes catastrophic trojan instability past ~1 orbit.
- v17 - **stability overhaul**: velocity Verlet integrator (50-800x better energy conservation), auto-cull escapees at 5000 COM-frame radius, live G slider (log scale 0.05-5.0), velocity vector arrows toggle, mobile UI moved to bottom + auto-collapse
- v18 - **Roche tidal disruption**: bodies fragment when tidal forces from a massive neighbor exceed self-gravity (fluid Roche limit, 20:1 mass ratio threshold, 120-frame cooldown, 0.5 min mass). Keyboard shortcuts (space/d/p/c/v/t/f/r). Tidal toggle in physics section.
- v19 - **Mobile UI overhaul**: persistent bottom toolbar (type cycle, pause, pin/circ/delete, drawer expand), mini stats overlay, drawer auto-dismiss on canvas tap, haptic on merge+shatter, safe-area-inset for notched phones. Desktop unchanged.
- v22 - **Body info tooltip**: hover over any body on desktop to see type, mass, speed, orbital radius to parent. Hidden on mobile (no hover). **Save/load state**: S to save, L to load. Persists to localStorage. Full snapshot: all bodies, cam position, G, time scale, ejection count. **Galaxy collision scenario**: two opposing dust clouds with central stars, counter-rotating disks.
- v23 - **Galaxy trajectory fix**: v22 stars merged on first contact (offset too small). Redesigned as wide flyby: offset 200 (was 80), infall 0.18 (was 0.6), tangential 0.38 (was 0.15). Closest approach 103.5 units, 7x clear of merge. Stars now do 2-3 tidal-stripping passes before eventual merger via dynamical friction.
- v24 - **Elastic bounce mode**: CoR slider (0=merge, 1=perfect elastic). Bounce uses sum-of-radii contact + impulse with coefficient of restitution + overlap separation. Original merge path (center-overlap + speed > 0.8) preserved at CoR=0. Orange flash on bounce. **Thrust mode**: click+drag a body to apply continuous force toward cursor. Screen-space displacement controls acceleration (mass-independent). Dead zone 10px, cap 300px, strength 0.0004/px/dt. Pulsing orange ring + arrow visual. Keyboard 'h'.
- v25 - **Real 8-planet solar system** (real eccentricities, COM-corrected, validated). CoR slider removed. orbitalElements() ported in. Verdict-colored plunger trace (cyan=bound, red=escape).
- v26 - **Trail fade** (6-chunk quadratic alpha). Fixed-pair rebuilt as slingshot playground. Dust cloud rebuilt as real accretion. Pinned-satellite Stark gotcha documented.
- v27 - **Orbit lines** (osculating ellipses) replace velocity arrows toggle. Solar name labels removed.
- v28 - **Orbit rendering cleanup**: dashed → sparse dots on orbit lines (dash-crawl fix), plunger/tap traces → solid. Plunger trace draws full osculating ellipse for bound orbits (decoupled from pull distance).
- v29 - **Orbit lines: setLineDash replaced with drawOscDots** (dots locked to orbital geometry, not canvas path - kills shimmer for real). **gravParent()** filters orbit lines to first-level satellites. **orbitalVelocity** fixed to sqrt(G*(M+m)/r) so charged bodies circularize.
- v30 - **UI redesign only, physics untouched** ("tracking console"). Intent-colored command deck, browsable experiments rail, console bottom-sheet (closed by default on all sizes), unified dock mode-track, zoom-fit button. No engine changes - see v30 section below.
- v31 - **trisolaris scenario** (chaotic three-body + a world, the Liu Cixin setup) replaces galaxy collision. **Experiments rail moved into the console**, taking the spawn-type group's slot (the dock chip already cycles type with the console closed). First validated physics change since v29 - see v31 section.
- v32 - **trisolaris planet fix**: the v31 world (m=4) Roche-shattered on its first close sun pass because the v31 validator didn't model Roche. Reworked as a solid mass-50 world (shatter-immune to a single sun), widened to r=680, with auto-fit-on-load so it's framed on a phone. Validator now includes Roche shatter. See v32 section.

## ~~v22/v23 galaxy collision scenario~~ (removed in v31, replaced by trisolaris)

*Galaxy collision was cut in v31 - it read as visual mush (two dust blobs smearing together) rather than a legible gravitational story. Kept here for reference; the trisolaris design notes are in the v31 section below.*

**Design.** Two disk galaxies separated by 900 units, each with a central star (m=2000) and 130 dust particles in a disk (r=35-180). The disks spin in opposite directions (one prograde, one retrograde) so when they interpenetrate the relative velocities are high - produces longer tidal tails and more ejections than same-direction spirals.

**v23 trajectory fix.** v22 had the galaxies nearly head-on (offset=80, approach=0.6) which caused the stars to merge on first contact. v23 redesigns the approach as a wide flyby:
- Vertical offset = 200 units (was 80)
- Infall velocity = 0.18 (was 0.6)
- Tangential velocity = 0.38 (was 0.15)
- Two-body closest approach = 103.5 units (7x clear of merge threshold)
- Orbit is bound - stars loop back for second pass around step 5000

**Why multi-pass matters.** The fun of galaxy collisions is the tidal stripping, not the stellar merger. Each close pass rips dust from the opposing disk, forming the characteristic S-shaped tidal bridges. Dynamical friction from the dust bleeds kinetic energy, tightening each subsequent pass until eventual merger after 2-3 orbits. This mirrors real galaxy merger timescales (Toomre sequence).

**Not COM-corrected** on purpose. The two galaxies are symmetric in mass, so net COM velocity is already ~zero by construction. Any tiny drift from random dust positions is negligible.

## things to consider when adding scenarios

Run through this checklist before declaring a new scenario stable:

1. Is the heaviest planet's mass ratio against the central star above ~1%? If yes, use proper two-body velocity and apply COM correction.
2. Are there adjacent satellites of the same parent? Check mutual Hill separation > 3.46.
3. Are any satellites past ~0.4 R_H of their parent? Pull them in.
4. Are any orbits below ~150 integrator steps per period? Loosen them.
5. Does any "fixed" body have neighbors heavy enough to need its momentum balanced?
6. Is the test detecting escapes in COM frame, not origin frame?
7. Does the scenario have light bodies passing near massive ones? Then validate **with Roche shatter on** - it's a live default. Any body with a neighbor ≥20× heavier (ROCHE_MASS_RATIO) shatters when it comes within `d_Roche = R_small · cbrt(2 · M_big/m_small)`. A headless clone that omits this will greenlight scenarios that vaporize on first contact live (the v31 trisolaris bug). Either keep light bodies clear of `d_Roche`, or lift their mass above the ratio so they're exempt.

If you bend any of these rules, document why in this file.

## v17 new mechanics

**Auto-cull.** Bodies past 5000 units from the system COM (computed each physics step) are automatically removed with a visual pop effect. This prevents ejected bodies from perturbing the system from far away - previously, a slingshot ejection at v~10 would travel thousands of units but still exert gravity on the central system, slowly pumping energy in. The ejection counter in the stats panel tracks how many were culled.

**Live G slider.** G is adjustable from ~0.05 to ~5.0 on a logarithmic scale. This invalidates the Verlet acceleration cache on each change (recomputed next frame). Scenarios spawned while G is non-default will use the current G for orbital velocity calculations, so they'll be correctly initialised. Resetting G after spawning a scenario will break those orbits - intentionally, since that's the fun of the slider.

**Velocity arrows.** Toggleable display of velocity vectors on all bodies. Arrow length = speed × cam.zoom × 12, capped at 80 screen pixels. Useful for visualising Kepler's second law (speed at periapsis vs apoapsis), slingshot assists, and diagnosing why a body is drifting.

## v18 Roche tidal disruption

**The physics.** When tidal force from a massive body exceeds a smaller body's self-gravity, it fragments. The Roche limit (fluid body approximation):

```
d_Roche = R_small * cbrt(2 * M_big / m_small)
```

Only triggers when mass ratio exceeds 20:1 (ROCHE_MASS_RATIO). Below that, bodies are too similar in mass for tidal disruption to make physical sense - they'd merge instead.

**Typical Roche distances at current tuning:**
- planet (m=50, r=6) → star (m=2200): 27px
- planet → black hole (m=9000): 43px
- planet (m=18, r=10) → solar sun (m=3500): 73px
- asteroid (m=1, r=2) → star: 33px
- asteroid → planet (m=50): 9px

**Fragment mechanics.** Body shatters into 4-10 fragments (count = mass/2, clamped). Fragments:
- Spread along the tidal axis (radial line to the massive body)
- Inherit parent orbital velocity + tidal stretching along axis
- Have a 120-frame (2s) shatter cooldown to prevent immediate re-shattering
- Created as asteroids regardless of parent type
- Total mass conserved

**Cascade prevention.** Two mechanisms: (1) minimum mass threshold (ROCHE_MIN_MASS = 0.5) prevents tiny fragments from shattering further, (2) shatterAge cooldown of 120 frames gives fragments time to spread before becoming eligible.

**Interaction with other systems.** Fragments can merge (if merge is on). Fragments can be culled by auto-cull. Fragments inherit the parent's color for visual continuity. The shatter visual effect is an elongated burst aligned to the tidal axis.

**Keyboard shortcuts** (also v18): Space=pause, D=delete, P=pin, C=circularize, V=velocity, T=trails, F=follow, R=recenter, Escape=clear modes.

## v24 elastic bounce mode

**Two collision paths.** The CoR slider selects between:
- CoR = 0: original merge path. Center-overlap within min(radius) + relative speed > 0.8. Conserves mass and momentum via weighted merge.
- CoR > 0: elastic bounce. Sum-of-radii contact detection (surface touching). Impulse-based collision response with coefficient of restitution. Conserves momentum exactly; kinetic energy conservation proportional to CoR (1 = perfect elastic, energy-conserving).

**Impulse formula.** Standard 1D normal impulse: `j = -(1 + e) * v_n / (1/m_a + 1/m_c)` where `v_n` is the relative velocity along the collision normal and `e` is CoR. Fixed bodies have infinite effective mass (invMass = 0). Two fixed bodies skip collision entirely.

**Overlap separation.** After applying impulse, bodies are pushed apart proportional to inverse mass ratio to prevent persistent overlap. This is a position correction, not a force - it doesn't affect energy.

**Known limitation.** At very high speeds bodies can tunnel through each other in a single timestep (CCD not implemented). Not worth fixing for a toy - the SOFT parameter and reasonable dt keep this rare.

**Interaction with G slider.** Negative G + high CoR = expanding bouncing gas simulation. Low G + CoR near 1 = gravity billiards. High G + CoR = 0 = normal accretion. The design space is surprisingly rich.

## v24 thrust mode

**Design.** Click a body, drag to apply continuous acceleration toward the cursor. Force direction and magnitude update every frame while held. Mass-independent acceleration (all bodies respond equally to the same screen gesture). This is deliberate - makes it feel like "pushing" rather than "propelling."

**Tuning.** Strength = 0.0004 per screen-pixel per timestep. At max drag (300px) with dt=0.5: delta-v = 0.06 units/tick = ~3.6 units/sec at 60fps. That's enough to noticeably bend an orbit over a few seconds without being instant. Dead zone = 10px to prevent jitter when trying to hold position.

**Energy implications.** Thrust injects energy into the system. This is fine - it's the whole point. But it means thrust + high CoR can create runaway situations. Also fine - that's fun.

## solar_kepler_v1 - the kepler edition (separate app)

A focused fork, not v25 of the sandbox. Single-file recreation of the real solar system with three interactive Kepler lessons layered on top. Shares the engine (Verlet KDK, G=0.5, SOFT=6, dt=0.5) and the v24 interaction grammar (tap-to-place with auto orbital velocity, hold-to-charge, screen-space plunger, frozen-bodies orbit prediction, touch guard). Versioned independently: solar_kepler_v1.html, v2, ...

**Validated configuration.** Sun m=4000. Planets at real eccentricities, compressed radii:

| planet | a | e | mass | steps/period |
|---|---|---|---|---|
| mercury | 80 | 0.206 | 0.8 | 200 |
| venus | 115 | 0.007 | 1.5 | 350 |
| earth | 160 | 0.017 | 1.5 | 575 |
| mars | 215 | 0.093 | 1.0 | 883 |
| jupiter | 330 | 0.049 | 6.0 | 1681 |
| saturn | 470 | 0.056 | 4.0 | 2877 |
| uranus | 650 | 0.046 | 3.0 | ~4400 |
| neptune | 880 | 0.009 | 3.0 | ~6800 |

Headless validation, 6 seeds x 90 wall-seconds: 0 escapes, COM drift < 2 units, max |da/a| < 2%, max |de| < 0.04 for every planet.

**Mutual Hill bug caught at design time.** First draft put Uranus/Neptune at 620/780 with m=4 each: separation = 2.6 mutual Hill radii (threshold 3.46), and Uranus showed |da/a| up to 10% in validation - the v12 moons failure mode again. Fix: widen to 650/880, lighten the giants (jupiter 8->6, saturn 6->4, uranus/neptune 4->3). All adjacent pairs now > 3.5 mutual Hill radii. The checklist works; run it.

**Osculating elements from state vectors** - the teaching core, computed live every frame:

```
mu = G * (M_primary + m)
E  = v_rel^2/2 - mu/r            // < 0 = bound
a  = -mu / (2E)
h  = rx*vy - ry*vx               // 2D specific angular momentum
e_vec = (v x h)/mu - r_hat       // in 2D: ((vy*h)/mu - rx/r, (-vx*h)/mu - ry/r)
e  = |e_vec|,  omega = atan2(e_vec)   // periapsis direction
T  = 2*pi*sqrt(a^3/mu)
```

Validated round-trip in node: place a body at periapsis with known (a, e, omega), recover all three to 4 decimals. Hyperbolic case (E >= 0) correctly reports e > 1, unbound. Ellipse render: center = focus - a*e along omega, semi-minor = a*sqrt(1-e^2), second focus mirrored through center.

**The three lessons:**
- K1: selected body's full osculating ellipse with sun-at-focus cross, empty second focus, peri/apo markers, live a/e/T readout. Mercury (e=0.206) is the default selection because the stretch is visible.
- K2: area sweep. Each physics step accumulates dA = 0.5*|r x dr| (relative to sun) into the current wedge; wedge closes every T/14 of sim time and the last 6 are drawn as alternating filled sectors anchored to the live sun position (points stored as sun-relative offsets so the wedges ride with the sun). Readout lists per-wedge areas - they come out equal, which is the lesson.
- K3: log-log chart of (a, T) for every body bound to the sun, against the theory line of slope 3/2. New dropped bodies land on the line within ~1 frame of being placed - the "it has no choice" moment.

**Plunger as a Kepler teacher.** During drag, the would-be state vector is fed through the same element math: predicted path is cyan + "ellipse e=..." when bound, red + "escape (hyperbola)" when E >= 0. Cheap, and quietly teaches the bound/unbound energy threshold.

**Input rule: tap-on-body selects, tap-empty places.** Hit radius max(r*zoom, 7px) + pad. On touch, dragging from a body pans instead (a moving planet under your finger is hard to re-tap otherwise). Selection survives merges (selected/sun references handed to the surviving body).

**Engine notes specific to this app:** no G slider (stability is the product), merge-only collisions (CoR path omitted), CULL_RADIUS raised to 8000 so long-period plunger comets aren't eaten at apoapsis, single merge resolved per step.

## v25 - real solar system + makeover

- **Solar scenario rebuilt** from the kepler edition's validated config: sun m=4000, 8 named planets at real eccentricities (mercury e=0.206 through neptune e=0.009), each starting at its own periapsis with vis-viva velocity, COM-corrected. Replaces the old 6-planet circular config whose jupiter (m=18) and saturn (m=12) ran perturbation-hot. Re-validated against the shipped table verbatim: 6 seeds x 90s, 0 escapes, worst |da/a| 2.7%, worst |de| 0.06.
- **CoR slider and elastic bounce removed.** Two collision paths doubled the surface area for one mechanic that rarely changed outcomes at typical speeds (sum-of-radii contact almost always co-occurred with merge-grade overlap). Merge-only path restored verbatim. If bounce comes back it should be a discrete mode, not a slider.
- **orbitalElements() ported into the sandbox** (same math as kepler edition, validated there). Powers two things: (1) the plunger trace is now verdict-colored - cyan dashed for a bound ellipse, red for escape, with a live "ellipse · e 0.42" / "escape · e 1.3" label at the cursor, computed relative to the heaviest body; (2) tooltips show a / e / T instead of just orbital radius.
- **Contrast pass on indicators**: trails 0.32->0.5 alpha at 1.5px, parent tether 0.25->0.6 at 1.6px, parent ring 0.35->0.75 at 2px, tap-mode trace 0.3->0.65 at 2px with longer dashes, plunger trace 2.2px. Panel borders 0.12->0.22.
- **Named bodies + saturn ring**: makeBody takes `name` and `saturnRing` overrides, both persisted through save/load. Labels render at zoom > 0.28 for bodies under mass 1000 (so a planet promoted to star drops its label).
- **Zoom to fit**: double-click or Z frames every body. Clears follow mode.
- Gotcha caught during the edit: zoomToFit referenced a `syncFollowBtn()` that doesn't exist - follow button state in the sandbox is toggled inline via classList, not through a sync function. Cleared the class directly.

## v26 - trail fade + scenario triage

**Trails fade to nothing.** Drawn in 6 chunks of quadratically rising alpha (oldest end invisible, newest at peak). Peak alpha is mass-gated: 0.5 for mass >= 5, 0.26 below, so particle scenarios don't wash out. Quadratic ramp reads better than linear - the tail dies faster than the head brightens.

**The pinned-satellite gotcha (the big v26 lesson).** Satellites of a pinned star in a multi-star system are structurally doomed. In a real binary, a star and its satellites fall toward the companion together - the uniform field component cancels by the equivalence principle, leaving only weak (r/d)^3 tides. A pinned star refuses to fall, so its satellites feel the companion's FULL uniform acceleration relative to their anchor: a constant Stark-like perturbation of strength G*M_comp/d^2 against local gravity G*M_home/r^2. At the old fixed-pair geometry that's ~2%, which pumps eccentricity to ~0.15 within 3 orbits (~3 wall-seconds) - confirmed by tracking isolated locals with zero other perturbers. No ladder spacing, mass reduction, or radius choice survives it; every config tested (8+) lost bodies. This generalizes the existing rule: "pinning a body that should be moving makes things worse" now includes "pinning a body whose satellites should be falling with it."

Diagnosis trail worth remembering: the visible deaths were local-local merges and star plunges, which first looked like mutual-Hill spacing (fixed - still died), then steps-per-period violations (real! r=36 around m=2000 is only 86 steps/orbit, also fixed - still died), then wanderer strafing (also real - canned chaotic particles fall through satellite zones and score direct hits). All three were contributing, but the Stark pumping was the floor under everything.

**Fixed pair rebuilt as the slingshot playground.** Pinned pair + 36-asteroid circumbinary ring at 900-990 (far field = single 4000 point mass), nothing else. 0 losses over 90s x 4 seeds. Design principle applied: the spectacle is the structure, the user supplies the chaos - the static potential conserves energy, so plunger launches through the double well trace clean, repeatable curves, and the v25 verdict trace makes aiming them a game. Canned chaos (divers, axis oscillators through the saddle, rosettes) was tried and rejected: short-lived AND it strafes the stable populations.

**Dust cloud rebuilt as real accretion.** The old version claimed accretion but was circular orbits + 0.015 jitter - relative speeds never reached the 0.8 merge gate, making it a dimmer ringed giant. Now: core 600 -> 2000 (orbital v scales sqrt(M)), speed spread +-17.5%, radial kicks +-12.5% of v_circ. Crossing orbits collide above the gate: ~35 merges per 90s with a steady timeline (no instant collapse), biggest grain grows 3-5x from the seed mass. Rings (ordered, cold) and dust (hot, accreting) are now genuinely different scenarios instead of near-duplicates.

**Binary**: outer ring trimmed 50 -> 30 asteroids. The circumbinary planets at 440/620 sit at 2.75x/3.9x the binary separation (Holman-Wiegert threshold 2.4x) - the inner one is closer to the line than the old notes implied (the "5.5x" figure was measured against the 80-unit half-separation), but it holds in practice. Unchanged otherwise.

**Kept as-is**: solar (just rebuilt in v25), figure-8 (canonical), trojans + moons (validated, documented limits), ringed giant (the cold-disk counterpart to dust), galaxy (v23 multi-pass design), empty. Nothing recommended for deletion now that dust earns its slot - it was the cut candidate before the rework.

## v27 - orbit lines replace velocity arrows; solar labels removed

**Name labels gone.** The solar scenario's text labels (mercury, venus, ...) added clutter the project didn't want. Removed from render. `name` is still stored on bodies and round-trips through save/load - it's data, not decoration, and the tooltip can still surface it on hover if wanted later.

**Velocity arrows -> orbit lines.** Velocity vectors were marginal (the trail already shows direction, and speed is in the tooltip). Replaced the toggle with osculating orbit ellipses: for every bound body, draw the two-body ellipse it would trace around the dominant attractor (heaviestBody) if every other body vanished. Same orbitalElements() math as the v25 plunger verdict.

Geometry (validated in node, body-lies-on-ellipse residual = 1.0000 across e = 0.017 to 0.5): recover periapsis direction omega from the eccentricity vector, semi-minor b = a*sqrt(1-e^2), ellipse center = focus - a*e along omega. ~~Drawn dashed at 0.5 alpha in each body's color.~~ v29: `drawOscDots()` at fixed eccentric-anomaly positions (see v29 section). Skips the primary, fixed bodies, near-parabolic (e >= 0.985), and a > 6000. v29 also filters to first-level satellites only (gravParent = system primary).

The teaching point this unlocks: an orbit is not a fixed track, it's a readout of position+velocity *right now*. Toggle it on in a perturbed system (trojans, moons, a slingshot flyby) and the ellipses visibly breathe as bodies tug each other - the instantaneous orbit is always a clean conic even when the real path isn't. Pairs naturally with the K-laws framing from the kepler edition.

Toggle is the `orbits` button; keys `v` and `o` both fire it. (The `o` key was previously unbound; `v` kept for muscle memory.)

## v28 - orbit rendering cleanup

**The dash-crawl problem.** `setLineDash([5, 5])` on the orbit ellipses caused visible shimmer/strobing as bodies moved - the dash phase shifted every frame because the ellipse geometry changed. Same issue on the plunger trace. Affected both desktop and phone. Fix: orbit lines switched to sparse round dots (`setLineDash([1.5, 11])` + `lineCap: 'round'`). Tiny dots don't shimmer the way long dashes did because there's no leading edge to visually track. Plunger and tap-hold traces switched to solid. Dashes remain on UI indicators (pin ring, parent tether, crosshairs) where the effect is negligible.

**Plunger trace decoupled from pull distance.** The old plunger trace used a 300-step forward sim, meaning the predicted arc length scaled with launch velocity - a gentle toss barely showed a partial curve, while a hard pull covered the full orbit. You couldn't preview the shape of a soft orbit without pulling into escape territory. Fix: bound orbits now draw the full osculating ellipse directly (same geometry as orbit lines), using the launch state vector. The ellipse IS the predicted orbit regardless of speed. Escape trajectories (E >= 0) still use the forward sim since there's no closed curve. Relaxed cutoffs for the plunger vs orbit lines: maxE 0.998 (was 0.985), maxA 20000 (was 6000) - the plunger shows a single highlighted prediction, so large/eccentric ellipses are useful information rather than visual clutter.

**drawOscEllipse() helper extracted.** The osculating ellipse geometry (e-vector → omega → center → semi-axes → Canvas ellipse) was inlined in three places with near-identical code. Refactored into a shared helper that takes (x, y, vx, vy, mass, primary) and optional {maxE, maxA} cutoffs. Returns the orbital elements on success, null on skip. Used by orbit lines, tap-hold trace, and plunger trace.

**Tap-hold trace also upgraded.** Was a 300-step forward sim (showed partial arc of the auto-circular orbit). Now draws the full osculating ellipse - shows the complete circle you'd place into. Uses the nearest massive body as primary (matching orbitalVelocity), not the system-wide heaviest.

## v29 - drawOscDots, gravParent, orbitalVelocity fix

**setLineDash is fundamentally incompatible with moving orbital geometry.** v28's sparse dots (`setLineDash([1.5, 11])` + `lineCap: 'round'`) still shimmered because `setLineDash` ties the pattern phase to the canvas path parameterization, not the geometric shape. As the ellipse center/orientation shifts each frame, all dots slide along the curve. Long dashes made this obvious; short dots made it subtler but still visible. The conclusion: **any** `setLineDash` pattern on a moving ellipse will crawl. The fix is architectural, not tuning.

**drawOscDots() replaces setLineDash entirely for orbit lines.** Samples the ellipse at N equally-spaced angles in world space (around the primary) and draws filled `arc()` circles. Critical detail: dots are positioned at fixed θ_i = 2πi/N angles using the conic equation `r = p/(1 + e·cos(θ-ω))`, NOT at eccentric anomaly positions relative to periapsis. This matters because on circular orbits (e≈0), the periapsis angle omega is degenerate - numerical noise makes it jitter wildly frame to frame. With eccentric-anomaly sampling, all dots rotate with omega. With fixed-angle sampling, the e·cos term vanishes as e→0, so dots are rock-stable: validated at 0.000850 px max shift between frames on a circular orbit (was tens of pixels with eccentric-anomaly sampling). For eccentric orbits, omega is well-defined and stable anyway.

**gravParent() + first-level filter.** Orbit lines used `heaviestBody()` as the primary for all bodies, so a moon orbiting a planet showed its orbit relative to the distant star (wrong ellipse, wrong parent). New `gravParent(body)` finds the body exerting the strongest gravitational acceleration (max M/r²). Orbit lines now skip bodies whose gravParent isn't the system primary - only first-level satellites (planets) get orbit lines, not moons. Validated: moon at r=20 from planet (M=50) correctly picks planet (M/r²=0.125) over star at r=320 (M/r²=0.039).

**orbitalVelocity() fixed for charged mass.** Was `sqrt(G*M/r)` (test-particle approximation). Now `sqrt(G*(M+m)/r)` (relative circular velocity). Without the placed body's mass, a 20x-charged body was too slow by ~1% and fell inward, producing an eccentric orbit with lower periapsis. Validated: e=0.000000 across m=1, m=80, m=2000 against a 4000-mass star. The old formula gave e=0.019608 for m=80 - visible as a non-circular orbit that tightened with charge.

Note: this is "periapsis" in the general case. "Perigee" is specifically Earth-centric, "perihelion" is Sun-centric, but "periapsis" works for any central body.

## v30 - UI redesign ("tracking console"), physics untouched

**Scope: zero engine changes.** Velocity Verlet, G=0.5, SOFT=6, STEP_DT=0.5, merge-only collisions, Roche tidal, gravParent/drawOscDots/orbitalVelocity - all byte-identical to v29. Only the `<style>` block and the UI DOM were rebuilt. All canvas/physics/interaction JS untouched. The 2-layer button architecture (canonical buttons hold state + mutex logic; dock mirrors delegate via `.click()` and reflect via `syncMobileBar()`) was preserved deliberately - it's why the redesign didn't touch physics wiring.

**Identity: evolved, not replaced.** Kept the "observatory terminal" feel - Fraunces italic display, JetBrains Mono workhorse, deep-space dark, film grain. New layer is the **intent-colored command deck**: the dock + telemetry glow in the hue of the current mode. place=amber, pin=violet, circ=green, thrust=orange, delete=red; armed-scenario channel=cyan via the status pill. Driven by `body[data-mode]` setting `--intent` + `color-mix()` glow. (Known cosmetic mismatch: the *canvas* pin ring is still drawn amber in the untouched JS while the UI pin hue is violet - left as-is to avoid touching the render path. Easy one-line change in the draw code if it bugs.)

**New structural pieces:**
- `#rail` - experiments card rail above the dock. Horizontal scroll-snap; each scenario is a card with an inline-SVG schematic sigil + name + one-line teaser + `data-preset`. Replaces the old buried scenario list. All 10 scenarios unchanged (solar/binary/three/trojans/moons/rings/dust/lagrange/galaxy/empty).
- `#mobile-bar` (the dock) - now visible on **all** sizes (was mobile-only). Spawn chip (cycles type) · mode-track (active segment expands + reveals its label) · pause · console toggle.
- `#ui` console - bottom sheet, **closed by default everywhere** (was: desktop left-panel always open, mobile auto-collapse). Opens via dock `≡`, closes via `#collapse` or `#scrim` tap. Groups: spawn, physics, time, view, state, gestures.
- `#rack` - off-screen holder for canonical pin/circ/thrust/delete buttons so keyboard shortcuts + dock mirrors + `clearTransientModes()` keep real click targets.
- `#zoomfit` (NEW capability) - "zoom to fit" button in the view group; frames all bodies via `zoomToFit()`. Only genuinely new control; everything else is a re-presentation.

**Removed:** old left `#ui` panel, permanent `#mobile-g` G-strip (its `mb-g-*` refs are all guarded by `if`/`?.`, so removal is inert), standalone `#help` panel (folded into the console's gestures group).

**Validation:** main script passes `node --check`. Missing-ID audit returns only the three intentionally-removed `mb-g-*` refs (all guarded). No physics re-validation needed - the engine didn't move.

## v31 - trisolaris scenario + rail moved into console

**Engine still untouched.** Same Verlet/G/SOFT/STEP_DT/merge/tidal. This is a scenario + UI-placement change, not an integrator change. But it IS the first new *validated* scenario since the v25-era rebuilds, so it went through the headless harness.

**Trisolaris - the chaotic three-body from Liu Cixin's novel.** Three near-equal suns and a single world that endures Stable Eras and Chaotic Eras until it doesn't. This is the *generic* three-body case, deliberately NOT the figure-8 (which we keep as `three` - the rare periodic solution). Design:
- Three suns from an equilateral triangle, side `s=380`, launched tangentially at `spin=0.96` of the rigid-rotation rate. The rigid rate for equal masses is `v = sqrt(G·m/s)` (derived: centripetal `ω²R` = net inward grav `√3·G·m/s²` at `R=s/√3` gives `ω²=3Gm/s³`, so `v=ωR=sqrt(Gm/s)`). Launching sub-rigid (0.96) + a 4% mass spread + a 4% position perturbation seeds the chaos: the triangle holds for ~30s (the opening "Stable Era") then decays.
- One world, `m=4`, on a wide circular orbit (`r=560`, `v=sqrt(G·M_total/r)`) around the whole triple - starts orbiting the system as a point mass before the chaos reaches it.
- COM-corrected respecting launch offset/velocity (`b.vx -= px - vox`, `b.x -= cx - ox`), so an armed/slingshot spawn still lands clean and escape detection stays honest.

**The critical lever was star radius, not velocity.** The merge gate is `centers within min(radius) AND relSpeed > 0.8`. A compact chaotic triple has repeated deep close passes (separations collapsing to ~7-15u), so at the default star radius=14 (or even 10) the suns merge within ~15-40s - the dance collapses to a boring binary almost immediately. Dropping star **radius to 5** shrinks the merge cross-section enough that those near-passes whiff instead of sticking, and the trisolar dance survives. Compensated visually with `glow=34` so a radius-5 body still reads as a sun. This is the same "radius controls merge rate" insight as elsewhere, but here it's the whole ballgame.

**Validated** (headless clone of computeAccelerations + KDK + merge + cull, 8 seeds × 12000 steps ≈ 200s/1x):
- Suns survive the full run in 6/8 seeds; a sun-merge catastrophe in 2/8.
- Planet ejected in 2/8 (late, step 8000+); otherwise endures with wild swings.
- Planet-to-nearest-sun distance ranges **5u (scorching grazing pass) to ~5300u (frozen, near the cull edge)** - the Stable/Chaotic Era swing, literally.
- Star-star separation per window confirms real weaving, not near-rigid rotation: opens at 320-400u (gentle), then by ~50s swings 13u..1400u as one sun is flung onto a long arc while the other two pair up, then they trade. Classic resonant scattering.
- In-file code path re-run through the engine clone: 4 bodies (3 stars + 1 planet), COM ≈ (0,0), COM velocity ≈ 0, masses [624,650,676,4], no NaN over 3000 steps. ✓

Chaotic by design - **no stability target**, same disclaimer the galaxy scenario carried. Eventual catastrophe (a sun-merge or a planet ejection) is on-theme; re-arm for a fresh fate. This replaced the galaxy slot (net scenario count stays 10).

**Sigil:** three filled suns in a triangle, a faint chaotic loop through them, a small planet dot off to the corner.

**Rail moved into the console.** The experiments rail was eating a fixed strip above the dock. It now lives inside the console sheet, taking the slot the **spawn-type group** used to occupy. Rationale: the dock spawn chip (`#mb-type`) already cycles type with the console closed, so the in-console type swatches were redundant. The cards keep their horizontal scroll-snap format (unchanged styling, just relocated).
- **Architecture note:** `#mb-type` cycles `currentType` directly via `TYPE_ORDER`; the old `[data-type]` swatches were only an *alternate direct-pick* that reflected state. Removing them is safe - every `querySelectorAll('[data-type]')` loop simply no-ops on an empty set, and `syncMobileBar()` reads `currentType` not the swatches.
- **Tradeoff shipped:** you lose direct-pick (tap straight to "black hole"); spawn type is now cycle-only from the dock chip. Accepted per the design call - flag if it chafes in use.

## v32 - trisolaris planet fix (Roche shatter)

**The bug.** Live, the v31 world flew apart almost immediately. The v31 headless validator said 6/8 seeds kept the planet for 200s, so the validator was wrong about something. It was: **the validator never modeled Roche tidal shatter, which is a live default** (`tidal on`). A body shatters into fragments when it comes within `d_Roche = R_small · cbrt(2 · M_big/m_small)` of something at least `ROCHE_MASS_RATIO=20` times heavier. For the v31 world (m=4, R=5) against a sun (m≈650), `d_Roche = 5·cbrt(2·162) ≈ 34u`. The chaotic dance guarantees a sub-34u pass quickly, so the world detonated into asteroid fragments that sprayed off - read by the user as "flies away."

**Why the un-Roched validator missed it.** Without the shatter step, the clone just tracked a point mass swinging around; it never deleted the planet on a close pass. The fix to the *process* matters more than the fix to the scenario: the headless harness now includes the Roche pass (see `tri_roche.js` / `tri_fix.js`), and the checklist gained item 7.

**The fix.** Make the world a *solid planet* that's exempt from single-sun shatter: mass 50 gives ratio 650/50 = 13 < 20, so a scorching close pass sears it rather than vaporizing it - which is also more physically honest (Roche shatter is a rubble-pile behavior; a solid planet wouldn't tidally disrupt from a star at these distances). Re-validated **with Roche on**, 16 seeds: planet survives 13/16 over 150s, deaths are mid-run gravitational ejections (steps 3800-7300, i.e. 60-120s) not load-time detonations, a sun merges in only 1/16, planet still swings 5u (scorching) to ~3600u (frozen). Caveat: if two suns merge into a ~1300 body, 1300/50 = 26 > 20, so the world becomes shatter-vulnerable to the *merged* super-sun - rare and late, and on-theme (the world finally torn apart).

**Framing.** The world orbits at r=680 around a compact sun-knot, far wider than any other scenario, so at the default zoom-1 it spawned off the edge of a phone screen. `loadPreset('trisolaris')` now calls `zoomToFit()` after spawning so the whole system is framed on load. As the chaos expands the suns past the frame, the user re-fits or follows. (Only trisolaris gets the auto-fit; the other scenarios are tuned for zoom 1.)

**Lesson for the file:** validate new scenarios against the engine's *defaults*, not a stripped engine. Merge + cull were in the clone; Roche shatter wasn't, and it's the one that bit. If a scenario puts light bodies near heavy ones, the Roche pass is not optional in the harness.

## v33 - tatooine (replaces binary)

**Binary → Tatooine.** The old binary scenario (two stars + two planets + 30-particle dust ring) was cluttered and lacked a clear identity. Tatooine focuses it: two suns, one circumbinary world, the double-sunset experience. Same binary core (equal-mass stars, m=1500 each, sep=80, total separation=160), one planet at r=420 (2.625× the total separation, above Holman-Wiegert 2.4× threshold), mass 40, named "tatooine." Planet uses proper two-body circular velocity `sqrt(G·M_stars²/(M_total·r))` since m_planet/M_stars ≈ 1.3% (above the 1% threshold where test-particle approximation causes visible eccentricity). COM correction: stars absorb planet momentum equally.

**Roche check.** Planet mass=40 vs star mass=1500 gives ratio=37.5 > ROCHE_MASS_RATIO=20, so the planet IS Roche-vulnerable in principle. But `d_Roche = 6·cbrt(75) ≈ 25u` and the planet's closest approach to a star is ~303u (orbital radius 420 minus stellar offset 80, minus orbital wobble). Never triggers.

**Validated** (headless, Roche included per checklist item 7): 8 seeds × 7200 steps (3600 time units ≈ 120s at 1×). 8/8 planet alive, 8/8 both stars alive. Planet orbits at 383-423u from stellar COM - the ~4% distance oscillation is expected from the time-varying binary potential (the planet feels two point masses, not one, so it wobbles slightly each binary period). No merges, no shatters, no ejections across all seeds.
## v34 - rogue star (new slot #11)

**The galaxy replacement, done right.** Galaxy was a pretty mushball that lacked dynamics. Rogue star delivers the same core fantasy - a dramatic gravitational encounter - with clean physics: a settled system (sun + 4 planets at r=130/230/370/540) disrupted by a fast interloper star (mass 1200, radius 11) on a hyperbolic trajectory. The interloper starts at -900u, approaching at v=2.2, with a randomized impact parameter (180-400u) that determines encounter severity. `zoomToFit()` on load so the interloper is visible during approach.

**Hyperbolic confirmation.** v_esc at starting distance ≈ 2.16, interloper v=2.2 → v_inf² ≈ 0.17. Genuine flyby, not capture. Marginal enough that the encounter is lingering (the interloper spends ~20 wall-seconds in the interaction zone) rather than a zip-through.

**Roche is a feature here.** All four planets have mass ratios >20 against the interloper (ratios: 150, 48, 30, 100), so any planet that gets within 20-27u of the interloper shatters into debris. This happens naturally on close-pass loads and creates dramatic tidal disruption events - physically accurate (the planet is in the interloper's Roche lobe during the closest approach). Close passes (impact 180-250u) can scatter or shatter 3-4 planets; wider passes (350-400u) may only perturb the outer orbits.

**Validated** (headless, 12 seeds × 12000 steps ≈ 200s, Roche on): central sun survives 12/12. Rogue always exits (never merges with the central star - closest approach 43-231u vs merge radius ~11u). Planets surviving: 0-3 out of 4 depending on impact geometry. Roche shattering events in 8/12 seeds. Every load is a different fate, which is the design intent.

**No COM correction on the interloper.** The settled system (sun + planets) is COM-corrected internally. The interloper is an external perturbation that carries its own momentum. The system COM drifts during the encounter, which is physically correct. Follow-biggest tracks the central star (mass 3000).

## v35 - hierarchical triple (new slot #12)

**Three-body flavor #3.** Figure-8 is periodic, trisolaris is chaotic, hierarchical triple is the stable one. This is the configuration ~80% of real triple star systems use: a tight inner binary + a distant third circling both. The hierarchy is the key - the outer body sees the inner pair as effectively a single point mass, and the inner pair barely feels the outer body's tidal influence.

**Parameters.** Inner binary: m1=1100, m2=900 (slightly asymmetric for visual interest), total separation a=120 (d1=54, d2=66 from inner COM). Outer star: m3=800, r_outer=500 from inner binary COM. Triple COM frame: binary COM orbits at 142.9u from triple COM, third star at 357.1u. All masses below 1500 → trails on for all three stars (the inner pair draws spirograph loops, the outer traces a wide arc).

**Timescales.** T_inner ≈ 261 time units (17s at 1×) - fast enough to see the binary spin clearly. T_outer ≈ 1877 time units (125s at 1×) - slow and stately. Period ratio 7.2:1 (inner binary completes ~7 orbits per outer orbit). The visual hierarchy is the point: fast clockwork inside, slow sweep outside.

**Stability.** Outer/inner separation ratio = 4.2× (well above Holman-Wiegert 2.4×). Tidal force from outer on inner binary ≈ 0.00014% of binary self-gravity - negligible. No Roche risk (mass ratios 1.22, 1.375, 1.125 - all far below the threshold of 20).

**Validated** (headless, 8 seeds × 18000 steps ≈ 5 min at 1×): all 3 bodies survive 8/8. Inner binary separation 117-126u (2.5% wobble around nominal 120). Outer distance 487-500u (2.6% wobble around nominal 500). Energy conservation ΔE/E ≈ 10⁻⁷ to 10⁻⁶. The small oscillations are the expected mutual perturbation - physically correct, visually natural.

**`zoomToFit()` on load** - outer star at 357u from triple COM is off-screen on phone at zoom 1.

## v36 - kirkwood belt (replaces ringed giant)

**Rings → Kirkwood.** The old ringed giant (250 particles in a tight ring around a massive body) was pretty but static - no story, no dynamics. Kirkwood replaces it with a star + Jupiter + asteroid belt showing the structure that mean-motion resonances carve.

**Pre-carved gaps.** Real Kirkwood gap formation takes millions of orbits (secular eccentricity pumping at resonance radii). At sandbox timescales, even an inflated Jupiter (13% of star mass) doesn't produce clean gaps within minutes - it just scatters the entire outer belt. Tested empirically: gap formation failed at Jupiter masses from 3.3% to 13.3% over 500-1000 wall-seconds. The gaps are therefore pre-carved at initialization at the 3:1 (r≈264) and 2:1 (r≈346) resonance radii with Jupiter at r=550. This is the mature belt after resonance depletion has run its course.

**Parameters.** Star mass=3000, Jupiter mass=80 (2.7% of star, planet type), r=550, color #e8a54a. Belt: 120 asteroids from r=180 to r=380, with ±10u gaps at the two strongest resonance radii. Resonance formula: `r_res = r_J × (p/q)^(2/3)` where p:q is the asteroid:Jupiter period ratio. COM correction: star absorbs Jupiter's orbital momentum.

**Stability.** Jupiter's Hill sphere inner edge at ~437u, belt outer edge at 380u - safely outside. Gravitational perturbation at outer belt ≈ 13% of star gravity (moderate). At inner belt ≈ 1% (negligible). Validated headless (8 seeds × 12000 steps ≈ 400s): 99-100/100 asteroids survive. Jupiter orbit stable at r=543-556 (nominal 550). Some asteroids drift radially (max ~600-900%), adding visual dynamism - outer-belt asteroids slowly feel Jupiter's influence.

**The teaching moment.** The scenario's interactive value: arm it and launch a test particle into a gap → watch it get scattered by the resonance. Launch into the stable zone → peaceful orbit. That contrast IS the resonance lesson, even though the gap formation itself is pre-computed.

**`zoomToFit()` on load** - Jupiter at r=550 is off-screen on phone at zoom 1.

## v37 - dust layer (massless tracers, DustSim-inspired)

**What.** Roche shatters and merges now spray dust: up to `DUST_MAX = 30000` grains in typed arrays (`dX/dY/dVX/dVY`), separate from `bodies[]`. Grains feel gravity, exert none, never merge with each other. Cost is O(grains × pullers), not O(n²). Measured headless: 30k grains, stepDust 0.6-0.8ms, drawDust ~4ms per frame (server CPU, not phone).

**Physics.** Symplectic Euler (kick then drift), same `G`, `SOFT`, `STEP_DT`, called at the end of `physicsStep()`. Removed when inside a puller's radius (swallowed) or past `CULL_RADIUS` from body COM. Full buffer overwrites random grains. Pullers = bodies with mass >= `DUST_PULL_MIN` (5) and past shatter cooldown.

**Emission.** Shatter: `800 × cbrt(mass)` grains (cap 4000) at the body, its velocity ± 0.25. Merge: `600 × cbrt(small.mass)` (cap 4000) on big's surface at the impact side, big's velocity + half the impactor's relative velocity ± 0.3 × relSpeed.

**Gotchas paid for.**
- Merge ejecta spawned at the impact point sits inside big's radius - swallowed the same step, zero visible dust.
- Fresh shatter fragments (mass ≥5) sit inside the spray; letting them swallow it killed over half the grains on step one. Hence the cooldown exclusion.
- Shatter cascades (fragments re-shattering) fill the 30k cap in ~15s from one grazing planet.

**Render.** 1-css-px grains written to an ImageData buffer, drawn with `lighter` compositing between trails and bodies. Color = speed relative to smoothed mean grain speed (red slow, green at mean, blue fast) so every scene spans the ramp. Dust clears with `bodies` on preset load, clear and state restore; it is not serialized.

**Dust placing (v38).** `dust` is a 5th spawn type on the chip (`TYPES.dust`, mass 0, never becomes a body). It places like any body: tap = cloud in orbit, drag = plunger slingshot, hold = charge packs in more grains (`mult`, cap 8000). *(v42: dust hold now streams instead - see v42.)* Diameter 8-240 screen px from a slider shown only while dust is selected. Density 0.4 grains per screen px², spread ± 0.03. A paint-brush version was tried first and dropped: strokes came out as a chain of round blobs, and placing like the other types felt more natural.

## v38 - tap-to-orbit host fix (`findOrbitHost`)

**Bug.** Tap-to-orbit used the *nearest* body heavier than the placed one, then added that body's velocity. In multi-star systems, a tap outside the stars inherited one star's fast swing around the others and got flung away.

**Fix.** `findOrbitHost(wx, wy, minMass)` replaces `findNearestMassive`. The primary is the body with the strongest pull (M/r²). Any body with ≥5% of the group mass that sits closer to the group's COM than the tap point (< 0.7 r) joins the group, repeating until nothing changes. A single-body host returns the body itself, so `orbitalVelocity()` still inherits a planet's velocity for moons. A multi-body host is a pseudo-body: COM position, COM velocity, total mass, radius 0.

**Measured** (200 single grains placed with the app's own logic, 1500 steps):

| placed 250-450u from COM | flung >1200u, old → new | lost, old → new |
|---|---|---|
| three | 68 → 7 | 75 → 19 |
| tatooine | 98 → 22 | 50 → 25 |

solar (80-380u): flung 63 → 0. Taps among the stars (80-200u) stay chaotic under both rules. That's real three-body physics, not the bug.

## v39 - dust coalescence (planet building)

Dust still never pulls bodies. It joins them instead. Console toggle `dust: off / snap / gravity` (`dustMode`, default `grav`).

**Grain mass.** `GRAIN_M = 0.01` is notional. It only becomes real when grains join a body. Main tuning knob: 100 grains = mass 1 = `DUST_PULL_MIN`, the smallest body dust can form. `DUST_PULL_MIN` dropped 5 → 1 so planetesimals pull and accrete dust. This makes shatter cascades with many fragments cost more (about 1.3ms/step).

**Collapse test** (every 10 steps, 20u grid, cell mass M, half-size h):
- tidal: `M ≥ 3 h³ Σ(M_b / d³)`. The clump's Hill radius must beat the cell size. This is the Roche shatter math in reverse.
- cold: grain speed variance `< G M / h`. The grains must not outrun the clump's escape speed.
- Pass → the grains become an asteroid (color `#c9b48a`, radius `2·cbrt(M)`) at the cell COM, with mean velocity. The strongest-pull host is COM-corrected by the new body's relative momentum.

**Gravity mode** adds coarse dust-on-dust pull: neighbour cells within ±2 (COM to COM, softening h) plus each grain's pull toward its own cell COM. That's O(grains + cells·25) per step, about 2ms/step at 20k grains.

**Accretion.** Swallowed grains add mass and momentum to the swallowing body (radius scales with cbrt of mass), but only if it's under `ACCRETE_MAX = 200`.

**Gotchas paid for.**
- Without a maturity gate, a tapped cloud is a dense, cold clump and turned to rock on the very first check (step 10). `DUST_MATURE = 300` steps: only grains that old count toward collapse. Tracked by `dBorn[]` plus `dustStep`, carried through every swap-remove.
- Unlimited accretion let a star swallow a shatter cascade and grow 2200 → 13400. Hence `ACCRETE_MAX`.

**Measured** (60s = 3600 steps, star 2200):

| setup | snap | gravity |
|---|---|---|
| smooth 20k disk r 40-260 | 0 born | 48 born (from step 300), merged up to a 204-mass planet |
| 12 tapped 48px clouds | 15 born, max 4.3 | 9 born, max 29.7 |
| grazing planet shatter | 0 born | 7 born (from step 1550) |

Gravity mode collapses the outer disk first: tidal shear falls as 1/d³.

## v40 - softer dust onset

Startup read as too aggressive: square grid lanes within seconds, and bodies popping in at 5s.
- Lanes were a grid artifact, not timing: every grain was pulled to the same fixed cell centres each step. Fix: grid origin shifts by a random sub-cell offset on every bin, so cell edges never line up twice.
- `DUST_RAMP = 600` steps: dust-on-dust pull scales 0→1 with grain age.
- `DUST_MATURE` 300 → 600 (~10s).

Measured (60s, gravity mode unless noted):

| setup | before | after |
|---|---|---|
| 20k disk, first birth | 5s, 55 born | 10s, 27 born, max 286 |
| 240px tap, 150u from star | 4 born, max 36.5 | 0 born - sheared into a halo first |
| 240px tap, 300u (45s) | 10 born, max 223 | 6 born, max 25 |
| 240px tap, 600u (45s) | 27 born | 47 born, max 227 |
| 240px tap, no star (45s) | 1 born, 414 | 1 born, 359 |

Gotcha: a longer delay hands the win to tides - big clouds tapped right beside a star now shred before they can collapse. Physically fair, but that's the cost of a calmer start.

## v41 - dust that orbits, weightless spray, dust prefabs

**Tapped dust: one orbit per grain.** A tapped cloud used to get one shared velocity (the orbital velocity of its centre). Away from the centre that is the wrong speed and direction, so the cloud smeared into crossing eccentric streams: tails flung outward near a star, and dust too hot to pass the cold test. v40's 10s maturity gave it time to smear, which killed coalescence in real use (v40 was only tested on a synthetic cold disk - lesson: test on the tap flow, sim running between taps). `emitDust(..., host)` now gives each grain the softened circular velocity about the host. Slingshot (drag) dust still gets one shared velocity - that's the throw.

Also: `findOrbitHost` takes `soft2`; dust passes its cloud radius^2 so a dust-born pebble under the tap can't hijack a cloud far wider than the pebble's Hill sphere.

| measure | before | after |
|---|---|---|
| median eccentricity, 240px tap 150u from star | 0.54 | 0.01 |
| p90 eccentricity, 48px tap, 40 pebbles around | 0.72 | 0.02 |
| bodies in 60s after filling pool by 48px taps (sim running, gravity mode) | 6-7 | 15-65 |

Body taps were never affected (planet/asteroid taps: e = 0 at 30/60/200u).

**Collision spray is weightless.** Merge and shatter spray grains counted as GRAIN_M each, so every merge minted mass: a 2-pebble merge emitted 600 grains = 6 mass from nothing. A 199-mass disk reached 1296 in 2 minutes; the v39/v40 tables ("204-mass planet", "shatter: 7 born") were inflated by this, and it is what ACCRETE_MAX was papering over. Spray now gets `dBorn = SPRAY` (2^30, "born in the future"): never binned, so it can't collapse, pull or be pulled by dust, and it doesn't accrete. Looks identical. Consequence: shatter debris no longer re-forms bodies - only placed/prefab dust has mass. Disk budget over 4 min: 2400 -> 2392, 0 gained.

**Prefabs** (validated in the real engine via their buttons, Roche live, 5 runs each, gravity mode):
- `dust` "dust disk" - star + 20k grains, r <= 260. Replaces the v26 disk of 200 asteroids (it was a stand-in for exactly this). First pebbles 10.0-10.2s; ~35 bodies at 30s; biggest 60-106 at 2 min. Snap mode: nothing forms (too thin without dust-on-dust pull).
- `dustcloud` "dust cloud" - star + one 6k-grain cloud (r 70) at 220u. Shears into an arc; 17-22 pebbles at 30s merging to 9-13 by 4 min, biggest 10-31. Mass 60.0 throughout.

Gotchas:
- A starless spinning cloud does not hold together: dust-on-dust pull only reaches +-2 grid cells (~50u), so a 150u cloud can't bind itself and flies apart. Dust needs a body to orbit.
- Late in the dust disk, body count can jump to 100+: grown planets Roche-shatter passing pebbles. Expected.

## v42 - dust hold-to-stream

**Gesture.** Dust only. Tap = one cloud (unchanged). Press and move >= `MIN_DRAG` within `STREAM_DELAY = 250` ms = fling (unchanged). Press and stay still 250ms = stream starts, then moving paints. Lift stops it. Dust hold-to-charge is gone: the difference was hard to see, and charge needed hold-still, which stream now owns. `getChargeMultiplier()` returns 1 for dust. Bodies still charge.

**Emission.** `updateDustStream(now)` runs each frame from `loop()`. Rate = `STREAM_CLOUDS = 2` brush-sized clouds per second (48px brush = ~1450 grains/s). Frame dt is clamped to 50ms so a stalled tab can't dump a burst. Each grain lands at a random point on the segment the finger moved this frame, so a fast stroke paints a band instead of the v38 chain of blobs. Grains get per-grain circular orbits about `findOrbitHost(..., r^2)` like a tapped cloud, and they carry real mass (`dBorn = dustStep`, not spray). It keeps streaming while paused. At `DUST_MAX` it overwrites random grains, which drops their mass (existing behavior).

**Validated** (headless Chromium, real touch events, solar running, 448x998): tap 724 grains; fast fling 724 grains with shared velocity, never streamed; hold 1s = 0 grains at 150ms, 1080 at 1s; painted ring around the sun = 5718 grains as a continuous band; +20s running, total mass 4079.60 -> 4079.60 (grains moving into bodies, none minted); planet hold 0.9s still charges (mass 50).
