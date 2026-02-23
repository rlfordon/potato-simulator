# Performance TODO

Remaining optimizations identified during the performance investigation.
Fixes #1 and #2 are done and pushed.

## Completed

- [x] **Fix splice-in-forEach bugs** — Replaced `forEach` + `splice(i,1)` with reverse `for` loops in `updateEnemies`, `updateAllies`, `updateParticles`, health drops, and food pickups. Also deferred pizza slice spawning to after the enemy loop. (correctness fix)
- [x] **Cache `Date.now()` per frame** — Added `frameNow` variable set once at the top of `gameLoop()`, replacing 40+ `Date.now()` calls per frame.

## Remaining

- [ ] **Reduce `Math.hypot` / `dist()` calls in hot loops** — `updateEnemies` and `updateAllies` both call `dist()` or `Math.hypot()` multiple times per entity per frame for the same pair. Cache the distance value once per entity and reuse it for AI decisions, attack range checks, etc.

- [ ] **Offscreen culling for draw calls** — Skip rendering entities, particles, and health drops that are outside the visible canvas. Currently everything is drawn every frame regardless of position.

- [ ] **Particle object pooling** — `spawnParticles` creates new objects every call and they get GC'd when removed. Use a pool (pre-allocated array) to recycle particle objects and reduce garbage collection pressure. Especially relevant during boss fights with heavy particle effects.

- [ ] **Reduce canvas state changes in `drawBackground`** — Several level backgrounds (especially levels 5-8) do many `ctx.save()`/`ctx.restore()`, gradient creation, and style changes per frame. Consider caching static background elements to an offscreen canvas and blitting once per frame.

- [ ] **Batch similar draw operations** — Group draws by fill/stroke style where possible (e.g., all particles of the same color) to reduce canvas state switches.
