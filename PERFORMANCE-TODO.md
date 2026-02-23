# Performance TODO

Remaining optimizations identified during the performance investigation.
Fixes #1, #2, and #3 are done and pushed.

## Completed

- [x] **Fix splice-in-forEach bugs** — Replaced `forEach` + `splice(i,1)` with reverse `for` loops in `updateEnemies`, `updateAllies`, `updateParticles`, health drops, and food pickups. Also deferred pizza slice spawning to after the enemy loop. (correctness fix)
- [x] **Cache `Date.now()` per frame** — Added `frameNow` variable set once at the top of `gameLoop()`, replacing 40+ `Date.now()` calls per frame.
- [x] **Reduce `Math.hypot` / `dist()` calls in hot loops** — In `updateEnemies`, reuse the already-computed `d` (player-to-enemy distance) for the boss AOE range check instead of calling `dist()` again. In `updateAllies`, reuse `nearDist` from the nearest-enemy search instead of recomputing `Math.hypot()`.

## Remaining

- [ ] **Offscreen culling for draw calls** — Skip rendering entities, particles, and health drops that are outside the visible canvas. Currently everything is drawn every frame regardless of position.

- [ ] **Particle object pooling** — `spawnParticles` creates new objects every call and they get GC'd when removed. Use a pool (pre-allocated array) to recycle particle objects and reduce garbage collection pressure. Especially relevant during boss fights with heavy particle effects.

- [ ] **Reduce canvas state changes in `drawBackground`** — Several level backgrounds (especially levels 5-8) do many `ctx.save()`/`ctx.restore()`, gradient creation, and style changes per frame. Consider caching static background elements to an offscreen canvas and blitting once per frame.

- [ ] **Batch similar draw operations** — Group draws by fill/stroke style where possible (e.g., all particles of the same color) to reduce canvas state switches.
