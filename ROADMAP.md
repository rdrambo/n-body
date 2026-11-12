# Roadmap

Parked ideas. Nothing here is decided.

## Graduate to a proper web app (parked 2026-11-12)

Dustin wants to grow this up. Questions to think through before building anything:

- **Installable.** Should it become a PWA (an installable web app with its own home-screen icon, offline support and true fullscreen)? It's the cheapest step and it fixes the camera-hole and browser-chrome problems at the same time.
- **One file vs modules.** `index.html` is about 3,500 lines. Splitting it means a build step, which the repo currently forbids. Where is the line?
- **Physics off the main thread.** Moving the sim into a Web Worker (a background thread) would keep touch input smooth at high dust counts.
- **Real tests.** Turn the headless scenario validator into a test suite that runs before every push.
- **Who is it for?** A toy for you, something to share, or something for teaching? The answer sets everything above.

## Claude's picks, if left to its own devices

1. **Share a system as a link.** Save/load only writes to this browser's storage. Putting the state in the URL means "look what I made" is one tap. That's the biggest lever for a toy like this.
2. **Rewind.** Keep the last ~30s of snapshots and add a scrub-back. Chaos experiments (trisolaris, rogue star) are more fun when you can replay the moment it broke, and it also undoes "clear field".
3. **Goals inside experiments.** For example, "keep a moon alive around tatooine for 5 minutes". It gives the sandbox a reason to come back to without turning it into a game.
4. **Design critique leftovers.** The sheet hides the sim, mode buttons are under 48px, and dim text is 2.55:1 contrast.
