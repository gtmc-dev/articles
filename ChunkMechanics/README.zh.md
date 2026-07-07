---
slug: chunk-mechanics
index: 0
chapter-title: 区块机制
intro-title: 从区块到世界
---

## 概述

当我们按下 `F3+G`，地面上出现了彩色网格——那是区块的边界。但这张网格所代表的，远不止"16×16 的格子"那么简单。

在 Minecraft 服务端的底层，区块是被精心管理的基本单元。它的生命周期——从无人问津到被请求、从读盘到生成、从升温到强加载、从降温到卸载——每一步都由一个被称为"区块管理系统"的复杂机制精确控制。

本章将带你完整走完这个过程。读完本章后，你应该能回答：

- 区块是什么？它在源码中如何被存储和管理？
- `ChunkStatus` 和 `ChunkLevelType` 有什么区别？为什么"已生成"不等于"正在运算"？
- 加载票（Ticket）如何决定一个区块的"热度"？
- `ThreadedAnvilChunkStorage` 这个庞大的调度器到底在做什么？
- 一次 `getChunk()` 调用背后，到底发生了什么？
- `ChunkHolder` 如何像一个温度计一样控制区块的三种"运行视图"？
- 玩家看见的地形和服务端运算的区块，边界在哪里？

## 结构

本章共 7 篇：

1. **[绪论](./00-绪论.zh.md)** —— 区块管理系统的研究背景与基本约定
2. **[基础概念与结构](./01-基础概念与结构.zh.md)** —— 区块、子区块、ProtoChunk 与 WorldChunk、客户端管线
3. **[生成状态与 ChunkStatus](./02-生成状态与ChunkStatus.zh.md)** —— 12 步生成管线、外圈依赖、taskMargin、光照与存档升级
4. **[级别与运行态分层](./03-级别与运行态分层.zh.md)** —— ChunkLevels、ChunkLevelType、INACCESSIBLE/FULL/BLOCK_TICKING/ENTITY_TICKING 的本质区别
5. **[服务端总调度器 TACS](./04-服务端总调度器TACS.zh.md)** —— ThreadedAnvilChunkStorage 的职责、核心表结构、ChunkTaskPrioritySystem
6. **[主线程异步任务](./05-主线程异步任务.zh.md)** —— CompletableFuture 在区块管理中的作用、任务队列与执行时机
7. **[加载票机制](./06-加载票机制.zh.md)** —— Ticket 的类型、生命周期、玩家加载、票的传播
8. **[ChunkHolder 生命周期](./07-ChunkHolder生命周期.zh.md)** —— 三条核心 Future、tick() 升温降温、区块运行视图切换

## 术语与参考

本章以 Minecraft Java Edition 1.20.1、Yarn Mapping `1.20.1+build.10` 为准，部分内容基于 1.16.4 的早期研究。基础概念部分参考了 [Discovering Minecraft](https://github.com/lovexyn0827/Discovering-Minecraft)（CC0 协议）的区块管理系统分析。

## 前置知识

- [重新认识世界](../WorldsAndBlocks/重新认识世界.zh.md)
- Java 基本语法与 `CompletableFuture` 概念
