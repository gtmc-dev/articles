---
translates: ./02-生成状态与ChunkStatus.zh.md
translated-from-revision: fa9abfa0c6a083d3e2dbbb8565313207b2287cd7
title: Generation State and ChunkStatus
description: 12-stage chunk generation pipeline, outer ring dependencies, taskMargin and Chebyshev distance, lighting and save upgrade bypass.
---

We already know chunks have two forms: `ProtoChunk` and `WorldChunk`. But how does a `ProtoChunk` evolve from nothing, step by step, into a `WorldChunk`?

The answer lies in `ChunkStatus`.

## Chunk Generation Pipeline

`ChunkStatus` corresponds to a stage in chunk generation. In 1.20.1, there are **12 stages**, arranged in order:

| Index | Status                 | Primary Task                                                                         |
| ----- | ---------------------- | ------------------------------------------------------------------------------------ |
| 0     | `EMPTY`                | Starting point, contains no data                                                     |
| 1     | `STRUCTURE_STARTS`     | Determine which structures (villages, strongholds) may start in the chunk            |
| 2     | `STRUCTURE_REFERENCES` | Add structure references for coordination between structures during world generation |
| 3     | `BIOMES`               | Determine biomes (based on noise sampling)                                           |
| 4     | `NOISE`                | Generate rough terrain outline and bedrock layer                                     |
| 5     | `SURFACE`              | Replace blocks near the surface (e.g., stone → dirt, grass)                          |
| 6     | `CARVERS`              | Generate caves and canyons                                                           |
| 7     | `FEATURES`             | Generate features (trees, ores, flowers, etc.)                                       |
| 8     | `INITIALIZE_LIGHT`     | Initialize lighting calculations (added in 1.20)                                     |
| 9     | `LIGHT`                | Complete lighting calculations                                                       |
| 10    | `SPAWN`                | Spawn initial creatures (wild animals, etc.)                                         |
| 11    | `FULL`                 | Convert `ProtoChunk` to `WorldChunk`                                                 |

Each stage has a corresponding **`GenerationTask`** (when generating from scratch) or **`LoadTask`** (when loading from disk), defining the operations needed to advance from the previous state to the current state. The `ChunkStatus.ChunkType` enum distinguishes two chunk types:

- `PROTOCHUNK`: Under generation, uses `ProtoChunk`;
- `LEVELCHUNK`: Completed (`FULL` stage), uses `WorldChunk`.

> [!TIP]
> A mnemonic for generation-to-completion stages (1.16.4 version, slightly different from 1.20.1):
>
> Empty, starts, references, biomes, noise, surface.
> Carvers, features, lighting, spawn — done at last.

## Outer Ring Dependencies

While `FULL` marks "chunk generation complete", this doesn't mean a single isolated chunk can progress all the way to `FULL` alone.

**Outer ring dependencies** (outskirt dependency) mean: for a chunk to advance to a certain `ChunkStatus`, chunks within a certain distance must reach at least a minimum state. This distance is determined by **`taskMargin`**.

Take noise terrain as an example: to generate terrain at `(0, 0)`, the generator needs to read biome information from surrounding chunks to complete interpolation. If surrounding chunks haven't even determined biomes, the central chunk's terrain cannot be correctly generated. Therefore, the `NOISE` stage's `taskMargin` is set to 8 — chunks within an 8-chunk radius must have at least completed `BIOMES`.

Each stage's `taskMargin` is roughly:

| Stage                  | taskMargin | Explanation                                                     |
| ---------------------- | ---------- | --------------------------------------------------------------- |
| `STRUCTURE_STARTS`     | 0          | Structure starting positions depend only on this chunk          |
| `STRUCTURE_REFERENCES` | 8          | Needs reference information from surrounding structures         |
| `BIOMES`               | 8          | Biome interpolation needs surrounding noise data                |
| `NOISE`                | 8          | Terrain generation needs surrounding biome information          |
| `SURFACE`              | 8          | Surface building needs surrounding noise information            |
| `CARVERS`              | 8          | Cave generation needs to connect to surrounding chunks          |
| `FEATURES`             | 8          | Features (like large trees) may cross chunk boundaries          |
| `INITIALIZE_LIGHT`     | 0          | Only initializes this chunk's lighting                          |
| `LIGHT`                | 1          | Lighting calculations need coordination with neighboring chunks |
| `FULL`                 | 0          | Conversion requires no external information                     |

> [!IMPORTANT]
> The existence of `taskMargin` means: **even if loading tickets only require loading chunks in the central area, chunks within a certain surrounding range will also be triggered to the corresponding ChunkStatus**. This is why terrain shown by F3+G is always one ring larger than the strong loading range — those outer ring chunks don't participate in ticking, but their generation is partially complete.

### taskMargin and Chebyshev Distance

`taskMargin` uses **Chebyshev Distance** as the distance metric. For chunk coordinates `(x₁, z₁)` to `(x₂, z₂)`:

$$\text{Distance} = \max(|x_1 - x_2|, |z_1 - z_2|)$$

For example, distance from `(0, 0)` to `(8, 0)` = 8, and to `(8, 8)` is also 8. This means when `taskMargin = 8`, generation affects a square region centered on the target chunk with side length `2×8+1 = 17`.

## Handling Unreachable Generation

Not all chunks can reach `FULL`:

- A chunk far from any ticket may only reach `BIOMES` and stop;
- A chunk close to a ticket but with old save data may load to `SURFACE` and stop (if further stages aren't needed).

When requesting a higher `ChunkStatus` than a chunk can reach:

```java
// In ChunkHolder.getOrCreateFuture()
if (targetStatusIndex > completedStatusIndex) {
    // Target status not yet reached → return pending Future, suspend task
}
```

The caller receives a `CompletableFuture` that remains **pending** until the chunk reaches that state or is unloaded — at which point the Future completes with `ChunkHolder.UNLOADED_WORLD_CHUNK`.

## Lighting and Save Upgrade Bypass

### Why Lighting Has Two Stages

1.20 introduced `INITIALIZE_LIGHT` between `FEATURES` and `LIGHT`. Why two stages?

**The problem:** Old lighting algorithms (`LIGHT` stage) required reading a large area around the chunk — if neighboring chunks weren't generated, the operation would block. This caused stuttering.

**The solution:** Split into two stages:

- **`INITIALIZE_LIGHT`** (taskMargin=0): Pre-calculate this chunk's own block light and skylight, without reading neighbors;
- **`LIGHT`** (taskMargin=1): Use neighboring chunk data to refine edge lighting.

This way, most lighting work completes in the first stage (no wait), with only edge refinement in the second stage (short wait).

### Save Upgrade Path

When loading chunks saved in old versions (e.g., 1.12 → 1.20), the generation pipeline can't simply replay all 12 stages — old terrain doesn't need regeneration, only adaptation.

`ChunkSerializer` defines a **save upgrade path**:

```java
// ChunkSerializer.deserialize() simplified
if (chunkNbt.contains("Status", NbtType.STRING)) {
    String statusName = chunkNbt.getString("Status");
    ChunkStatus savedStatus = ChunkStatus.byName(statusName);
    // Apply DataFixer for version migration
    chunkNbt = DataFixTypes.CHUNK.update(...);
}
```

**Upgrade logic:**

- If saved status ≥ `FEATURES` → skip all generation stages, directly set to saved status;
- If saved status < `FEATURES` → replay from `EMPTY` (old terrain incompatible, must regenerate);
- Lighting stages (`INITIALIZE_LIGHT`, `LIGHT`) always re-execute even if saved (marked `shouldAlwaysUpgrade=true`).

Why force lighting recalculation? Because surrounding chunks may have changed after saving. Old lighting data is no longer trustworthy.

## Summary

- `ChunkStatus` represents a stage in the chunk generation pipeline, with 12 stages total;
- Each stage may depend on surrounding chunks (taskMargin), using Chebyshev distance;
- Chunks may stop at any stage, with the furthest reachable stage determined by loading tickets;
- Lighting is split into two stages (`INITIALIZE_LIGHT` + `LIGHT`) to reduce blocking;
- Old saves bypass partial generation stages but force lighting recalculation.

## :advanced Code Walkthrough

### Why Generation Tasks Use Executor

Each `ChunkStatus` has an associated `Executor` type:

```java
// ChunkStatus.java (excerpt)
public enum TaskType {
    MAIN_THREAD,      // Execute on server main thread
    ASYNC             // Execute on independent generation thread pool
}
```

Most generation stages (NOISE, SURFACE, CARVERS, FEATURES) use `ASYNC` — they run on independent threads without blocking the main thread. Only a few stages (like FULL conversion, certain lighting operations) use `MAIN_THREAD` — because they need to access runtime state (entities, ticking logic).

**Why can't FULL stage be ASYNC?** Because `WorldChunk` participates in runtime scheduling — it needs to register scheduled ticks, load entities, trigger adjacent block updates. These operations can only run on the main thread, otherwise they'd face concurrency conflicts.

### taskMargin Design Details

```java
// ChunkStatus.java (excerpt)
private final int taskMargin;

public int getDistanceFromFull() {
    return ChunkStatus.FULL.index - this.index;
}

public int getMaxDistanceFromFull() {
    return this.getDistanceFromFull() + this.taskMargin;
}
```

`getMaxDistanceFromFull()` returns "how many chunks away from the center, at minimum, must outer ring chunks reach to allow center chunk to advance to this stage". For example:

- `NOISE.taskMargin = 8` → outer ring chunks need to reach `BIOMES` (one stage before NOISE);
- Center chunk reaching `NOISE` requires: `taskMargin=8` chunks away reach at least `BIOMES`.

Scheduling logic uses this value to pre-trigger outer ring chunk generation:

```java
// ChunkHolder.createFuture() simplified
for (int r = 0; r <= status.taskMargin; r++) {
    // Ensure chunks at distance r reach the required preceding status
    if (!canReach(r, status.getPreviousStatus())) {
        // Cannot reach → returned CompletableFuture stays pending, task suspended
    }
}
```

**Why does `CARVERS` need taskMargin=8 while `LIGHT` only needs 1?** During cave generation, a cave may span multiple chunks. If caves are only generated within one chunk, "suddenly severed caves" appear at boundaries — a visual error. taskMargin=8 ensures chunks within 8 blocks of the cave-generating center chunk have at least completed preceding stages (`SURFACE`), allowing caves across the entire region to generate seamlessly.

Lighting calculations only need taskMargin=1 because light propagation only needs brightness values from adjacent chunks — after the first layer of neighbors completes, lighting can be correctly calculated.

**The meaning of `shouldAlwaysUpgrade`**: Certain states (like `INITIALIZE_LIGHT` and `LIGHT`) are marked `shouldAlwaysUpgrade=true`. This means even when loading old chunks from disk where that state has already "completed", the system will **forcibly re-execute** that stage's calculations. Why does lighting need this? Because lighting depends on surrounding chunk states — and surrounding chunks may have been modified after saving. Forcing lighting recalculation guarantees loaded chunk lighting is correct.

### GenerationTask Scheduling

Each `ChunkStatus` carries a `GenerationTask` when registered (using the default `STATUS_BUMP_LOAD_TASK` if not specified):

```java
// ChunkStatus.java
interface GenerationTask {
    CompletableFuture<Either<Chunk, Unloaded>> doWork(
        ChunkStatus targetStatus, Executor executor, ServerWorld world,
        ChunkGenerator generator, ..., List<Chunk> chunks, Chunk chunk);
}
```

Scheduling logic in `runGenerationTask()`: it takes the middle chunk from the `chunks` list (collected by `getRegion` from surrounding chunks) as the target, calls `generationTask.doWork()`. After task completion, it marks the current stage complete with `chunk.setStatus(this)`, then passes the result to `futuresByStatus[this.index]`, triggering Futures waiting for this stage.

Two design details worth noting:

1. **Tasks execute on independent generation threads** (`worldGenExecutor`), not the main thread. This is why `CompletableFuture` is necessary — the main thread cannot block waiting for generation to complete.
2. **getRegion's margin parameter**: For `makeChunkTickable`, `margin=1` means requiring all surrounding 9 chunks (3×3) to reach `FULL`. This is not taskMargin — taskMargin controls propagation **within the same stage**, while margin controls **dependencies between different stages**.

## References

- [Discovering Minecraft - ChunkStatus and World Generation](https://github.com/lovexyn0827/Discovering-Minecraft) (CC0 License)
- `net.minecraft.world.chunk.ChunkStatus`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ServerLightingProvider`
- `net.minecraft.world.chunk.ChunkSerializer`
- `com.mojang.datafixers.DataFixer`
