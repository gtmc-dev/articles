---
slug: block-mechanics
index: -1
chapter-title: 方块机制
intro-title: 方块的本质
---

## 概述

方块是 Minecraft 世界的基本组成单位。但"方块"一词掩盖了一个重要的区分：在源码中，`Block` 和 `BlockState` 是两种完全不同的概念。

本章从这种区分出发，依次介绍方块状态的设计、放置与破坏的完整流程，以及与区块机制、更新系统的衔接。

## 结构

1. **[方块与方块状态](./01-方块与方块状态.zh.md)** —— Block 与 BlockState 的本质区别、属性与值的硬性限制、状态与方块实体的分界、流体状态的导出
2. **[方块的放置、改变与破坏](./02-方块的放置改变与破坏.zh.md)** —— `setBlockState` 的共同入口、放置流程的精确步骤、破坏的两种入口、flags 各标记的含义、写入后触发的事件链

## 前置知识

- [重新认识世界](../WorldsAndBlocks/重新认识世界.zh.md)
- [区块机制](../ChunkMechanics/README.zh.md)
