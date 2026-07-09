---
slug: chunk-tacs
title: 服务端总调度器 TACS
description: ThreadedAnvilChunkStorage 的职责总览、核心表结构、三类执行器，以及 ChunkTaskPrioritySystem 的优先级调度。
index: 4
is-advanced: false
---

如果把区块管理系统比作一个工厂，`ThreadedAnvilChunkStorage`（以下简称 **TACS**）就是车间主任——它不直接生产产品，但协调着每一条生产线的运转。

## 名称解析

`ThreadedAnvilChunkStorage` 这个名称本身说明了它的职责：

- **Threaded**：线程化的——区块生成在独立线程上运行；
- **Anvil**：铁砧——指现代 Minecraft 中的 Anvil 区块存储格式（`.mca`）；
- **ChunkStorage**：区块存储——管理区块的持久化。

连起来就是"**线程化 Anvil 区块存储管理器**"。尽管多线程的使用实际上非常有限——主要是世界生成和 NBT 存取，且后者甚至不完全在 TACS 中管理。

## 职责总览

TACS 在区块管理系统中处于底层协调位置，与几乎所有其他子系统都有联系：

- **管理 `ChunkHolder` 形式的区块**：包括 `ChunkHolder` 的创建与卸载、共同管理其加载等级和状态。
- **定义并计划区块加载、生成和光照更新任务**：通过 `CompletableFuture` 将任务异步化。
- **定义向玩家发送区块数据的逻辑**：`sendWatchPackets()`、`sendChunkDataPackets()` 等方法。
- **管理实体的数据跟踪**：与客户端的数据同步，`EntityTracker` 管理实体位置追踪。
- **处理玩家里去与加入**：`handlePlayerAddedOrRemoved()` 和 `updatePosition()`。

## 核心表结构

TACS 维护了多张映射表，每张表有明确的用途：

### currentChunkHolders

```java
Long2ObjectMap<ChunkHolder> currentChunkHolders
```

当前维度中所有**存在 ChunkHolder** 的区块。key 是 `ChunkPos.toLong()` 序列化后的 `long`，value 是对应的 `ChunkHolder`。

一个 `ChunkHolder` 被创建时（即区块首次进入可访问范围时），被放入此表；被卸载时从此表移除。

### chunkHolders

```java
// ThreadedAnvilChunkStorage.java (Yarn 1.20.1)
private final Long2ObjectLinkedOpenHashMap<ChunkHolder> currentChunkHolders = new Long2ObjectLinkedOpenHashMap<>();
private volatile Long2ObjectLinkedOpenHashMap<ChunkHolder> chunkHolders = this.currentChunkHolders.clone();
```

`ThreadedAnvilChunkStorage` 确实有名为 `chunkHolders` 的字段——它是 `currentChunkHolders` 的 **volatile 快照副本**，用于 `tick()` 和 `save()` 等需要稳定遍历的场景。更新发生在主线程（Minecraft Server），而读取端无需加锁即可安全遍历，避免 `ConcurrentModificationException`。这是一种 **Copy-On-Write** 模式：写入端在主线程更新，读取端零开销。

> [!NOTE] 与 `ChunkTicketManager.chunkHolders` 的区别
> - `TACS.chunkHolders`：`Long2ObjectLinkedOpenHashMap<ChunkHolder>`，当前维度**所有**活跃 `ChunkHolder` 的完整映射表，key 为 `ChunkPos.toLong()` 序列化后的 `long`。
> - `ChunkTicketManager.chunkHolders`：`Set<ChunkHolder>`，仅收集本轮 tick 中**加载等级发生变化的** `ChunkHolder`，用于后续通知回调。

### chunksToUnload

```java
LongSet chunksToUnload
```

记录即将被卸载的区块的 `long` 坐标。当 `ChunkHolder` 的加载等级从可访问降为不可访问时，它从 `currentChunkHolders` 移入 `chunksToUnload`，等待适当时机真正卸载。

### loadedChunks

```java
// 通过 VersionedChunkStorage 间接管理
```

已从磁盘加载或已生成完毕、可供访问的区块集合。"loaded" 在此处的含义是"数据在内存中"而非"正在 tick"。

### chunkToType

```java
// 不直接作为一个独立字段存在；通过 ChunkStatus 和 ChunkLevels 共同表达
```

区块的"类型"——即它是 `ProtoChunk` 还是 `WorldChunk`——由 `ChunkStatus` 的 `ChunkType` 枚举决定：`PROTOCHUNK` 或 `LEVELCHUNK`。

## TACS 同时管理的系统

TACS 不是一个孤立的类——它在同一维度下协调着多个独立子系统：

| 子系统 | 在 TACS 中的接入点 |
|---|---|
| **区块管理器**（`ServerChunkManager`） | TACS 是它的核心字段 `this.chunkStorage` |
| **加载票管理器**（`ChunkTicketManager`） | TACS 持有 `this.ticketManager`，并在 `tick()` 中调用它 |
| **光照引擎**（`ServerLightingProvider`） | TACS 持有 `this.lightingProvider`，在区块状态变化时通知它 |
| **POI 存储**（`PointOfInterestStorage`） | TACS 持有 `this.pointOfInterestStorage`，区块保存时同步保存 POI |
| **实体跟踪**（`EntityTracker`） | TACS 持有 `this.entityTrackers`，管理实体位置追踪与客户端同步 |
| **NBT 存取**（`StorageIoWorker`） | TACS 通过 `VersionedChunkStorage` 间接管理 |

## 三类执行器

TACS 涉及三种线程/执行器：

| 执行器 | 用途 | 线程 |
|---|---|---|
| **主线程执行器**（`mainThreadExecutor`） | 执行不涉及 IO 和生成的区块管理任务 | 服务端主线程 |
| **生成线程**（`worldGenExecutor`） | 执行世界生成任务（噪声、地物、光照初始化等） | `Worker-Main-N` 线程池 |
| **光照线程**（通过 `ServerLightingProvider`） | 执行光照计算和传播 | Light Engine 专用线程 |

> [!NOTE]
> 有关三线程模型的完整创建链路（构造函数的 `TaskExecutor.create()` 调用）、`ChunkTaskPrioritySystem` 的 `createExecutor()` 封装机制，以及 `LevelPrioritizedQueue` 的优先级调度细节，参见 **05-异步任务与三线程模型** 章节。04 本章聚焦于 TACS 本身的协调职责，不重复展开。

## ChunkTaskPrioritySystem

`ChunkTaskPrioritySystem` 是 TACS 内部的优先级任务调度器。每一个维度有两个 `ChunkTaskPrioritySystem` 实例：

1. **TACS 的 `chunkTaskPrioritySystem`**：管理加载、生成、保存等与区块数据直接相关的任务。
2. **`ChunkTicketManager` 的 `levelUpdateListener`**：管理和调度加载等级更新相关的任务——当加载等级变化需要通知 `ChunkHolder` 时使用。

它维护一个按**优先级**排列的任务队列。优先级由加载等级和任务类型共同决定——等级越低（越"强"的加载）的区块，其任务优先级越高。

```java
// 任务创建示例
ChunkTaskPrioritySystem.createMessage(holder, task)
// 向主线程执行器发送，优先级由 holder 的加载等级决定
```

这种设计保证了：最需要立刻处理的区块（玩家脚下的区块，level ≤ 31）的任务总是优先于远处区块（level = 33）的任务。

## 为什么 TACS 才是区块生命周期的主脑

从上面的分析可以看出，TACS 是整个区块管理系统的事实中心：

1. **`ServerChunkManager`** 是高级抽象——把"获取一个区块"翻译为 TACS 能理解的调用；
2. **`ChunkTicketManager`** 决定"哪些区块应该加载"——但它不执行加载，只计算并广播加载等级；
3. **`ChunkHolder`** 记录某个区块的状态——但它不自发行动，行为由 TACS 驱动；
4. **`ChunkSerializer`** 负责读写磁盘——但它只在 TACS 安排时才工作。

TACS 收到的每一个 `setLevel` 回调、每一个 `tick` 调用、每一个 `save` 请求，最终都转化为对 `ChunkHolder` 状态的变更、对 `CompletableFuture` 的创建与 complete、对 `ChunkTaskPrioritySystem` 的任务提交。

## 小结

- TACS（`ThreadedAnvilChunkStorage`）是区块管理系统的底层协调中心。
- 它维护 `currentChunkHolders`、`chunksToUnload` 等核心映射表，管理 `ChunkHolder` 的全生命周期。
- 它同时协调加载票管理器、光照引擎、POI 存储、实体跟踪和 NBT 存取。
- 三类执行器（主线程、生成线程、光照线程）分工协作。
- `ChunkTaskPrioritySystem` 按加载等级优先级调度任务，保证关键区块优先处理。

## [!ADVANCED] 代码走读

### setLevel：生命周期管理的核心方法

```java
// ThreadedAnvilChunkStorage.java
ChunkHolder setLevel(long pos, int level, @Nullable ChunkHolder holder, int oldLevel) {
    // 第1条规则：如果新旧都是不可访问，什么都不做
    if (!ChunkLevels.isAccessible(oldLevel) && !ChunkLevels.isAccessible(level)) {
        return holder;  // 不可访问→不可访问：生成中的区块不需要 ChunkHolder 跟进
    }

    // 第2条规则：已有 holder 时直接更新 level
    if (holder != null) {
        holder.setLevel(level);
    }

    // 第3条规则：不可访问时标记为可卸载
    if (holder != null) {
        if (!ChunkLevels.isAccessible(level)) {
            this.unloadedChunks.add(pos);  // 只是标记，不立即卸载
        } else {
            this.unloadedChunks.remove(pos);
        }
    }

    // 第4条规则：可访问但没有 holder → 创建或从卸载队列复用
    if (ChunkLevels.isAccessible(level) && holder == null) {
        holder = this.chunksToUnload.remove(pos);  // 先尝试复用
        if (holder != null) {
            holder.setLevel(level);  // 旧 holder 换新 level，直接复活
        } else {
            holder = new ChunkHolder(new ChunkPos(pos), level, ...);  // 全新创建
        }
        this.currentChunkHolders.put(pos, holder);
    }
    return holder;
}
```

**为什么第 4 条规则要优先从 `chunksToUnload` 复用？** 一个区块可能经历"加载 → 卸载 → 再次加载"的循环（玩家来回走动）。如果每次卸载都 new、每次加载都 new，频繁的对象创建会导致 GC 压力。`chunksToUnload` 存储了尚未被真正释放的 `ChunkHolder`——它里面的 `ChunkSection[]` 可能还有数据。复用它意味着跳过构造器中的 `fillSectionArray()` 填充，直接以一个完好的 `sectionArray` 重新进入运行态。

**为什么第 1 条规则不返回 null？** 如果 holder 已存在（在不可访问范围内有生成任务进行中），`setLevel` 被调用只是更新 level 值而不销毁 holder——生成任务需要 holder 作为容器来存储生成的中间结果。返回 null 会让上层认为"这个区块不需要关注"，而实际上生成线程正在往这个 holder 里写数据。

### setLevel 的完整链路

`setLevel()` 是 TACS 的核心方法，但它不是直接被调用的。从 `ChunkTicketManager` 的票等级传播到 TACS 的 `setLevel()`，再到最终的区块生成完成，是一个完整的闭环。

#### 调用入口：谁调用了 setLevel？

不是直接调用。通过 `ThreadedAnvilChunkStorage.TicketManager.setLevel()` 间接调用：

```java
// ThreadedAnvilChunkStorage.java line 1491-1492
protected ChunkHolder setLevel(long pos, int level, @Nullable ChunkHolder holder, int i) {
    return ThreadedAnvilChunkStorage.this.setLevel(pos, level, holder, i);
}
```

`TicketManager` 是 `ChunkTicketManager` 的子类，TACS 构造函数中创建（line 197）：
```java
this.ticketManager = new ThreadedAnvilChunkStorage.TicketManager(executor, mainThreadExecutor);
```

票的等级传播 → `TicketManager.setLevel()` → TACS 的 `setLevel()`。

#### getRegion() 如何驱动生成？

`setLevel()` 创建 `ChunkHolder` 后，生成任务并不是立即开始。当外部调用 `getChunk()` 获取区块时，`getRegion()` 被调用（lines 293-362）：

- `getRegion()` 遍历以目标区块为中心、`(2*margin+1)²` 区域内的所有区块
- 对每个区块调用 `holder.getChunkAt(requiredStatus, this)`
- 返回 `CompletableFuture<List<Chunk>>`——当所有周边区块都达到 requiredStatus 时 complete

设计关键：`getRegion()` 收集的不只是目标区块，而是整个外圈依赖区域的所有区块。这是 `taskMargin` 的实现——如果周围区块不满足 status 要求，整个 Future 就不会 resolved。

#### setLevel 的完整时序

1. **票传播完成**：`TicketDistanceLevelPropagator` 更新某个区块的 level
2. **TicketManager 通知**：`TicketManager.setLevel(pos, newLevel, holder, oldLevel)` 被调用
3. **TACS.setLevel()**：创建或更新 `ChunkHolder`（4 条规则），设置为 `chunkHolderListDirty = true`
4. **ChunkHolder.setLevel(level)**：更新内部 level 值，触发 holder 的 `processLevelChange()`
5. **下次 tick**：`ticketManager.tick()` → `holder.tick(chunkStorage, executor)` → 根据新旧 ChunkLevelType 判断升降温 → 创建对应的 `CompletableFuture`（如 `getChunkAt(FULL, chunkStorage)`）
6. **异步任务执行**：`CompletableFuture` 被提交到 `ChunkTaskPrioritySystem` → 按优先级排队 → 在 worldgen/light/main 线程上执行
7. **生成完成**：`chunk.setStatus(nextStatus)` + `holder.setCompletedLevel(completedStatusIndex)` → 触发下一阶段的生成任务
8. **最终到达 FULL**：`holder.combineSavingFuture(chunk)` → 区块正式变为可访问

#### 为什么要通过 TicketManager 间接调用？

`ChunkTicketManager` 负责"哪些区块需要什么加载等级"，TACS 负责"区块的容器管理"。通过 `TicketManager.setLevel()` 这个中间层，TACS 不需要知道票的存在——它只接受 level 变化通知，然后更新容器状态。

这种职责分离让两个系统解耦：
- `ChunkTicketManager` 关注"哪些区块应该保持加载（加载票逻辑）"
- TACS 关注"如何让区块从磁盘/生成器变为可访问（生成链路）"

当票系统认为某个区块不再需要时，只需调用 `setLevel(pos, 33, holder, 31)` ——TACS 会自动处理"如何安全地卸载这个区块"的细节。反之，当票系统认为需要加载时，只需调用 `setLevel(pos, 31, null, 33)` ——TACS 会自动处理"如何创建 holder 并启动生成任务"的细节。

### chunkHolders 快照：volatile + 克隆的线程安全模式

```java
private volatile Long2ObjectLinkedOpenHashMap<ChunkHolder> chunkHolders
    = this.currentChunkHolders.clone();
```

这个字段的写入频率很低（只在需要稳定迭代时克隆），读取频率很高（每个 tick 遍历多次）。`volatile` 保证了**跨线程的可见性**——当主线程更新 `chunkHolders` 的引用时，任何读取它的线程都能立即看到新值。克隆操作本身在主线程上执行（写入端），而读取端可能在 tick 保存或 dump 等异步路径中。

这种模式是典型的**写时复制（Copy-On-Write）**：只在需要迭代时支付克隆开销，读取端零开销。

### setViewDistance：视距变化时的客户端数据重发

```java
protected void setViewDistance(int watchDistance) {
    int i = MathHelper.clamp(watchDistance, 2, 32);  // 硬限制 2~32
    if (i != this.watchDistance) {
        int oldDist = this.watchDistance;
        this.watchDistance = i;
        this.ticketManager.setWatchDistance(i);
        // 遍历所有现有 ChunkHolder
        for (ChunkHolder holder : this.currentChunkHolders.values()) {
            ChunkPos pos = holder.getPos();
            // 对每个正在观察此区块的玩家
            this.getPlayersWatchingChunk(pos, false).forEach(player -> {
                // 判断该区块在新旧视距下是否在玩家视野内
                boolean wasInRange = isWithinDistance(pos, playerPos, oldDist);
                boolean nowInRange = isWithinDistance(pos, playerPos, i);
                // 只在边界变化时发送/取消数据包
                if (wasInRange != nowInRange) {
                    this.sendWatchPackets(player, pos, ...);
                }
            });
        }
    }
}
```

注意这里的逻辑：**只在区块的"进入视野/离开视野"状态改变时才发送数据包**。如果一个区块在新旧视距下都在视野内，不会重发——避免了玩家调整视距时的网络风暴。但如果一个区块在新视距下首次进入视野，整个 `ChunkDataS2CPacket` 会被发送到客户端。

### TACS 的 tick 调度循环

TACS 的 `tick()` 不负责生成任务、不负责方块刻——它只做两件"收尾"工作：POI 数据持久化和卸载不需要的区块（回收资源 + 保存脏数据）。

#### 调用入口：ServerChunkManager.tick()

```java
// ServerChunkManager.java lines 339-351
public void tick(BooleanSupplier shouldKeepTicking, boolean tickChunks) {
    this.world.getProfiler().push("purge");
    this.ticketManager.purge();           // 1. 清理过期票
    this.tick();                          // 2. 处理票导致的 level 变化
    this.world.getProfiler().swap("chunks");
    if (tickChunks) {
        this.tickChunks();                // 3. 方块刻/实体刻
    }
    this.world.getProfiler().swap("unload");
    this.threadedAnvilChunkStorage.tick(shouldKeepTicking);  // 4. TACS tick
    this.world.getProfiler().pop();
    this.initChunkCaches();
}
```

步骤 1（`ticketManager.purge()`）清理过期票后，步骤 2（`this.tick()`）调用的不是 TACS 的 `tick()`，而是 `ServerChunkManager.tick()` 的另一个重载：

```java
// ServerChunkManager.java lines 301-310
boolean tick() {
    boolean bl = this.ticketManager.tick(this.threadedAnvilChunkStorage);
    // ticketManager.tick() 遍历 chunkHolders 集合
    // 对每个 level 变化的 holder 调用 holder.tick(chunkStorage, mainThreadExecutor)
    // holder.tick() 根据新旧 ChunkLevelType 升级/降级相应的 Future
    ...
}
```

`ChunkTicketManager.tick()` 内部（line 122）：
```java
this.chunkHolders.forEach(holder -> holder.tick(chunkStorage, this.mainThreadExecutor));
```

完整流程：
1. `ticketManager.purge()` 清理过期票 → 重新计算 level
2. 票变化导致 level 变化 → ChunkHolder 被加入 `ticketManager.chunkHolders` 集合
3. `ticketManager.tick()` 遍历集合 → 每个 holder 调用 `holder.tick()` 完成升降温
4. 区块的生成任务已被创建（通过 CompletableFuture），但尚未执行
5. 在主线程异步任务执行阶段（05 章节详解），这些 CompletableFuture 被取出执行
6. 生成完成后，`holder.setCompletedLevel()` 被调用，TACS 的 `setLevel()` 触发

#### TACS.tick() 的两个阶段

```java
// ThreadedAnvilChunkStorage.java lines 474-484
protected void tick(BooleanSupplier shouldKeepTicking) {
    Profiler profiler = this.world.getProfiler();
    profiler.push("poi");
    this.pointOfInterestStorage.tick(shouldKeepTicking);  // 阶段1：POI
    profiler.swap("chunk_unload");
    if (!this.world.isSavingDisabled()) {
        this.unloadChunks(shouldKeepTicking);              // 阶段2：卸载
    }
    profiler.pop();
}
```

**阶段 1（POI）**：`PointOfInterestStorage.tick()` 处理脏 POI 数据的保存。

**阶段 2（卸载）**：`unloadChunks(shouldKeepTicking)` 内部分三步：

```java
// ThreadedAnvilChunkStorage.java lines 500-530
private void unloadChunks(BooleanSupplier shouldKeepTicking) {
    // 步骤 2a：unloadedChunks → chunksToUnload（最多200，积压超过2000则不限）
    LongIterator longIterator = this.unloadedChunks.iterator();
    for (int i = 0; longIterator.hasNext()
        && (shouldKeepTicking.getAsBoolean() || i < 200 || this.unloadedChunks.size() > 2000);
        longIterator.remove()) {
        long l = longIterator.nextLong();
        ChunkHolder chunkHolder = this.currentChunkHolders.remove(l);
        if (chunkHolder != null) {
            this.chunksToUnload.put(l, chunkHolder);
            this.tryUnloadChunk(l, chunkHolder);
            i++;
        }
    }

    // 步骤 2b：执行卸载任务队列（最多 backlog-2000 个）
    int j = Math.max(0, this.unloadTaskQueue.size() - 2000);
    Runnable runnable;
    while ((shouldKeepTicking.getAsBoolean() || j > 0)
        && (runnable = this.unloadTaskQueue.poll()) != null) {
        j--;
        runnable.run();
    }

    // 步骤 2c：保存最多 20 个脏区块
    int k = 0;
    ObjectIterator<ChunkHolder> objectIterator = this.chunkHolders.values().iterator();
    while (k < 20 && shouldKeepTicking.getAsBoolean() && objectIterator.hasNext()) {
        if (this.save(objectIterator.next())) {
            k++;
        }
    }
}
```

**步骤 2a 的卸载条件**：
- `shouldKeepTicking = true`（不掉刻）：卸载至多 200 个区块
- `shouldKeepTicking = false`（掉刻）：如果 `unloadedChunks.size() > 2000`（积压严重），不限数量强制卸载
- 每次从 `currentChunkHolders` 移除 holder → 移入 `chunksToUnload` → 调用 `tryUnloadChunk()` 执行异步保存 + 卸载

**步骤 2b**：`unloadTaskQueue` 是 `tryUnloadChunk()` 的保存回调队列（`thenAcceptAsync(..., this.unloadTaskQueue::add)`）。如果队列积压超过 2000，强制执行以减少内存压力。

**步骤 2c**：遍历 `chunkHolders` 快照，保存最多 20 个 `needsSaving=true` 的区块。这 20 个上限是**每个 tick 的保存节流**——避免一次性保存大量区块导致硬盘 IO 峰值。

> [!NOTE]
> **设计意图**：TACS 的 `tick()` 不负责生成任务、不负责方块刻——它只做两件"收尾"工作：POI 数据持久化和卸载不需要的区块。生成任务和方块刻需要 CPU 资源，应该在其他阶段（05 章节的异步任务、ChunkHolder 的升降温）中处理。TACS 的 `tick()` 只在 tick 末尾做必要的清理和保存。

## 参考

- [Discovering Minecraft - 区块管理系统的基本架构](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTaskPrioritySystem`
- `net.minecraft.server.world.ServerChunkManager`
