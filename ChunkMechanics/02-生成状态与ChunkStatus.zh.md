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

光照计算在 `INITIALIZE_LIGHT` 和 `LIGHT` 两个阶段执行。它由 `ServerLightingProvider` 管理：

- `INITIALIZE_LIGHT`：为刚生成完地物和洞穴的区块建立初始的天空光和方块光；
- `LIGHT`：与周围区块协调，处理光照传播。这个阶段要求周围区块也至少达到了 `LIGHT` 或更高。

光照系统的独立线程（Light Engine Thread）与生成线程协作——光照更新器监听区块的状态变化，当相邻区块完成生成时自动触发重新计算。

### 存档升级

当从磁盘加载旧版本的区块数据时，`UpgradeData` 和 `DataFixer` 子系统会在区块被转换前介入：

- `ChunkSerializer` 解析 NBT 数据时，检测 `DataVersion`；
- 如果 `DataVersion` 低于当前版本，`DataFixer` 会执行一系列转换——从方块 ID 的映射到 NBT 结构的重组；
- `UpgradeData` 记录哪些子区块和哪些位置需要特殊处理（如"这个区块被升级过，方块实体的 `CustomName` 标签格式已改变"）。

## 小结

- `ChunkStatus` 定义了 12 个生成阶段，从 `EMPTY` 到 `FULL`，每个阶段对应一个 `GenerationTask` 或 `LoadTask`。
- `taskMargin` 决定了外圈依赖：一个区块要推进到某阶段，周围一定切比雪夫距离内的区块必须达到最低状态。
- 生成不可达时，区块的 ChunkStatus 停留在与距离匹配的阶段——不是"不生成"，而是"只生成到能停住的地方"。
- 光照系统在两个专用阶段运行，由独立线程管理；存档升级系统在 `ChunkSerializer` 中通过 `DataFixer` 介入。
- 世界生成的详细内容超出了本章范围，将在后续"世界生成"专章详细展开。

## [!ADVANCED] 源码验证

### ChunkStatus 枚举（1.20.1）

```java
// ChunkStatus.java - 注册顺序
public static final ChunkStatus EMPTY = register("empty", ...);
public static final ChunkStatus STRUCTURE_STARTS = register("structure_starts", EMPTY, ...);
public static final ChunkStatus STRUCTURE_REFERENCES = register("structure_references", STRUCTURE_STARTS, ...);
public static final ChunkStatus BIOMES = register("biomes", STRUCTURE_REFERENCES, ...);
public static final ChunkStatus NOISE = register("noise", BIOMES, ...);
public static final ChunkStatus SURFACE = register("surface", NOISE, ...);
public static final ChunkStatus CARVERS = register("carvers", SURFACE, ...);
public static final ChunkStatus FEATURES = register("features", CARVERS, ...);
public static final ChunkStatus INITIALIZE_LIGHT = register("initialize_light", FEATURES, ...);
public static final ChunkStatus LIGHT = register("light", INITIALIZE_LIGHT, ...);
public static final ChunkStatus SPAWN = register("spawn", LIGHT, ...);
public static final ChunkStatus FULL = register("full", SPAWN, ...);
```

1.20.1 相比 1.16.4 移除了 `liquid_carvers` 和 `heightmaps`，新增了 `INITIALIZE_LIGHT`。`isAtLeast()` 方法通过比较 `index`（注册序号）判断"是否至少达到某阶段"。

### taskMargin 在注册中的体现

每个 `ChunkStatus` 注册时携带 `taskMargin`，例如：

```java
// CARVERS: taskMargin=8, shouldAlwaysUpgrade=false
// LIGHT: taskMargin=1, shouldAlwaysUpgrade=true
// FULL: taskMargin=0, shouldAlwaysUpgrade=false, chunkType=LEVELCHUNK
```

`taskMargin` 的实际影响在 `ChunkStatus.getTaskMargin()` 返回后，由 `ThreadedAnvilChunkStorage.getRegion()` 用于确保周围指定切比雪夫距离内的区块达到要求状态。

### ChunkHolder.futuresByStatus

```java
// ChunkHolder.java
private final AtomicReferenceArray<CompletableFuture<Either<Chunk, Unloaded>>> futuresByStatus
    = new AtomicReferenceArray<>(ChunkStatus.createOrderedList().size());

public CompletableFuture<Either<Chunk, Unloaded>> getFutureFor(ChunkStatus leastStatus) {
    CompletableFuture<Either<Chunk, Unloaded>> f = this.futuresByStatus.get(leastStatus.getIndex());
    return f == null ? UNLOADED_CHUNK_FUTURE : f;
}
```

数组长度 = `ChunkStatus` 数量（12），每个索引对应一个 `ChunkStatus` 阶段的完成 Future。

## 参考

- [Discovering Minecraft - ChunkStatus 与世界生成](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.world.chunk.ChunkStatus`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ServerLightingProvider`
- `net.minecraft.world.chunk.ChunkSerializer`
- `com.mojang.datafixers.DataFixer`
