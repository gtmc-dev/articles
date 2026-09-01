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

```java file=net/minecraft/server/world/ChunkHolder.java
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> accessibleFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> tickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> entityTickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
```

| Future                | 含义             | 什么时候被 complete             | 对应 ChunkLevelType |
| --------------------- | ---------------- | ------------------------------- | ------------------- |
| `accessibleFuture`    | 区块数据可供读写 | level ≤ 33 且区块已加载         | `FULL`              |
| `tickingFuture`       | 方块运算可以执行 | level ≤ 32 且周围区块达 FULL    | `BLOCK_TICKING`     |
| `entityTickingFuture` | 实体运算可以执行 | level ≤ 31 且区块已进入 ticking | `ENTITY_TICKING`    |

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

## :advanced 代码走读

### 三个 Future 的状态机设计

```java file=net/minecraft/server/world/ChunkHolder.java
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> accessibleFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> tickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
private volatile CompletableFuture<Either<WorldChunk, Unloaded>> entityTickingFuture
    = UNLOADED_WORLD_CHUNK_FUTURE;
```

三个 Future 的初始值 `UNLOADED_WORLD_CHUNK_FUTURE` 是一个**已完成的** Future，值为 `Either.right(Unloaded.INSTANCE)`。这意味着任何在初始化时尝试等待这些 Future 的代码都会**立即得到"不可用"的结果**，不会阻塞主线程。

**为什么用 `Either` 而不是 `Optional`？** `Optional<WorldChunk>` 只能表示"有"或"没有"。但 `ChunkHolder.Unloaded` 不仅仅表示"没有"——它还携带信息（如含有该区块坐标的 `toString()`，用于调试和日志）。`Either<WorldChunk, Unloaded>` 明确区分了"可用"（Left）和"不可用的原因"（Right）两种状态。在降级时，`complete(UNLOADED_WORLD_CHUNK)` 将 Right 作为完成值注入，所有等待这个 Future 的代码都能区分"因为降级而不可用"和"因为 null 而不可用"。

**为什么三个 Future 都是 `volatile`？** 因为它们的读写可能发生在不同线程上：`tick()` 在主线程执行，`CompletableFuture` 的回调（`.thenAccept()` 等）可能在主线程执行器或生成线程上运行。`volatile` 保证了写入后对其他线程立即可见，避免了"写入了新 Future，但读线程看到的还是旧值"的并发问题。

### tick() 的状态机设计：为什么检查所有三个级别

`tick()` 是整个 `ChunkHolder` 的核心——它像一个**三层温控器**，对比 `lastTickLevel` 和 `level` 来决定执行什么操作。

关键设计决策：**即使只有一个级别发生变化，`tick()` 仍然检查所有三个级别。** 这是因为加载等级的跳跃可能一次跨越多级——比如加载票被直接移除，level 可能从 31 跳到 44。在这种情况下，`tick()` 需要降级所有三个 Future，而不仅仅是 `entityTickingFuture`。

**entityTickingFuture 的升温检查：**

```java
if (!bl7 && bl8) {  // 从非 ENTITY_TICKING → ENTITY_TICKING
    if (this.entityTickingFuture != UNLOADED_WORLD_CHUNK_FUTURE) {
        throw new IllegalStateException();  // 防御性断言
    }
    this.entityTickingFuture = chunkStorage.makeChunkEntitiesTickable(this);
}
```

这里的 `if (this.entityTickingFuture != UNLOADED_WORLD_CHUNK_FUTURE)` 断言是一个**防御性编程**的例子。正常流程下，从非 ENTITY_TICKING 升温时，`entityTickingFuture` 应该处于初始状态（`UNLOADED_WORLD_CHUNK_FUTURE`）。如果它不是，说明之前的降温操作没有正确 complete 这个 Future，或者升温被错误地重复触发了——这是一种不该发生但不完全不可能发生的情况（比如由于异步任务执行顺序的异常）。抛出 `IllegalStateException` 而不是静默覆盖，**让 bug 在第一时间暴露**，而不是让一个"半完成"的 Future 继续传递下去，在不知名的地方引发更难追踪的问题。

**`accessible |= bl4` 的设计：**

```java
this.accessible |= bl4;
```

`accessible` 是一个粘性标记——一旦被设为 true，永远不会被重置为 false。即使区块的 level 从 33 降到 34，`accessible` 保持 true。它的语义是"这个区块是否曾经达到过 FULL"——用于 `ThreadedAnvilChunkStorage` 中判断是否需要在保存时处理这个区块（曾经可访问过的区块可能有 `needsSaving = true`，需要写回磁盘）。

### makeChunkTickable 的 margin=1：为什么要求周边区块

```java file=net/minecraft/server/world/ThreadedAnvilChunkStorage.java
public CompletableFuture<Either<WorldChunk, Unloaded>> makeChunkTickable(ChunkHolder holder) {
    // 等待以 holder 为中心、margin=1 范围内所有区块达到 FULL
    CompletableFuture<Either<List<Chunk>, Unloaded>> f = this.getRegion(
        holder, 1, distance -> ChunkStatus.FULL);

    return f.thenApplyAsync(chunks ->
        chunks.mapLeft(cs -> (WorldChunk) cs.get(cs.size() / 2)), ...)
        .thenApplyAsync(either -> either.ifLeft(chunk -> {
            chunk.runPostProcessing();          // 执行后处理列表中的任务
            this.world.disableTickSchedulers(chunk);  // 禁用旧的 tick 调度器
        }), this.mainThreadExecutor);
}
```

`getRegion(holder, 1, distance -> ChunkStatus.FULL)` 收集了以 holder 为中心，超出 1 个切比雪夫距离外的所有区块（即 3×3 = 9 个区块），等待它们全部达到 `FULL`。

**为什么需要周边区块？** 使一个区块进入 ticking 意味着它开始执行方块刻和随机刻——这些运算需要查询相邻区块的方块状态（例如红石信号传播、流体流动、活塞推拉检查）。如果相邻区块还在生成中（还是 `ProtoChunk`），查询结果可能是不完整的——导致红石装置行为异常。要求周边区块全部达到 FULL 保证了 tick 运算的可预测性。

注意 `chunk.runPostProcessing()` 这一行：在生成过程中，某些操作（如更新栅栏的连接状态、调整红石粉的形状）不能在第 10 阶段（SPAWN）立即执行，因为当时的相邻区块可能还不存在。这些操作被推迟到 `postProcessingLists` 中，在 `makeChunkTickable` 时集中执行——此时 3×3 范围内的区块都已完成生成，这些延迟操作有了完整的上下文。

### makeChunkEntitiesTickable 的 margin=2：为什么实体需要更大范围

与 `makeChunkTickable` 不同，`makeChunkEntitiesTickable` 对实体运算进行准备，它要求更大的范围：

```java
public CompletableFuture<Either<WorldChunk, ChunkHolder.Unloaded>> makeChunkEntitiesTickable(ChunkHolder chunk) {
    return this.getRegion(chunk, 2, distance -> ChunkStatus.FULL)
        .thenApplyAsync(either -> either.mapLeft(chunks -> (WorldChunk)chunks.get(chunks.size() / 2)), this.mainThreadExecutor);
}
```

`getRegion(holder, 2, distance -> ChunkStatus.FULL)` 收集了以 holder 为中心，超出 2 个切比雪夫距离外的所有区块（即 5×5 = 25 个区块），等待它们全部达到 `FULL`。

**为什么实体需要 5×5 而不是 3×3？** 实体运算涉及三个方面：

1. **碰撞检测**：实体移动时，其碰撞箱可能跨越多个区块。如果实体位于区块边界附近，它的碰撞检测需要查询相邻区块的方块碰撞箱和实体列表。5×5 的范围保证了即使实体在角落移动，其碰撞检测所需的所有区块都已就绪。

2. **AI 寻路**：实体的寻路系统会在多个区块范围内评估路径。如果路径经过的区块尚未加载，寻路可能失败或产生死路。5×5 的范围为大部分实体（包括玩家、生物、掉落物）的寻路提供了足够的缓冲。

3. **刷怪判定**：敌对生物的生成需要检查 5×5 区块范围内的玩家距离、光照等级和方块类型。如果你的刷怪范围只覆盖 3×3，那么靠近边缘的刷怪判定会失效——因为缺少对更远处玩家的距离判定。

因此，`makeChunkEntitiesTickable` 的 margin=2 是一个**安全边界**，它为实体运算中可能跨区块的所有查询提供了完整的上下文，避免了实体行为异常。

### getChunkAt() —— 生成请求的入口

**getChunkAt()** 是整个区块生成管线的统一入口。无论是 TACS 内部的 `getRegion()` 等待周围区块，还是 `makeChunkTickable()` 请求区块进入 ticking，还是外部代码通过 `World.getBlockState()` 间接触发，最终都收敛到：

```java
holder.getChunkAt(someStatus, chunkStorage)
```

它负责三件事：**去重**（同一区块同一 status 的多次请求共享同一个 Future），**授权**（检查 level 是否允许该 status），**调度**（委托给 TACS 启动异步生成）。

#### getChunkAt() 的核心逻辑

```java file=net/minecraft/server/world/ChunkHolder.java lines=276-299
public CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> getChunkAt(
    ChunkStatus targetStatus, ThreadedAnvilChunkStorage chunkStorage) {
    int i = targetStatus.getIndex();

    // 第一步：检查 futuresByStatus 缓存
    CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> completableFuture = this.futuresByStatus.get(i);
    if (completableFuture != null) {
        Either<Chunk, ChunkHolder.Unloaded> either = completableFuture.getNow(field_36388);
        // 防御性检查：null 值是不合法的
        if (either == null) {
            throw chunkStorage.crash(new IllegalStateException(...), "...");
        }
        // 如果是正在进行的 Future（未完成），直接返回它
        if (either == field_36388 || either.right().isEmpty()) {
            return completableFuture;
        }
        // 如果 Future 已经完成但有值（Left），说明已经生成完成但缓存没更新
        // 继续往下走到生成逻辑（会更新缓存）
    }

    // 第二步：level 允许 → 启动生成
    if (ChunkLevels.getStatus(this.level).isAtLeast(targetStatus)) {
        CompletableFuture<Either<Chunk, ChunkHolder.Unloaded>> completableFuture2 =
            chunkStorage.getChunk(this, targetStatus);
        this.combineSavingFuture(completableFuture2, "schedule " + targetStatus);
        this.futuresByStatus.set(i, completableFuture2);
        return completableFuture2;
    } else {
        // level 不允许 → 返回旧 Future 或 UNLOADED
        return completableFuture == null ? UNLOADED_CHUNK_FUTURE : completableFuture;
    }
}
```

#### 三步流程解析

**第一步：检查 futuresByStatus 缓存**

如果 `futuresByStatus.get(i)` 返回了一个已存在的 Future：

- 调用 `getNow(field_36388)` 立即获取当前值（不会阻塞）
- 如果返回的是 `field_36388`（哨兵对象，代表"Future 正在被处理中"）→ 直接返回这个 Future，多个调用者共享同一个 Future
- 如果返回的不是 `field_36388` 且 `either.right().isEmpty()`（说明 Future 已经完成且返回了 Left 值，即区块已生成）→ 直接返回这个 Future
- 如果返回 null → 触发崩溃，这是一致性断言（futuresByStatus 中不应该存储 null）
- 如果 Future 已经完成但返回了 `Unloaded`（Right 值）→ 说明之前的生成失败了，继续往下走重新启动

**第二步：判断 level 是否允许**

`ChunkLevels.getStatus(this.level).isAtLeast(targetStatus)` 检查当前加载等级是否允许目标 ChunkStatus。例如：

- level=32（BLOCK_TICKING）允许请求 FULL、FEATURES 等
- level=45（INACCESSIBLE）不允许请求任何 ChunkStatus

如果 level 不允许 → 返回缓存的 Future（如果有）或 `UNLOADED_CHUNK_FUTURE`（如果没有）。这意味着"等级不够，无法启动生成"。

**第三步：启动生成**

- 委托给 `chunkStorage.getChunk(this, targetStatus)` —— TACS 负责实际的生成调度
- 将这个 Future 通过 `combineSavingFuture()` 链接到 `savingFuture`（下一节详述）
- 写入 `futuresByStatus.set(i, completableFuture2)` 以便后续复用

#### 为什么需要第一步检查？

第 280 行的 `if (either == field_36388)` 是关键：

- `field_36388` 是一个哨兵对象，代表"Future 正在被处理中"
- 当 `getNow(defaultValue)` 返回 `field_36388` 时，说明 Future 还没完成——返回它是安全的（等待者可以注册回调）
- 如果 Future 已经完成（不是 `field_36388`），但 `either.right()` 有值（`Unloaded`）—— 说明之前的生成失败了，需要重新启动
- 如果 Future 已经完成且 `either.right().isEmpty()`（说明是 Left 值）—— 说明区块已经生成完成，直接返回这个 Future

这个检查保证了：同一区块同一 status 的多次请求**不会启动多次生成**——它们共享同一个 Future，只有一个生成任务会被提交。

#### getChunkAt() 在生成管线中的位置

`getChunkAt()` 是整个区块生成管线的**统一入口**。它的调用者包括：

- `TACS.getRegion()` —— 等待周围区块达到某个 status
- `TACS.makeChunkTickable()` —— 请求区块进入 BLOCK_TICKING
- `TACS.makeChunkEntitiesTickable()` —— 请求区块进入 ENTITY_TICKING
- `TACS.getChunk()` —— 递归生成某个 status 的区块
- 外部代码通过 `World.getBlockState()` 间接触发（如果区块不在内存中）

无论从哪条路径进入，最终都会调用 `holder.getChunkAt(someStatus, chunkStorage)`。它负责：

- **去重**：同一区块同一 status 的多次请求共享同一个 Future
- **授权**：检查 level 是否允许该 status
- **调度**：委托给 TACS 启动异步生成

### completedLevel 与 savingFuture 链

在前面的章节中，我们提到 `ChunkTaskPrioritySystem` 使用 **completedLevel** 而不是 **level** 作为优先级的依据（见 05 章）。这一节解释 `completedLevel` 的含义，以及它与 `savingFuture` 的关系。

#### completedLevel 的定义

`completedLevel` 是 `ChunkHolder` 的一个字段，表示**当前已完成的最高 ChunkStatus 的 index**：

```java file=net/minecraft/server/world/ChunkHolder.java lines=96-98
this.lastTickLevel = ChunkLevels.INACCESSIBLE + 1;
this.level = this.lastTickLevel;
this.completedLevel = this.lastTickLevel;  // 初始化为不可访问
```

```java file=net/minecraft/server/world/ChunkHolder.java lines=329-335
public int getCompletedLevel() {
    return this.completedLevel;  // 当前已完成的最高 ChunkStatus 的 index
}

private void setCompletedLevel(int level) {
    this.completedLevel = level;  // 被 chunkStatus 推进时调用
}
```

初始值为 `ChunkLevels.INACCESSIBLE + 1`，表示"尚未完成任何生成阶段"。当区块的生成阶段推进时（如从 FEATURES 进入 INITIALIZE_LIGHT），`setCompletedLevel(newStatusIndex)` 被调用。

#### completedLevel 的更新时机

当 `tick()` 推进区块的生命周期时，会调用 `levelUpdateListener.updateLevel()`：

```java file=net/minecraft/server/world/ChunkHolder.java lines=430
this.levelUpdateListener.updateLevel(
    this.pos, this::getCompletedLevel, this.level, this::setCompletedLevel
);
```

这意味着当一个区块完成了一个生成阶段：

1. `completedLevel` 增加（如从 7 到 8）
2. `levelUpdateListener` 被通知
3. `ChunkTaskPrioritySystem` 更新该区块在所有优先级队列中的位置
4. 结果：该区块的后续任务（如下一个 ChunkStatus）优先级降低（因为已经"完成度更高"）

这与 05 章节中"为什么用 completedLevel 而不是 level 作为优先级"直接相关：已经完成到 FEATURES 的区块，其后续 LIGHT 任务的优先级低于一个虽 level 更高但尚未完成 BIOMES 的区块的任务。

#### savingFuture 的链式构建

`savingFuture` 是 `ChunkHolder` 的核心 Future，初始值为已完成：

```java file=net/minecraft/server/world/ChunkHolder.java lines=57
private CompletableFuture<Chunk> savingFuture = CompletableFuture.completedFuture(null);
```

每次 `combineSavingFuture()` 调用都在链上追加一个新环节：

```java file=net/minecraft/server/world/ChunkHolder.java lines=301-315
protected void combineSavingFuture(String thenDesc, CompletableFuture<?> then) {
    if (this.actionStack != null) {
        this.actionStack.push(new ChunkHolder.MultithreadAction(Thread.currentThread(), then, thenDesc));
    }

    this.savingFuture = this.savingFuture.thenCombine(
        (CompletionStage<? extends Object>)then,
        (chunk, object) -> (Chunk)chunk
    );
}

// 重载版本：接受 Either<Chunk, Unloaded> 类型的 Future
private void combineSavingFuture(CompletableFuture<? extends Either<? extends Chunk, ChunkHolder.Unloaded>> then, String thenDesc) {
    if (this.actionStack != null) {
        this.actionStack.push(new ChunkHolder.MultithreadAction(Thread.currentThread(), then, thenDesc));
    }

    this.savingFuture = this.savingFuture.thenCombine(then, (chunk, either) ->
        either.map(chunkx -> chunkx, unloaded -> (Chunk)chunk)
    );
}
```

这个链式构建的意义：

- 区块生成管线的每个阶段（EMPTY → STRUCTURE_STARTS → ... → FULL）都会追加到 `savingFuture`
- 当所有阶段都完成时，`savingFuture` 才 complete
- 任何需要"等待这个区块彻底生成完毕"的代码，只需等待 `savingFuture`

#### savingFuture 的实际使用

当 `tick()` 推进 `accessibleFuture`/`tickingFuture`/`entityTickingFuture` 时，它们也被链接到 `savingFuture`：

```java file=net/minecraft/server/world/ChunkHolder.java
// ChunkHolder.java lines 388, 401, 418（升温时）
this.combineSavingFuture(this.accessibleFuture, "full");
this.combineSavingFuture(this.tickingFuture, "ticking");
this.combineSavingFuture(this.entityTickingFuture, "entity ticking");
```

这保证了：只有当所有生成阶段**和**所有运行视图都完成后，`savingFuture` 才会 complete。任何降级（UNLOADED）都会导致 `suchUnloaded` 异常，标记链为失败。

#### 为什么 savingFuture 需要这么复杂？

如果没有 `savingFuture`，外部代码需要自己追踪"这个区块的哪个生成阶段完成了"——这需要暴露 `futuresByStatus` 并让外部代码注册多个回调。`savingFuture` 把复杂度封装在 `ChunkHolder` 内部：外部只需要等待一个 Future，内部通过 `thenCombine` 链自动管理所有子 Future 的依赖性。

## 参考

- [Discovering Minecraft - ChunkHolder](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.server.world.ChunkHolder`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTicketManager`
- `net.minecraft.world.chunk.ChunkStatus`
