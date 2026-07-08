---
slug: save-throttling-savestate
title: Save Throttling and Savestate Phenomenon
description: How chunk save cooldown, async disk write, Watchdog, and OOM collectively determine the state fork of Chunk Savestate.
index: 15
is-advanced: true
---

> This phenomenon was discovered by BFladderbean. The principle document was written by [HackerRouter](https://space.bilibili.com/335688294) and first published in "[Chunk Savestate Principle Analysis](https://space.bilibili.com/335688294)" (BV1uV4y1a7Xx). This article expands on OOM paths and source code line anchors based on that work and incorporates it into the ChunkMechanics main line.

## Core Causal Chain

Chunk Savestate appears like this:

1. TACS (`ThreadedAnvilChunkStorage`) saves chunk A (old state) in an incremental save during a regular tick and sets a 10-second cooldown.
2. The states of chunk A and chunk B are changed (e.g., a player moves items between A and B).
3. In subsequent save attempts, chunk A is skipped due to cooldown; chunk B is saved normally.
4. The main thread hangs, and Watchdog force-exits (or OOM kills the process).
5. After server restart, chunk A reads the old snapshot, chunk B reads the new snapshot, and items exist in both locations.

This is not random save contention, but the inevitable result of **throttling cooldown** and **forced exit** working together: the cooldown protects the outdated snapshot, and the forced exit prevents subsequent correction.

## TACS Incremental Save

You've already seen TACS's unload pipeline in Chapter 11. Incremental saves occur in every tick, not just the auto-save every 6000 ticks:

```
MinecraftServer.tick()
  → tickWorlds()
    → serverWorld.tick()
      → getChunkManager().tick()
        → threadedAnvilChunkStorage.tick()
          → unloadChunks()
```

At the end of `ThreadedAnvilChunkStorage.unloadChunks()` (`ThreadedAnvilChunkStorage.java`:522-529):

```java
int k = 0;
ObjectIterator<ChunkHolder> objectIterator = this.chunkHolders.values().iterator();
while (k < 20 && shouldKeepTicking.getAsBoolean() && objectIterator.hasNext()) {
    if (this.save(objectIterator.next())) {
        k++;
    }
}
```

Up to 20 chunks are saved per tick. `k` only increments when `save(ChunkHolder)` returns `true`, meaning the counter only moves forward when a chunk was actually written, cooldown didn't block it, and it's not in an unload task.

This loop is independent of the 6000-tick auto-save and is one of the server tick's regular processes.

## 10-Second Save Cooldown

TACS uses a `chunkToNextSaveTimeMs` (`Long2LongOpenHashMap`) to maintain the next saveable timestamp for each chunk. When `save(ChunkHolder)` finds "it's not yet time for the next save," it directly returns `false`, not triggering serialization or incrementing the save count.

In `ThreadedAnvilChunkStorage.save(ChunkHolder)` (`ThreadedAnvilChunkStorage.java`:790-813):

```java
long l = chunk.getPos().toLong();
long m = this.chunkToNextSaveTimeMs.getOrDefault(l, -1L);
long n = System.currentTimeMillis();
if (n < m) {
    return false;
}

boolean bl = this.save(chunk);
chunkHolder.updateAccessibleStatus();
if (bl) {
    this.chunkToNextSaveTimeMs.put(l, n + 10000L);
}
```

If the current time `n` is less than the recorded `m`, it's skipped immediately. Only when the save truly succeeds (`save(chunk)` returns `true`) is the next saveable time set to `n + 10000L`, i.e., 10 seconds later.

This cooldown only applies at the `save(ChunkHolder)` entry point, not in the `save(Chunk)` or forced flush path.

## Direct Chunk Save

The `save(Chunk)` overload directly serializes a `Chunk` into NBT and queues it to `StorageIoWorker`. It checks the dirty flag before calling `ChunkSerializer.serialize()`:

```java
private boolean save(Chunk chunk) {
    this.pointOfInterestStorage.saveChunk(chunk.getPos());
    if (!chunk.needsSaving()) {
        return false;
    }
    chunk.setLastSaveTime(this.world.getTime());
    chunk.setShouldSave(false);

    ChunkPos chunkPos = chunk.getPos();
    try {
        ChunkStatus chunkStatus = chunk.getStatus();
        if (chunkStatus.getChunkType() != ChunkStatus.ChunkType.LEVELCHUNK) {
            // ...omitted
        }
        this.world.getProfiler().visit("chunkSave");
        CompoundTag compoundTag = ChunkSerializer.serialize(this.world, chunk);
        this.setNbt(chunkPos, compoundTag);
        this.markAsChanged(chunkPos);
        return true;
    } catch (Exception e) {
        // ...omitted
        return false;
    }
}
```

The key is `setNbt(chunkPos, compoundTag)`:

```java
private void setNbt(ChunkPos pos, @Nullable CompoundTag nbt) {
    if (nbt == null) {
        this.worker.setResult(pos, null);
    } else {
        this.worker.setResult(pos, ChunkSerializer.writeToTag(this.world, nbt));
    }
}
```

The `worker` is `StorageIoWorker`, which is the async region file write layer. `setResult(pos, tag)` puts a task into the write queue but doesn't wait for completion. The NBT is serialized into bytes and waits for the worker thread to write to the region file in batches.

This explains an important characteristic: **calling `save(Chunk)` doesn't guarantee immediate disk write; it only guarantees the data has entered the persistent write queue.**

## Async Write Worker

`StorageIoWorker.setResult()` looks like this (`StorageIoWorker.java`:120-130):

```java
public CompletableFuture<Void> setResult(ChunkPos pos, @Nullable NbtCompound nbt) {
    return this.<CompletableFuture<Void>>run(() -> {
        StorageIoWorker.Result result = this.results.computeIfAbsent(pos, pos2 -> new StorageIoWorker.Result(nbt));
        result.nbt = nbt;
        return Either.left(result.future);
    }).thenCompose(Function.identity());
}
```

This method puts the NBT into `results` (a `ChunkPos → Result` map). `Result` internally has a `CompletableFuture<Void>`, which only completes with `complete(null)` after the actual disk write finishes.

The actual disk write executes asynchronously in the background. `writeRemainingResults()` chains `writeResult()` calls, each time taking one entry from `results`, calling `storage.write(pos, nbt)`, then completing the corresponding `future` (`StorageIoWorker.java`:201-223).

In regular ticks, these futures may not have completed yet; when forcing flush, `completeAll(true)` waits for all futures and calls `storage.sync()`.

## Normal Flush

When `ServerWorld.save()` receives `flush=true`, the call chain looks like this (`ThreadedAnvilChunkStorage.java`:440-471):

```java
protected void save(boolean flush) {
    if (flush) {
        List<ChunkHolder> list = this.chunkHolders.values().stream()
            .filter(ChunkHolder::isAccessible)
            .peek(ChunkHolder::updateAccessibleStatus)
            .collect(Collectors.toList());
        MutableBoolean mutableBoolean = new MutableBoolean();

        do {
            mutableBoolean.setFalse();
            list.stream()
                .map(chunkHolder -> {
                    CompletableFuture<Chunk> completableFuture;
                    do {
                        completableFuture = chunkHolder.getSavingFuture();
                        this.mainThreadExecutor.runTasks(completableFuture::isDone);
                    } while (completableFuture != chunkHolder.getSavingFuture());

                    return completableFuture.join();
                })
                .filter(chunk -> chunk instanceof WrapperProtoChunk || chunk instanceof WorldChunk)
                .filter(this::save)
                .forEach(chunk -> mutableBoolean.setTrue());
        } while (mutableBoolean.isTrue());

        this.unloadChunks(() -> true);
        this.completeAll();
    } else {
        this.chunkHolders.values().forEach(this::save);
    }
}
```

Note that this calls `save(Chunk)`, **bypassing the cooldown check in `save(ChunkHolder)`**. This is why the flush path is not subject to the 10-second cooldown limitation.

Then `completeAll()` calls `StorageIoWorker.completeAll(true)` (`StorageIoWorker.java`:154-168):

```java
public CompletableFuture<Void> completeAll(boolean sync) {
    CompletableFuture<Void> completableFuture = this.<CompletableFuture<Void>>run(
            () -> Either.left(CompletableFuture.allOf(this.results.values().stream().map(result -> result.future).toArray(CompletableFuture[]::new)))
        )
        .thenCompose(Function.identity());
    return sync ? completableFuture.thenCompose(void_ -> this.run(() -> {
        try {
            this.storage.sync();
            return Either.left(null);
        } catch (Exception exception) {
            LOGGER.warn("Failed to synchronize chunks", exception);
            return Either.right(exception);
        }
    })) : completableFuture.thenCompose(void_ -> this.run(() -> Either.left(null)));
}
```

When `sync=true`, it waits for all `future` completions, then calls `RegionBasedStorage.sync()`, which iterates through all open `.mca` region files and calls `RegionFile.sync()` on each (`RegionBasedStorage.java`:91-94):

```java
public void sync() throws IOException {
    for (RegionFile regionFile : this.cachedRegionFiles.values()) {
        regionFile.sync();
    }
}
```

And `RegionFile.sync()` directly calls `FileChannel.force(true)` (`RegionFile.java`:251-253):

```java
public void sync() throws IOException {
    this.channel.force(true);
}
```

`force(true)` writes the buffer to disk and flushes file metadata. This is the only true "disk flush" guarantee in the normal flush process.

## Auto-Save vs. Normal Shutdown

The server calls `saveAll(true, false, false)` once every 6000 ticks (5 minutes) (call site `MinecraftServer.java`:873-876, `saveAll` definition 566-602):

```java
public boolean saveAll(boolean suppressLogs, boolean flush, boolean force) {
    try {
        this.saving = true;
        this.getPlayerManager().saveAllPlayerData();
        return this.save(suppressLogs, flush, force);
    } finally {
        this.saving = false;
    }
}
```

**Note `flush=false`**. This means normal auto-save only hands NBT to `StorageIoWorker`, doesn't wait for futures, doesn't flush disk. In 1.20.1, auto-save doesn't guarantee that `.mca` files have actually landed on disk.

Normal shutdown (`/stop` command or `Ctrl+C`) enters `MinecraftServer.shutdown()` (`MinecraftServer.java`:609-665):

```java
public void shutdown() {
    LOGGER.info("Stopping server");
    if (this.getNetworkIo() != null) {
        this.getNetworkIo().stop();
    }

    this.saving = true;
    if (this.playerManager != null) {
        LOGGER.info("Saving players");
        this.playerManager.saveAllPlayerData();
        this.playerManager.disconnectAllPlayers();
    }

    LOGGER.info("Saving worlds");

    for (ServerWorld serverWorld : this.getWorlds()) {
        if (serverWorld != null) {
            serverWorld.savingDisabled = false;
        }
    }

    while (this.worlds.values().stream().anyMatch(world -> world.getChunkManager().threadedAnvilChunkStorage.shouldDelayShutdown())) {
        this.timeReference = Util.getMeasuringTimeMs() + 1L;

        for (ServerWorld serverWorld2 : this.getWorlds()) {
            serverWorld2.getChunkManager().removePersistentTickets();
            serverWorld2.getChunkManager().tick(() -> true, false);
        }

        this.runTasksTillTickEnd();
    }

    this.save(false, true, false);

    for (ServerWorld serverWorld3 : this.getWorlds()) {
        if (serverWorld3 != null) {
            try {
                serverWorld3.close();
            } catch (IOException iOException) {
                LOGGER.error("Exception closing the level", iOException);
            }
        }
    }

    this.saving = false;
    // ...
}
```

Key points:

- It first drains delayed chunk tasks (the `shouldDelayShutdown()` loop).
- Then calls `save(false, true, false)`, where **`flush=true`**, which bypasses cooldown, waits for all futures, and flushes disk.
- Only after that does it close worlds.

This is why normal shutdown doesn't cause Savestate: all dirty chunks are written and flushed in the final flush, with no lingering old snapshots.

## Watchdog Forced Exit

`DedicatedServerWatchdog` monitors main thread tick time in a separate thread. When it detects a single tick exceeding `maxTickTime` (default 60 seconds), it:

1. Prints error log: `FATAL_MARKER`, "Considering it to be crashed, server will forcibly shutdown."
2. Collects all thread stacks and writes a crash report.
3. Calls the `shutdown()` method, **which does two things** (`DedicatedServerWatchdog.java`:90-103):

```java
private void shutdown() {
    try {
        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                Runtime.getRuntime().halt(1);
            }
        }, 10000L);
        System.exit(1);
    } catch (Throwable throwable) {
        Runtime.getRuntime().halt(1);
    }
}
```

It first schedules a `Runtime.halt(1)` after 10 seconds, then immediately calls `System.exit(1)`. `System.exit(1)` starts the JVM shutdown process and registered shutdown hooks, but it doesn't unwind arbitrary stuck threads or execute those threads' `finally` blocks. Since the server thread is already stuck in the main loop, the `finally` block in `runServer()` (which calls `shutdown()` and the final `save(false, true, false)`) won't execute from that thread at all. After 10 seconds, `halt(1)` directly kills the process without waiting for any cleanup.

**Result: The final `save(false, true, false)` doesn't get to execute, and chunks previously skipped by cooldown permanently remain in their old snapshot state.**

## OOM Savestate and Chunk Save Suppression

You may have heard "OOM can also trigger Savestate." This is a completely different path from Watchdog forced exit: Watchdog causes cross-chunk old snapshot A + new snapshot B splits, while the OOM method causes specific chunks to be regenerated through chunk save suppression.

### Causal Chain

Core principle: Under memory pressure, a chunk stuffed with massive NBT data experiences serialization, write, or read failure during save, causing that chunk's `.mca` data to become corrupted or unreadable. After server restart, unable to read this chunk's NBT from disk, it falls back to creating a new `ProtoChunk`, then regenerates this chunk.

Complete path:

1. A player stuffs massive items into a chunk's container (e.g., nested shulker boxes, large numbers of unique NBT enchanted books), causing this chunk's NBT data to balloon to several MB or even tens of MB.
2. When the server attempts to save this chunk, OOM, buffer overflow, or other exceptions occur during serialization, write, or subsequent read, causing the `.mca` entry to be truncated, corrupted, or not written at all.
3. After server restart, `RegionFile.getChunkInputStream(pos)` can't return a valid stream, `StorageIoWorker.readChunkData(pos)` catches an exception, or `RegionBasedStorage.getTagAt(pos)` returns `null`.
4. `ThreadedAnvilChunkStorage.loadChunk(pos)` finds NBT unavailable, returns `getProtoChunk(pos)` creating a new `ProtoChunk`.
5. Subsequent `upgradeChunk()` calls `requiredStatus.runGenerationTask(...)`, regenerating this chunk's terrain, structures, biomes, etc., completely replacing the entire chunk.

### Difference from Watchdog

- **Watchdog Savestate**: Save cooldown causes some chunks to keep old snapshots while other chunks save new snapshots, producing a split between chunk snapshots from different timelines. The old chunk data is a consistent old snapshot, not regenerated.
- **OOM Chunk Save Suppression**: The target chunk cannot be read because its NBT is corrupted, so the entire chunk is regenerated to the world's seed-initial state. Terrain, chests, and entities all return to how they were at generation time. Non-target chunks still load normal snapshots.

Comparison table:

| Path                         | Save Behavior                   | Post-Restart Chunk State        | Typical Phenomenon                               |
| :--------------------------- | :------------------------------ | :------------------------------ | :----------------------------------------------- |
| `/stop` or `save-all flush`  | All chunks flush, cooldown nullified | All load the same new snapshot  | Data consistency guaranteed                      |
| Watchdog forced exit         | Cooldown skips some chunks      | Old snapshot A + new snapshot B coexist | Cross-chunk item duplication, entity duplication |
| OOM chunk save suppression   | Target chunk write fails, data corrupted | Target chunk is regenerated     | Target chunk terrain, chests, and entities return to initial state |
| Ordinary OOM recovery        | Depends on whether flush fully executes | Uncertain                       | Unreliable                                       |

### Source Code Anchors

`RegionFile.getChunkInputStream(pos)` returns `null` under multiple conditions (`RegionFile.java`:102-142):

```java
@Nullable
public synchronized DataInputStream getChunkInputStream(ChunkPos pos) throws IOException {
    int i = this.getSectorData(pos);
    if (i == 0) {
        return null;  // Chunk entry does not exist
    }
    // ... read sector data ...
    if (byteBuffer.remaining() < 5) {
        LOGGER.error("Chunk {} header is truncated: expected {} but read {}", pos, l, byteBuffer.remaining());
        return null;  // Header truncated
    }
    int m = byteBuffer.getInt();
    byte b = byteBuffer.get();
    if (m == 0) {
        LOGGER.warn("Chunk {} is allocated, but stream is missing", pos);
        return null;  // Entry allocated but stream missing
    }
    // ... check stream integrity ...
    if (n > byteBuffer.remaining()) {
        LOGGER.error("Chunk {} stream is truncated: expected {} but read {}", pos, n, byteBuffer.remaining());
        return null;  // Stream truncated
    } else if (n < 0) {
        LOGGER.error("Declared size {} of chunk {} is negative", m, pos);
        return null;  // Declared size negative
    }
    // ...
}
```

An invalid version marker also returns `null` (`RegionFile.java`:157-165):

```java
@Nullable
private DataInputStream decompress(ChunkPos pos, byte flags, InputStream stream) throws IOException {
    ChunkStreamVersion chunkStreamVersion = ChunkStreamVersion.get(flags);
    if (chunkStreamVersion == null) {
        LOGGER.error("Chunk {} has invalid chunk stream version {}", pos, flags);
        return null;
    }
    return new DataInputStream(chunkStreamVersion.wrap(stream));
}
```

`RegionBasedStorage.getTagAt(pos)` returns `null` when the stream is `null` (`RegionBasedStorage.java`:47-52):

```java
@Nullable
public NbtCompound getTagAt(ChunkPos pos) throws IOException {
    RegionFile regionFile = this.getRegionFile(pos);
    try (DataInputStream dataInputStream = regionFile.getChunkInputStream(pos)) {
        return dataInputStream == null ? null : NbtIo.read(dataInputStream);
    }
}
```

`StorageIoWorker.readChunkData(pos)` catches exceptions and logs them (`StorageIoWorker.java`:137-150):

```java
public CompletableFuture<Optional<NbtCompound>> readChunkData(ChunkPos pos) {
    return this.run(() -> {
        // ...
        try {
            NbtCompound nbtCompound = this.storage.getTagAt(pos);
            return Either.left(Optional.ofNullable(nbtCompound));
        } catch (Exception exception) {
            LOGGER.warn("Failed to read chunk {}", pos, exception);
            return Either.right(exception);
        }
    });
}
```

## Load Chunk and Exception Recovery

Back to the load path. When TACS tries to load a chunk, it calls `loadChunk(pos)` (`ThreadedAnvilChunkStorage.java`:597-619):

```java
private CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> loadChunk(ChunkPos pos) {
    return this.getUpdatedChunkNbt(pos).thenApply(nbt -> nbt.filter(nbt2 -> {
        boolean bl = containsStatus(nbt2);
        if (!bl) {
            LOGGER.error("Chunk file at {} is missing level data, skipping", pos);
        }
        return bl;
    })).thenApplyAsync(nbt -> {
        this.world.getProfiler().visit("chunkLoad");
        if (nbt.isPresent()) {
            Chunk chunk = ChunkSerializer.deserialize(this.world, this.pointOfInterestStorage, pos, nbt.get());
            this.mark(pos, chunk.getStatus().getChunkType());
            return Either.left(chunk);
        } else {
            return Either.left(this.getProtoChunk(pos));  // NBT missing, create new ProtoChunk
        }
    }, this.mainThreadExecutor).exceptionallyAsync(throwable -> this.recoverFromException(throwable, pos), this.mainThreadExecutor);
}
```

Exception recovery also returns `getProtoChunk(pos)` (`ThreadedAnvilChunkStorage.java`:620-638):

```java
private Either<Chunk, ChunkHolder.Unloaded> recoverFromException(Throwable throwable, ChunkPos chunkPos) {
    if (throwable instanceof CrashException crashException) {
        Throwable throwable2 = crashException.getCause();
        if (!(throwable2 instanceof IOException)) {
            this.markAsProtoChunk(chunkPos);
            throw crashException;
        }
        LOGGER.error("Couldn't load chunk {}", chunkPos, throwable2);
    } else if (throwable instanceof IOException) {
        LOGGER.error("Couldn't load chunk {}", chunkPos, throwable);
    }
    return Either.left(this.getProtoChunk(chunkPos));
}

private Chunk getProtoChunk(ChunkPos chunkPos) {
    this.markAsProtoChunk(chunkPos);
    return new ProtoChunk(chunkPos, UpgradeData.NO_UPGRADE_DATA, this.world, this.world.getRegistryManager().get(RegistryKeys.BIOME), null);
}
```

`getProtoChunk()` returns a brand new `ProtoChunk` with none of the old snapshot's state.

`ThreadedAnvilChunkStorage.upgradeChunk()` checks the chunk's current status, and if below the required status, calls `requiredStatus.runGenerationTask(...)` to regenerate (`ThreadedAnvilChunkStorage.java`:649-679):

```java
private CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> upgradeChunk(ChunkHolder holder, ChunkStatus requiredStatus) {
    // ...
    return completableFuture.thenComposeAsync(
        either -> either.map(
            chunks -> {
                try {
                    Chunk chunk = chunks.get(chunks.size() / 2);
                    CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> completableFuturex;
                    if (chunk.getStatus().isAtLeast(requiredStatus)) {
                        completableFuturex = requiredStatus.runLoadTask(...);
                    } else {
                        completableFuturex = requiredStatus.runGenerationTask(...);  // Regenerate
                    }
                    // ...
                    return completableFuturex;
                } catch (Exception exception) {
                    // ...
                }
            },
            // ...
        ),
        // ...
    );
}
```

The write path is in `RegionFile.writeChunk(pos, buf)` (`RegionFile.java`:267-293). If OOM or exceptions occur during write, the `.mca` entry may be truncated, corrupted, or not written at all, causing subsequent reads to fail:

```java
protected synchronized void writeChunk(ChunkPos pos, ByteBuffer buf) throws IOException {
    // ... allocate sectors, write data ...
    this.channel.write(buf, o * 4096);
    // ... update header info ...
    this.writeHeader();
    // ...
}
```

Under memory pressure, any `ByteBuffer.allocate()`, `channel.write()`, or `writeHeader()` can fail, leaving incomplete data.

## Inevitability of State Fork

Back to the original causal chain:

1. Chunk A is normally saved in a certain tick, gets 10-second cooldown.
2. Player changes A and B in the following seconds (e.g., moves items from A's chest to B's chest).
3. Chunk B is normally saved in subsequent tick (cooldown expired or first save).
4. Chunk A is still in cooldown, save is skipped, dirty mark remains in memory.
5. Watchdog detects main thread hang, forces `halt(1)` after 10 seconds.
6. Final flush doesn't get to execute.
7. Server restarts, chunk A reads old snapshot (written in step 1), chunk B reads new snapshot (written in step 3).

Items exist simultaneously in old A snapshot and new B snapshot, because there's no atomicity guarantee between the two saves, and the cooldown mechanism doesn't care about cross-chunk consistency. This isn't a bug, but the intersection of throttling design and forced exit reality.

## Summary

Chunk Savestate is not a random event, but the result of these mechanisms working together:

- **Incremental save**: Up to 20 chunks saved per tick, independent of auto-save.
- **10-second cooldown**: `save(ChunkHolder)` rejects overly frequent saves, protecting skipped chunks in old state.
- **Async disk write**: `StorageIoWorker` queues NBT, only waits for futures and flushes disk during flush.
- **Auto-save doesn't flush**: The 6000-tick auto-save has `flush=false`, doesn't guarantee disk write.
- **Normal shutdown flush**: `save(false, true, false)` in `shutdown()` bypasses cooldown, waits for all futures, flushes disk.
- **Watchdog forced exit**: `halt(1)` kills process after 10 seconds, normal flush doesn't get to execute, cooldown-skipped chunks retain old snapshots.
- **OOM chunk save suppression**: Target chunk corrupted during save due to massive NBT data, can't be read after restart, gets regenerated (different path from Watchdog's cross-chunk old/new snapshot split).
- **Normal OOM recovery**: `finally` attempts `shutdown()`, but highly unreliable in OOM environment.

If you need cross-chunk old/new snapshot split Savestate, use Watchdog forced exit (hang main thread to trigger timeout). If you need data consistency, use `/stop` or manual `save-all flush`, or after understanding the cooldown mechanism, use scripts to construct timing windows for cross-chunk operations, leaving 10 seconds before normal shutdown for all chunks' cooldowns to expire and be re-marked dirty.

**Don't rely on normal OOM recovery path as a Savestate method**: its "success" is accidental, not guaranteed.
