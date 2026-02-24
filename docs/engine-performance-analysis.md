# OpenMW engine FPS improvement analysis (city hotspots)

Scope: engine-side changes that improve FPS/frame-time in CPU-heavy scenes (large cities, many actors, dense static clutter), without renderer migration.

## 1) Actor update/combat logic has O(N^2) behavior

### Evidence
- `Actors::update` iterates actors and, in combat engagement windows, loops over **all other actors** for each actor (`for actor` + nested `for otherActor`).
- This can scale poorly in city scenes with many NPCs.

Relevant code:
- `apps/openmw/mwmechanics/actors.cpp` lines around 1597-1605.

### Improvement
- Introduce a spatial partition (uniform grid / hash / BVH) for nearby actor queries used by `engageCombat` and related awareness checks.
- Replace full `mActors` scans with neighborhood candidate lists.
- Cache per-frame neighbor sets keyed by active cell + grid bucket.

### Expected impact
- Major CPU reduction in dense actor scenes.
- Better frame-time stability when entering busy districts or markets.

---

## 2) LOS cache lookup is linear and globally locked

### Evidence
- `PhysicsTaskScheduler::getLineOfSight` does `std::find` over `mLOSCache` (vector), under `mLOSCacheMutex`.
- Heavy LOS callsites exist in AI/combat update paths.

Relevant code:
- `apps/openmw/mwphysics/mtphysics.cpp` lines around 681-696.
- Many LOS consumers in `apps/openmw/mwmechanics/actors.cpp` and `aicombat.cpp`.

### Improvement
- Replace vector+linear-search cache with hash map keyed by canonicalized actor-pair IDs.
- Use sharded locks or lock-free read path for cache hits.
- Batch refresh/update of only hot keys touched this frame.

### Expected impact
- Lower AI/physics overhead in scenes with many potential LOS checks.
- Reduced lock contention with multithreaded physics.

---

## 3) Object paging reference collection repeatedly scans cells and uses expensive containers

### Evidence
- `collectESM3References` / `collectESM4References` build `std::map<RefNum, PagedCellRef>` each chunk request.
- Per-ref checks include linear `std::find` against `cell->mMovedRefs`.
- `createChunk` then does repeated cache mutex acquisitions (`mSizeCacheMutex`, `mLODNameCacheMutex`, `mRefTrackerMutex`) per-reference.

Relevant code:
- `apps/openmw/mwrender/objectpaging.cpp` lines around 547-613, 615-632, 676-802.

### Improvement
- Use per-cell immutable snapshot/index of pageable references, updated on cell mutation events.
- Convert moved-ref membership checks to hash sets.
- Batch lock cache accesses (or use concurrent map) instead of lock-per-ref.
- Replace `std::map` with flatter/hash structure where ordering is not required.

### Expected impact
- Lower CPU spikes while paging objects in city traversal and camera turns.

---

## 4) Preload trigger scans active cells/doors/NPC transports every update

### Evidence
- `Scene::preloadCells` calls routines that iterate active cells and door/transport content frequently.
- Door preload: collects teleport doors by scanning all active cells and doors.
- Fast-travel preload: scans NPC/creature transport lists in active cells.

Relevant code:
- `apps/openmw/mwworld/scene.cpp` lines around 1136-1164, 1169-1181, 1341-1347.

### Improvement
- Maintain incremental trigger indices (nearby teleport doors, nearby transport providers) updated on activation/deactivation.
- Add hysteresis and cooldown to prevent repeated scheduling of same destinations.
- Prioritize by camera velocity vector and predicted heading.

### Expected impact
- Reduced main-thread overhead in dense cells; fewer stutters near city entrances and travel hubs.

---

## 5) View reuse search in terrain view cache is linear over active views

### Evidence
- `ViewDataMap::getViewData` linearly scans `mUsedViews` to find the most suitable reusable view.
- This occurs on cull traversal path.

Relevant code:
- `components/terrain/viewdata.cpp` lines around 149-160.

### Improvement
- Add spatial index (ring buffer by quantized camera position or small k-d tree) for candidate view reuse.
- Keep fast-path cache for last successful reusable view per camera.

### Expected impact
- Smaller but measurable CPU wins with distant terrain + multiple camera contexts.

---

## 6) Build a dedicated perf regression harness for city scenarios

### Evidence
- OpenMW already has frame stats capture and analysis tooling and benchmark targets, but no city-path regression suite.

Relevant code/docs:
- `scripts/HOWTO-benchmark.md`
- `apps/benchmarks/*`

### Improvement
- Add scripted benchmark scenarios (camera pan in Balmora market, Vivec canton bridge traversal, interior market crowd).
- Track p50/p95/p99 frame time and subsystem timings in CI gates.

### Expected impact
- Makes optimization work repeatable and prevents regressions.

---

## Priority order for implementation

1. Actor neighborhood query system (replace O(N^2) combat neighbor scans)
2. LOS cache redesign (hash + lower lock contention)
3. Object paging reference/index and lock batching
4. Preload trigger indexing/hysteresis
5. Terrain view reuse lookup optimization
6. CI city perf harness and gates

These are the most likely to produce noticeable FPS and frame-time gains in “bad spots” without drastic architectural changes.
