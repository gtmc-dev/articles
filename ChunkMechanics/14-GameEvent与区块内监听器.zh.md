---
slug: game-event-listeners
title: Game Event 与区块内监听器
index: 14
is-advanced: false
---

前面几篇主要讨论区块怎样被加载、运行、保存和卸载。本篇补一个较小但容易混淆的系统：**游戏事件**（`GameEvent`）。

它常常和幽匿感测体、幽匿尖啸体、悦灵等机制一起出现，但本篇不把重点放在幽匿玩法本身，而是看区块如何保存和管理这些监听器。

**基础篇**

- `GameEvent` 与 NC/PP 更新的区别
- `WorldChunk` 中按 section 组织的游戏事件分发器
- 游戏事件监听器的注册、移除和卸载生命周期
- `VibrationListener` 与游戏事件系统的关系

## Game Event ≠ 方块更新

NC 更新和 PP 更新是方块之间的通知机制。它们关心的是：某个方块或邻近位置发生变化后，其他方块是否应该重新判断自己的状态。例如活塞是否应当伸出、侦测器是否应当亮起、墙和栅栏是否需要更新连接形状。也就是说，方块更新管的是“方块应该怎么变”。

`GameEvent` 则是另一套事件系统。它把“世界里发生了什么事”发射给附近的监听器，但事件本身不直接要求方块改变状态。常见事件包括 `GameEvent.BLOCK_PLACE`、`GameEvent.BLOCK_DESTROY`、`GameEvent.STEP`、`GameEvent.PROJECTILE_LAND`、`GameEvent.ENTITY_DAMAGE`。这些事件可以被幽匿感测体、幽匿尖啸体或其他监听器感知，再由监听器自己的逻辑决定是否响应。

因此，不要把 `emitGameEvent()` 理解成一种新的 NC 更新或 PP 更新。方块更新的消息很“粗”：这里变了，你自己检查。游戏事件的消息更像记录事实：这里发生了 `STEP`，这里发生了 `BLOCK_PLACE`。前者服务于方块状态传播，后者服务于监听器感知。

## WorldChunk 中的事件分发器

`WorldChunk` 里维护了一张游戏事件分发器映射：

```java
private final Int2ObjectMap<GameEventDispatcher> gameEventDispatchers = new Int2ObjectOpenHashMap<>();
```

键是 section Y 坐标，也就是 `ChunkSectionPos.getSectionCoord(y)` 得到的纵向 16 格区间；值是这个 section 对应的 `GameEventDispatcher`。这说明游戏事件监听器不是整块区块统一塞进一个大列表，而是按 section 拆开管理。

当外部需要某个 section 的分发器时，会调用：

```java
public GameEventDispatcher getGameEventDispatcher(int ySectionCoord) {
    return this.world instanceof ServerWorld serverWorld
        ? this.gameEventDispatchers.computeIfAbsent(
            ySectionCoord,
            sectionCoord -> new SimpleGameEventDispatcher(
                serverWorld,
                ySectionCoord,
                this::removeGameEventDispatcher
            )
        )
        : super.getGameEventDispatcher(ySectionCoord);
}
```

服务端 `WorldChunk` 会懒创建 `SimpleGameEventDispatcher`：只有某个 section 真的需要监听器时，映射里才出现对应分发器。客户端或非服务端路径则回到父类逻辑，不在这里维护服务端监听器集合。

每个 section 一个分发器，有两个直观好处。第一，发射事件时不必把整块区块所有监听器混在一起筛选，section 可以先做一层空间分组。第二，监听器的增删集中在更小的列表里，不需要把整块区块的监听器压到单个 dispatcher 上，也避免单个 dispatcher 成为统一的争用点。

`SimpleGameEventDispatcher` 的内部结构很轻：

```java
private final List<GameEventListener> listeners = Lists.newArrayList();
private final Set<GameEventListener> toRemove = Sets.newHashSet();
private final List<GameEventListener> toAdd = Lists.newArrayList();
private boolean dispatching;
```

`listeners` 保存当前 section 的游戏事件监听器。若正在分发事件，新增和移除不会立刻改动正在遍历的列表，而是先进入 `toAdd` / `toRemove`，分发结束后再合并。这和方块实体 ticker 的 pending 列表思路相似：避免遍历中修改当前集合。

## 监听器注册与生命周期

以幽匿感测体为例。幽匿感测体被放置后，方块状态对应的方块实体会被创建为 `SculkSensorBlockEntity`。这个方块实体实现了 `GameEventListener.Holder<Vibrations.VibrationListener>`，它的 `getEventListener()` 返回自己的 `Vibrations.VibrationListener`。幽匿尖啸体 `SculkShriekerBlockEntity` 也是同样的模式。

`WorldChunk` 并不需要认识每一种幽匿方块。它只通过 `BlockEntityProvider.getGameEventListener(ServerWorld, blockEntity)` 询问：这个方块实体是否提供游戏事件监听器？默认实现会检查方块实体是否是 `GameEventListener.Holder<?>`，如果是，就取出 `holder.getEventListener()`。

注册发生在 `WorldChunk.updateGameEventListener()`：

```java
private <T extends BlockEntity> void updateGameEventListener(T blockEntity, ServerWorld world) {
    Block block = blockEntity.getCachedState().getBlock();
    if (block instanceof BlockEntityProvider) {
        GameEventListener gameEventListener =
            ((BlockEntityProvider)block).getGameEventListener(world, blockEntity);
        if (gameEventListener != null) {
            this.getGameEventDispatcher(
                ChunkSectionPos.getSectionCoord(blockEntity.getPos().getY())
            ).addListener(gameEventListener);
        }
    }
}
```

这段代码把监听器注册到“方块实体所在 Y 坐标对应的 section 分发器”中。`addListener(GameEventListener)` 之后，分发器就会在本 section 的事件分发过程中考虑这个监听器；真正是否接收，还要由监听器的位置、范围、标签和回调逻辑决定。

注意这里的生命周期与方块实体绑定。方块实体加入区块时，`WorldChunk.addBlockEntity()` 会在区块已进入世界运行状态时调用 `updateGameEventListener()`，然后再更新方块实体 ticker。区块从 NBT 或网络包补齐方块实体时，`updateAllBlockEntities()` 也会对已有方块实体重新注册监听器。

移除则走反向路径。方块被破坏、替换，或方块实体被移出区块时，`WorldChunk.removeBlockEntity()` 会取出该方块实体对应的 `GameEventListener`，找到它所在 section 的 `GameEventDispatcher`，然后调用：

```java
gameEventDispatcher.removeListener(gameEventListener);
```

`SimpleGameEventDispatcher.removeListener()` 删除监听器后，如果 `listeners` 已经为空，就调用构造时传入的 disposal callback：

```java
if (this.listeners.isEmpty()) {
    this.disposalCallback.apply(this.ySectionCoord);
}
```

在 `WorldChunk` 中，这个回调正是 `removeGameEventDispatcher(int ySectionCoord)`：

```java
private void removeGameEventDispatcher(int ySectionCoord) {
    this.gameEventDispatchers.remove(ySectionCoord);
}
```

所以，dispatcher 的清理不是另开一张全局表扫描，而是 section 内最后一个监听器移除时顺手把这个 section 的分发器从 `gameEventDispatchers` 中删除。空 section 不保留空 dispatcher。

区块卸载时也可以按这个思路理解。第 11 篇已经讲过，卸载是 TACS 把 holder 从当前表移到待卸载结构，再在卸载任务中执行保存、实体卸载、光照状态刷新等收尾。对于游戏事件监听器来说，关键是方块实体随区块退出运行集合时要被移除或标记移除，对应监听器也应从所在 section 的 dispatcher 中注销；当某个 section 的监听器全部离开后，`removeGameEventDispatcher()` 会把分发器映射项清掉。

这也是为什么 `gameEventDispatchers` 放在 `WorldChunk` 上，而不是放在世界级全局列表里：监听器和区块 section 的生命周期天然绑定。区块加载时，方块实体恢复，监听器重新注册；方块实体消失或区块卸载时，监听器移除，空 dispatcher 跟着消失。

## 与振动系统的关系

`VibrationListener` 是游戏事件监听器的一种特化实现。幽匿感测体、幽匿尖啸体会用它接收特定 `GameEvent`，再根据距离、频率、标签和自身状态决定是否激活。悦灵（`AllayEntity`）也使用振动相关监听器，只是它挂在实体的 `EntityGameEventHandler` 上。

换句话说，振动系统是建立在游戏事件监听器之上的一组规则：哪些事件能被听见、传播距离是多少、如何换算频率、监听器收到后怎样响应。这些振动传播和频率分类细节，不在本篇展开。

## 本节要点回顾

- NC/PP 更新是方块状态传播机制，`GameEvent` 是发给监听器的事件通知。
- `GameEvent` 记录“什么事情发生了”，不会直接要求方块改变状态。
- `WorldChunk.gameEventDispatchers` 是 `Int2ObjectMap<GameEventDispatcher>`，按 section Y 坐标保存分发器。
- `getGameEventDispatcher(int ySectionCoord)` 在服务端懒创建 `SimpleGameEventDispatcher`。
- 方块实体若实现 `GameEventListener.Holder<?>`，可通过 `BlockEntityProvider.getGameEventListener()` 暴露监听器。
- 幽匿感测体、幽匿尖啸体的 `Vibrations.VibrationListener` 会注册到所在 section 的 dispatcher。
- 监听器移除后，空的 `SimpleGameEventDispatcher` 会回调 `removeGameEventDispatcher()`，从 `WorldChunk` 映射中删除。
- 区块卸载的收尾流程会让方块实体和监听器退出运行结构，section dispatcher 也随监听器生命周期被清理。
