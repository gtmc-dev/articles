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
Long2ObjectMap<ChunkHolder> chunkHolders  // 注意：这个表在 ChunkTicketManager 中
```

注意这里有一处命名混淆：`ThreadedAnvilChunkStorage` 本身没有名为 `chunkHolders` 的字段（它用的是 `currentChunkHolders`），但在 Yarn Mapping 较早版本或不同的分析文中，"chunkHolders" 可能指代列表形式的所有 `ChunkHolder`。在 1.20.1 的源码中：

- `TACS.currentChunkHolders`：按 `long` 索引所有活跃的 `ChunkHolder`
- `ChunkTicketManager.chunkHolders`：`Set<ChunkHolder>`，收集本轮 tick 中加载等级发生变化的 holder

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

## 参考

- [Discovering Minecraft - 区块管理系统的基本架构](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ChunkTaskPrioritySystem`
- `net.minecraft.server.world.ServerChunkManager`
