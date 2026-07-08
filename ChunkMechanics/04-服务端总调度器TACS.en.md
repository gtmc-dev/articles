---
slug: chunk-tacs
title: Server Master Dispatcher TACS
description: ThreadedAnvilChunkStorage responsibilities overview, core table structures, three types of executors, and ChunkTaskPrioritySystem priority scheduling.
index: 4
is-advanced: false
---

If the chunk management system is a factory, `ThreadedAnvilChunkStorage` (hereafter **TACS**) is the workshop director — it doesn't directly produce products, but coordinates every production line's operation.

## Name Breakdown

The name `ThreadedAnvilChunkStorage` itself explains its responsibilities:

- **Threaded**: Threaded — chunk generation runs on independent threads;
- **Anvil**: Anvil — refers to the modern Minecraft Anvil chunk storage format (`.mca`);
- **ChunkStorage**: Chunk Storage — manages chunk persistence.

Together: "**Threaded Anvil chunk storage manager**". Though multithreading use is actually quite limited — mainly world generation and NBT access, with the latter not even fully managed in TACS.

## Responsibilities Overview

TACS sits at the bottom coordination position in the chunk management system, connecting to almost all other subsystems:

- **Manages chunks in `ChunkHolder` form**: Including `ChunkHolder` creation and unloading, jointly managing their loading levels and states.
- **Defines and schedules chunk loading, generation, and light update tasks**: Through `CompletableFuture`, tasks are made asynchronous.
- **Defines logic for sending chunk data to players**: Methods like `sendWatchPackets()`, `sendChunkDataPackets()`, etc.
- **Manages entity data tracking**: Data synchronization with clients, `EntityTracker` manages entity position tracking.
- **Handles player addition and removal**: `handlePlayerAddedOrRemoved()` and `updatePosition()`.

## Core Table Structures

TACS maintains multiple mapping tables, each with a clear purpose:

### currentChunkHolders

```java
Long2ObjectMap<ChunkHolder> currentChunkHolders
```

All chunks with **existing ChunkHolders** in the current dimension. Key is `ChunkPos.toLong()` serialized `long`, value is the corresponding `ChunkHolder`.

A `ChunkHolder` is placed in this table when created (when the chunk first enters accessible range); removed from this table when unloaded.

### chunkHolders

```java
// ThreadedAnvilChunkStorage.java (Yarn 1.20.1)
private final Long2ObjectLinkedOpenHashMap<ChunkHolder> currentChunkHolders = new Long2ObjectLinkedOpenHashMap<>();
private volatile Long2ObjectLinkedOpenHashMap<ChunkHolder> chunkHolders = this.currentChunkHolders.clone();
```

`ThreadedAnvilChunkStorage` does have a field named `chunkHolders` — it's a **volatile snapshot copy** of `currentChunkHolders`, used for scenarios like `tick()` and `save()` that need stable traversal. Updates happen on the main thread (Minecraft Server), while the read side can safely traverse without locks, avoiding `ConcurrentModificationException`. This is a **Copy-On-Write** pattern: write side updates on main thread, read side has zero overhead.

> [!NOTE] Difference from `ChunkTicketManager.chunkHolders`
> - `TACS.chunkHolders`: `Long2ObjectLinkedOpenHashMap<ChunkHolder>`, complete mapping table of **all** active `ChunkHolder` in current dimension, key is `ChunkPos.toLong()` serialized `long`.
> - `ChunkTicketManager.chunkHolders`: `Set<ChunkHolder>`, only collects `ChunkHolder` whose **loading levels changed** in this tick round, used for subsequent notification callbacks.

### chunksToUnload

```java
LongSet chunksToUnload
```

Records `long` coordinates of chunks about to be unloaded. When a `ChunkHolder`'s loading level drops from accessible to inaccessible, it moves from `currentChunkHolders` into `chunksToUnload`, waiting for an appropriate time to actually unload.

### loadedChunks

```java
// Indirectly managed via VersionedChunkStorage
```

Set of chunks loaded from disk or finished generating, available for access. "Loaded" here means "data in memory", not "currently ticking".

### chunkToType

```java
Long2ByteMap chunkToType
```

Records each chunk's `ChunkLevelType`. Key is chunk pos long, value is `ChunkLevelType.ordinal()` (byte). Used for quickly checking "is this chunk in ENTITY_TICKING state without having to find its ChunkHolder?"

This is a performance-optimized redundant cache — `ChunkHolder` already stores level type, but looking it up requires one more hash table query. `chunkToType` allows direct querying during high-frequency operations (like entity movement collision detection).

### unloadedTaskQueue

```java
Queue<Runnable> unloadedTaskQueue
```

Stores tasks (usually save and cleanup) that need to be executed after a chunk unloads. When a chunk is completely removed from memory, tasks in the queue execute sequentially — typical use: `saveChunk()`, `removeFromTracking()`.

## Three Executors

TACS delegates different types of tasks to three executors:

### mainThreadExecutor

```java
private final Executor mainThreadExecutor;
```

**Main thread executor**. All operations that modify world state (placing/breaking blocks, entity movement, chunk state changes) must run here. Guarantees serialized execution, avoiding concurrency issues.

### worldGenExecutor

```java
private final Executor worldGenExecutor;
```

**World generation thread pool**. Chunk generation tasks (like `NOISE`, `FEATURES`, `CARVERS` stages) run here. Multiple chunks can generate in parallel, but each chunk's generation is still single-threaded (same chunk's different stages execute sequentially).

### lightingExecutor

```java
private final Executor lightingExecutor;
```

**Lighting calculation thread pool**. Light propagation calculations (like `LIGHT` stage) run here. Separated from generation threads because lighting depends on neighboring chunks — dedicated threads avoid blocking generation tasks.

> [!IMPORTANT]
> These three executors **don't directly execute tasks** — they're wrappers for the underlying `ChunkTaskPrioritySystem`, ensuring tasks enter a priority queue first and are scheduled uniformly, rather than executing immediately.

## ChunkTaskPrioritySystem: Priority Scheduling

All asynchronous tasks are coordinated by `ChunkTaskPrioritySystem` before execution. It's not a simple FIFO queue, but a **priority queue** — priority determined by **loading level**:

```java
// ChunkTaskPrioritySystem.java
public static Task<Runnable> createMessage(ChunkHolder holder, Runnable task) {
    // Task priority determined by holder.getCompletedLevel()
    // Lower completedLevel (stronger loading) → higher priority
    return createMessage(task, holder.getPos().toLong(), holder::getCompletedLevel);
}
```

Design principle: **Chunks closer to completion, closer to target level, get higher priority**. This avoids "a level=22 chunk with EMPTY progress" taking priority over "a level=33 chunk that only needs disk loading" — the latter might finish in 1 tick.

> [!TIP]
> Why use `completedLevel` rather than `level` to determine priority?
> `level` is the "target" (what level we want this chunk to reach), `completedLevel` is the "status quo" (what level it's actually reached). Sorting by `completedLevel` means: **tasks already largely complete, closest to target, get highest priority**.

## [!ADVANCED] Code Walkthrough

### TACS Lifecycle and Chunk Holder Creation

TACS is created when `ServerWorld` initializes:

```java
// ServerWorld.java
this.chunkManager = new ServerChunkManager(
    this, session, dataFixer, structureManager, executor,
    chunkGenerator, viewDistance, simulationDistance, ...);
// ServerChunkManager constructor creates TACS internally
```

When a chunk holder is needed:

```java
// ThreadedAnvilChunkStorage.setLevel()
ChunkHolder holder = this.currentChunkHolders.get(pos);
if (holder == null) {
    // Chunk doesn't exist → create new holder
    holder = new ChunkHolder(pos, level, ...);
    this.currentChunkHolders.put(pos, holder);
}
// Update level and schedule generation/loading tasks
```

Key decision: **ChunkHolder is created lazily** — only when a ticket requires it. Chunks outside any ticket range don't have holders, saving memory.

### Why Use Volatile Snapshot Copy

```java
private volatile Long2ObjectLinkedOpenHashMap<ChunkHolder> chunkHolders = 
    this.currentChunkHolders.clone();

public void tick() {
    // Clone before traversal
    this.chunkHolders = this.currentChunkHolders.clone();
    // Traverse cloned copy
    for (ChunkHolder holder : this.chunkHolders.values()) {
        holder.tick();
    }
}
```

**Why clone?** `tick()` may indirectly modify `currentChunkHolders` (creating new holders or removing them). If directly traversing `currentChunkHolders`, `ConcurrentModificationException` would result. Cloning creates a snapshot — the original may change during traversal, but the copy remains stable.

**Volatile ensures visibility**: Main thread updates `currentChunkHolders`, other threads (like save threads) read `chunkHolders`. `volatile` guarantees visibility — once main thread updates the snapshot, other threads immediately see the new value.

This pattern is typical **Copy-On-Write**: only pay cloning cost when iteration is needed, read side has zero overhead.

### setViewDistance: Client Data Resend When View Distance Changes

```java
protected void setViewDistance(int watchDistance) {
    int i = MathHelper.clamp(watchDistance, 2, 32);  // Hard limit 2~32
    if (i != this.watchDistance) {
        int oldDist = this.watchDistance;
        this.watchDistance = i;
        this.ticketManager.setWatchDistance(i);
        // Traverse all existing ChunkHolders
        for (ChunkHolder holder : this.currentChunkHolders.values()) {
            ChunkPos pos = holder.getPos();
            // For each player watching this chunk
            this.getPlayersWatchingChunk(pos, false).forEach(player -> {
                // Check if chunk is in player view under old/new distance
                boolean wasInRange = isWithinDistance(pos, playerPos, oldDist);
                boolean nowInRange = isWithinDistance(pos, playerPos, i);
                // Only send/cancel packets when boundary changes
                if (wasInRange != nowInRange) {
                    this.sendWatchPackets(player, pos, ...);
                }
            });
        }
    }
}
```

Note the logic here: **Only send packets when a chunk's "entering view/leaving view" state changes**. If a chunk is in view under both old and new distances, no resend — avoids network storms when players adjust view distance. But if a chunk first enters view under the new distance, the entire `ChunkDataS2CPacket` is sent to the client.

## References

- [Discovering Minecraft - Basic Architecture of the Chunk Management System](https://github.com/lovexyn0827/Discovering-Minecraft) (CC0 License)
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTaskPrioritySystem`
- `net.minecraft.server.world.ServerChunkManager`