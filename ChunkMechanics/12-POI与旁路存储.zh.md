---
slug: point-of-interest
title: POI 与旁路存储
description: Point of Interest 为什么独立于普通区块保存、它如何按 section 组织数据，以及它在村民、袭击、流浪商人和区块生命周期中的位置。
index: 12
is-advanced: false
---

前面几篇把区块的加载、降温、保存和卸载串了起来，但还有一类数据一直贴着区块运行，却不住在普通区块 NBT 里：`PointOfInterestStorage`。它保存的是 AI 和世界规则经常要查询的“兴趣点”，也就是 POI。

这一篇专门补上这条旁路：POI 为什么独立保存，它在内存里长什么样，方块变化怎样更新它，以及它如何参与村民、袭击和流浪商人的逻辑。

## POI 是什么

`Point of Interest`（兴趣点，简称 **POI**）是“可被 AI 或世界逻辑查询的方块位置”：床、工作站、钟、蜂巢、下界传送门、磁石、避雷针等都可能注册成 POI。它不是 `BlockState` 本身的一部分，也不随普通区块方块状态一起查询；它是一份从方块状态旁边抽出的索引。村民找床和工作站、袭击系统判断村庄中心、流浪商人寻找集会点，都会绕到 `PointOfInterestStorage` 里查，而不是临时扫描附近所有方块。

## 为什么 POI 要独立存储

如果每次村民想找工作站，都从当前位置开始扫描几十格范围内每个方块的 `BlockState`，服务器会把大量时间浪费在重复遍历上。一个半径 48 的水平范围已经跨过多个区块；再乘上垂直 section、村民数量和每 tick 的 AI 调度，这种“现查现扫”的成本很快会失控。

POI 存储把问题换成了索引查询。`PointOfInterestStorage` 按 chunk section 组织 `PointOfInterestSet`，每个集合内部又按类型保存 POI。查询时，代码先用半径推导需要看的区块，再只遍历这些区块中已经登记的 POI，最后按类型、位置谓词和 `OccupationStatus` 过滤。换句话说，服务器查询的是“附近有哪些已知兴趣点”，而不是“附近每个方块是不是兴趣点”。

这也是它要独立保存的原因。普通区块方块数据写在维度目录的 `region/` 子目录下；POI 数据写在同一维度目录的 `poi/` 子目录下。TACS 构造时就把这条旁路接好：

```java
// ThreadedAnvilChunkStorage.java (Yarn 1.20.1+build.10)
super(session.getWorldDirectory(world.getRegistryKey()).resolve("region"), dataFixer, dsync);
Path path = session.getWorldDirectory(world.getRegistryKey());
this.pointOfInterestStorage = new PointOfInterestStorage(path.resolve("poi"), dataFixer, dsync, dynamicRegistryManager, world);
```

于是同一个维度下至少有两套与区块坐标相关的持久化数据：`region/` 保存方块、地形、方块实体和计划刻等普通区块内容；`poi/` 保存兴趣点索引。第 11 篇提到 TACS 保存区块时会先调用 `pointOfInterestStorage.saveChunk(chunk.getPos())`，就是为了让这条旁路和普通区块保存节奏保持一致。

## POI 的数据结构

`PointOfInterestStorage` 是顶层管理类。它继承 `SerializingRegionBasedStorage<PointOfInterestSet>`，父类负责按 `ChunkSectionPos` 加载、缓存、标脏和写回 region 文件；子类负责 POI 查询、距离传播和从区块 section 扫描方块状态。虽然文件目录按 chunk 保存，内存索引的 key 实际是 section 坐标：`ChunkSectionPos.toLong(pos)`。

`PointOfInterestSet` 是单个 chunk section 内所有 POI 的集合。它有两张表：`pointsOfInterestByPos` 用 section 内局部坐标快速找到某个位置的 POI；`pointsOfInterestByType` 用 `RegistryEntry<PointOfInterestType>` 分组，方便按类型流式查询。这个类名容易让人误以为“一个区块一个 set”，但在 1.20.1 的源码里，它对应的是一个 section；一个 chunk 的 POI 文件会在 `Sections` 下保存多个 y section。

`PointOfInterest` 是单个 POI 记录，包含方块位置 `pos`、兴趣点类型 `type`、空闲票数 `freeTickets` 和更新回调 `updateListener`。占用状态不是另存一个布尔值，而是由票数推出来：`hasSpace()` 判断 `freeTickets > 0`，`isOccupied()` 判断 `freeTickets != type.value().ticketCount()`。在 Yarn 1.20.1+build.10 中，`PointOfInterest` 没有独立的“最后引用时间”字段；一次占用或释放会修改 `freeTickets`，再通过 `updateListener.run()` 标记需要保存。

`PointOfInterestType` 则描述“哪些 `BlockState` 属于同一种 POI、最多能被多少对象占用、搜索距离是多少”。严格说它不是 Java `enum`，而是注册进 `RegistryKeys.POINT_OF_INTEREST_TYPE` 的 record；`PointOfInterestTypes` 是兴趣点类型注册表，负责注册 `HOME`、`MEETING`、各职业工作站、`NETHER_PORTAL`、`LODESTONE`、`LIGHTNING_ROD` 等类型，并维护 `BlockState -> RegistryEntry<PointOfInterestType>` 的反向映射。如果你从旧版村庄系统记得“门”，要注意 Yarn 1.20.1+build.10 的默认 `PointOfInterestTypes` 已经不把门注册为 POI。

## 方块变化如何影响 POI

POI 索引不是生成后永远不变。服务端每次看到方块状态变化时，`ServerWorld.onBlockChanged()` 会分别查旧 `BlockState` 和新 `BlockState` 是否对应 POI 类型：

```java
Optional<RegistryEntry<PointOfInterestType>> oldType = PointOfInterestTypes.getTypeForState(oldBlock);
Optional<RegistryEntry<PointOfInterestType>> newType = PointOfInterestTypes.getTypeForState(newBlock);
if (!Objects.equals(oldType, newType)) {
    oldType.ifPresent(type -> this.getServer().execute(() -> this.getPointOfInterestStorage().remove(blockPos)));
    newType.ifPresent(type -> this.getServer().execute(() -> this.getPointOfInterestStorage().add(blockPos, type)));
}
```

放置一个 POI 方块，例如织布机，对应的新 `BlockState` 会在 `PointOfInterestTypes` 里匹配到 `SHEPHERD` 类型，然后进入 `PointOfInterestStorage.add(pos, type)`。`add()` 根据方块位置算出 section 坐标，取出或创建对应的 `PointOfInterestSet`，再把新的 `PointOfInterest` 放进位置索引和类型索引。破坏这个方块时，旧状态仍能匹配到 POI 类型，新状态匹配不到，于是调用 `remove(pos)`，从同一个 section 的集合里删掉记录。

添加、删除、占用和释放都会触发更新回调。源码里没有一个直接叫 `markDirty()` 的公开方法；这个角色由 `SerializingRegionBasedStorage.onUpdate(long pos)` 承担。`PointOfInterestSet` 收到变化后运行 `updateListener`，父类把该 section 的 long 坐标加入 `unsavedElements`。之后 TACS 的 `tick()` 会调用 `pointOfInterestStorage.tick(shouldKeepTicking)`，或者 TACS 保存某个区块时调用 `pointOfInterestStorage.saveChunk(chunk.getPos())`，再把同一 chunk 下的脏 section 序列化进 `poi/`。

还有一条容易忽略的路径是预加载。`PointOfInterestStorage.preloadChunks(world, pos, radius)` 会在指定范围内检查 POI section 是否已经有效；如果无效，就把对应 chunk 坐标加入 `preloadedChunks`，并用 `world.getChunk(chunkX, chunkZ, ChunkStatus.EMPTY)` 触发最低状态加载。这不是为了生成完整区块，而是为了让经常被查询的 POI 数据尽早进入内存，典型调用者包括传送门搜索这类需要反复查 POI 的逻辑。

## POI 与村民/袭击系统

村民找工作站、床或集会点时，通常不是直接读方块，而是让 `PointOfInterestStorage` 在范围内找满足类型和占用条件的位置。例如“找最近的某类 POI”会走 `getNearestPosition(typePredicate, pos, radius, occupationStatus)`；需要预定工作站时，还可能走带 `HAS_SPACE` 的 `getPosition(...)`，找到后调用 `reserveTicket()` 占用一张票。

袭击系统也把 POI 当作村庄判断的输入。`RaidManager.startRaid()` 会查询玩家附近半径 64 内所有属于 `PointOfInterestTypeTags.VILLAGE` 且 `OccupationStatus.IS_OCCUPIED` 的 POI，把这些位置取平均，作为袭击中心。这里的“村庄”不是一个单独保存在磁盘上的 Village 对象，而是由已占用的床和钟等 POI 在运行时共同推导出来。

流浪商人则会寻找集会点。`WanderingTraderManager` 使用 `getPosition(poiType -> poiType.matchesKey(PointOfInterestTypes.MEETING), pos -> true, blockPos, 48, OccupationStatus.ANY)` 找 48 格内的 `MEETING` POI，也就是钟；找不到时才退回玩家位置。`OccupationStatus` 的三个值很直白：`ANY` 表示不看占用，`HAS_SPACE` 表示还有空闲票，`IS_OCCUPIED` 表示已经被至少一个对象占用。

## POI 在区块生命周期中的位置

POI 不是普通区块的一部分，但它跟区块生命周期贴得很紧。区块 section 初次加载或调色板需要补齐时，`PointOfInterestStorage.initForPalette(sectionPos, chunkSection)` 会先用 `shouldScan(chunkSection)` 判断这个 section 是否含有任何 POI 方块状态；如果有，就逐格扫描这个 section，把 `PointOfInterestTypes.getTypeForState(blockState)` 找到的类型填进对应 `PointOfInterestSet`。这一步把真实方块状态转换成旁路索引。

运行期间，方块变化让 POI 增删；AI 查询读取 POI 并可能改动票数；TACS tick 推进 POI 的脏数据保存；TACS 保存 chunk 时又先调用 `saveChunk()` 兜底。关服判断里也包含 `pointOfInterestStorage.hasUnsavedElements()`，说明只要 POI 还有没写回的 section，服务器就不能认为这个维度已经完全收尾。

可以把 POI 理解成“区块旁边的一张 AI 索引表”：它从区块方块状态中派生，按区块坐标保存，但查询、占用和写盘都有自己的生命周期。普通区块负责回答“这里是什么方块”；POI 负责快速回答“附近有哪些 AI 关心的点”。

## 小结

- POI 是床、工作站、钟等可被 AI 和世界逻辑查询的兴趣点，不是 `BlockState` 的内嵌字段。
- `PointOfInterestStorage` 独立保存在维度目录的 `poi/` 下，与普通区块 `region/` 分离。
- `PointOfInterestSet` 在 1.20.1 中按 chunk section 保存 POI，并同时维护按位置和按类型的索引。
- 方块变化通过 `ServerWorld.onBlockChanged()` 转成 `add()` / `remove()`，再由更新回调标记脏 section。
- 村民、袭击、流浪商人都依赖 POI 查询，但它们各自的 AI 细节不属于 POI 存储本身。

## [!ADVANCED] 源码走读

### PointOfInterestStorage 构造器

`PointOfInterestStorage` 的构造器非常短：

```java
public PointOfInterestStorage(Path path, DataFixer dataFixer, boolean dsync, DynamicRegistryManager registryManager, HeightLimitView world) {
    super(path, PointOfInterestSet::createCodec, PointOfInterestSet::new, dataFixer, DataFixTypes.POI_CHUNK, dsync, registryManager, world);
    this.pointOfInterestDistanceTracker = new PointOfInterestStorage.PointOfInterestDistanceTracker();
}
```

关键都藏在 `super(...)` 的参数里。`path` 是 TACS 传入的 `path.resolve("poi")`，决定了磁盘目录；`PointOfInterestSet::createCodec` 决定每个 section 怎样序列化；`PointOfInterestSet::new` 是缺失 section 的工厂；`DataFixTypes.POI_CHUNK` 表示读旧版本 POI NBT 时走 POI 专用的数据修复类型；`registryManager` 让 codec 能解析 `PointOfInterestType` 注册表项；`world` 提供高度范围，用来知道一个 chunk 有哪些 section。

父类 `SerializingRegionBasedStorage` 会创建自己的 `StorageIoWorker`：

```java
this.worker = new StorageIoWorker(path, dsync, path.getFileName().toString());
```

所以 POI 的异步 IO worker 和普通区块的 `VersionedChunkStorage` 不是同一个对象。TACS 只是同时持有它们，并在保存和 tick 的时候把两边都推进。

构造器最后创建 `PointOfInterestDistanceTracker`。这是一个 `SectionDistanceLevelPropagator`，用来维护“某个 section 离最近的已占用村庄 POI 有多远”。`ServerWorld.isNearOccupiedPointOfInterest()` 最终会读这个距离；它服务于村庄相关判断，但仍然只是一层 POI 距离缓存，不是完整村民 AI。

### add() / remove() 的内部逻辑

`PointOfInterestStorage.add()` 本身只做一次路由：

```java
public void add(BlockPos pos, RegistryEntry<PointOfInterestType> type) {
    this.getOrCreate(ChunkSectionPos.toLong(pos)).add(pos, type);
}
```

`ChunkSectionPos.toLong(pos)` 把方块位置压成 section 坐标。`getOrCreate()` 来自父类：如果 section 已加载，就返回已有的 `PointOfInterestSet`；如果磁盘里没有这个 section 的 POI 数据，就用 `PointOfInterestSet::new` 创建一个空集合，并把更新回调绑定到 `onUpdate(pos)`。

真正插入发生在 `PointOfInterestSet.add()`：

```java
public void add(BlockPos pos, RegistryEntry<PointOfInterestType> type) {
    if (this.add(new PointOfInterest(pos, type, this.updateListener))) {
        this.updateListener.run();
    }
}
```

内部私有 `add(PointOfInterest poi)` 会先用 `ChunkSectionPos.packLocal(blockPos)` 得到 section 内短坐标。如果同一局部位置已经存在相同类型的 POI，就返回 `false`，避免重复标脏；如果已有 POI 但类型不同，会记录“POI data mismatch”。成功时，它同时写入 `pointsOfInterestByPos` 和 `pointsOfInterestByType`。

`remove()` 也分两层：

```java
public void remove(BlockPos pos) {
    this.get(ChunkSectionPos.toLong(pos)).ifPresent(poiSet -> poiSet.remove(pos));
}
```

它不会无条件创建集合，而是只在对应 section 存在时删除。`PointOfInterestSet.remove(pos)` 用局部短坐标从 `pointsOfInterestByPos` 移除，再从 `pointsOfInterestByType` 中对应类型的集合移除。成功删除后运行 `updateListener`；如果找不到记录，则记录“never registered”的错误。

占用状态也走同一套保存链。`reserveTicket()` 会减少 `freeTickets`，`releaseTicket()` 会增加 `freeTickets`，二者都会调用 `updateListener.run()`。因此“村民占用了工作站”这种逻辑变化也会让 POI section 变脏，而不需要方块本身发生变化。

### markDirty() 触发的保存机制

在这组源码里，POI 的“标脏”动作不是一个名为 `markDirty()` 的公开方法，而是 `SerializingRegionBasedStorage.onUpdate(long pos)`：

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

`PointOfInterestSet` 的 `updateListener` 最终调用这里。`unsavedElements` 保存的是 section 坐标，不是 chunk 坐标；这让 POI 可以精确知道哪个 section 发生了变化。真正写盘时再把同一 chunk 的所有 section 合并成一个 NBT：

```java
private <T> Dynamic<T> serialize(ChunkPos chunkPos, DynamicOps<T> ops) {
    Map<T, T> map = Maps.newHashMap();
    for (int y = this.world.getBottomSectionCoord(); y < this.world.getTopSectionCoord(); y++) {
        long sectionPos = chunkSectionPosAsLong(chunkPos, y);
        this.unsavedElements.remove(sectionPos);
        Optional<R> optional = this.loadedElements.get(sectionPos);
        if (optional != null && optional.isPresent()) {
            // 写入 Sections[y]
        }
    }
    // 返回包含 Sections 和 DataVersion 的 NBT
}
```

保存入口有两个。第一个是 POI 自己的 tick：

```java
protected void tick(BooleanSupplier shouldKeepTicking) {
    while (this.hasUnsavedElements() && shouldKeepTicking.getAsBoolean()) {
        ChunkPos chunkPos = ChunkSectionPos.from(this.unsavedElements.firstLong()).toChunkPos();
        this.save(chunkPos);
    }
}
```

TACS 每 tick 在 `poi` profiler 段调用它：

```java
profiler.push("poi");
this.pointOfInterestStorage.tick(shouldKeepTicking);
```

第二个入口是普通区块保存前的兜底：

```java
private boolean save(Chunk chunk) {
    this.pointOfInterestStorage.saveChunk(chunk.getPos());
    if (!chunk.needsSaving()) {
        return false;
    }
    // 普通 chunk NBT 保存
}
```

`saveChunk(chunkPos)` 会检查这个 chunk 的各个 section 是否在 `unsavedElements` 中；只要命中一个，就保存整个 POI chunk。注意它发生在 `chunk.needsSaving()` 判断之前，所以即使普通区块本体没有脏数据，POI 仍然可以被写回。这正是“旁路存储”的核心：POI 从区块派生、跟区块同坐标推进，但它的脏标记和持久化判断独立存在。

## 参考

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
