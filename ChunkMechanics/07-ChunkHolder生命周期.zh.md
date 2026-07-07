---
slug: chunk-holder-lifecycle
title: ChunkHolder 生命周期
description: 三条核心 Future 如何控制区块的三种运行视图，tick() 如何升温降温，以及从 ProtoChunk 到 WorldChunk 到卸载的完整周期。
index: 7
is-advanced: false
---

如果加载票系统是"大脑"（决定该做什么），TACS 是"手臂"（协调各部门），那么 `ChunkHolder` 就是"体温计"——它测量并响应一个区块当前的"热度"，然后精确地控制区块处于哪种运行态。

## ChunkHolder 的职责

在前几章中我们已多次提到 `ChunkHolder`。它是区块在区块管理系统中的**内部表示**（或容器）。之所以需要一个单独的 `Holder` 对象，是因为：

- 加载范围边界的区块**在内存中不一定存在**，但仍有必要存储其加载等级；
- 区块的加载等级和生成状态**对外部应该不可见**——它们是内部管理信息；
- 不同运行视图下区块的可用性**由 Holder 统一管理**，避免了"这个区块现在到底能不能用"的混乱；
- 便于对单个区块进行设定加载等级、触发加载、安排卸载等原子操作。

## 三条核心 Future

`ChunkHolder` 用三个 `CompletableFuture` 来管理区块的三种"运行视图"：

```java
// ChunkHolder.java
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> accessibleFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> tickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> entityTickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
```

| Future | 含义 | 什么时候被 complete | 对应 ChunkLevelType |
|---|---|---|---|
| `accessibleFuture` | 区块数据可供读写 | level ≤ 33 且区块已加载 | `FULL` |
| `tickingFuture` | 方块运算可以执行 | level ≤ 32 且周围区块达 FULL | `BLOCK_TICKING` |
| `entityTickingFuture` | 实体运算可以执行 | level ≤ 31 且区块已进入 ticking | `ENTITY_TICKING` |

初始值 `UNLOADED_WORLD_CHUNK_FUTURE` 是一个**已经完成**的 Future，值为 `Either.right(Unloaded.INSTANCE)`——表示"不可用"。任何等待此 Future 的代码都会立即得到一个"不可用"的结果。

> [!IMPORTANT]
> 三条 Future **逐级递进**：`entityTickingFuture` 完成时，`tickingFuture` 和 `accessibleFuture` 必然已完成。但反过来不成立——一个区块可以"可访问"但不 "ticking"，也可以 "ticking" 但不 "entity ticking"。

## tick()：升温与降温

`ChunkHolder.tick()` 是真正让区块"活过来"或"冷下去"的方法。它被 `ChunkTicketManager.tick()` 调用——只有加载等级发生变化的 `ChunkHolder` 才会被收集到 `chunkHolders` 集合并触发 `tick()`。

`tick()` 的核心逻辑是对比**上一次 tick 的 ChunkLevelType** 和**当前的 ChunkLevelType**，然后按级别差执行升温或降温操作：

```java
// ChunkHolder.tick() 简化逻辑
protected void tick(ThreadedAnvilChunkStorage chunkStorage, Executor executor) {
    ChunkLevelType oldType = ChunkLevels.getType(this.lastTickLevel);
    ChunkLevelType newType = ChunkLevels.getType(this.level);

    // 升温：从不可访问 → FULL
    if (!oldType.isAfter(FULL) && newType.isAfter(FULL)) {
        this.accessibleFuture = chunkStorage.makeChunkAccessible(this);
    }
    // 降温：从 FULL → 不可访问
    if (oldType.isAfter(FULL) && !newType.isAfter(FULL)) {
        this.accessibleFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    // 升温：从 FULL → BLOCK_TICKING
    if (!oldType.isAfter(BLOCK_TICKING) && newType.isAfter(BLOCK_TICKING)) {
        this.tickingFuture = chunkStorage.makeChunkTickable(this);
    }
    // 降温
    if (oldType.isAfter(BLOCK_TICKING) && !newType.isAfter(BLOCK_TICKING)) {
        this.tickingFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    // 升温：从 BLOCK_TICKING → ENTITY_TICKING
    if (!oldType.isAfter(ENTITY_TICKING) && newType.isAfter(ENTITY_TICKING)) {
        this.entityTickingFuture = chunkStorage.makeChunkEntitiesTickable(this);
    }
    // 降温
    if (oldType.isAfter(ENTITY_TICKING) && !newType.isAfter(ENTITY_TICKING)) {
        this.entityTickingFuture.complete(UNLOADED_WORLD_CHUNK);
    }

    this.lastTickLevel = this.level;
}
```

**升温时**，`chunkStorage` 的 `makeChunkAccessible`、`makeChunkTickable`、`makeChunkEntitiesTickable` 分别触发真正的加载、生成和 tick 挂载操作。这些方法返回的 `CompletableFuture` 会在操作完成时自动 complete——外部代码可以通过 thenAccept/handle 注册回调。

**降温时**，对应的 Future 被**立即** complete 为 `UNLOADED`。这意味着：一旦级别下降，对该视图的所有等待都立刻被通知"不可用"——不需要等到区块真正被卸载。

## 降温不等于卸载

一个重要的区分：**Future 被 complete(UNLOADED) ≠ 区块被卸载**。

降温只是关闭了某个运行视图——外部代码无法再通过 `getTickingFuture()` 等获取区块了。但区块数据可能仍然在内存中（ `accessibleFuture` 可能仍然有效），加载等级也可能只是从 31 降到了 32 而非 34。

真正的卸载发生在 `ThreadedAnvilChunkStorage.setLevel()` 中——当 level 退出 `INACCESSIBLE` 范围时：

```java
// ThreadedAnvilChunkStorage.setLevel() 简化逻辑
if (!ChunkLevels.isAccessible(level)) {
    this.unloadedChunks.add(pos);  // 加入卸载队列
}
```

## futuresByStatus：生成状态的异步视图

除了三条运行视图 Future，`ChunkHolder` 还有一个更底层的数组：

```java
private final AtomicReferenceArray<CompletableFuture<Either<Chunk, Unloaded>>> futuresByStatus;
```

这个数组按 `ChunkStatus` 的索引存储每个生成阶段的 Future。例如：

- `futuresByStatus[FULL.getIndex()]`：区块达到 FULL 的 Future
- `futuresByStatus[BIOMES.getIndex()]`：区块达到 BIOMES 的 Future

当区块的加载等级降级时，`tick()` 会将不再需要的 `futuresByStatus` 条目清空——这意味着外部的生成任务不再被等待，可以提前取消。

## 完整的生命周期

结合前七章的内容，一个区块的完整生命周期可以这样描述：

### 阶段一：无人问津（level > INACCESSIBLE）

区块不在 `currentChunkHolders` 中。没有 `ChunkHolder` 存在。对它的任何查询都会失败或返回默认值。

### 阶段二：进入视野（level ≤ INACCESSIBLE）

一张票被添加（如玩家靠近、传送门激活、`/forceload`）。`TicketDistanceLevelPropagator` 算出该区块的 level ≤ `INACCESSIBLE`。`ThreadedAnvilChunkStorage.setLevel()` 创建或复用 `ChunkHolder`，放入 `currentChunkHolders`。

### 阶段三：升温（level 逐步降低）

`ChunkTicketManager.tick()` 检测到 level 变化，将 `ChunkHolder` 加入待处理集合。`ChunkHolder.tick()` 被调用：

- 进入 **FULL**（level≤33）：TACS 安排区块生成或读盘任务。如果区块不存在于内存，先读盘；如果是全新区块，启动生成管线（`EMPTY` → `STRUCTURE_STARTS` → ... → `FULL`）。生成完成时 `accessibleFuture` 被 complete。
- 进入 **BLOCK_TICKING**（level≤32）：TACS 确保周围 1 区块内的区块均达到 FULL，然后调用 `makeChunkTickable()`——执行后处理、挂载方块 tick 调度器。`tickingFuture` 被 complete。
- 进入 **ENTITY_TICKING**（level≤31）：TACS 调用 `makeChunkEntitiesTickable()`——激活实体运算、挂载实体 tick 调度器。`entityTickingFuture` 被 complete。

### 阶段四：稳定运行

区块处于 `ENTITY_TICKING`。每 gt 中：方块刻、随机刻、实体 AI、刷怪判定全部正常执行。`needsSaving` 为 true 的区块在适当时间被保存到磁盘。

### 阶段五：降温（level 逐步升高）

票被移除（玩家走远、传送门票过期）。level 逐步上升：

- 退出 **ENTITY_TICKING**：`entityTickingFuture` 被 complete(UNLOADED)。实体运算停止，但实体数据仍在内存中。
- 退出 **BLOCK_TICKING**：`tickingFuture` 被 complete(UNLOADED)。方块刻和随机刻停止。
- 退出 **FULL**：`accessibleFuture` 被 complete(UNLOADED)。区块对外不可访问。
- 退出 **INACCESSIBLE**：`ThreadedAnvilChunkStorage.setLevel()` 将 `ChunkHolder` 加入 `unloadedChunks`。

### 阶段六：卸载

在适当的时机（保存流程中），`unloadedChunks` 中的区块被处理：

1. 如果 `needsSaving` 为 true：`ChunkSerializer` 将区块数据序列化为 NBT，通过 `StorageIoWorker` 写入 `.mca` 文件。
2. `ChunkHolder` 从 `currentChunkHolders` 中移除。
3. 区块从内存中释放。

### 阶段七：重生（再次被需要）

如果在此之后同一区块再次被请求（玩家走回来），上述流程重新开始——但这次很可能不需要重新生成，直接从磁盘加载即可。

## 为什么区块不是简单的 loaded / unloaded

现在我们可以回答这个核心问题了。

"Minecraft 中的区块要么被加载了，要么没有"——这是一个过于简单的二分法。真相是：

- 一个区块可以**存在于内存中**（有 `ChunkHolder`），但**不可访问**（level = 34）；
- 一个区块可以**可访问**，但**方块不运算**（level = 33）；
- 一个区块可以**方块在运算**，但**实体不移动**（level = 32）；
- 一个区块可以**一切正常**（level ≤ 31）；
- 一个区块可以**生成只完成了一半**（`ChunkStatus = CARVERS`），但仍然在内存中；
- 一个区块可以**三个 Future 都 UNLOADED**，但数据还没来得及写盘。

这种精细的分层机制保证了 Minecraft 在"加载玩家周围的一切"和"节省资源"之间取得了平衡。

## 小结

- `ChunkHolder` 是区块在管理系统中的内部容器，三条 `CompletableFuture` 控制三种运行视图。
- `tick()` 通过对比新旧 `ChunkLevelType` 执行升温和降温——温度的变化由加载票系统驱动。
- 降温不等于卸载：Future 被 complete(UNLOADED) 后运算停止，但数据可能仍在内存中等待保存。
- 完整生命周期：无人问津 → 进入视野 → 升温 → 稳定 → 降温 → 卸载 → 重生。
- 区块的"存在"不是二元的——它是一个由 level 和 ChunkStatus 共同决定的精细光谱。

## [!ADVANCED] 源码验证

### ChunkHolder 的三个 Future

```java
// ChunkHolder.java
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> accessibleFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> tickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> entityTickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;

// 初始值 UNLOADED_WORLD_CHUNK_FUTURE 是一个已完成 Future：
// = CompletableFuture.completedFuture(Either.right(Unloaded.INSTANCE))
```

### tick() 完整源码（关键片段）

```java
// ChunkHolder.tick()
protected void tick(ThreadedAnvilChunkStorage chunkStorage, Executor executor) {
    ChunkStatus chunkStatus = ChunkLevels.getStatus(this.lastTickLevel);
    ChunkStatus chunkStatus2 = ChunkLevels.getStatus(this.level);
    ChunkLevelType chunkLevelType = ChunkLevels.getType(this.lastTickLevel);
    ChunkLevelType chunkLevelType2 = ChunkLevels.getType(this.level);

    // ===== 降温：清空不再需要的 futuresByStatus =====
    boolean bl = ChunkLevels.isAccessible(this.lastTickLevel);
    boolean bl2 = ChunkLevels.isAccessible(this.level);
    if (bl) {
        for (int i = bl2 ? chunkStatus2.getIndex() + 1 : 0;
             i <= chunkStatus.getIndex(); i++) {
            if (this.futuresByStatus.get(i) == null) {
                this.futuresByStatus.set(i,
                    CompletableFuture.completedFuture(Either.right(new Unloaded() {...})));
            }
        }
    }

    // ===== 升温/降温: accessible → FULL =====
    boolean bl3 = chunkLevelType.isAfter(ChunkLevelType.FULL);
    boolean bl4 = chunkLevelType2.isAfter(ChunkLevelType.FULL);
    this.accessible |= bl4;
    if (!bl3 && bl4) {
        this.accessibleFuture = chunkStorage.makeChunkAccessible(this);
    }
    if (bl3 && !bl4) {
        this.accessibleFuture.complete(UNLOADED_WORLD_CHUNK);
        this.accessibleFuture = UNLOADED_WORLD_CHUNK_FUTURE;
    }

    // ===== 升温/降温: ticking → BLOCK_TICKING =====
    boolean bl5 = chunkLevelType.isAfter(ChunkLevelType.BLOCK_TICKING);
    boolean bl6 = chunkLevelType2.isAfter(ChunkLevelType.BLOCK_TICKING);
    if (!bl5 && bl6) {
        this.tickingFuture = chunkStorage.makeChunkTickable(this);
    }
    if (bl5 && !bl6) {
        this.tickingFuture.complete(UNLOADED_WORLD_CHUNK);
        this.tickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
    }

    // ===== 升温/降温: entityTicking → ENTITY_TICKING =====
    boolean bl7 = chunkLevelType.isAfter(ChunkLevelType.ENTITY_TICKING);
    boolean bl8 = chunkLevelType2.isAfter(ChunkLevelType.ENTITY_TICKING);
    if (!bl7 && bl8) {
        if (this.entityTickingFuture != UNLOADED_WORLD_CHUNK_FUTURE) {
            throw new IllegalStateException();  // 防御：不允许从非 UNLOADED 状态升温
        }
        this.entityTickingFuture = chunkStorage.makeChunkEntitiesTickable(this);
    }
    if (bl7 && !bl8) {
        this.entityTickingFuture.complete(UNLOADED_WORLD_CHUNK);
        this.entityTickingFuture = UNLOADED_WORLD_CHUNK_FUTURE;
    }

    // ===== 完成：更新 completedLevel =====
    this.levelUpdateListener.updateLevel(
        this.pos, this::getCompletedLevel, this.level, this::setCompletedLevel);
    this.lastTickLevel = this.level;
}
```

### makeChunkTickable 的周边依赖

```java
// ThreadedAnvilChunkStorage.java
public CompletableFuture<Either<WorldChunk, Unloaded>> makeChunkTickable(ChunkHolder holder) {
    // 确保周围 margin=1 区块距离内的区块都达到 FULL
    CompletableFuture<Either<List<Chunk>, Unloaded>> f = this.getRegion(holder, 1,
        distance -> ChunkStatus.FULL);
    return f.thenApplyAsync(chunks ->
        chunks.mapLeft(cs -> (WorldChunk) cs.get(cs.size() / 2)), ...)
        .thenApplyAsync(either -> either.ifLeft(chunk -> {
            chunk.runPostProcessing();
            this.world.disableTickSchedulers(chunk);
        }), this.mainThreadExecutor);
}
```

`getRegion(holder, 1, ...)` 中的 `margin=1` 要求：要使中心区块进入 ticking，其周围 1 个切比雪夫距离内的所有区块必须达到 `FULL`。如果达不到，`makeChunkTickable` 返回的 Future 不会 complete——该区块的 ticking 被挂起，等待异步任务队列再次推动。

## 参考

- [Discovering Minecraft - ChunkHolder](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.server.world.ChunkHolder`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTicketManager`
