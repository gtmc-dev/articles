---
slug: chunk-system-intro
intro-title: Introduction
chapter-title: Chunks
index: -1
---

## Research Background and Significance

For a long time, community discourse on 1.14+ chunk loading has focused on analyzing **steady states** — we know that nether portal loaders will cause 3×3 chunks in the target dimension to be loaded, but are these chunks loaded instantly or after multiple game ticks? This question is rarely addressed.

Meanwhile, chunk management is an important and quite complex system in Minecraft, intimately related to nearly every aspect of the world. From the most fundamental "block queries" to "entity simulation," from "player view distance" to "chunk saving," the chunk management system precisely controls "which chunks exist, which chunks are simulating, which chunks should be saved" behind the scenes.

This chapter aims to enhance readers' understanding of the underlying working mode of the chunk management system, help readers understand the source code organization structure, and thereby discover more creative applications based on chunk management mechanisms.

## Research Environment and Conventions

This chapter is based on the following environment:

- Minecraft Java Edition **1.20.1**
- Deobfuscation mapping: **Yarn Mapping `1.20.1+build.10`**
- Parts of the analysis are based on lovexyn0827's early research on 1.16.4 and 1.20.1 (CC0 license)

Special conventions used in the text:

- **Module** refers to an object unique within an entire dimension, usually corresponding to a Java class, sometimes also referring to a relatively independent subsystem.
- **"Loading"** does not specifically refer to the commonly understood accessible, weak-loaded, or strong-loaded states, but rather the state of chunks existing in memory and the process of chunks being read from disk into memory.
- Distances in the text, unless otherwise specified, are all **Chebyshev distance** (Chebyshev Distance), not Euclidean distance. For example, the Chebyshev distance from chunk `(0, 0)` to chunk `(2, 2)` is `max(2, 2) = 2`.

## Basic Structure of This Chapter

1. **Basic Concepts and Structure**: Starting from the mathematical definition of chunks, understand sub-chunks, local coordinate packing, differences between ProtoChunk and WorldChunk, and the client chunk pipeline.
2. **Generation Status and ChunkStatus**: Deep dive into the 12-step chunk generation pipeline, understand outer ring dependencies, taskMargin, and bypass systems like lighting/storage upgrades.
3. **Level and Runtime Stratification**: Distinguish ChunkStatus (generation state) and ChunkLevelType (runtime qualification), establish the mental model "generated ≠ simulating."
4. **Server Master Scheduler TACS**: The complete responsibilities of ThreadedAnvilChunkStorage — from currentChunkHolders to chunksToUnload, from worldgen/light/main three executor types to ChunkTaskPrioritySystem's priority scheduling.
5. **Main Thread Async Tasks**: Understand the core role of CompletableFuture in chunk management — almost all operations aren't executed directly, but planned as Futures and then executed in async task queues.
6. **Loading Ticket Mechanism**: Full table of Ticket types, lifecycle, player loading, ticket propagation and expiry reclamation.
7. **ChunkHolder Lifecycle**: How the three core Futures (accessibleFuture, tickingFuture, entityTickingFuture) act like a thermometer controlling three "runtime views" of chunks.
8. **Scheduled Ticks and Chunk Runtime Level**: How scheduled ticks depend on a chunk's BLOCK_TICKING level — tickingFutureReadyPredicate determines whether scheduled ticks can execute, and where scheduled ticks go when chunks aren't ticking.
9. **Random Ticks and Block Entity Ticks**: Random tick sampling mechanism, how ChunkSection counters skip empty sections, block entity ticker registration and execution, and how all three collectively constitute "block ticking."
10. **Entity Loading and Entity Tracking**: The design philosophy of two independent management systems for entities and blocks, ServerEntityManager's responsibilities, how EntityTracker synchronizes — and why ENTITY_TICKING requires 5×5 FULL.
11. **Save and Unload Pipeline**: Three-level unload structure (unloadedChunks → chunksToUnload → unloadTaskQueue), save throttling, ChunkSerializer's serialization responsibilities, and how StorageIoWorker ultimately writes to .mca.
12. **POI and Bypass Storage**: Why Point of Interest is stored independently from regular chunks — its data structure, how block changes trigger updates, and query relationships with villagers, raids, and wandering traders.
13. **Client Visibility and Chunk Packets**: How the server decides which chunks to send to clients, division of labor between chunk data packets and incremental update packets, geometric determination of watchDistance, and the batch sending mechanism of flushUpdates.
14. **Game Event and In-Chunk Listeners**: The essential difference between the game event system and NC/PP updates, WorldChunk's section-based management of event dispatchers, and cleanup of sculk listeners during chunk unload.
15. **Save Throttling and Savestate Phenomenon**: TACS's 10-second save cooldown, StorageIoWorker async disk write, differences between auto-save/flush/shutdown, and how Watchdog and OOM form or break cross-chunk state forks.

## Prerequisites

This text assumes readers have some familiarity with basic concepts in Minecraft:

- [Blocks and Block States](../BlockMechanics/01-方块与方块状态.zh.md) — The difference between Block and BlockState
