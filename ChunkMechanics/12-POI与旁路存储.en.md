---
slug: point-of-interest
title: POI and Bypass Storage
description: Why Point of Interest is stored independently from regular chunks, how it organizes data by section, and its role in villagers, raids, wandering traders, and chunk lifecycle.
index: 12
is-advanced: false
---

Previous chapters covered chunk loading, cooling, saving, and unloading, but there's one type of data that runs alongside chunks yet doesn't live in regular chunk NBT: `PointOfInterestStorage`. It stores "points of interest" frequently queried by AI and world logic, known as POI.

This chapter fills in this bypass path: why POI is stored independently, how it's structured in memory, how block changes update it, and how it participates in villager, raid, and wandering trader logic.

## What is POI

`Point of Interest` (POI) refers to "block positions queryable by AI or world logic": beds, job sites, bells, beehives, nether portals, lodestones, lightning rods, etc. can all be registered as POI. It's not part of `BlockState` itself, nor is it queried alongside regular chunk block states; it's an index extracted from block state. When villagers look for beds and job sites, when the raid system determines village centers, or when wandering traders find meeting points, they query `PointOfInterestStorage` rather than scanning nearby blocks.

## Why POI Requires Independent Storage

If every time a villager wants to find a job site, the server scanned every block's `BlockState` within a range of several dozen blocks, it would waste massive time on repeated traversal. A 48-block horizontal radius already spans multiple chunks; multiply that by vertical sections, villager count, and per-tick AI scheduling, and this "scan-on-demand" cost quickly spirals out of control.

POI storage converts the problem into an index query. `PointOfInterestStorage` organizes `PointOfInterestSet` by chunk section, with each set internally storing POI by type. During queries, code first derives which chunks to check from the radius, then only traverses POI already registered in those chunks, finally filtering by type, position predicate, and `OccupationStatus`. In other words, the server queries "what known points of interest are nearby," not "is every nearby block a point of interest."

This is why it requires independent storage. Regular chunk block data is written to the `region/` subdirectory under the dimension directory; POI data is written to the `poi/` subdirectory under the same dimension directory. TACS sets up this bypass path during construction:

```java
// ThreadedAnvilChunkStorage.java (Yarn 1.20.1+build.10)
super(session.getWorldDirectory(world.getRegistryKey()).resolve("region"), dataFixer, dsync);
Path path = session.getWorldDirectory(world.getRegistryKey());
this.pointOfInterestStorage = new PointOfInterestStorage(path.resolve("poi"), dataFixer, dsync, dynamicRegistryManager, world);
```

Thus, a single dimension has at least two sets of persistent data related to chunk coordinates: `region/` stores blocks, terrain, block entities, and scheduled ticks; `poi/` stores the point-of-interest index. Chapter 11 mentioned that when TACS saves a chunk, it first calls `pointOfInterestStorage.saveChunk(chunk.getPos())` to keep this bypass path in sync with the regular chunk save rhythm.

## POI Data Structure

`PointOfInterestStorage` is the top-level manager. It extends `SerializingRegionBasedStorage<PointOfInterestSet>`, with the parent class handling loading, caching, marking dirty, and writing back region files by `ChunkSectionPos`; the child class handles POI queries, distance propagation, and scanning block states from chunk sections. Although the file directory saves by chunk, the memory index key is actually the section coordinate: `ChunkSectionPos.toLong(pos)`.

`PointOfInterestSet` is the set of all POI within a single chunk section. It has two tables: `pointsOfInterestByPos` uses section-local coordinates to quickly find the POI at a position; `pointsOfInterestByType` groups by `RegistryEntry<PointOfInterestType>`, facilitating type-based streaming queries. This class name can be misleading ("one set per chunk"), but in 1.20.1 source, it corresponds to one section; a chunk's POI file will save multiple y sections under `Sections`.

`PointOfInterest` is a single POI record, containing block position `pos`, interest point type `type`, free tickets `freeTickets`, and an update callback `updateListener`. Occupation status isn't stored as a separate boolean, but derived from ticket count: `hasSpace()` checks `freeTickets > 0`, `isOccupied()` checks `freeTickets != type.value().ticketCount()`. In Yarn 1.20.1+build.10, `PointOfInterest` has no standalone "last reference time" field; an occupation or release modifies `freeTickets`, then triggers `updateListener.run()` to mark for save.

`PointOfInterestType` describes "which `BlockState` belong to the same POI type, how many objects can occupy it, and what search distance is allowed." Strictly speaking, it's not a Java `enum`, but a record registered in `RegistryKeys.POINT_OF_INTEREST_TYPE`; `PointOfInterestTypes` is the interest point type registry, responsible for registering `HOME`, `MEETING`, profession job sites, `NETHER_PORTAL`, `LODESTONE`, `LIGHTNING_ROD`, etc., and maintaining the reverse mapping `BlockState -> RegistryEntry<PointOfInterestType>`. If you remember "doors" from the old village system, note that Yarn 1.20.1+build.10's default `PointOfInterestTypes` no longer registers doors as POI.

## How Block Changes Affect POI

POI indices aren't static after generation. Every time the server detects a block state change, `ServerWorld.onBlockChanged()` checks whether the old and new `BlockState` correspond to POI types:

```java
Optional<RegistryEntry<PointOfInterestType>> oldType = PointOfInterestTypes.getTypeForState(oldBlock);
Optional<RegistryEntry<PointOfInterestType>> newType = PointOfInterestTypes.getTypeForState(newBlock);
if (!Objects.equals(oldType, newType)) {
    oldType.ifPresent(type -> this.getServer().execute(() -> this.getPointOfInterestStorage().remove(blockPos)));
    newType.ifPresent(type -> this.getServer().execute(() -> this.getPointOfInterestStorage().add(blockPos, type)));
}
```

Placing a POI block, like a loom, matches the new `BlockState` to the `SHEPHERD` type in `PointOfInterestTypes`, then enters `PointOfInterestStorage.add(pos, type)`. `add()` derives the section coordinate from the block position, retrieves or creates the corresponding `PointOfInterestSet`, and puts the new `PointOfInterest` into both the position index and type index. When destroying this block, the old state still matches the POI type, the new state doesn't, so it calls `remove(pos)`, removing the record from the same section's set.

Adding, removing, occupying, and releasing all trigger update callbacks. The source doesn't have a public method directly called `markDirty()`; this role is fulfilled by `SerializingRegionBasedStorage.onUpdate(long pos)`. After `PointOfInterestSet` receives a change, it runs `updateListener`, and the parent class adds the section's long coordinate to `unsavedElements`. Later, TACS's `tick()` calls `pointOfInterestStorage.tick(shouldKeepTicking)`, or when TACS saves a chunk it calls `pointOfInterestStorage.saveChunk(chunk.getPos())`, serializing dirty sections under the same chunk into `poi/`.

Another easily overlooked path is preloading. `PointOfInterestStorage.preloadChunks(world, pos, radius)` checks whether POI sections are valid within a specified range; if invalid, it adds the corresponding chunk coordinate to `preloadedChunks` and triggers minimum-status loading with `world.getChunk(chunkX, chunkZ, ChunkStatus.EMPTY)`. This isn't to generate complete chunks, but to get frequently queried POI data into memory early; typical callers include portal search logic that repeatedly queries POI.

## POI and Villagers/Raid System

When villagers look for job sites, beds, or meeting points, they typically don't read blocks directly, but ask `PointOfInterestStorage` to find positions satisfying type and occupation conditions within range. For example, "find nearest POI of a certain type" goes through `getNearestPosition(typePredicate, pos, radius, occupationStatus)`; when reserving a job site, it may go through `getPosition(...)` with `HAS_SPACE`, then call `reserveTicket()` after finding one to occupy a ticket.

The raid system also uses POI as input for village determination. `RaidManager.startRaid()` queries all POI within a 64-block radius around the player that belong to `PointOfInterestTypeTags.VILLAGE` and have `OccupationStatus.IS_OCCUPIED`, averages these positions, and uses that as the raid center. The "village" here isn't a separate Village object saved to disk, but derived at runtime from occupied POI like beds and bells.

Wandering traders seek meeting points. `WanderingTraderManager` uses `getPosition(poiType -> poiType.matchesKey(PointOfInterestTypes.MEETING), pos -> true, blockPos, 48, OccupationStatus.ANY)` to find `MEETING` POI (bells) within 48 blocks; if none found, it falls back to the player's position. The three values of `OccupationStatus` are straightforward: `ANY` means ignore occupation, `HAS_SPACE` means free tickets remain, `IS_OCCUPIED` means at least one object has occupied it.

## POI in the Chunk Lifecycle

POI isn't part of regular chunks, but it's tightly integrated with the chunk lifecycle. When a chunk section is first loaded or the palette needs to be filled, `PointOfInterestStorage.initForPalette(sectionPos, chunkSection)` first uses `shouldScan(chunkSection)` to determine whether this section contains any POI block states; if so, it scans every position in the section and fills types found by `PointOfInterestTypes.getTypeForState(blockState)` into the corresponding `PointOfInterestSet`. This step converts real block states into the bypass index.

During runtime, block changes cause POI additions/deletions; AI queries read POI and may modify ticket counts; TACS tick advances POI dirty data saves; when TACS saves a chunk, it first calls `saveChunk()` as a fallback. The shutdown check also includes `pointOfInterestStorage.hasUnsavedElements()`, indicating that as long as POI still has unwritten sections, the server cannot consider this dimension fully finalized.

POI can be understood as "an AI index table alongside chunks": it derives from chunk block states, saves by chunk coordinates, but queries, occupation, and disk writes all have their own lifecycle. Regular chunks answer "what block is here"; POI quickly answers "what AI-relevant points are nearby."

## [!ADVANCED] Source Code Walkthrough

### PointOfInterestStorage Constructor

The `PointOfInterestStorage` constructor is very short:

```java
public PointOfInterestStorage(Path path, DataFixer dataFixer, boolean dsync, DynamicRegistryManager registryManager, HeightLimitView world) {
    super(path, PointOfInterestSet::createCodec, PointOfInterestSet::new, dataFixer, DataFixTypes.POI_CHUNK, dsync, registryManager, world);
    this.pointOfInterestDistanceTracker = new PointOfInterestStorage.PointOfInterestDistanceTracker();
}
```

The important parts are hidden inside the `super(...)` arguments. `path` is the `path.resolve("poi")` passed in by TACS, determining the disk directory. `PointOfInterestSet::createCodec` determines how each section is serialized. `PointOfInterestSet::new` is the factory for missing sections. `DataFixTypes.POI_CHUNK` means old-version POI NBT uses the dedicated POI data-fixer type. `registryManager` lets the codec resolve `PointOfInterestType` registry entries. `world` provides the height range, telling it which sections a chunk contains.

The parent class `SerializingRegionBasedStorage` creates its own `StorageIoWorker`:

```java
this.worker = new StorageIoWorker(path, dsync, path.getFileName().toString());
```

So the async I/O worker for POI is not the same object as the `VersionedChunkStorage` used by ordinary chunks. TACS merely holds both and advances both during saving and ticking.

At the end of the constructor, it creates `PointOfInterestDistanceTracker`. This is a `SectionDistanceLevelPropagator` used to maintain "how far a section is from the nearest occupied village POI." `ServerWorld.isNearOccupiedPointOfInterest()` ultimately reads this distance. It serves village-related checks, but it is still only a POI distance cache, not full villager AI.

### Internal Logic of add() / remove()

`PointOfInterestStorage.add()` itself only performs one routing step:

```java
public void add(BlockPos pos, RegistryEntry<PointOfInterestType> type) {
    this.getOrCreate(ChunkSectionPos.toLong(pos)).add(pos, type);
}
```

`ChunkSectionPos.toLong(pos)` packs the block position into section coordinates. `getOrCreate()` comes from the parent class. If the section is already loaded, it returns the existing `PointOfInterestSet`. If the disk has no POI data for that section, it creates an empty set with `PointOfInterestSet::new` and binds the update callback to `onUpdate(pos)`.

The actual insertion happens in `PointOfInterestSet.add()`:

```java
public void add(BlockPos pos, RegistryEntry<PointOfInterestType> type) {
    if (this.add(new PointOfInterest(pos, type, this.updateListener))) {
        this.updateListener.run();
    }
}
```

The internal private `add(PointOfInterest poi)` first uses `ChunkSectionPos.packLocal(blockPos)` to get the section-local short coordinate. If the same local position already has a POI of the same type, it returns `false` to avoid repeated dirty marking. If a POI already exists there but with a different type, it records a "POI data mismatch". On success, it writes to both `pointsOfInterestByPos` and `pointsOfInterestByType`.

`remove()` is also split into two layers:

```java
public void remove(BlockPos pos) {
    this.get(ChunkSectionPos.toLong(pos)).ifPresent(poiSet -> poiSet.remove(pos));
}
```

It does not create a set unconditionally. It only removes when the corresponding section exists. `PointOfInterestSet.remove(pos)` removes the entry from `pointsOfInterestByPos` using the local short coordinate, then removes it from the corresponding typed set in `pointsOfInterestByType`. On successful deletion it runs `updateListener`. If no record is found, it logs a "never registered" error.

Occupation state also uses the same save chain. `reserveTicket()` decreases `freeTickets`, and `releaseTicket()` increases `freeTickets`. Both call `updateListener.run()`. Therefore, a logic change such as "a villager occupied a workstation" also makes the POI section dirty even though the block itself did not change.

### Save Mechanism Triggered by markDirty()

In this set of source files, the "mark dirty" action for POI is not a public method called `markDirty()`, but `SerializingRegionBasedStorage.onUpdate(long pos)`:

```java
protected void onUpdate(long pos) {
    Optional<R> optional = this.loadedElements.get(pos);
    if (optional != null && optional.isPresent()) {
        this.unsavedElements.add(pos);
    } else {
        LOGGER.warn("No data for position: {}", ChunkSectionPos.from(pos));
    }
}
```

`PointOfInterestSet`'s `updateListener` ultimately calls this. `unsavedElements` stores section coordinates rather than chunk coordinates. This lets POI know exactly which section changed. During actual disk write, all sections under the same chunk are merged into one NBT:

```java
private <T> Dynamic<T> serialize(ChunkPos chunkPos, DynamicOps<T> ops) {
    Map<T, T> map = Maps.newHashMap();
    for (int y = this.world.getBottomSectionCoord(); y < this.world.getTopSectionCoord(); y++) {
        long sectionPos = chunkSectionPosAsLong(chunkPos, y);
        this.unsavedElements.remove(sectionPos);
        Optional<R> optional = this.loadedElements.get(sectionPos);
        if (optional != null && optional.isPresent()) {
            // Write Sections[y]
        }
    }
    // Return NBT containing Sections and DataVersion
}
```

There are two save entry points. The first is POI's own tick:

```java
protected void tick(BooleanSupplier shouldKeepTicking) {
    while (this.hasUnsavedElements() && shouldKeepTicking.getAsBoolean()) {
        ChunkPos chunkPos = ChunkSectionPos.from(this.unsavedElements.firstLong()).toChunkPos();
        this.save(chunkPos);
    }
}
```

TACS calls it every tick in the `poi` profiler section:

```java
profiler.push("poi");
this.pointOfInterestStorage.tick(shouldKeepTicking);
```

The second entry point is the fallback before regular chunk save:

```java
private boolean save(Chunk chunk) {
    this.pointOfInterestStorage.saveChunk(chunk.getPos());
    if (!chunk.needsSaving()) {
        return false;
    }
    // Regular chunk NBT save
}
```

`saveChunk(chunkPos)` checks whether any section of that chunk is in `unsavedElements`. As long as one section matches, it saves the whole POI chunk. Note that this happens before the `chunk.needsSaving()` check. So even if the regular chunk body has no dirty data, POI can still be written back. This is exactly the core of "bypass storage": POI is derived from chunks and advances alongside chunk coordinates, but its dirty marking and persistence logic exist independently.

## References

- `net.minecraft.world.poi.PointOfInterestStorage`
- `net.minecraft.world.poi.PointOfInterestSet`
- `net.minecraft.world.poi.PointOfInterest`
- `net.minecraft.world.poi.PointOfInterestType`
- `net.minecraft.world.poi.PointOfInterestTypes`
- `net.minecraft.world.storage.SerializingRegionBasedStorage`
- `net.minecraft.server.world.ThreadedAnvilChunkStorage`
- `net.minecraft.server.world.ServerWorld`
- `net.minecraft.village.raid.RaidManager`
- `net.minecraft.world.WanderingTraderManager`
