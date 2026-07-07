---
slug: worlds-and-blocks
index: 0
chapter-title: 世界与方块
intro-title: 从眼前的方块开始
---

## 概述

当我们站在 Minecraft 的一个维度中时，眼前似乎只有一整片连续的地形：脚下是方块，远处仍然是方块，穿过传送门后又会到达另一个维度。

但是，当游戏需要查找、保存或改变其中的某个方块时，“一整片地形”并不是一个足够精确的描述。游戏需要知道这个方块属于哪个维度、哪个区块、哪组坐标；还需要分辨“这是一个活塞”和“这是一个朝东且没有伸出的活塞”之间的区别。

本章将从这些可以直接观察的现象出发，依次介绍：

1. 存档与维度；
2. 区块与子区块；
3. 方块、方块状态与方块实体之间的区别；
4. 放置、改变和破坏一个方块时发生的事情。

读完本章后，你不需要记住 Minecraft 的完整运算流程，但应该能够回答三个问题：

- 给出一个坐标时，游戏去哪里寻找对应的方块？
- 同一种方块为什么能够表现出不同朝向、能量或形状？
- 一个方块发生变化时，哪些事情属于“写入新状态”，哪些事情是由写入操作进一步引发的？

这些概念将作为[方块更新](../BlockUpdate/README.zh.md)、[微时序](../MicroTiming/README.zh.md)和[加载票](../LoadingTicket/README.zh.md)等章节的共同基础。

> [!IMPORTANT]
> 本章源码结论以 Minecraft Java Edition 1.20.1、Yarn Mapping `1.20.1+build.10` 为准。正文先建立能够解释常见现象的模型；更精确的类、字段和方法放在各篇末尾的进阶部分。

## 术语与参考

本章采用“维度（World）”“维度类型（Dimension Type）”“区块（Chunk）”“子区块（Chunk Section）”“方块状态（Block State）”等译名。`BlockPos` 直接称为“坐标”。术语选择参考了 [Discovering Minecraft](https://github.com/lovexyn0827/Discovering-Minecraft)，并以 1.20.1 Yarn 中的实际类型与方法进行校对。
