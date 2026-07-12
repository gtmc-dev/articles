---
slug: pendingchunk-phenomenon
title: PendingChunk 现象
description: ServerEntityManager 的 PENDING 状态卡住后，计划刻失效、实体卸载重试与关服 flush 死锁的现象链路。
index: 21
is-advanced: true
---

## PendingChunk 现象

### `ServerEntityManager` 的三状态机

实体数据独立保存到 `entities` 区域文件中，由 **`ServerEntityManager`** 管理。它用 **`managedStatuses`** 记录每个 chunk 的实体数据加载状态（`ServerEntityManager.java`:54、62、512-516）：

```java
private final Long2ObjectMap<ServerEntityManager.Status> managedStatuses = new Long2ObjectOpenHashMap<>();

this.managedStatuses.defaultReturnValue(ServerEntityManager.Status.FRESH);

enum Status {
    FRESH,
    PENDING,
    LOADED;
}
```

三个状态分别表示：

| 状态      | 含义                                                |
| --------- | --------------------------------------------------- |
| `FRESH`   | 默认状态，实体管理器还没有读过这个 chunk 的实体数据 |
| `PENDING` | 已经发起异步读取，正在等待实体数据返回              |
| `LOADED`  | 实体数据已经读入内存，或确认该 chunk 没有实体数据   |

正常转换路径是 `FRESH -> scheduleRead() -> PENDING -> loadChunks() -> LOADED`。这个状态机的设计重点是避免覆盖旧实体数据：`FRESH` 必须先读，再合并或确认，再保存。问题也出在这里：`PENDING` 是等待态，如果异步读取永远不完成，它没有自动超时和回滚。

### `scheduleRead()`：从 `FRESH` 到 `PENDING`

当保存或卸载路径遇到 `FRESH` 状态时，会调用 **`scheduleRead()`**（`ServerEntityManager.java`:261-268）：

```java
private void scheduleRead(long chunkPos) {
    this.managedStatuses.put(chunkPos, ServerEntityManager.Status.PENDING);
    ChunkPos chunkPos2 = new ChunkPos(chunkPos);
    this.dataAccess.readChunkData(chunkPos2)
        .thenAccept(this.loadingQueue::add)
        .exceptionally(throwable -> {
            LOGGER.error("Failed to read chunk {}", chunkPos2, throwable);
            return null;
        });
}
```

关键点是第一行：状态会立刻从 `FRESH` 变成 **`PENDING`**（`ServerEntityManager.java`:262），不等 `readChunkData()` 完成，也不等数据进入 **`loadingQueue`**。

读取成功时，`thenAccept(this.loadingQueue::add)` 会把结果加入队列；读取以异常形式完成时，`exceptionally` 只记录日志（`ServerEntityManager.java`:265-267），不会把状态改回 `FRESH`，也不会标成失败态。

> [!WARNING]
> OOM 路径的严重性在于：它不一定表现成一个普通可恢复异常。如果读取任务在内存耗尽时中断，外层 Future 可能没有成功进入 `thenAccept`，也没有以这里能处理的方式完成。此时 `PENDING` 就会变成永久等待。

一旦发生这种情况，`loadingQueue` 收不到数据包，`loadChunks()` 没机会改成 `LOADED`，`trySave()` 和 `isLoaded()` 也会一直看到 `PENDING`。

### `loadChunks()`：从 `PENDING` 到 `LOADED`

`ServerEntityManager.tick()` 每 gt 会调用 **`loadChunks()`**，把已经读完的实体数据挂进管理器（`ServerEntityManager.java`:289-295）：

```java
private void loadChunks() {
    ChunkDataList<T> chunkDataList;
    while ((chunkDataList = this.loadingQueue.poll()) != null) {
        chunkDataList.stream().forEach(entity -> this.addEntity((T)entity, true));
        this.managedStatuses.put(chunkDataList.getChunkPos().toLong(), ServerEntityManager.Status.LOADED);
    }
}
```

只要 `loadingQueue.poll()` 取到了数据包，它就把包里的实体逐个 `addEntity()`，然后把对应 chunk 标为 `LOADED`（`ServerEntityManager.java`:293）。如果 `poll()` 返回 `null`，循环直接结束；它不会扫描所有 `PENDING` 状态，也不会主动检查某个 Future 是否超时。

### 计划刻的完整检查链

计划刻失效看起来像方块运行级别问题，但在这条异常路径中，真正卡住的是实体加载状态。`WorldTickScheduler.collectTickableChunkTickSchedulers()` 在收集到期计划刻时，会先检查 **`tickingFutureReadyPredicate`**（`WorldTickScheduler.java`:102-126）：

```java
if (m <= time) {
    ChunkTickScheduler<T> chunkTickScheduler = this.chunkTickSchedulers.get(l);
    OrderedTick<T> orderedTick = chunkTickScheduler.peekNextTick();
    if (orderedTick.triggerTick() > time) {
        entry.setValue(orderedTick.triggerTick());
    } else if (this.tickingFutureReadyPredicate.test(l)) {
        objectIterator.remove();
        this.tickableChunkTickSchedulers.add(chunkTickScheduler);
    }
}
```

如果 `this.tickingFutureReadyPredicate.test(l)` 返回 `false`，该区块的 `ChunkTickScheduler` 不会进入 `tickableChunkTickSchedulers`。`ServerWorld` 提供的 predicate 是 **`isTickingFutureReady()`**（`ServerWorld.java`:1766-1768）：

```java
private boolean isTickingFutureReady(long chunkPos) {
    return this.isChunkLoaded(chunkPos) && this.chunkManager.isTickingFutureReady(chunkPos);
}
```

第一半是 **`isChunkLoaded()`**（`ServerWorld.java`:1762-1764）：

```java
public boolean isChunkLoaded(long chunkPos) {
    return this.entityManager.isLoaded(chunkPos);
}
```

最后落到 **`ServerEntityManager.isLoaded()`**（`ServerEntityManager.java`:362-364）：

```java
public boolean isLoaded(long chunkPos) {
    return this.managedStatuses.get(chunkPos) == ServerEntityManager.Status.LOADED;
}
```

完整链条是 `WorldTickScheduler.collectTickableChunkTickSchedulers()` -> `ServerWorld.isTickingFutureReady()` -> `ServerWorld.isChunkLoaded()` -> `ServerEntityManager.isLoaded()` -> `managedStatuses == LOADED`。如果该 chunk 卡在 `PENDING`，最底层返回 `false`，区块层计划器不会被加入本刻可执行队列。

> [!IMPORTANT]
> 这里的“计划刻失效”不是计划刻对象消失。它们仍可能留在区块层计划器中，只是世界层计划器每次收集时都因为 `isChunkLoaded()` 为 false 而跳过。

### 实体卸载的重试路径与崩服链路

**正常 tick 中的重试路径**

实体数据的卸载走 `ServerEntityManager` 的独立路径,与 TACS 的区块卸载是两条独立的线。

`updateTrackingStatus` 将 HIDDEN 状态 chunk 加入待卸载队列(`ServerEntityManager.java`:184-192):

```java
public void updateTrackingStatus(long chunkPos, EntityTrackingStatus trackingStatus) {
    EntityTrackingStatus entityTrackingStatus = this.trackingStatuses.put(chunkPos, trackingStatus);
    if (trackingStatus == EntityTrackingStatus.HIDDEN) {
        this.pendingUnloads.add(chunkPos);
    } else {
        this.pendingUnloads.remove(chunkPos);
    }
    this.updateLoadStatus(chunkPos);
}
```

每 gt 的 `unloadChunks()` 尝试处理 `pendingUnloads`(`ServerEntityManager.java`:285-286):

```java
private void unloadChunks() {
    this.pendingUnloads.removeIf(pos ->
        this.trackingStatuses.get(pos) != EntityTrackingStatus.HIDDEN || this.unload(pos));
}
```

`unload()` 内部调用 `trySave()`,如果 PENDING 则返回 false(`ServerEntityManager.java`:270-278):

```java
private boolean unload(long chunkPos) {
    if (!this.trySave(chunkPos, entity -> entity.setChangeListener(EntityChangeListener.NONE))) {
        return false;
    }
    EntityTrackingSection<T> section = this.trackingSections.remove(chunkPos);
    if (section != null) {
        section.forEach(entity -> this.stopTicking(entity));
    }
    this.managedStatuses.remove(chunkPos);
    return true;
}
```

两条线的对比:

|              | 区块数据卸载(TACS)                    | 实体数据卸载(ServerEntityManager)                          |
| ------------ | ------------------------------------- | ---------------------------------------------------------- |
| 触发         | `setLevel()` → `unloadedChunks`       | `ChunkHolder.tick()` 降温 → `updateTrackingStatus(HIDDEN)` |
| 卸载入口     | `unloadChunks()` → `tryUnloadChunk()` | `pendingUnloads.removeIf(..., this::unload(pos))`          |
| 保存检查     | `chunk.needsSaving()`                 | `trySave()` → 检查 `managedStatuses`                       |
| PENDING 影响 | ❌ 不受影响(正常完成)                 | ✅ 返回 false,留在 `pendingUnloads` 重试                   |
| 设计意图     | 保证区块数据一致写盘                  | 保证实体数据不被覆盖旧数据                                 |

**正常 tick 中 PENDING 的表现**:

- 区块方块数据和流体照常保存,`ChunkHolder` 正常从 `currentChunkHolders` 移除
- 每 gt 实体管理器尝试保存,但因为 `trySave()` 返回 false,一直失败
- 这是一个**无上限的重试循环**:chunk 永远留在 `pendingUnloads` 中,每 gt 浪费一次 `trySave()` 调用

**崩服链路:save-all / 关服时的 flush()**

问题爆发在需要**强制完成所有实体 IO** 的时候。`ServerEntityManager.flush()` 在 save-all 和关服时被调用(lines 325-338):

```java
public void flush() {
    LongSet longSet = this.getLoadedChunks();

    while (!longSet.isEmpty()) {
        this.dataAccess.awaitAll(false);       // 等待所有 pending IO 完成
        this.loadChunks();                     // 处理加载完成的实体数据
        longSet.removeIf(pos -> {
            boolean bl = this.trackingStatuses.get(pos) == EntityTrackingStatus.HIDDEN;
            return bl ? this.unload(pos) : this.trySave(pos, entity -> {});
        });
    }

    this.dataAccess.awaitAll(true);             // 等待所有排队的写入完成
}
```

`flush()` 的 `while (!longSet.isEmpty())` 循环处理所有 LOADED 状态的 chunk。但 `getLoadedChunks()` 只收集 `managedStatuses == LOADED` 的条目,不收集 PENDING。所以**这个循环不会直接卡在 PENDING chunk 上**。

**真正的死锁发生在 `dataAccess.awaitAll(false)`**:

- `EntityChunkDataAccess.readChunkData(chunkPos)` 返回的 `CompletableFuture` 由 `StorageIoWorker` 管理
- 如果 OOM 导致 IO worker 线程崩溃或任务被拒绝,该 Future **永不 complete**
- `flush()` 中 line 329: `this.dataAccess.awaitAll(false)` 会**阻塞等待所有 pending IO**
- 卡住的 PENDING IO 导致 `awaitAll()` 永远不返回
- 调用 `flush()` 的线程被阻塞 → 关服流程卡住
- Watchdog 检测到卡死 → `halt(1)`

**所以 "区块卸载触发崩服" 的完整链路是**:

```
OOM 导致 entity readChunkData Future 永不 complete
  ↓
managedStatuses.put(pos, PENDING)  // 已在 scheduleRead 中设置
  ↓
正常 tick:ChunkHolder 降温 → HIDDEN → pendingUnloads → 重试 unload(失败但不崩溃)
  ↓
关服/save-all:ServerEntityManager.flush() 被调用
  ↓
while (!longSet.isEmpty()) { ... }  // 正常处理 LOADED chunks
  ↓
this.dataAccess.awaitAll(false)
  ↓
卡在 PENDING chunk 的未完成 IO Future 上
  ↓
主线程阻塞,Watchdog 检测
  ↓
halt(1)
```

> [!WARNING]
> **为什么这是 "PendingChunk 更常见的用法"**:
>
> 相比正常用户关服遇到死锁(被动触发),更常见的场景是**主动构造 PENDING 状态**(通过 OOM 或实体数据损坏),然后:
>
> 1. 触发 save-all 或关服命令
> 2. `flush()` 被调用 → 阻塞在 `awaitAll()`
> 3. 服务器 "卡死" → Watchdog 强制 `halt(1)`
> 4. 实现强制崩服
>
> 这与 `shouldDelayShutdown()` 死锁(line 492 的 `currentChunkHolders`)是**两条不同的死锁路径**:
>
> - `shouldDelayShutdown()` 死锁:区块数据路径(currentChunkHolders 清不空)
> - `flush()` 死锁:实体数据路径(IO Future 永不 complete)

### `PENDING` 卡住的现象

当一个区块的实体状态永久停在 `PENDING`，表面现象会非常混合：

| 方面           | 表现                                                                    |
| -------------- | ----------------------------------------------------------------------- |
| 方块数据       | 通常仍可见,玩家可能还能交互                                             |
| 方块刻         | 取决于区块运行级别和具体路径                                            |
| 计划刻         | 永久无法通过 `tickingFutureReadyPredicate`                              |
| 流体更新       | 依赖计划刻,可能停止扩散                                                 |
| 红石元件       | 中继器、比较器、活塞等行为会停住                                        |
| 随机刻         | 仍受区块运行级别控制,但常与异常区域一起出现                             |
| 实体           | 可能部分缺失或完全未挂载                                                |
| 保存           | `trySave()` 遇到 `PENDING` 返回 `false`                                 |
| 卸载(区块数据) | ✅ 正常完成(`tryUnloadChunk` → `save(chunk)` → `chunksToUnload.remove`) |
| 卸载(实体数据) | ❌ 每 gt 重试失败(`pendingUnloads` 中永久保留,`unload()` 返回 false)    |
| 关服/save-all  | ❌ `flush()` 阻塞在 `dataAccess.awaitAll()` → Watchdog → halt(1)        |

玩家最容易观察到的是红石和流体：中继器到点不翻转、比较器输出不更新、活塞不按预约时间推出或收回，水和岩浆也可能停止继续流动。这里的区块可能看起来仍然加载着，真正失败的是计划刻执行前的实体加载就绪检查。

### 为什么只影响计划刻链

几类 tick 的检查条件不同：

| 机制          | 主要检查                                               | 是否经过 `ServerWorld.isChunkLoaded()` |
| ------------- | ------------------------------------------------------ | -------------------------------------- |
| 计划刻        | `tickingFutureReadyPredicate`                          | 是                                     |
| 方块实体 tick | `WorldChunk` 内部 tickers 与运行级别                   | 否                                     |
| 随机刻        | `ServerWorld.tickChunk()` 与方块 ticking 资格          | 否                                     |
| 实体 tick     | `ChunkLevels.shouldTickEntities(level)` 与实体追踪状态 | 否                                     |

计划刻的特殊之处在于，`ServerWorld.isTickingFutureReady()` 在检查 chunk manager 的 ticking future 之前，先检查 `this.isChunkLoaded(chunkPos)`。这个名字容易误导：它不是问“方块区块对象是否存在”，而是问实体管理器是否认为该 chunk 的实体数据已经 `LOADED`。一旦实体加载状态没有从 `PENDING` 走到 `LOADED`，计划刻会被永久挡在门外。

### 为什么 `trySave()` 无法保存 `PENDING` 区块

保存路径进一步放大了问题。`ServerEntityManager.trySave()` 的开头会先检查状态（`ServerEntityManager.java`:235-259）：

```java
ServerEntityManager.Status status = this.managedStatuses.get(chunkPos);
if (status == ServerEntityManager.Status.PENDING) {
    return false;
}
...
} else if (status == ServerEntityManager.Status.FRESH) {
    this.scheduleRead(chunkPos);
    return false;
} else {
    this.dataAccess.writeChunkData(new ChunkDataList<>(new ChunkPos(chunkPos), list));
    list.forEach(action);
    return true;
}
```

`PENDING` 的处理是最硬的：直接 `return false`（`ServerEntityManager.java`:237）。它不会保存，也不会卸载，也不会把状态改成失败。

这是合理的保守策略：`PENDING` 表示磁盘实体数据还没读完，此时写盘可能覆盖尚未读出的实体数据。但在 OOM 卡死路径中，它会变成死锁式等待。

### 关服时的主线程死锁与 `halt(1)`

**`PENDING` 状态卡住的最严重后果不是计划刻失效，而是关服时导致主线程死锁，最终触发 Watchdog 强制 `Runtime.getRuntime().halt(1)`。**

当服务器执行关服流程时，`MinecraftServer` 会进入一个等待循环（`MinecraftServer.java`:634-643）：

```java
while (this.worlds.values().stream().anyMatch(
    world -> world.getChunkManager().threadedAnvilChunkStorage.shouldDelayShutdown())) {
    this.timeReference = Util.getMeasuringTimeMs() + 1L;

    for (ServerWorld serverWorld2 : this.getWorlds()) {
        serverWorld2.getChunkManager().removePersistentTickets();
        serverWorld2.getChunkManager().tick(() -> true, false);
    }

    this.runTasksTillTickEnd();
}
```

只要 `shouldDelayShutdown()` 返回 `true`，服务器就会继续 tick 区块管理器，试图清空所有待卸载任务。**这个循环没有超时机制**——它会一直运行，直到所有维度的 `ThreadedAnvilChunkStorage` 报告"可以关服"。

`shouldDelayShutdown()` 的检查项（`ThreadedAnvilChunkStorage.java`:489-498）：

```java
public boolean shouldDelayShutdown() {
    return this.lightingProvider.hasUpdates()
        || !this.chunksToUnload.isEmpty()
        || !this.currentChunkHolders.isEmpty()  // line 492：关键！
        || this.pointOfInterestStorage.hasUnsavedElements()
        || !this.unloadedChunks.isEmpty()
        || !this.unloadTaskQueue.isEmpty()
        || this.chunkTaskPrioritySystem.shouldDelayShutdown()
        || this.ticketManager.shouldDelayShutdown();
}
```

**line 492 是死锁的根源**：`!this.currentChunkHolders.isEmpty()`。一个 `ChunkHolder` 要从 `currentChunkHolders` 移除，必须先被成功卸载。但实体数据的卸载与区块数据的卸载走**两条独立的路径**。

**区块数据的卸载**（由 TACS 的 `unloadChunks()` 处理）不受 PENDING 状态影响：

- `tryUnloadChunk()` → `save(chunk)` 保存区块 NBT → `chunksToUnload.remove(pos, holder)`
- `ServerWorld.unloadEntities(chunk)` **不调用 `ServerEntityManager.unload()`**——它只执行 `chunk.clear()` 和 `chunk.removeChunkTickSchedulers()`
- 区块数据路径可以正常完成

**实体数据的卸载**（由 `ServerEntityManager` 独立处理）受 PENDING 状态阻塞：

- `ChunkHolder.tick()` 降温 → `updateTrackingStatus(pos, HIDDEN)` → 加入 `pendingUnloads`
- 每 gt `unloadChunks()` 尝试 `unload(pos)` → 内部调用 `trySave()` → PENDING 返回 false
- **chunk 留在 `pendingUnloads` 中，下个 tick 重试**

此时 Watchdog 线程会检测到主线程卡死超过 60 秒（默认），触发强制退出（`DedicatedServerWatchdog.java`:90-103）：

```java
private void shutdown() {
    try {
        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                Runtime.getRuntime().halt(1);  // 10 秒后强制 halt
            }
        }, 10000L);
        System.exit(1);  // 尝试正常退出
    } catch (Throwable throwable) {
        Runtime.getRuntime().halt(1);  // 如果 exit 也失败，立即 halt
    }
}
```

`halt(1)` 是**立即终止 JVM**，不执行 shutdown hooks，不 flush 缓冲区，不等待线程结束。相比正常关服（flush 所有数据、等待异步任务、生成完整日志），`halt(1)` 相当于进程被 `kill -9`。

> [!DANGER]
> **`PENDING` 状态卡住 + 关服 = 数据丢失高风险**
>
> 当 Watchdog 触发 `halt(1)` 时：
>
> - 所有 `PENDING` 区块的实体数据**未保存**（`trySave` 返回 false）
> - 所有 `chunksToUnload` 中的区块可能**未写盘**
> - 所有 `unloadTaskQueue` 中的异步保存任务**被中断**
> - POI 数据、光照数据、区块 NBT 可能**部分写入**（缓冲区未 flush）
>
> 重启后，这些区块会回滚到上次成功保存的快照，或者触发新的加载失败。如果 OOM 问题未解决，可能进入**加载 → `PENDING` 卡住 → 关服死锁 → `halt(1)` → 重启 → 再次卡住**的死循环。

**诊断标志**：

服务器日志中出现以下特征时，可能是 `PENDING` 死锁：

| 阶段          | 日志特征                                                        |
| ------------- | --------------------------------------------------------------- |
| 关服开始      | "Stopping server" / "Saving worlds..."                          |
| 卡住表现      | 控制台长时间无新日志输出，CPU 单核占用 100%                     |
| Watchdog 警告 | "A single server tick took X seconds (should be max 0.05)"      |
| 强制退出      | "Forcing crash... Server has stopped responding" 或进程直接退出 |
| 重启后        | 某些区块的红石、流体、计划刻仍然失效                            |

**完整的死锁链路**：

```
关服命令执行
  ↓
MinecraftServer 进入关服等待循环 (line 634)
  ↓
while (shouldDelayShutdown() == true) {
    tick 区块管理器...
}  ← 无超时机制
  ↓
shouldDelayShutdown() 检查 currentChunkHolders
  ↓
!currentChunkHolders.isEmpty() == true (line 492)
  ↓
为什么不为空？
  ↓
unloadChunks() → tryUnloadChunk() → save(chunk)
  ↓
ServerEntityManager.trySave(chunkPos)
  ↓
if (status == PENDING) { return false; } (line 237)
  ↓
保存失败 → 卸载失败 → ChunkHolder 无法移除
  ↓
shouldDelayShutdown() 永远返回 true
  ↓
主线程陷入无限循环（持续 tick 但永远清不空）
  ↓
Watchdog 检测到主线程卡死超过 60 秒
  ↓
DedicatedServerWatchdog.shutdown() 触发
  ↓
Runtime.getRuntime().halt(1) (line 96)
  ↓
JVM 立即终止，不执行 shutdown hooks，不 flush 缓冲区
  ↓
数据丢失：PENDING 区块实体未保存、chunksToUnload 未写盘、
异步任务被中断、POI/光照/NBT 可能部分写入
```

### `scheduleRead()` 的 Future 链

`ServerEntityManager.scheduleRead()` 的危险点是状态更新早于 I/O 完成：

```java
this.managedStatuses.put(chunkPos, ServerEntityManager.Status.PENDING);
this.dataAccess.readChunkData(chunkPos2)
    .thenAccept(this.loadingQueue::add)
    .exceptionally(throwable -> {
        LOGGER.error("Failed to read chunk {}", chunkPos2, throwable);
        return null;
    });
```

`exceptionally` 不把状态改回 `FRESH`，是因为失败后立即重试也不一定安全：下一次保存会再次发起读取，可能形成频繁重试，甚至在半损坏数据上引发覆盖风险。源码选择记录异常并保持当前状态，但在 OOM 或 Future 未完成时，这会留下永久 `PENDING` 的窗口。

### `loadChunks()` 的批处理循环

`loadChunks()` 没有 Future 列表，它只消费队列：

```java
while ((chunkDataList = this.loadingQueue.poll()) != null) {
    chunkDataList.stream().forEach(entity -> this.addEntity((T)entity, true));
    this.managedStatuses.put(chunkDataList.getChunkPos().toLong(), ServerEntityManager.Status.LOADED);
}
```

状态推进的唯一证据是 `ChunkDataList` 已经进入 `loadingQueue`。如果 `poll()` 为 `null`，函数不会扫描 `managedStatuses` 找出 `PENDING` 项；`PENDING` 本身不会主动过期，只有队列里的数据包能把它改成 `LOADED`。

### `tickingFutureReadyPredicate` 的完整调用栈

计划刻从世界层进入执行队列前，会经过以下源码路径：

```java
// WorldTickScheduler.java lines 119-122
} else if (this.tickingFutureReadyPredicate.test(l)) {
    objectIterator.remove();
    this.tickableChunkTickSchedulers.add(chunkTickScheduler);
}
```

这个 predicate 来自 `ServerWorld`：

```java
// ServerWorld.java lines 1766-1768
private boolean isTickingFutureReady(long chunkPos) {
    return this.isChunkLoaded(chunkPos) && this.chunkManager.isTickingFutureReady(chunkPos);
}
```

前半段继续进入：

```java
// ServerWorld.java lines 1762-1764
public boolean isChunkLoaded(long chunkPos) {
    return this.entityManager.isLoaded(chunkPos);
}
```

最后是实体管理器状态：

```java
// ServerEntityManager.java lines 362-364
public boolean isLoaded(long chunkPos) {
    return this.managedStatuses.get(chunkPos) == ServerEntityManager.Status.LOADED;
}
```

所以，计划刻是否能执行，不只取决于 `ChunkHolder.getTickingFuture()` 是否已经得到 `WorldChunk`，还取决于实体管理器是否认为该 chunk 已 `LOADED`；`PENDING` 在这里和 `FRESH` 一样都会返回 `false`。

### 为什么 `trySave()` 无法保存 `PENDING` 区块

`trySave()` 的第一道门就是：

```java
ServerEntityManager.Status status = this.managedStatuses.get(chunkPos);
if (status == ServerEntityManager.Status.PENDING) {
    return false;
}
```

这行提前返回阻止了后面所有保存逻辑，也会让 `unload(chunkPos)` 失败，因为 `unload()` 只有在 `trySave()` 返回 true 后才会移除管理状态。从数据安全角度看，这是正确的；从运行稳定性角度看，一旦读取 Future 不再推进，保存、卸载、计划刻就会一起被这个状态卡住。

## 参考

- `net.minecraft.server.world.ServerEntityManager`
- `net.minecraft.server.world.ServerWorld`
- `net.minecraft.world.tick.WorldTickScheduler`
