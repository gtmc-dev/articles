---
slug: savestate-save-suppression
title: Savestate 与区块保存抑制
description: Watchdog Savestate 的跨区块状态分叉、OOM 区块保存抑制与目标区块重新生成现象。
index: 18
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

## 机制前置

本章只讨论现象链路。保存冷却、`StorageIoWorker`、flush、自动保存与 Watchdog 强退路径的源码细节，见 [保存节流与刷盘机制](./15-保存节流与刷盘机制.zh.md)。

## 保存失败的路径与后果

加载失败会产生降级区块或崩溃，但**保存失败**的后果更隐蔽——区块在内存中正常运行，玩家看不出异常，但数据无法写入磁盘，重启后会回滚到上次成功保存的状态。

### 磁盘空间不足

**触发条件**：`StorageIoWorker` 在写入 region 文件时，操作系统返回"磁盘已满"错误（通常是 `IOException: No space left on device`）。

**发生路径**：

```
ServerWorld.save() / 自动保存 / 关服
  ↓
ThreadedAnvilChunkStorage.save(boolean flush)
  ↓
tryUnloadChunk() → save(chunk)
  ↓
setNbt(chunkPos, nbt)
  ↓
StorageIoWorker.setResult(chunkPos, nbt)
  ↓
worker 线程执行 writeChunk()
  ↓
RegionFile.getChunkOutputStream() 申请扇区
  ↓
磁盘空间不足 → IOException
  ↓
Future.completeExceptionally(exception)
```

**后果**：

1. **区块数据丢失**：内存中的修改无法写入 `.mca` 文件，重启后回滚到上次成功保存的快照
2. **实体数据也可能丢失**：`ServerEntityManager` 的 `trySave()` 也走 `StorageIoWorker`，磁盘满时实体保存也会失败
3. **POI 数据丢失**：`PointOfInterestStorage` 独立保存到 `poi` 目录，磁盘满时也无法写入
4. **日志会记录异常**：`StorageIoWorker` 在 `writeChunk()` 的 `exceptionally` 中记录错误
5. **服务器不会崩溃**：保存失败被当作"非致命错误"，服务器继续运行，玩家可能完全不知道数据没保存

> [!WARNING]
> 磁盘空间不足时，服务器**不会主动告警或强制关闭**，而是静默失败。玩家可能玩了几个小时，以为进度都保存了，重启后发现全部回滚。

### 磁盘写入权限被拒绝

**触发条件**：操作系统拒绝写入 region 文件（通常是 `IOException: Permission denied`）。

**常见原因**：

1. 世界文件夹的所有者/权限被错误修改
2. SELinux / AppArmor 安全策略阻止写入
3. 文件系统被挂载为只读
4. Windows 下文件被其他程序锁定

### region 文件损坏或锁定

**触发条件**：

1. **region 文件损坏**：`.mca` 文件的头部损坏、扇区索引错误
2. **文件被其他进程锁定**：多个服务器实例误启动到同一世界目录

> [!DANGER]
> **多个服务器实例同时运行到同一世界目录**是最危险的情况——两个 `StorageIoWorker` 同时写入同一个 `.mca` 文件，导致 region 文件损坏、实体/POI 数据损坏、玩家数据损坏。

### 对比表格

| 失败类型 | 触发原因 | 服务器表现 | 数据后果 | 诊断难度 |
|---|---|---|---|---|
| **磁盘空间不足** | No space left on device | 继续运行，日志记录错误 | 所有保存失败，重启回滚 | 中 |
| **权限被拒绝** | Permission denied | 继续运行，日志记录错误 | 所有保存失败，重启回滚 | 高 |
| **region 文件损坏** | 头部损坏、扇区索引错误 | 可能继续运行，可能触发降级 | 单个 region 受影响 | 高 |
| **文件被锁定** | 多实例同时运行 | Windows 可能崩溃 | 数据竞争，严重损坏 | 极高 |

### 与 Savestate 的关系

保存失败与 **Savestate 现象** 有本质区别：

- **Savestate**：保存**成功**但**不完整**（部分区块因保存冷却未写入）
- **保存失败**：保存**失败**（IOException、权限、磁盘满），区块数据**完全未写入**

但两者可以叠加：磁盘满 + Watchdog 强退 = 部分区块成功保存 + 部分区块失败，造成更复杂的数据不一致。

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

### Java 数组长度限制的确定性 OOM

**前文讲的 OOM 路径依赖"内存压力"，但还有一条更可靠的路径：利用 Java 数组长度限制，让单个区块的序列化数据超过 2.147 GB，触发确定性 OOM。**

#### Java 数组的硬限制

Java 数组的最大长度是 `Integer.MAX_VALUE - 8 = 2,147,483,639` 个元素（约 2.147 GB）。这是 JVM 规范的硬限制，**无论服务器有多少可用内存，都无法创建更大的数组**。

当 `ChunkSerializer.serialize()` 将区块序列化为 NBT，然后 `RegionFile` 通过 GZIP 压缩后写入 `.mca` 文件时，如果压缩后的数据超过 2.147 GB，在尝试分配数组时会抛出：

```
java.lang.OutOfMemoryError: Required array length 2147483639 + 152 is too large
```

**关键点**：这个错误**不是因为内存不足**，而是因为**Java 数组长度限制**。即使服务器有 64 GB 可用内存，也会失败。

#### 触发路径

完整的触发链路：

```
ServerWorld.save() / 关服
  ↓
ThreadedAnvilChunkStorage.save(chunk)
  ↓
ChunkSerializer.serialize(world, chunk)  // 序列化为 NbtCompound
  ↓
setNbt(chunkPos, nbt)
  ↓
StorageIoWorker.setResult(chunkPos, nbt)
  ↓
worker 线程执行 writeChunk()
  ↓
RegionFile.writeChunk(pos, buf)
  ↓
GZIP 压缩 → 尝试分配超过 2.147 GB 的 byte[]
  ↓
OutOfMemoryError: Required array length ... is too large
  ↓
Future.completeExceptionally(exception)
  ↓
区块保存失败，数据未写入 .mca
```

#### 如何构造 2.147 GB 的区块数据

要让单个区块的序列化数据超过 2.147 GB，需要：

#### 预期行为与日志

**服务器表现**：

- 保存失败时，控制台会输出：
  ```
  [19:10:30] [IO-Worker-27/ERROR]: Caught exception in thread Thread[IO-Worker-27,10,main]
  java.lang.OutOfMemoryError: Required array length 2147483639 + 152 is too large
  ```
  （具体时间、数组长度、线程 ID 会不同）

- **服务器不会崩溃**，继续运行，但该区块的保存失败。
- 玩家在游戏中看不出异常，区块照常加载、运算。

**重启后**：

- 该区块的数据未写入 `.mca` 文件，或者写入了损坏的数据。
- `RegionFile.getChunkInputStream(pos)` 返回 `null` 或抛出异常。
- `ThreadedAnvilChunkStorage.loadChunk(pos)` 调用 `getProtoChunk(pos)` 创建新的 `ProtoChunk`。
- 该区块被**重新生成**为世界种子的初始状态——地形、箱子、实体全部回到生成时的样子。

#### 为什么需要"几 GB 可用内存"

虽然 OOM 是由数组长度限制触发，但**在序列化和压缩过程中**，JVM 仍然需要分配临时内存：

1. `ChunkSerializer.serialize()` 创建 `NbtCompound` 对象（可能几百 MB）
2. GZIP 压缩时创建中间缓冲区
3. `RegionFile` 尝试分配目标数组时失败

如果服务器可用内存不足（如只有几百 MB），可能在达到数组长度限制之前就因为**真正的内存不足**而 OOM，导致：
- 其他区块也保存失败（不只是目标区块）
- 服务器崩溃或 Watchdog 触发 `halt(1)`

**因此，建议有 2-4 GB 可用内存**，确保只有目标区块因数组长度限制而失败，其他区块正常保存。

#### 与"内存压力 OOM"的对比

| | 内存压力 OOM | Java 数组长度限制 OOM |
|---|---|---|
| **触发条件** | 服务器可用内存不足 | 区块序列化数据 > 2.147 GB |
| **可靠性** | 不可靠（取决于内存状态） | **确定性触发**（无论内存多少） |
| **影响范围** | 可能影响多个区块 | 只影响目标区块 |
| **所需资源** | 无特殊要求 | 需要 2-4 GB 可用内存 + 大量 NBT 数据 |
| **错误信息** | `OutOfMemoryError: Java heap space` | `OutOfMemoryError: Required array length ... is too large` |

#### 实际应用场景

这种方法常用于：
- **测试区块重新生成机制**：验证服务器在区块损坏时的降级行为。
- **清除特定区块的数据**：如果一个区块因为 bug 或误操作陷入异常状态，可以通过触发重新生成来"重置"该区块。
- **研究 Savestate 现象**：对比 Watchdog 强退（跨区块分裂）和 OOM 保存抑制（单区块重生）的不同表现。

> [!WARNING]
> 在生产服务器上使用此方法需要谨慎：
> - 目标区块的所有数据（地形、建筑、箱子、实体）会**永久丢失**。
> - 如果周围区块有依赖关系（如红石线路跨区块），可能导致不一致。
> - 建议在测试服或备份后尝试。

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

## 小结

Chunk Savestate 是以下机制共同作用的结果：

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
