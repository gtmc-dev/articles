---
translates: ./14-GameEvent与区块内监听器.zh.md
translated-from-revision: 5cd3ed8426fb74699ad86c3870e73a4d03ce43a0
title: Game Event and In-Chunk Listeners
---

Previous chapters mainly discussed how chunks are loaded, run, saved, and unloaded. This chapter supplements a smaller but easily confused system: **game events** (`GameEvent`).

It often appears alongside sculk sensors, sculk shriekers, allays, and related mechanics, but this chapter doesn't focus on sculk gameplay itself, but rather how chunks save and manage these listeners.

**Basics**

- The difference between `GameEvent` and NC/PP updates
- Game event dispatchers organized by section in `WorldChunk`
- Registration, removal, and unload lifecycle of game event listeners
- The relationship between `VibrationListener` and the game event system

## Game Event ≠ Block Update

NC updates and PP updates are notification mechanisms between blocks. They care about: after a block or neighboring position changes, should other blocks re-evaluate their own state? For example, should a piston extend, should an observer light up, should walls and fences update connection shapes. In other words, block updates handle "how blocks should change."

`GameEvent` is a different event system. It broadcasts "what happened in the world" to nearby listeners, but the event itself doesn't directly require blocks to change state. Common events include `GameEvent.BLOCK_PLACE`, `GameEvent.BLOCK_DESTROY`, `GameEvent.STEP`, `GameEvent.PROJECTILE_LAND`, `GameEvent.ENTITY_DAMAGE`. These events can be sensed by sculk sensors, sculk shriekers, or other listeners, which then decide whether to respond based on their own logic.

Therefore, don't understand `emitGameEvent()` as a new type of NC or PP update. Block update messages are "coarse": something changed here, check it yourself. Game event messages are more like recording facts: a `STEP` happened here, a `BLOCK_PLACE` happened here. The former serves block state propagation, the latter serves listener perception.

## Event Dispatchers in WorldChunk

`WorldChunk` maintains a game event dispatcher mapping:

```java
private final Int2ObjectMap<GameEventDispatcher> gameEventDispatchers = new Int2ObjectOpenHashMap<>();
```

The key is the section Y coordinate, i.e., the vertical 16-block interval obtained by `ChunkSectionPos.getSectionCoord(y)`; the value is the `GameEventDispatcher` corresponding to that section. This indicates game event listeners aren't stuffed into one big list for the entire chunk, but are managed split by section.

When external code needs a dispatcher for a section, it calls:

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

Server-side `WorldChunk` lazily creates `SimpleGameEventDispatcher`: a dispatcher appears in the mapping only when a section truly needs listeners. Client or non-server paths fall back to parent class logic, not maintaining server-side listener collections here.

One dispatcher per section has two intuitive benefits. First, when emitting events, you don't need to mix all listeners from the entire chunk and filter them; sections provide an initial spatial grouping. Second, listener addition and removal are concentrated in smaller lists, avoiding piling all chunk listeners onto a single dispatcher or making a single dispatcher a unified contention point.

`SimpleGameEventDispatcher`'s internal structure is lightweight:

```java
private final List<GameEventListener> listeners = Lists.newArrayList();
private final Set<GameEventListener> toRemove = Sets.newHashSet();
private final List<GameEventListener> toAdd = Lists.newArrayList();
private boolean dispatching;
```

`listeners` stores the current section's game event listeners. If an event is being dispatched, additions and removals don't immediately modify the list being traversed, but first enter `toAdd` / `toRemove`, then merge after dispatch ends. This is similar to the pending list approach for block entity tickers: avoid modifying the current collection during traversal.

## Listener Registration and Lifecycle

Take sculk sensors as an example. When a sculk sensor is placed, the block state's corresponding block entity is created as `SculkSensorBlockEntity`. This block entity implements `GameEventListener.Holder<Vibrations.VibrationListener>`, and its `getEventListener()` returns its own `Vibrations.VibrationListener`. Sculk shriekers `SculkShriekerBlockEntity` follow the same pattern.

`WorldChunk` doesn't need to recognize every type of sculk block. It only queries through `BlockEntityProvider.getGameEventListener(ServerWorld, blockEntity)`: does this block entity provide a game event listener? The default implementation checks if the block entity is a `GameEventListener.Holder<?>`, and if so, extracts `holder.getEventListener()`.

Registration occurs in `WorldChunk.updateGameEventListener()`:

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

This code registers the listener to "the section dispatcher corresponding to the block entity's Y coordinate." After `addListener(GameEventListener)`, the dispatcher will consider this listener during this section's event dispatch; whether it actually receives depends on the listener's position, range, tags, and callback logic.

Note this lifecycle is bound to the block entity. When a block entity joins a chunk, `WorldChunk.addBlockEntity()` calls `updateGameEventListener()` when the chunk has already entered world runtime state, then updates the block entity ticker. When the chunk fills block entities from NBT or network packets, `updateAllBlockEntities()` also re-registers listeners for existing block entities.

Removal takes the reverse path. When a block is destroyed, replaced, or a block entity is removed from a chunk, `WorldChunk.removeBlockEntity()` extracts the `GameEventListener` corresponding to that block entity, finds the `GameEventDispatcher` for its section, then calls:

```java
gameEventDispatcher.removeListener(gameEventListener);
```

After `SimpleGameEventDispatcher.removeListener()` removes the listener, if `listeners` is now empty, it calls the disposal callback passed during construction:

```java
if (this.listeners.isEmpty()) {
    this.disposalCallback.apply(this.ySectionCoord);
}
```

In `WorldChunk`, this callback is precisely `removeGameEventDispatcher(int ySectionCoord)`:

```java
private void removeGameEventDispatcher(int ySectionCoord) {
    this.gameEventDispatchers.remove(ySectionCoord);
}
```

Thus, dispatcher cleanup isn't a separate global table scan, but when the last listener in a section is removed, it removes that section's dispatcher from `gameEventDispatchers` on the side. Empty sections don't keep empty dispatchers.

Chunk unloading can also be understood following this logic. As Chapter 11 explained, unloading isn't an instant "evaporation" when `ChunkHolder` disappears, but TACS moving the holder from the current table to the pending unload structure, then executing save, entity unload, light state refresh, and other cleanup in unload tasks. For game event listeners, the key is that block entities must be removed or marked for removal when exiting the running set with the chunk, and corresponding listeners should be unregistered from their section's dispatcher; when all listeners in a section leave, `removeGameEventDispatcher()` clears the dispatcher mapping entry.

This is also why `gameEventDispatchers` is placed on `WorldChunk`, not in a world-level global list: listener and chunk section lifecycles are naturally bound. When a chunk loads, block entities restore and listeners re-register; when block entities disappear or the chunk unloads, listeners are removed and empty dispatchers disappear with them.

## Relationship with the Vibration System

`VibrationListener` is a specialized implementation of game event listeners. Sculk sensors and sculk shriekers use it to receive specific `GameEvent`, then decide whether to activate based on distance, frequency, tags, and their own state. Allays (`AllayEntity`) also use vibration-related listeners, but they're attached to the entity's `EntityGameEventHandler`, not in `WorldChunk`'s block entity listener mapping.

In other words, the vibration system isn't a separate parallel entry to `GameEvent`, but a set of rules built on top of game event listeners: which events can be heard, what transmission distance is allowed, how to convert frequency, and how listeners respond upon receiving. These vibration propagation and frequency classification details aren't covered in this chapter.

## Key Points Review

- NC/PP updates are block state propagation mechanisms; `GameEvent` is event notification for listeners.
- `GameEvent` records "what happened," it doesn't directly require blocks to change state.
- `WorldChunk.gameEventDispatchers` is `Int2ObjectMap<GameEventDispatcher>`, storing dispatchers by section Y coordinate.
- `getGameEventDispatcher(int ySectionCoord)` lazily creates `SimpleGameEventDispatcher` on the server.
- If a block entity implements `GameEventListener.Holder<?>`, it can expose a listener via `BlockEntityProvider.getGameEventListener()`.
- Sculk sensors and sculk shriekers' `Vibrations.VibrationListener` register to their section's dispatcher.
- After listener removal, empty `SimpleGameEventDispatcher` callbacks `removeGameEventDispatcher()`, removing itself from the `WorldChunk` mapping.
- The chunk unload cleanup process exits block entities and listeners from running structures; section dispatchers are also cleaned with listener lifecycle.
