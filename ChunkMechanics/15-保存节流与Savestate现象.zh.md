---
slug: save-throttling-savestate
title: 保存节流与 Savestate 现象
description: 区块保存冷却、异步写盘、Watchdog 与 OOM 如何共同决定 Chunk Savestate 的状态分叉。
index: 15
is-advanced: true
---

> 此现象由 BFladderbean 发现。原理文档由 [HackerRouter](https://space.bilibili.com/335688294) 撰写并首次发表于《[Chunk Savestate 原理分析](https://space.bilibili.com/335688294)》（BV1uV4y1a7Xx），本文在其基础上扩充 OOM 路径和源码行锚，并纳入 ChunkMechanics 主线。

## 核心因果链

Chunk Savestate 是这样出现的：

1. TACS（`ThreadedAnvilChunkStorage`）在普通 tick 的增量保存中保存了区块 A（旧状态）并设置 10 秒冷却。
2. 区块 A 和区块 B 的状态被改变（例如玩家在 A 和 B 之间移动物品）。
3. 后续保存尝试中，区块 A 因冷却被跳过、区块 B 正常保存。
4. 主线程卡死，Watchdog 强退（或 OOM 杀进程）。
5. 服务器重启后，区块 A 读出旧快照、区块 B 读出新快照，物品同时存在于两处。

这不是随机的保存竞争，而是**节流冷却**和**强制退出**共同作用的必然结果：冷却保护了过时快照，强退阻止了后续修正。

## TACS 增量保存

你已经在第 11 篇中见过 TACS 的卸载流水线。增量保存发生在每个 tick 中，不是只有 6000 tick 一次的自动保存：

```
MinecraftServer.tick()
  → tickWorlds()
    → serverWorld.tick()
      → getChunkManager().tick()
        → threadedAnvilChunkStorage.tick()
          → unloadChunks()
```

在 `ThreadedAnvilChunkStorage.unloadChunks()` 末尾（`ThreadedAnvilChunkStorage.java`:522-529）：

```java
int k = 0;
ObjectIterator<ChunkHolder> objectIterator = this.chunkHolders.values().iterator();
while (k < 20 && shouldKeepTicking.getAsBoolean() && objectIterator.hasNext()) {
    if (this.save(objectIterator.next())) {
        k++;
    }
}
```

每 tick 最多保存 20 个区块。`k` 只在 `save(ChunkHolder)` 返回 `true` 时增加，也就是说只有真的写了区块、冷却没有拦截、不在卸载任务中，计数器才往前走。

这个循环和 6000 tick 的自动保存相互独立，是服务端 tick 的常态流程之一。

## 10 秒保存冷却

TACS 用一个 `chunkToNextSaveTimeMs`（`Long2LongOpenHashMap`）维护每个区块的下次可保存时间戳。当 `save(ChunkHolder)` 发现"现在还不到下次保存时刻"，直接返回 `false`，不触发序列化也不增加保存计数。

在 `ThreadedAnvilChunkStorage.save(ChunkHolder)` 中（`ThreadedAnvilChunkStorage.java`:790-813）：

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

如果当前时间 `n` 小于上次记录的 `m`，直接跳过。只有真的保存成功（`save(chunk)` 返回 `true`），才把下次可保存时间设置为 `n + 10000L`，也就是 10 秒后。

这个冷却只作用在 `save(ChunkHolder)` 的入口，不作用在 `save(Chunk)` 或强制 flush 路径。

## 直接区块保存

`save(Chunk)` 是最终做实际序列化的地方（`ThreadedAnvilChunkStorage.java`:816-846）：

```java
private boolean save(Chunk chunk) {
    this.pointOfInterestStorage.saveChunk(chunk.getPos());
    if (!chunk.needsSaving()) {
        return false;
    }

    chunk.setNeedsSaving(false);
    ChunkPos chunkPos = chunk.getPos();

    try {
        ChunkStatus chunkStatus = chunk.getStatus();
        // 跳过空区块和低状态 ProtoChunk
        if (chunkStatus.getChunkType() != ChunkStatus.ChunkType.LEVELCHUNK) {
            if (this.isLevelChunk(chunkPos)) {
                return false;
            }
            if (chunkStatus == ChunkStatus.EMPTY && chunk.getStructureStarts().values().stream().noneMatch(StructureStart::hasChildren)) {
                return false;
            }
        }

        this.world.getProfiler().visit("chunkSave");
        NbtCompound nbtCompound = ChunkSerializer.serialize(this.world, chunk);
        this.setNbt(chunkPos, nbtCompound);
        this.mark(chunkPos, chunkStatus.getChunkType());
        return true;
    } catch (Exception exception) {
        LOGGER.error("Failed to save chunk {},{}", chunkPos.x, chunkPos.z, exception);
        return false;
    }
}
```

它先检查 `needsSaving()`，如果区块没脏标记就跳过。然后把脏标记清掉（`setNeedsSaving(false)`），调用 `ChunkSerializer.serialize()` 把区块转成 NBT，最后通过 `setNbt()` 交给存储层。

这个调用是主线程上的同步操作。区块对象在这时被读取，之后 NBT 进入写盘队列。

## StorageIoWorker 写盘队列

`setNbt()` 最终调用 `VersionedChunkStorage.setNbt()`，它会把 NBT 交给 `StorageIoWorker`：

```java
public CompletableFuture<Void> setResult(ChunkPos pos, @Nullable NbtCompound nbt) {
    return this.<CompletableFuture<Void>>run(() -> {
        StorageIoWorker.Result result = this.results.computeIfAbsent(pos, pos2 -> new StorageIoWorker.Result(nbt));
        result.nbt = nbt;
        return Either.left(result.future);
    }).thenCompose(Function.identity());
}
```

这个方法把 NBT 放进 `results`（一个 `ChunkPos → Result` 的 map）。`Result` 内部有一个 `CompletableFuture<Void>`，在实际写盘完成后才会 `complete(null)`。

真正的写盘在后台异步执行。`writeRemainingResults()` 链式调度 `writeResult()`，每次从 `results` 取一个条目，调用 `storage.write(pos, nbt)`，然后 complete 对应的 `future`（`StorageIoWorker.java`:201-223）。

普通 tick 中，这些 future 可能还没 complete；强制 flush 时，`completeAll(true)` 会等所有 future 并调用 `storage.sync()`。

## 正常 Flush

当 `ServerWorld.save()` 收到 `flush=true` 时，调用链是这样的（`ThreadedAnvilChunkStorage.java`:440-471）：

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

注意这里调用的是 `save(Chunk)`，**绕过了 `save(ChunkHolder)` 的冷却检查**。这就是为什么 flush 路径不受 10 秒冷却限制。

然后 `completeAll()` 会调用 `StorageIoWorker.completeAll(true)`（`StorageIoWorker.java`:154-168）：

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

`sync=true` 时会等所有 `future` 完成，然后调用 `RegionBasedStorage.sync()`，它遍历所有打开的 `.mca` 区域文件，对每个调用 `RegionFile.sync()`（`RegionBasedStorage.java`:91-94）：

```java
public void sync() throws IOException {
    for (RegionFile regionFile : this.cachedRegionFiles.values()) {
        regionFile.sync();
    }
}
```

而 `RegionFile.sync()` 直接调用 `FileChannel.force(true)`（`RegionFile.java`:251-253）：

```java
public void sync() throws IOException {
    this.channel.force(true);
}
```

`force(true)` 把缓冲区写盘并刷新文件元数据。这是正常 flush 流程中唯一真正的"刷盘"保证。

## 自动保存 vs. 正常关服

服务端每 6000 tick（5 分钟）调用一次 `saveAll(true, false, false)`（调用点 `MinecraftServer.java`:873-876，`saveAll` 定义 566-602）：

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

**注意 `flush=false`**。这意味着普通自动保存只是把 NBT 交给 `StorageIoWorker`，不等 future、不刷盘。在 1.20.1 中，自动保存不保证 `.mca` 文件已经落到磁盘上。

正常关服（`/stop` 命令或 `Ctrl+C`）会进入 `MinecraftServer.shutdown()`（`MinecraftServer.java`:609-665）：

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

关键点：

- 它先排空延迟区块任务（`shouldDelayShutdown()` 循环）。
- 然后调用 `save(false, true, false)`，这里 **`flush=true`**，会绕过冷却、等所有 future、刷盘。
- 之后才关闭世界。

这就是为什么正常关服不会出现 Savestate：所有脏区块都被最后一次 flush 写完并刷盘，没有遗漏的旧快照。

## Watchdog 强制退出

`DedicatedServerWatchdog` 在单独线程中监控主线程 tick 时间。当发现单个 tick 超过 `maxTickTime`（默认 60 秒），它会：

1. 打印错误日志：`FATAL_MARKER`，"Considering it to be crashed, server will forcibly shutdown."
2. 收集所有线程的堆栈并写崩溃报告。
3. 调用 `shutdown()` 方法，**而这个方法做了两件事**（`DedicatedServerWatchdog.java`:90-103）：

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

它先调度一个 10 秒后的 `Runtime.halt(1)`，然后立刻调用 `System.exit(1)`。`System.exit(1)` 会启动 JVM 关闭流程和注册的关闭钩子，但它不会展开任意卡住的线程或执行那些线程的 `finally` 块。由于服务器线程已经卡在主循环中，`runServer()` 的 `finally` 块（调用 `shutdown()` 和最后的 `save(false, true, false)`）根本不会从该线程执行。10 秒后 `halt(1)` 直接杀进程，不等任何清理。

**结果：最后一次 `save(false, true, false)` 来不及执行，之前被冷却跳过的区块就永久留在旧快照状态。**

## OOM Savestate 与区块保存抑制

你可能听说过"OOM 也能触发 Savestate"。这是一条和 Watchdog 强退完全不同的路径：Watchdog 导致跨区块的旧快照 A + 新快照 B 分裂，而 OOM 方法通过区块保存抑制导致特定区块被重新生成。

### 因果链

核心原理：在内存压力下，一个被塞入大量 NBT 数据的区块在保存时发生序列化、写入或读取失败，导致该区块的 `.mca` 数据损坏或无法读取。服务器重启后，无法从磁盘读出这个区块的 NBT，就回退到新建 `ProtoChunk`，然后重新生成这个区块。

完整路径：

1. 玩家往某个区块的容器中塞入海量物品（例如潜影盒套娃、大量独特 NBT 附魔书），使这个区块的 NBT 数据膨胀到几 MB 甚至几十 MB。
2. 服务端尝试保存该区块时，在序列化、写入或后续读取过程中发生 OOM、缓冲区溢出、或其他异常，导致 `.mca` 条目被截断、损坏或根本没有写入。
3. 服务器重启后，`RegionFile.getChunkInputStream(pos)` 无法返回有效流、`StorageIoWorker.readChunkData(pos)` 捕获异常、或 `RegionBasedStorage.getTagAt(pos)` 返回 `null`。
4. `ThreadedAnvilChunkStorage.loadChunk(pos)` 发现 NBT 不可用，返回 `getProtoChunk(pos)` 创建一个新的 `ProtoChunk`。
5. 后续 `upgradeChunk()` 调用 `requiredStatus.runGenerationTask(...)`，重新生成该区块的地形、结构、生物群系等，整个区块被完全替换。

### 和 Watchdog 的区别

- **Watchdog Savestate**：保存冷却导致某些区块保留旧快照，其他区块保存新快照，区块之间呈现不同时间线的快照分裂。旧区块的数据是一致的老快照，不是被重新生成的。
- **OOM 区块保存抑制**：目标区块因 NBT 损坏无法读取，整个区块被重新生成为世界种子的初始状态，地形、箱子、实体全部回到生成时的样子。非目标区块仍然读出正常快照。

对比表：

| 路径                         | 保存行为                     | 重启后区块状态           | 典型现象                         |
| :--------------------------- | :--------------------------- | :----------------------- | :------------------------------- |
| `/stop` 或 `save-all flush` | 全部区块 flush，冷却无效化   | 全部读出一致的新快照     | 数据一致性保证                   |
| Watchdog 强退                | 冷却跳过部分区块             | 旧快照 A + 新快照 B 共存 | 跨区块物品复制、实体复制         |
| OOM 区块保存抑制             | 目标区块写入失败、数据损坏   | 目标区块被重新生成       | 目标区块地形、箱子、实体回到初始 |
| 普通 OOM 恢复                | 取决于 flush 是否完整执行    | 不确定                   | 不可靠                           |

### 源码锚点

`RegionFile.getChunkInputStream(pos)` 在多个条件下返回 `null`（`RegionFile.java`:102-142）：

```java
@Nullable
public synchronized DataInputStream getChunkInputStream(ChunkPos pos) throws IOException {
    int i = this.getSectorData(pos);
    if (i == 0) {
        return null;  // 区块条目不存在
    }
    // ... 读取扇区数据 ...
    if (byteBuffer.remaining() < 5) {
        LOGGER.error("Chunk {} header is truncated: expected {} but read {}", pos, l, byteBuffer.remaining());
        return null;  // 头被截断
    }
    int m = byteBuffer.getInt();
    byte b = byteBuffer.get();
    if (m == 0) {
        LOGGER.warn("Chunk {} is allocated, but stream is missing", pos);
        return null;  // 条目已分配但流缺失
    }
    // ... 检查流完整性 ...
    if (n > byteBuffer.remaining()) {
        LOGGER.error("Chunk {} stream is truncated: expected {} but read {}", pos, n, byteBuffer.remaining());
        return null;  // 流被截断
    } else if (n < 0) {
        LOGGER.error("Declared size {} of chunk {} is negative", m, pos);
        return null;  // 声明大小为负
    }
    // ...
}
```

无效版本标识也返回 `null`（`RegionFile.java`:157-165）：

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

`RegionBasedStorage.getTagAt(pos)` 在流为 `null` 时返回 `null`（`RegionBasedStorage.java`:47-52）：

```java
@Nullable
public NbtCompound getTagAt(ChunkPos pos) throws IOException {
    RegionFile regionFile = this.getRegionFile(pos);
    try (DataInputStream dataInputStream = regionFile.getChunkInputStream(pos)) {
        return dataInputStream == null ? null : NbtIo.read(dataInputStream);
    }
}
```

`StorageIoWorker.readChunkData(pos)` 捕获异常并记录（`StorageIoWorker.java`:137-150）：

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

`ThreadedAnvilChunkStorage.loadChunk(pos)` 过滤 NBT 并在缺失时返回 `getProtoChunk(pos)`（`ThreadedAnvilChunkStorage.java`:596-613）：

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
            return Either.left(this.getProtoChunk(pos));  // NBT 缺失，创建新 ProtoChunk
        }
    }, this.mainThreadExecutor).exceptionallyAsync(throwable -> this.recoverFromException(throwable, pos), this.mainThreadExecutor);
}
```

异常恢复也返回 `getProtoChunk(pos)`（`ThreadedAnvilChunkStorage.java`:620-638）：

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

`getProtoChunk()` 返回一个全新的 `ProtoChunk`，没有旧快照的任何状态。

`ThreadedAnvilChunkStorage.upgradeChunk()` 会检查 chunk 的当前状态，如果低于所需状态，就调用 `requiredStatus.runGenerationTask(...)` 重新生成（`ThreadedAnvilChunkStorage.java`:649-679）：

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
                        completableFuturex = requiredStatus.runGenerationTask(...);  // 重新生成
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

写入路径在 `RegionFile.writeChunk(pos, buf)` 中（`RegionFile.java`:267-293）。如果写入过程中发生 OOM 或异常，`.mca` 条目可能被截断、损坏或根本没有写入，导致后续读取失败：

```java
protected synchronized void writeChunk(ChunkPos pos, ByteBuffer buf) throws IOException {
    // ... 分配扇区、写入数据 ...
    this.channel.write(buf, o * 4096);
    // ... 更新头信息 ...
    this.writeHeader();
    // ...
}
```

在内存压力下，任何 `ByteBuffer.allocate()`、`channel.write()` 或 `writeHeader()` 都可能失败，留下不完整的数据。

## 状态分叉的必然性

回到最初的因果链：

1. 区块 A 在某个 tick 被正常保存，获得 10 秒冷却。
2. 玩家在接下来几秒内改变了 A 和 B（例如从 A 的箱子移动物品到 B 的箱子）。
3. 区块 B 在后续 tick 中正常保存（冷却已过或首次保存）。
4. 区块 A 因冷却还在，保存被跳过，脏标记留在内存里。
5. Watchdog 检测到主线程卡死，10 秒后强制 `halt(1)`。
6. 最后的 flush 来不及执行。
7. 服务器重启，区块 A 读出旧快照（步骤 1 时写的），区块 B 读出新快照（步骤 3 时写的）。

物品在旧的 A 快照和新的 B 快照中同时存在，因为两次保存之间没有原子性保证，冷却机制也不关心跨区块一致性。这不是 bug，而是节流设计和强退现实的交集。

## 总结

Chunk Savestate 不是随机事件，而是以下机制共同作用的结果：

- **增量保存**：每 tick 保存最多 20 个区块，独立于自动保存。
- **10 秒冷却**：`save(ChunkHolder)` 拒绝过于频繁的保存，保护被跳过的区块处于旧状态。
- **异步写盘**：`StorageIoWorker` 把 NBT 排队，flush 时才等 future 和刷盘。
- **自动保存不 flush**：6000 tick 的自动保存 `flush=false`，不保证磁盘写入。
- **正常关服 flush**：`shutdown()` 中的 `save(false, true, false)` 绕过冷却、等所有 future、刷盘。
- **Watchdog 强退**：10 秒后 `halt(1)` 杀进程，正常 flush 来不及执行，冷却跳过的区块保留旧快照。
- **OOM 区块保存抑制**：目标区块因大量 NBT 数据在保存时损坏，重启后无法读取，被重新生成（和 Watchdog 的跨区块旧/新快照分裂是不同的路径）。
- **普通 OOM 恢复**：`finally` 会尝试 `shutdown()`，但 OOM 环境中高度不可靠。

如果你需要跨区块旧/新快照分裂的 Savestate，用 Watchdog 强退（卡主线程让它超时）。如果你需要数据一致，用 `/stop` 或手动 `save-all flush`，或者理解冷却机制后用脚本构造跨区块操作的时序窗口，在正常关服前留够 10 秒让所有区块冷却到期并重新标记脏。

**不要依赖普通 OOM 恢复路径作为 Savestate 手段**：它的"成功"是偶然，不是保证。
