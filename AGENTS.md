# AGENTS.md

Vanilla JS Asteroids clone rendered on HTML5 canvas. No dependencies, bundler, package.json, tests, lint, or build step — do not look for or create them.

## Run / verify

- Open `index.html` in a browser, or `npx serve .` (default `http://localhost:3000`).
- Verification is manual: play the game and confirm behavior in the browser. There is no automated test suite.

## Layout

- `index.html` — page shell only; loads `game.js` as a classic script (not a module). Canvas is fixed 800x600; the size is mirrored in game.js as `W`/`H`.
- `game.js` — entire game: input, utils, `Bullet`/`Asteroid`/`Ship`/`Particle` classes, update/draw, `requestAnimationFrame` loop.

## Conventions / gotchas

- Language: README, code comments, and all in-game UI text are in Spanish. Keep it that way when adding user-facing text.
- Fixed-timestep-style loop caps `dt` at 0.05s (`game.js:415`); use the passed `dt` for all physics, never assume a fixed frame rate.
- Space is toroidal: positions wrap via `wrap()` (`game.js:27`); collision/`dist()` does not account for edge wrapping.
- Input uses two maps: `keys` (held) and `justPressed` (edge-triggered, consumed by `pressed()`). Use `pressed()` for one-shot actions (shoot, restart), `keys` for continuous (rotate/thrust).
- Game state machine uses the `state` string: `'playing' | 'dead' | 'gameover'` (`game.js:241`). `initGame()`/`nextLevel()` reset all globals declared at `game.js:239`.
- Asteroid size is indexed 1..3 into parallel arrays `RADII`, `SPEEDS`, `POINTS` (`game.js:61`); size 0 is unused. `split()` decrements size and stops at 1.
- No semicolon-free style bikeshedding; file uses `'use strict'` and 2-space indent.
