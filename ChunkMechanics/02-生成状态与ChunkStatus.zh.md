---
slug: chunk-status
title: 生成状态与 ChunkStatus
description: 12步区块生成管线、外圈依赖、taskMargin 与切比雪夫距离、光照与存档升级旁路。
index: 2
is-advanced: false
---

我们已经知道区块有 `ProtoChunk` 和 `WorldChunk` 两种形式。但一个 `ProtoChunk` 是如何从无到有、一步一步变成 `WorldChunk` 的？

答案在 `ChunkStatus` 中。

## 区块生成管线

`ChunkStatus` 对应区块生成中的一个阶段。在 1.20.1 中，共有 **12 个阶段**，按顺序排列：

| 序号 | 状态 | 主要任务 |
|------|------|----------|
| 0 | `EMPTY` | 起点，不包含任何数据 |
| 1 | `STRUCTURE_STARTS` | 确定可能在区块中起始的结构（如村庄、要塞） |
| 2 | `STRUCTURE_REFERENCES` | 添加结构参照，用于世界生成中结构间的协调 |
| 3 | `BIOMES` | 确定生物群系（基于噪声采样） |
| 4 | `NOISE` | 生成地形的大致轮廓与基岩层 |
| 5 | `SURFACE` | 替换地表附近的方块（如替换石头为泥土、草方块） |
| 6 | `CARVERS` | 生成洞穴和峡谷 |
| 7 | `FEATURES` | 生成地物（树木、矿石、花等） |
| 8 | `INITIALIZE_LIGHT` | 初始光照计算（1.20 新增） |
| 9 | `LIGHT` | 完整光照计算 |
| 10 | `SPAWN` | 生成初始生物（如野生动物） |
| 11 | `FULL` | 将 `ProtoChunk` 转换为 `WorldChunk` |

每个阶段都有一个对应的 **`GenerationTask`**（从无到有时）或 **`LoadTask`**（从磁盘加载时），定义了从上一个状态推进到当前状态需要执行的运算。`ChunkStatus.ChunkType` 枚举区分两种区块类型：

- `PROTOCHUNK`：生成中，使用 `ProtoChunk`；
- `LEVELCHUNK`：已完成（`FULL` 阶段），使用 `WorldChunk`。

> [!TIP]
> 从生成到完成的顺口溜（1.16.4 版本，与 1.20.1 略有差异）：
> 
> 一空二始三参考，群系噪声再地表。
> 气流雕刻又结构，光生高度便终了。

## 外圈依赖

虽然 `FULL` 是"区块生成完成"的标志，但这不意味着孤立的单个区块就能一路生成到 `FULL`。

**外圈依赖**（outskirt dependency）指：一个区块要推进到某个 `ChunkStatus`，其周围一定距离内的区块至少需要达到某个最低状态。这个距离由 **`taskMargin`** 决定。

以噪声地形为例：要在 `(0, 0)` 生成地形，生成器需要读取周围区块的生物群系信息来完成插值。如果周围区块连生物群系都没确定，中央区块的地形就无法正确生成。因此，`NOISE` 阶段的 `taskMargin` 被设为 8——周围 8 区块范围内的区块必须至少完成了 `BIOMES`。

各个阶段的 `taskMargin` 大致如下：

| 阶段 | taskMargin | 说明 |
|------|-----------|------|
| `STRUCTURE_STARTS` | 0 | 结构的起始位置仅依赖本区块 |
| `STRUCTURE_REFERENCES` | 8 | 需要周围结构的参照信息 |
| `BIOMES` | 8 | 生物群系插值需要周围噪声数据 |
| `NOISE` | 8 | 地形生成需要周围生物群系信息 |
| `SURFACE` | 8 | 地表构建需要周围噪声信息 |
| `CARVERS` | 8 | 洞穴生成需要连通到周围区块 |
| `FEATURES` | 8 | 地物（如大树）可能跨越区块边界 |
| `INITIALIZE_LIGHT` | 0 | 仅初始化本区块光照 |
| `LIGHT` | 1 | 光照计算需要与邻块协调 |
| `FULL` | 0 | 转换不需要外部信息 |

> [!IMPORTANT]
> `taskMargin` 的存在意味着：**即使加载票只要求加载中心区的区块，周边一定范围内的区块也会被触发到对应的 ChunkStatus**。这就是为什么 F3+G 显示的地形总是比强加载范围大一圈——那些外圈区块虽然不参与 tick，但它们的生成已经部分完成。

### taskMargin 与切比雪夫距离

`taskMargin` 使用的距离度量是**切比雪夫距离**（Chebyshev Distance）。对于区块坐标 `(x₁, z₁)` 到 `(x₂, z₂)`：

$$\text{距离} = \max(|x_1 - x_2|, |z_1 - z_2|)$$

例如，`(0, 0)` 到 `(8, 0)` 的距离 = 8，到 `(8, 8)` 的距离也是 8。这意味着 `taskMargin = 8` 时，生成波及的是一个以目标区块为中心、边长为 `2×8+1 = 17` 的正方形区域。

## 生成不可达时的处理

当某个区块因为加载票的约束无法达到 `FULL` 时，它的 `ChunkStatus` 会停留在与"到最近可访问区块的距离"相匹配的阶段：

| 到最近可访问区块的距离 | 停留的 ChunkStatus |
|---|---|
| 0（自身可访问） | `FULL` |
| 1 | `FEATURES` |
| 2 | `CARVERS` |
| 3~10 | `STRUCTURE_STARTS` |
| 11 | `EMPTY` |

这意味着，1.20.1 中距离强加载区块正好 1 区块远的区块，只会生成到 `FEATURES`——洞穴有了、地物也有了，但光照是不完整的，也不能转换为 `WorldChunk`。只有当它收到更强的加载票，级别足够低时，生成才会继续向后推进。

## 旁路系统

区块生成不是孤立的——有两个重要的"旁路系统"与它并行运作。

### 光照系统

光照系统是区块生成管线中的特殊环节：它并不改变方块布局，而是为已生成的方块计算光照值（亮度）。光照计算由 `ServerLightingProvider` 统一管理，分为 `INITIALIZE_LIGHT` 和 `LIGHT` 两个阶段执行。这两个阶段看似相似，但分工明确：前者负责"告知"光照系统哪些区块列需要光照计算，后者负责实际执行光照传播。

#### INITIALIZE_LIGHT vs LIGHT

**INITIALIZE_LIGHT**（1.20 新增）对应 `ChunkStatus.INITIALIZE_LIGHT`（index=8），`taskMargin=0`。它的职责是为刚完成 `FEATURES` 阶段的区块建立初始光照状态：

1. 标记哪些 **chunk section** 非空（即包含方块，需要光照计算）；
2. 调用 `ServerLightingProvider.initializeLight(chunk, excludeBlocks)`，内部会为每个非空 section 调用 `setColumnEnabled(pos, true)` 和 `setSectionStatus(pos, false)`；
3. **不执行完整光照传播**，只是告诉光照系统"这些区块列需要计算光照"。

为什么需要这个阶段？因为光照传播依赖于方块的实际布局——光照系统需要知道哪些 section 有方块（非空），才能正确初始化光源图。

**LIGHT** 对应 `ChunkStatus.LIGHT`（index=9），`taskMargin=1`。它的职责是执行完整的光照传播，确保目标区块及其邻块的光照值正确：

1. 调用 `ServerLightingProvider.light(chunk, excludeBlocks)`，内部会调用 `propagateLight(chunkPos)` 和最终的 `doLightUpdates()`；
2. 光照传播是跨区块的——一个光源（如火把）可能影响周围多个区块的亮度值；
3. 完成后，调用 `chunk.setLightOn(true)` 标记该区块光照已就绪。

`taskMargin=1` 的含义是：周围 1 区块范围内的邻块也必须至少完成 `LIGHT`，否则边界光照会不正确（因为光照传播可能跨越边界）。

| 阶段 | taskMargin | 主要动作 | 是否传播光照 | 完成后状态 |
|---|---|---|---|---|
| `INITIALIZE_LIGHT` | 0 | `setColumnEnabled()`, `setSectionStatus()` | ❌ 否 | 光照系统知道哪些列需要计算 |
| `LIGHT` | 1 | `propagateLight()`, `doLightUpdates()` | ✅ 是 | 区块光照值正确，可以发送给客户端 |

#### 异步执行与 CompletableFuture

从 `ChunkStatus.LIGHT` 的 `GenerationTask` 注册代码可以看出，光照任务返回的是 `CompletableFuture<Chunk>`：

```java
// ChunkStatus.java (LIGHT 阶段注册时)
LIGHT = register("light", INITIALIZE_LIGHT, 1, PRE_CARVER_HEIGHTMAP, LIGHT_BLOCKING,
    (targetStatus, executor, world, generator, ..., chunks, chunk) -> {
        // ...
        return lightingProvider.light(chunk, isNewChunk)
            .thenApply(ch -> { chunk.setStatus(ChunkStatus.LIGHT); return ch; });
    }
);
```

这意味着：

1. `light()` 被调用时，任务进入 `ServerLightingProvider` 的队列；
2. 任务通过 `ChunkTaskPrioritySystem` 路由到 light 线程（Light Engine Thread）；
3. light 线程执行实际的光照传播（BFS/DFS 遍历光源图）；
4. 完成后 `CompletableFuture` complete，区块进入下一个状态。

**为什么光照必须异步？** 光照传播是 CPU 密集型操作：需要大范围遍历光源图，计算每个方块的亮度值衰减。如果在主线程执行，大规模世界生成时会导致严重掉刻。1.14 之前光照是同步的，玩家在高速飞行时经常遇到"卡顿 + 服务端无响应"——原因就是主线程被光照计算阻塞了。

1.14 引入异步光照后，light 线程独立于主线程运行，玩家移动时不再因为光照计算而卡顿。

#### 光照传播的基本原理

父类 `LightingProvider` 持有两个 `ChunkLightProvider`：

- **`blockLightProvider`**：处理方块光（火把、岩浆、荧石等光源）；
- **`skyLightProvider`**：处理天空光（来自天空的自然光，在下界和末地为 null）。

`doLightUpdates()` 调用两者的 `doLightUpdates()`：

```java
// LightingProvider.java
public int doLightUpdates() {
    int i = 0;
    if (this.blockLightProvider != null) {
        i += this.blockLightProvider.doLightUpdates();
    }
    if (this.skyLightProvider != null) {
        i += this.skyLightProvider.doLightUpdates();
    }
    return i;
}
```

`ChunkLightProvider.doLightUpdates()` 的核心：

1. 维护一个**待传播队列**（`field_44735`，类型是 `LongQueue`）；
2. 从光源位置开始，按 BFS/DFS 规则向周围传播；
3. 每传播一个方块，根据方块的**透光度**（opacity）衰减亮度值；
4. 当亮度值降到 0 或无法再传播时停止。

> [!TIP]
> 光照传播算法本身非常复杂（涉及光源图、增量更新、边界同步等），这超出 ChunkMechanics 的范围。02 章节只需讲"光照系统做什么"和"它如何融入生成管线"，不需深入算法实现——那是光照系统专题的内容。

#### 光照系统与生成管线的协作

光照系统在生成管线中的位置非常关键，时序关系如下：

1. 区块生成到 `FEATURES`（地物、洞穴、结构全部生成完毕）；
2. 进入 `INITIALIZE_LIGHT`：告诉光照系统"这些列需要光照"；
3. 进入 `LIGHT`：实际执行光照传播；
4. 光照完成后，区块可以进入 `SPAWN`（生成初始生物）；
5. 最终进入 `FULL`（转换为 `WorldChunk`）。

**为什么光照必须在 SPAWN 之前？** 因为生物生成依赖光照值。某些生物只在暗处生成（如僵尸、骷髅），某些只在亮处生成（如动物）。如果光照未计算完成，生物生成器无法正确判断哪些位置可以刷怪——这会导致怪物在明亮的草原上刷新，或者动物在洞穴里刷新。

**为什么光照不能更早？** 因为光照传播依赖最终的方块布局。如果在 `FEATURES` 之前计算光照，后续的地物放置（如大树）会改变方块布局，之前计算的光照就失效了，必须重新计算。放在 `FEATURES` 之后，保证了光照计算时方块布局已经稳定。

#### 光照票的自动管理

在 06 章节"加载票系统"中会详细讲解，这里只需知道：当一个区块需要光照计算时，光照系统会自动添加临时的 **`LIGHT` 票**：

```java
// 某个区块需要光照计算时
chunkTicketManager.addTicket(ChunkTicketType.LIGHT, pos, radius, pos);
```

这张票的作用：

- 确保光照计算期间，目标区块及其邻块保持加载状态；
- 光照计算完成后，票自动过期；
- 避免"光照计算到一半，邻块被卸载"的竞态条件。

这就是为什么你在调试时偶尔会在 F3 界面看到某些区块的加载原因是 `light`——那是光照系统自动添加的临时票。

#### shouldAlwaysUpgrade 的强制重算

`ChunkStatus.LIGHT` 和 `INITIALIZE_LIGHT` 都标记为 `shouldAlwaysUpgrade=true`：

```java
// ChunkStatus.java
INITIALIZE_LIGHT = register("initialize_light", ..., true, ...);  // shouldAlwaysUpgrade=true
LIGHT = register("light", ..., true, ...);                         // shouldAlwaysUpgrade=true
```

这意味着：即使从磁盘加载旧区块时，这两个阶段已经"完成"，系统也会**强制重新执行**。

为什么？因为光照依赖于周围区块的状态——而周围区块可能在保存后被修改了（如玩家放置/破坏方块，或者邻块重新生成）。强制重新计算光照保证了加载后的区块光照是正确的，不会出现"保存时是夜晚，加载后变白天，但光照还是旧的"的情况。

> [!IMPORTANT]
> 这也是为什么加载旧区块时会感觉"稍微卡一下"——因为即使区块数据已经存在磁盘上，光照系统仍然需要重新计算一遍。这是正确性的代价：与其让玩家看到错误的光照，不如每次加载时重新计算。

### 存档升级

当从磁盘加载旧版本的区块数据时，`UpgradeData` 和 `DataFixer` 子系统会在区块被转换前介入：

- `ChunkSerializer` 解析 NBT 数据时，检测 `DataVersion`；
- 如果 `DataVersion` 低于当前版本，`DataFixer` 会执行一系列转换——从方块 ID 的映射到 NBT 结构的重组；
- `UpgradeData` 记录哪些子区块和哪些位置需要特殊处理（如"这个区块被升级过，方块实体的 `CustomName` 标签格式已改变"）。

## 小结

- `ChunkStatus` 定义了 12 个生成阶段，从 `EMPTY` 到 `FULL`，每个阶段对应一个 `GenerationTask` 或 `LoadTask`。
- `taskMargin` 决定了外圈依赖：一个区块要推进到某阶段，周围一定切比雪夫距离内的区块必须达到最低状态。
- 生成不可达时，区块的 ChunkStatus 停留在与距离匹配的阶段——不是"不生成"，而是"只生成到能停住的地方"。
- 光照系统分为 `INITIALIZE_LIGHT`（标记需要光照的列）和 `LIGHT`（执行实际传播）两个阶段，异步执行在独立线程中，完成后强制重算以保证正确性；存档升级系统在 `ChunkSerializer` 中通过 `DataFixer` 介入。
- 世界生成的详细内容超出了本章范围，将在后续"世界生成"专章详细展开。

## [!ADVANCED] 代码走读

### ChunkStatus 的注册模式：为什么是链表而不是枚举

```java
// ChunkStatus.java（简化）
public static final ChunkStatus EMPTY = register("empty", null, -1, ...);
public static final ChunkStatus STRUCTURE_STARTS = register("structure_starts", EMPTY, 0, ...);
public static final ChunkStatus STRUCTURE_REFERENCES = register("structure_references", STRUCTURE_STARTS, 8, ...);
// ... 以此类推 ...
public static final ChunkStatus FULL = register("full", SPAWN, 0, false, ..., ChunkType.LEVELCHUNK, ...);
```

`ChunkStatus` 不是 Java 的 `enum`，而是一个手写的**链表结构**。每个状态通过 `previous` 字段指向前一个状态：

```java
// ChunkStatus 构造器
ChunkStatus(@Nullable ChunkStatus previous, int taskMargin, ...) {
    this.previous = previous == null ? this : previous;  // EMPTY 的前驱是自己
    this.index = previous == null ? 0 : previous.getIndex() + 1;
    // ...
}
```

**为什么不用 `enum`？** 如果 `ChunkStatus` 是一个 Java `enum`，它的值在编译时就固定了——模组无法在运行时插入新的生成阶段。用链表注册模式，模组可以通过 Mixin 向 `createOrderedList()` 返回的列表中添加新状态，从而在生成管线中插入自定义步骤。基础概念中提到的 `INITIALIZE_LIGHT`（1.20 新增）本身也证明了这种灵活性——Mojang 自己也需要在版本间增删阶段。

**`isAtLeast()` 的设计**：通过比较 `index` 判断进度：

```java
public boolean isAtLeast(ChunkStatus other) {
    return this.getIndex() >= other.getIndex();
}
```

注意这里的语义是"至少达到"，即"已完成到某个阶段或更远"。例如 `LIGHT.isAtLeast(FEATURES)` 返回 `true`——光照完成了，地物自然也已经完成。

### taskMargin：生成任务的外圈依赖机制

```java
// ChunkStatus 注册示例
CARVERS:  register("carvers", SURFACE, 8, ...)  // taskMargin=8
LIGHT:    register("light", INITIALIZE_LIGHT, 1, ...)  // taskMargin=1
FULL:     register("full", SPAWN, 0, ...)  // taskMargin=0
```

`taskMargin` 在 `ThreadedAnvilChunkStorage.getRegion()` 中被使用：

```java
// 伪代码
getRegion(holder, taskMargin, statusProvider) {
    // 收集以 holder 的 chunk 为中心、(2*taskMargin+1)² 区域内的所有区块
    // 等待它们都达到 statusProvider.apply(distance) 要求的最低状态
    // 如果达不到 → 返回的 CompletableFuture 一直 pending，任务被挂起
}
```

**为什么 `CARVERS` 需要 taskMargin=8 而 `LIGHT` 只需要 1？** 洞穴生成时，一条洞穴可能横跨多个区块。如果只在一个区块内生成洞穴，边界处就会出现"洞穴突然断裂"的视觉错误。taskMargin=8 保证了生成洞穴的中心区块周围 8 格以内的区块也至少完成了前置阶段（`SURFACE`），整个区域的洞穴可以连贯地生成。

光照计算只需要 taskMargin=1，因为光照传播只需相邻区块的亮度值——第一层邻块完成后，光照就已经可以正确计算了。

**`shouldAlwaysUpgrade` 的含义**：某些状态（如 `INITIALIZE_LIGHT` 和 `LIGHT`）标记为 `shouldAlwaysUpgrade=true`。这意味着即使从磁盘加载旧区块时该状态已经"完成"，系统也会**强制重新执行**这个阶段的运算。为什么光照需要这个？因为光照依赖于周围区块的状态——而周围区块可能在保存后被修改了。强制重新计算光照保证了加载后的区块光照是正确的。

### GenerationTask 的任务调度

每个 `ChunkStatus` 在注册时携带一个 `GenerationTask`（通过重载方法，未指定时使用默认的 `STATUS_BUMP_LOAD_TASK`）：

```java
// ChunkStatus.java
interface GenerationTask {
    CompletableFuture<Either<Chunk, Unloaded>> doWork(
        ChunkStatus targetStatus, Executor executor, ServerWorld world,
        ChunkGenerator generator, ..., List<Chunk> chunks, Chunk chunk);
}
```

调度逻辑在 `runGenerationTask()` 中：它从 `chunks` 列表（`getRegion` 收集的周边区块）取中间的区块作为目标，调用 `generationTask.doWork()`。任务完成后，将 `chunk.setStatus(this)` 标记当前阶段完成，然后编译器将结果传递给 `futuresByStatus[this.index]`，触发等待此阶段的 Future。

这里有两点设计细节值得注意：
1. **任务在独立生成线程上执行**（`worldGenExecutor`），不在主线程。这是 `CompletableFuture` 的必要性——主线程不能阻塞等待生成完成。
2. **getRegion 的 margin 参数**：对于 `makeChunkTickable`，`margin=1` 意味着要求周围 9 个区块（3×3）都达到 `FULL`。这不是 taskMargin——taskMargin 控制的是**同一个阶段**的传播，而 margin 控制的是**不同阶段之间的依赖**。

## 参考

- [Discovering Minecraft - ChunkStatus 与世界生成](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.world.chunk.ChunkStatus`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ServerLightingProvider`
- `net.minecraft.world.chunk.ChunkSerializer`
- `com.mojang.datafixers.DataFixer`
