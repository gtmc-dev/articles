---
translates: ./07-ChunkHolder生命周期.zh.md
translated-from-revision: 127319632f93b2099a9a13ceba705dd67e5f1952
title: ChunkHolder Lifecycle
description: How three core Futures control the three operational views of a chunk, how tick() warms up and cools down, and the complete cycle from ProtoChunk to WorldChunk to unloading.
---

If the ticket system is the "brain" (deciding what to do) and TACS is the "arm" (coordinating departments), then `ChunkHolder` is the "thermometer" — it measures and responds to a chunk's current "heat," then precisely controls which operational state the chunk is in.

## ChunkHolder's Responsibilities

We've mentioned `ChunkHolder` many times in previous chapters. It is the **internal representation** (or container) of a chunk within the chunk management system. A separate `Holder` object is needed because:

- Chunks at loading range boundaries **may not exist in memory**, but storing their load level is still necessary;
- A chunk's load level and generation state **should be invisible to external code** — they are internal management information;
- The availability of chunks under different operational views **is managed uniformly by Holder**, avoiding confusion about "is this chunk usable right now?";
- Facilitates atomic operations on individual chunks like setting load level, triggering loading, and scheduling unloading.

## Three Core Futures

`ChunkHolder` uses three `CompletableFuture` objects to manage a chunk's three "operational views":

```java
// ChunkHolder.java
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> accessibleFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> tickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> entityTickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
```

| Future                | Meaning                             | When completed                                 | Corresponding ChunkLevelType |
| --------------------- | ----------------------------------- | ---------------------------------------------- | ---------------------------- |
| `accessibleFuture`    | Chunk data available for read/write | level ≤ 33 and chunk loaded                    | `FULL`                       |
| `tickingFuture`       | Block ticking can execute           | level ≤ 32 and surrounding chunks reached FULL | `BLOCK_TICKING`              |
| `entityTickingFuture` | Entity ticking can execute          | level ≤ 31 and chunk entered ticking           | `ENTITY_TICKING`             |

The initial value `UNLOADED_WORLD_CHUNK_FUTURE` is an **already completed** Future with value `Either.right(Unloaded.INSTANCE)` — indicating "unavailable." Any code waiting on this Future immediately gets an "unavailable" result.

> [!IMPORTANT]
> The three Futures are **progressively tiered**: when `entityTickingFuture` completes, `tickingFuture` and `accessibleFuture` must already be complete. But the reverse is not true — a chunk can be "accessible" but not "ticking," or "ticking" but not "entity ticking."

## tick(): Warming Up and Cooling Down

`ChunkHolder.tick()` is what actually makes a chunk "come alive" or "cool down." It's called by `ChunkTicketManager.tick()` — only `ChunkHolder` objects with changed load levels are collected into the `chunkHolders` set and trigger `tick()`.

The core logic of `tick()` is to compare **the ChunkLevelType from the last tick** with **the current ChunkLevelType**, then execute warming or cooling operations based on the level difference:

```java
// ChunkHolder.tick() simplified logic
protected void tick(ThreadedAnvilChunkStorage chunkStorage, Executor executor) {
    ChunkLevelType oldType = ChunkLevels.getType(this.lastTickLevel);
    ChunkLevelType newType = ChunkLevels.getType(this.level);

    // Warm up: from inaccessible → FULL
    if (!oldType.isAfter(FULL) && newType.isAfter(FULL)) {
        this.accessibleFuture = chunkStorage.makeChunkAccessible(this);
    }
    // Cool down: from FULL → inaccessible
    if (oldType.isAfter(FULL) && !newType.isAfter(FULL)) {
        this.accessibleFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    // Warm up: from FULL → BLOCK_TICKING
    if (!oldType.isAfter(BLOCK_TICKING) && newType.isAfter(BLOCK_TICKING)) {
        this.tickingFuture = chunkStorage.makeChunkTickable(this);
    }
    // Cool down
    if (oldType.isAfter(BLOCK_TICKING) && !newType.isAfter(BLOCK_TICKING)) {
        this.tickingFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    // Warm up: from BLOCK_TICKING → ENTITY_TICKING
    if (!oldType.isAfter(ENTITY_TICKING) && newType.isAfter(ENTITY_TICKING)) {
        this.entityTickingFuture = chunkStorage.makeChunkEntitiesTickable(this);
    }
    // Cool down
    if (oldType.isAfter(ENTITY_TICKING) && !newType.isAfter(ENTITY_TICKING)) {
        this.entityTickingFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    this.lastTickLevel = this.level;
}
```

**Warming up** means making the chunk available — creating an incomplete Future and waiting for the generation or loading process to complete it.

**Cooling down** means immediately completing the corresponding Future with `UNLOADED_WORLD_CHUNK` — any code waiting on this Future immediately receives an "unavailable" signal and stops accessing the chunk.

## makeChunkAccessible: From ProtoChunk to WorldChunk

`makeChunkAccessible(holder)` is responsible for converting a chunk from "not yet loaded or generated" to "data accessible."

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkAccessible(ChunkHolder holder) {
    ChunkPos pos = holder.getPos();
    CompletableFuture<Either<List<Chunk>, ChunkHolder.Unloaded>> regionFuture =
        this.getRegion(holder, 1, distance -> ChunkStatus.FULL);

    return regionFuture.thenApplyAsync(either -> {
        return either.flatMap(chunks -> {
            WorldChunk worldChunk = (WorldChunk)chunks.get(chunks.size() / 2);
            worldChunk.runPostProcessing();
            this.world.startTickingChunk(worldChunk);
            return Either.left(worldChunk);
        });
    }, this.mainExecutor);
}
```

Key operations:

1. **getRegion(holder, 1, ...)**: Wait for a 3×3 chunk region centered on `holder` to all reach `ChunkStatus.FULL`.
2. **runPostProcessing()**: Execute delayed post-processing (like updating fence connections).
3. **startTickingChunk()**: Add the chunk to the world's ticking list.
4. **Return WorldChunk**: Complete the `accessibleFuture` with the completed `WorldChunk`.

Now this chunk's data can be read and written, `World.getChunk(pos)` returns a valid `WorldChunk`, and block entity data and scheduled ticks are all accessible.

## makeChunkTickable: Enabling Block Ticking

`makeChunkTickable(holder)` prepares the chunk to execute block ticking:

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkTickable(ChunkHolder holder) {
    return this.getRegion(holder, 1, distance -> ChunkStatus.FULL)
        .thenApplyAsync(either -> {
            return either.mapLeft(chunks -> {
                WorldChunk worldChunk = (WorldChunk)chunks.get(chunks.size() / 2);
                worldChunk.runPostProcessing();
                return worldChunk;
            });
        }, this.mainExecutor);
}
```

Wait for a 3×3 region, then run post-processing. After this completes, `tickingFuture` is resolved, the chunk can execute:

- Scheduled ticks (Scheduled Tick)
- Random ticks (Random Tick)
- Block entity ticks (Block Entity Tick)

## makeChunkEntitiesTickable: Enabling Entity Ticking

`makeChunkEntitiesTickable(holder)` prepares entity ticking:

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkEntitiesTickable(ChunkHolder holder) {
    return this.getRegion(holder, 2, distance -> ChunkStatus.FULL)
        .thenApplyAsync(either -> {
            return either.mapLeft(chunks -> (WorldChunk)chunks.get(chunks.size() / 2));
        }, this.mainExecutor);
}
```

Unlike `makeChunkTickable`, this waits for a **5×5 region** — entities need more context for movement, pathfinding, and collision detection.

## Common Lifecycle Paths

### Cold Start: From Nothing to Entity Ticking

1. **Add PLAYER ticket** → level=22 (assuming simulation distance 10)
2. **ChunkHolder.tick() triggered** → `oldType=INACCESSIBLE`, `newType=ENTITY_TICKING`
3. **All three Futures created simultaneously**:
   - `accessibleFuture = makeChunkAccessible(this)`
   - `tickingFuture = makeChunkTickable(this)`
   - `entityTickingFuture = makeChunkEntitiesTickable(this)`
4. **Generation chain starts** → `ChunkStatus.EMPTY` → ... → `ChunkStatus.FULL`
5. **3×3 region ready** → `accessibleFuture` completes
6. **3×3 region ready** → `tickingFuture` completes
7. **5×5 region ready** → `entityTickingFuture` completes
8. **Chunk becomes entity ticking**

### Gradual Cooling: From Entity Ticking to Unload

1. **Remove PLAYER ticket** → level rises
2. **ChunkHolder.tick() triggered** → level crosses 31 boundary
3. **entityTickingFuture.complete(UNLOADED_WORLD_CHUNK)** → entity ticking stops
4. **Level continues rising** → crosses 32 boundary
5. **tickingFuture.complete(UNLOADED_WORLD_CHUNK)** → block ticking stops
6. **Level crosses 33** → `accessibleFuture.complete(UNLOADED_WORLD_CHUNK)` → chunk marked for unload
7. **ChunkTicketManager collects unloadable chunks** → scheduled for saving
8. **TACS.save()** → writes to region file
9. **ChunkHolder removed from TACS** → memory freed

## :advanced Code Walkthrough

### ChunkHolder.tick() Complete Logic

```java
protected void tick(ThreadedAnvilChunkStorage chunkStorage, Executor executor) {
    ChunkStatus chunkStatus = ChunkHolder.getTargetGenerationStatus(this.level);
    ChunkStatus chunkStatus2 = ChunkHolder.getTargetGenerationStatus(this.lastTickLevel);
    boolean bl = ChunkLevels.isLoaded(this.level);
    boolean bl2 = ChunkLevels.isLoaded(this.lastTickLevel);

    ChunkLevelType chunkLevelType = ChunkLevels.getType(this.level);
    ChunkLevelType chunkLevelType2 = ChunkLevels.getType(this.lastTickLevel);

    // Generate if needed
    if (!chunkStatus.isAtLeast(chunkStatus2)) {
        this.combinedChunkFuture = chunkStorage.generate(this, chunkStatus);
    }

    // Warm up: inaccessible → FULL
    if (!chunkLevelType2.isAfter(ChunkLevelType.FULL) && chunkLevelType.isAfter(ChunkLevelType.FULL)) {
        this.accessibleFuture = chunkStorage.makeChunkAccessible(this);
        this.tickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
        this.entityTickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
        this.updateFutures(chunkStorage, this.accessibleFuture);
    }

    // Cool down: FULL → inaccessible
    if (chunkLevelType2.isAfter(ChunkLevelType.FULL) && !chunkLevelType.isAfter(ChunkLevelType.FULL)) {
        CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> future = this.accessibleFuture;
        this.accessibleFuture = UNLOADED_WORLD_CHUNK_FUTURE;
        this.updateFutures(chunkStorage,
            future.thenApply(either -> either.ifLeft(chunkStorage::onChunkStatusChange)));
    }

    // Warm up: FULL → BLOCK_TICKING
    if (!chunkLevelType2.isAfter(ChunkLevelType.BLOCK_TICKING) && chunkLevelType.isAfter(ChunkLevelType.BLOCK_TICKING)) {
        this.tickingFuture = chunkStorage.makeChunkTickable(this);
        this.entityTickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
        this.updateFutures(chunkStorage, this.tickingFuture);
    }

    // Cool down: BLOCK_TICKING → FULL
    if (chunkLevelType2.isAfter(ChunkLevelType.BLOCK_TICKING) && !chunkLevelType.isAfter(ChunkLevelType.BLOCK_TICKING)) {
        this.tickingFuture.complete(UNLOADED_WORLD_CHUNK);
        this.tickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
    }

    // Warm up: BLOCK_TICKING → ENTITY_TICKING
    if (!chunkLevelType2.isAfter(ChunkLevelType.ENTITY_TICKING) && chunkLevelType.isAfter(ChunkLevelType.ENTITY_TICKING)) {
        if (this.entityTickingFuture != UNLOADED_WORLD_CHUNK_FUTURE) {
            throw new IllegalStateException("Unexpected entity ticking future");
        }
        this.entityTickingFuture = chunkStorage.makeChunkEntitiesTickable(this);
        this.updateFutures(chunkStorage, this.entityTickingFuture);
    }

    // Cool down: ENTITY_TICKING → BLOCK_TICKING
    if (chunkLevelType2.isAfter(ChunkLevelType.ENTITY_TICKING) && !chunkLevelType.isAfter(ChunkLevelType.ENTITY_TICKING)) {
        this.entityTickingFuture.complete(UNLOADED_WORLD_CHUNK);
        this.entityTickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
    }

    // Schedule unload if no longer loaded
    if (bl2 && !bl) {
        this.sendPacketToPlayersWatching(new ChunkDataS2CPacket(this.getWorldChunk(), this.world.getLightingProvider(), null, null));
        this.scheduledForUnload = true;
    }

    this.lastTickLevel = this.level;
    this.levelsInRange = ChunkLevels.isOutOfRange(this.level);
}
```

### getRegion: Why Surrounding Chunks Are Needed

```java
public CompletableFuture<Either<List<Chunk>, ChunkHolder.Unloaded>> getRegion(
    ChunkHolder holder, int margin, IntFunction<ChunkStatus> distanceToStatus
) {
    List<CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>>> list = Lists.newArrayList();
    ChunkPos centerPos = holder.getPos();
    int centerX = centerPos.x;
    int centerZ = centerPos.z;

    // Collect chunks in (2*margin+1)×(2*margin+1) range
    for (int dx = -margin; dx <= margin; dx++) {
        for (int dz = -margin; dz <= margin; dz++) {
            int distance = Math.max(Math.abs(dx), Math.abs(dz));
            ChunkPos pos = new ChunkPos(centerX + dx, centerZ + dz);
            ChunkHolder neighborHolder = this.getCurrentChunkHolder(pos.toLong());

            if (neighborHolder == null) {
                return UNLOADED_CHUNKS_FUTURE;
            }

            ChunkStatus requiredStatus = distanceToStatus.apply(distance);
            CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> chunkFuture =
                neighborHolder.createFuture(requiredStatus, this);
            list.add(chunkFuture);
        }
    }

    return Util.combineSafe(list).thenApply(eithers -> {
        // If any chunk unavailable, whole region unavailable
        List<Chunk> chunks = Lists.newArrayList();
        for (Either<Chunk, ChunkHolder.Unloaded> either : eithers) {
            if (either.right().isPresent()) {
                return Either.right(ChunkHolder.Unloaded.INSTANCE);
            }
            chunks.add(either.left().get());
        }
        return Either.left(chunks);
    });
}
```

Key design:

- **All-or-nothing**: If any chunk in the region is unavailable, the entire region is considered unavailable.
- **Different margins for different needs**: `makeChunkTickable` uses margin=1 (3×3), `makeChunkEntitiesTickable` uses margin=2 (5×5).
- **Distance-based status requirement**: Center chunks need higher status (FULL), farther chunks may only need lower status.

### makeChunkTickable's margin=1: Why Block Ticking Needs 3×3

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkTickable(ChunkHolder chunk) {
    CompletableFuture<Either<List<Chunk>, ChunkHolder.Unloaded>> regionFuture =
        this.getRegion(chunk, 1, distance -> ChunkStatus.FULL);

    return regionFuture.thenApplyAsync(either -> {
        return either.mapLeft(chunks -> {
            WorldChunk worldChunk = (WorldChunk)chunks.get(chunks.size() / 2);
            worldChunk.runPostProcessing();
            return worldChunk;
        });
    }, this.mainThreadExecutor);
}
```

`getRegion(holder, 1, distance -> ChunkStatus.FULL)` collects all chunks within 1 Chebyshev distance from holder (9 chunks in 3×3), waiting for all to reach `FULL`.

**Why are surrounding chunks needed?** Making a chunk enter ticking means it starts executing block ticks and random ticks — these operations need to query block states in adjacent chunks (e.g., redstone signal propagation, fluid flow, piston push/pull checks). If adjacent chunks are still generating (still `ProtoChunk`), query results may be incomplete — causing redstone device behavior anomalies. Requiring all surrounding chunks to reach FULL ensures tick operation predictability.

Note the `chunk.runPostProcessing()` line: during generation, some operations (like updating fence connection states, adjusting redstone wire shapes) can't execute immediately at stage 10 (SPAWN) because adjacent chunks may not yet exist. These operations are deferred to `postProcessingLists`, executed centrally in `makeChunkTickable` — at this point chunks in the 3×3 range have all completed generation, giving these delayed operations complete context.

### makeChunkEntitiesTickable's margin=2: Why Entities Need Larger Range

Unlike `makeChunkTickable`, `makeChunkEntitiesTickable` prepares for entity ticking and requires a larger range:

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkEntitiesTickable(ChunkHolder chunk) {
    return this.getRegion(chunk, 2, distance -> ChunkStatus.FULL)
        .thenApplyAsync(either -> either.mapLeft(chunks -> (WorldChunk)chunks.get(chunks.size() / 2)), this.mainThreadExecutor);
}
```

`getRegion(holder, 2, distance -> ChunkStatus.FULL)` collects all chunks within 2 Chebyshev distance from holder (25 chunks in 5×5), waiting for all to reach `FULL`.

**Why do entities need 5×5 instead of 3×3?** Entity ticking involves three aspects:

1. **Collision detection**: When entities move, their bounding boxes may span multiple chunks. If an entity is near a chunk boundary, its collision detection needs to query block bounding boxes and entity lists in adjacent chunks. The 5×5 range ensures all chunks needed for collision detection are ready even when entities move in corners.

2. **AI pathfinding**: Entity pathfinding systems evaluate paths across multiple chunk ranges. If chunks along the path aren't loaded, pathfinding may fail or produce dead ends. The 5×5 range provides sufficient buffer for most entities (including players, mobs, item drops) pathfinding.

3. **Mob spawning checks**: Hostile mob spawning needs to check player distance, light levels, and block types within a 5×5 chunk range. If your spawning range only covers 3×3, spawning checks near edges will fail — lacking distance checks for more distant players.

Therefore, `makeChunkEntitiesTickable`'s margin=2 is a **safety margin**, providing complete context for all cross-chunk queries in entity ticking, avoiding entity behavior anomalies.

## References

- [Discovering Minecraft - ChunkHolder](https://github.com/lovexyn0827/Discovering-Minecraft) (CC0 License)
- `net.minecraft.server.world.ChunkHolder`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTicketManager`
