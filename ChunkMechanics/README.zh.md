---
slug: chunk-system-intro
intro-title: 绪论
chapter-title: 区块
index: -1
---

## 研究背景与意义

一直以来，社区内对 1.14+ 区块加载的论述多集中于对**稳定态**的分析——我们知道下界传送门加载器会让目标维度中 3×3 的区块被加载，但这些区块是瞬间加载还是在多个 gt 后才加载？这个问题很少被提及。

与此同时，区块管理是 Minecraft 中一个比较重要而又相当复杂的系统，与世界的几乎每一个方面都息息相关。从最根本的"方块查询"到"实体运算"，从"玩家视距"到"区块保存"，区块管理系统都在幕后精确地控制着"哪些区块存在、哪些区块在运算、哪些区块该被保存"。

本章旨在增强读者对区块管理系统底层工作模式的理解，帮助读者了解源码组织结构，进而发掘出更多基于区块管理机制的创造性应用。

## 研究环境与约定

本章以以下环境为基准：

- Minecraft Java Edition **1.20.1**
- 反混淆映射表：**Yarn Mapping `1.20.1+build.10`**
- 部分分析基于 lovexyn0827 对 1.16.4 和 1.20.1 的早期研究（CC0 协议）

文中使用的一些特殊约定：

- **模块**指一个在整个维度中唯一的对象，通常对应一个 Java 类，有时也指代一个较为独立的子系统。
- **"加载"**不特指通常意义上的可访问、弱加载或强加载这些状态，而是指区块存在于内存中的状态以及区块从磁盘读入内存的过程。
- 文中的距离如无特殊说明，均为**切比雪夫距离**（Chebyshev Distance），而不是欧几里得距离。例如，区块 `(0, 0)` 到区块 `(2, 2)` 的切比雪夫距离为 `max(2, 2) = 2`。

## 本章的基本结构

1. **基础概念与结构**：从区块的数学定义出发，理解子区块、局部坐标打包、ProtoChunk 与 WorldChunk 的差异，以及客户端区块管线。
2. **生成状态与 ChunkStatus**：深入12步区块生成管线，理解外圈依赖、taskMargin 和光照/存档升级等旁路系统。
3. **级别与运行态分层**：区分 ChunkStatus（生成状态）和 ChunkLevelType（运行资格），建立"已生成 ≠ 正在运算"的心智模型。
4. **服务端总调度器 TACS**：ThreadedAnvilChunkStorage 的完整职责——从 currentChunkHolders 到 chunksToUnload，从 worldgen/light/main 三类执行器到 ChunkTaskPrioritySystem 的优先级调度。
5. **主线程异步任务**：理解 CompletableFuture 在区块管理中的核心作用——几乎所有操作都不是直接被运行，而是作为 Future 被计划，然后在异步任务队列中被执行。
6. **加载票机制**：Ticket 的类型全表、生命周期、玩家加载、票的传播与过期回收。
7. **ChunkHolder 生命周期**：三条核心 Future（accessibleFuture、tickingFuture、entityTickingFuture）如何像一个温度计一样控制区块的三种"运行视图"。
8. **计划刻与区块运行级别**：计划刻如何依赖区块的 BLOCK_TICKING 级别——tickingFutureReadyPredicate 决定计划刻能否执行，以及区块未 ticking 时计划刻的去向。
9. **随机刻与方块实体刻**：随机刻的抽样机制、ChunkSection 计数器如何跳空、方块实体 ticker 的注册与执行，以及三者如何共同构成"方块刻"。
10. **实体加载与实体追踪**：实体与方块两套独立管理系统的设计哲学、ServerEntityManager 的职责、EntityTracker 如何同步——以及为什么 ENTITY_TICKING 需要 5×5 FULL。
11. **保存与卸载流水线**：三级卸载结构（unloadedChunks → chunksToUnload → unloadTaskQueue）、保存节流、ChunkSerializer 的序列化职责，以及 StorageIoWorker 怎么最终写进 .mca。
12. **POI 与旁路存储**：兴趣点（PointOfInterest）为什么独立于普通区块存储——它的数据结构、方块变化如何触发更新，以及与村民、袭击、流浪商人的查询关系。
13. **客户端可见性与区块数据包**：服务端如何决定向客户端发送哪些区块、区块数据包与增量更新包的分工、watchDistance 的几何判定，以及 flushUpdates 的批量发送机制。
14. **Game Event 与区块内监听器**：游戏事件系统与 NC/PP 更新的本质区别、WorldChunk 按 section 管理事件分发器，以及幽匿监听器在区块卸载时的清理。
15. **保存节流与 Savestate 现象**：TACS 的 10 秒保存冷却、StorageIoWorker 异步写盘、自动保存/flush/关服的差异，以及 Watchdog 与 OOM 如何形成或打断跨区块状态分叉。
16. **加载失败与降级机制**（高级专题）：IOException 降级、EmptyChunk 虚空区块、PENDING 状态卡住导致计划刻永久失效，以及关服时的 flush 死锁与 halt(1) 崩服链路。
17. **完整流程总览与跨章节索引**：从 Ticket 创建到区块卸载的完整时间线、五条并行管线的交互、关键机制速查表、异常路径速查，以及分路线的阅读建议。

> [!TIP]
> **阅读顺序建议**：01-03（建立基础概念）→ **17（建立全局视图）** → 06-07（理解核心加载/温度机制）→ 04-05（深入异步架构）→ 08-16（按需阅读各专题）

## 前置知识

本文假定读者已对 Minecraft 中的基本概念有一定了解：

- [方块与方块状态](../BlockMechanics/01-方块与方块状态.zh.md) —— Block 与 BlockState 的区别
