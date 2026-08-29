---
translates: ./README.zh.md
translated-from-revision: 7c7b4ca93524fd57af3bfe42221c1af2812e0166
chapter-title: Blocks
intro-title: The Nature of Blocks
---

## Overview

Blocks are the basic building units of the Minecraft world. But the word "block" hides an important distinction: in the source code, `Block` and `BlockState` are two completely different concepts.

This chapter starts from that distinction, then introduces the design of block states, the full process of placement and breaking, and how they connect to chunk mechanics and the update system.

## Structure

1. **[Blocks and Block States](./01-方块与方块状态.zh.md)** — the essential difference between `Block` and `BlockState`, the hard limits on properties and values, the subchunk palette storage model, the boundary between NBT format and block entities, and how fluid state is derived
2. **[Placing, Changing, and Breaking Blocks](./02-方块的放置改变与破坏.zh.md)** — the shared `setBlockState` entry point, the exact steps of the placement flow, the two breaking entry points, the meaning of each flag, and the chain of events triggered after a write
