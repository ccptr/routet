# Embedded routing engines: OsmAnd (main) and Navit (secondary)

*Research date: 2026-10-23. Sources were read from local clones:*

| repo | path | HEAD date | role |
|---|---|---|---|
| `osmandapp/OsmAnd` | `~/g/OsmAnd` | 2026-10-23 | Android app + **OsmAnd-java** (Java router, the reference implementation) + **OsmAnd-shared** (Kotlin Multiplatform copy of the router, for iOS) |
| `osmandapp/OsmAnd-core-legacy` | `~/g/OsmAnd-core-legacy` | 2026-10-21 | **native C++ router** (`native/src/*`), used through JNI on Android |
| `osmandapp/OsmAnd-core` | `~/g/OsmAnd-core` | – | the newer Qt-based core (rendering). Its `src/Routing/` is a 2014-era router that is effectively dead (last real change 2019). **Not** the production router. |
| `osmandapp/OsmAnd-tools` | `~/g/OsmAnd-tools` | 2026-10-22 | map generation (OsmAndMapCreator), **HH preprocessing**, OsmAndServer (web routing API) |
| `osmandapp/OsmAnd-resources` | `~/g/OsmAnd-resources` | – | `routing/routing.xml` profiles, `protos/OBF.proto`, voice scripts |
| `navit-gps/navit` | `~/g/navit` | 2026-09-07 (src), 2026-10-21 (route.c) | Navit |

Repo-relative paths below use the prefixes `OsmAnd:`, `legacy:`, `tools:`, `resources:`, `navit:`.
The Java router lives at `OsmAnd:OsmAnd-java/src/main/java/net/osmand/router/` (abbreviated **`router/`**) and the OBF readers at `OsmAnd:OsmAnd-java/src/main/java/net/osmand/binary/` (abbreviated **`binary/`**).

---

## Part I: OsmAnd

### 1. Overview

**What it is.** OsmAnd ("OSM Automated Navigation Directions") is a mobile-first, fully offline navigation app for Android and iOS. Its routing engine is not a separate product. It is a library inside the app's pure-Java core module (`OsmAnd-java`), with a hand-maintained C++ port for speed on Android and, since September 2026, a Kotlin Multiplatform port for iOS. The same Java code also runs server-side in `OsmAndServer` (the routing behind osmand.net/map) and in the desktop OsmAndMapCreator tooling.

**Three parallel implementations, synced by hand:**

| impl | location | used by | notes |
|---|---|---|---|
| Java | `router/*.java` (~20k LOC) | Android in "safe mode", server, tools, tests, HH preprocessing | reference implementation; new features land here first |
| C++ | `legacy:native/src/` (`binaryRoutePlanner.cpp` 1270, `hhRoutePlanner.cpp` 1536, `routeResultPreparation.cpp` 2710, `generalRouter.cpp` 885, `binaryRead.cpp` 4275 …) | Android default (JNI, `legacy:native/src/java_wrap.cpp` → `Java_net_osmand_NativeLibrary_nativeRouting`, `nativeSearchGpxRoute`, `nativeTransportRouting`) | commit messages such as "Port turn lanes fixes #21764 … from Java" (2026-10-05) show the porting workflow |
| Kotlin (KMP) | `OsmAnd:OsmAnd-shared/src/commonMain/kotlin/net/osmand/shared/routing/` (~17k LOC) | iOS (replacing C++) | added 2026-09-13/14 by `alex-dev` ("Copy the route planner to OsmAnd-shared", "Copy the HH route planner…"). The KDoc in `RoutePlannerFrontEnd.kt` says: *"A copy of `net.osmand.router.RoutePlannerFrontEnd`, which stays in OsmAnd-java for android and tools; this copy is for iOS … on iOS this planner is what replaces the C++ one."* Compatibility tests: `OsmAnd:OsmAnd-java/src/test/java/net/osmand/shared/compat/{RoutePlannerCompatTest,HHRoutePlannerCompatTest,ApproximationCompatTest}.java` |

**Licensing specifics** (`OsmAnd:LICENSE`, `OsmAnd:PULL_REQUEST.MIT.LICENSE`):
- **Code is GPLv3**: OsmAnd, OsmAnd-java, OsmAnd-core(-legacy), OsmAnd-tools, OsmAnd-resources. GPLv3 is copyleft. Linking the router into a non-GPL product is not allowed. Porting ideas (algorithms) is fine; translating code line by line creates a derivative work.
- **UI/UX artwork is CC-BY-NC-ND 4.0**. *"Publishing applications using the OsmAnd UI/UX code to Google Play, Amazon Market, or Apple Store must be done with written permission."* This does not affect the router, but it matters if anyone forks the app.
- **Inbound contributions are MIT** ("All pull requests are accepted under MIT License"). This asymmetric arrangement lets OsmAnd BV relicense its own code. Copyright holder: OsmAnd BV (Amstelveen, NL).
- Generated TTS voice files are proprietary. Gnu Trove (LGPL) is bundled as a patched jar (`OsmAnd-java/libs/gnu-trove-osmand.jar`).
- Map data: OBF files are derived from OSM (ODbL). The `.obf` files on download.osmand.net are produced by OsmAnd's own pipeline. OsmAnd-tools can regenerate them from any PBF.

**Maintainers / activity.** The router is essentially **Victor Shcherb's** work (OsmAnd founder/CEO): 515 + 99 (`vshcherb`) of all commits to `router/` since 2018 (the history starts at a 2018 repo restructure). Other notable contributors are RZR-UA (147), Dima-1 (98), ivanPyrohivskyi (80), MadWasp79, max-klaus, cepprice, Chumva, alex-dev.

Commits per year to `OsmAnd:…/net/osmand/router/`:

```
2018 36 | 2019 133 | 2020 155 | 2021 183 | 2022 196 | 2023 210 | 2024 300 | 2025 66 | 2026 64
```

In the last 12 months the authors were RZR-UA 39, vshcherb 28, Ivan 6, Vitaliy 3, and others. October 2026 alone saw boat/open-water routing, HH memory-budget work, turn-lane fixes and a native memory guard. The C++ side (`legacy:native/src`) is equally alive (2024: 224 commits, 2026: 118 so far). The 2024 peak corresponds to the HH ("fast routing") rollout. Activity is high and production-driven, but development is concentrated in a few people at one company.

### 2. Offline data model: OBF files

**Container.** `.obf` (OsmAnd Binary Format) is a hand-parsed protobuf-like format. The schema is in `resources:protos/OBF.proto`. Messages are not parsed by the protobuf runtime. Top-level sections are *"encoded as fixed32 length delimited"*, so a reader can seek and skip by byte offsets (random access). `OsmAndStructure` holds repeated sections:

```
addressIndex = 7, transportIndex = 4, poiIndex = 8, mapIndex = 6 (rendering),
routingIndex = 9 (OsmAndRoutingIndex), hhRoutingIndex = 10 (OsmAndHHRoutingIndex), versionConfirm = 32
```

One file per downloadable region (e.g. `Germany_saarland_europe_2.obf`), plus hourly/daily "OsmAnd Live" diff files. The router opens many files at once (`BinaryMapIndexReader[]`) and merges them. Live-update deletions are expressed as `osmand_change=delete` and handled by `access=-1` in routing.xml.

**Routing section (`OsmAndRoutingIndex`)** (`OBF.proto` lines ~737–812):
- `rules`: a per-file dictionary of `(tag, value)` encoding rules. Roads store their tags as varint arrays of rule ids (`RouteData.types`, `pointTypes`). OsmAnd keeps a curated set of OSM tags, chosen by `resources:obf_creation/rendering_types.xml`, so the routing profile can evaluate raw OSM-ish tags at runtime.
- `rootBoxes`: a quadtree of `RouteDataBox` (delta-encoded bboxes, `shiftToData` → `RouteDataBlock`).
- `basemapBoxes`: a second, generalized tree containing only major roads. It is used for the "BASE" phase of 2-phase long-distance routing.
- `RouteDataBlock { idTable, dataObjects (RouteData[]), restrictions (RestrictionData[]), stringTable }`.
- `RouteData { points (delta-coded x/y), pointTypes, pointNames, types, routeId, stringNames }`.
- `RestrictionData { type, from, to, via? }`: turn restrictions, including via-way restrictions.
- Coordinates are 31-bit tile coordinates (`MapUtils.get31TileNumberX/Y`) with `SHIFT_COORDINATES = 4` in `binary/BinaryMapRouteReaderAdapter.java`.
- Elevation is **baked in at generation time** from SRTM/ASTER/EU-DEM as synthetic tags `osmand_ele_start`, `osmand_ele_end`, and per-point `osmand_ele_asc`/`osmand_ele_desc` (`tools:…/obf/preparation/IndexHeightData.java`). These are decoded lazily in `binary/RouteDataObject.calculateHeightArray()`.

**Generation** (`tools:java-tools/OsmAndMapCreatorUtilities/src/main/java/net/osmand/obf/preparation/IndexRouteCreator.java`, 1751 LOC). It reads OSM (via an intermediate SQLite DB), keeps routable ways with their encoding rules and point tags, and processes restriction relations. This includes `no_entry`/`no_exit` mapped onto `no_straight_on`, and via-ways (`restrictionVia`). It writes `route_objects` and `baseroute_objects` (generalized: roundabout simplification, way combination, Douglas-Peucker). Long ways are kept whole; the comment says *"NEVER remove this simplify route due to memory limits in routing"*. Ids are `osmId << 6` plus split bits (`binary/ObfConstants.java`, `SHIFT_ID = 6`). The pipeline is packaged as OsmAndMapCreator (`generate-obf` in `tools:docker/Readme.md`).

**HH section (`OsmAndHHRoutingIndex`)**: see §4.2. The proto comment reads *"HH Routing index need to be consistent with global file and include fully referenced clusters"*.

```
OsmAndHHRoutingIndex { edition (timestamp), profile ("car"/"bicycle"/…), profileParams[] (e.g. "", "height_obstacles"),
  tagValuesTable, pointBoxes (quadtree of HHRouteNetworkPoint), pointSegments (HHRouteBlockSegments by id range & profileId) }
HHRouteNetworkPoint { id, dx, dy, globalId, roadId, roadStartEndIndex, clusterId, dualPointId, dualClusterId,
  profileIds[], partialInd, tagValueIds[] }
HHRoutePointSegments { segmentsIn: bytes, segmentsOut: bytes }   // shortcut costs, encoded per cluster in/out
```

**Measured section sizes.** I downloaded current files and parsed the top-level sections with a small script (`/tmp/claude-1002/-home-dev-p-routet/9a562bfa-8561-4774-9b54-e3d4f26bca5b/scripts/obfsec.py`):

| file (Oct 2026) | total | map (render) | **routing** | address | poi | transport | **HH car** | **HH bicycle** |
|---|---|---|---|---|---|---|---|---|
| Montenegro_europe_2.obf | 84.9 MB | 49.5 | **16.1** | 2.2 | 15.7 | 0.16 | **0.49** | **0.77** |
| Germany_saarland_europe_2.obf | 131.3 MB | 66.1 | **15.3** | 20.9 | 23.0 | 2.1 | **1.23** | **2.58** |
| Germany_saarland_europe_2.road.obf ("roads only" download) | 69.6 MB | 4.5 | 15.3 | 20.9 | 23.0 | 2.1 | 1.23 | 2.58 |

- The routing graph takes about 12–19% of a full map file, roughly 6 MB per 1000 km² in dense Germany (Saarland is 2,570 km²).
- HH adds about 8% (car) and 17% (bicycle) on top of the routing section, or about 1–2% of the full file. OsmAnd's blog says "~0.5% per profile" and about 800 MB for the whole-planet car routing data.
- Both files carry the **same HH edition timestamp** (`1790987038662` = 2026-10-03). HH is computed globally and then split per region, so maps of different editions do not combine (see §4.2).
- For scale, the download index lists Germany_berlin at 267 MB (full) and 161 MB (road), and Île-de-France at 678 MB and 361 MB.

### 3. Graph model and lazy loading

**No explicit node table.** A road is a `RouteDataObject` (`binary/RouteDataObject.java`):

```java
public final RouteRegion region;     // owns the tag dictionary
public int[] types;                  // encoding-rule ids (tags)
public int[] pointsX, pointsY;       // 31-bit coords
public int[][] pointTypes;           // per-node tags (traffic_signals, barrier, ele …)
public long[] restrictions, restrictionsVia;
public long id;                      // osmId<<6 | split bits
```

A graph vertex is implicit: any two roads sharing an identical `(x31, y31)` point are connected. When a tile is loaded, `RoutingContext.RoutingSubregionTile` indexes all road points into a `TLongObjectMap<RouteSegment> routes` keyed by `(x31 << 31) + y31`. Each entry starts a linked list (`RouteSegment.nextLoaded`) of every road segment touching that coordinate (`router/RoutingContext.java` ~752–830).

**The search state is edge-based.** `BinaryRoutePlanner.RouteSegment` (`router/BinaryRoutePlanner.java` ~1068) is a *directed sub-segment* `(road, segStart, segEnd)` between two consecutive points of a way. It carries `parentRoute`, `distanceFromStart` (seconds) and `distanceToEnd` (heuristic). The visited-set key is `(road.id << 11) + (pntIdx << 1) + dir` (`calculateRoutePointInternalId`). Because the search runs over directed edges, turn restrictions and turn penalties fall out naturally, and U-turn handling is explicit (`penaltyForReverseDirection`; `-1` = forbidden).

**Lazy tile loading** (`router/RoutingContext.java`):
- `ZOOM_TO_LOAD_TILES = 16` (configurable in routing.xml via `zoomToLoadTiles`). `getRoutingTile(x31, y31)` computes a z16 tile id. On first touch, `loadTileHeaders()` walks every open file's `RouteSubregion` quadtree (`BinaryMapRouteReaderAdapter.RouteSubregion`: bbox + `filePointer` + `shiftToData`) to find the intersecting data blocks. `loadSubregionTile()` then decodes the `RouteDataObject`s and indexes them.
- Memory is managed explicitly. When the estimated size exceeds 90% of `memoryLimitation`, `unloadUnusedTiles()` evicts least-used tiles down to 70% (the `access` counter on each tile). `DEFAULT_MEMORY_LIMIT = 30` MB (Java) and `DEFAULT_NATIVE_MEMORY_LIMIT = 256` MB in `router/RoutingConfiguration.java`. BRP also estimates its own queue/visited overhead per segment (`STANDARD_ROAD_IN_QUEUE_OVERHEAD = 220`, `STANDARD_ROAD_VISITED_OVERHEAD = 150` bytes) and throws "not enough memory" at 90% of the limit.
- Since 2026-10, the Android app also runs a watchdog, `OsmAnd:OsmAnd/src/net/osmand/plus/routing/NativeRoutingMemoryGuard.java`, with a budget of 512 MB–3 GB and 1 GB reserved RAM. It cancels native routing when the process grows past that budget.
- With multiple overlapping files (e.g. a region file plus OsmAnd Live diffs), duplicates are resolved per road id (`excludedIds`, `isExcluded`). `mapIndexReaderFilter` can restrict the search to the file group chosen by HH.

**Turn restrictions** (`BinaryRoutePlanner.proccessRestrictions` / `processRestriction`, ~727–836). At each intersection the planner collects candidate next segments into `segmentsToVisitPrescripted` and `segmentsToVisitNotForbidden`:
- `no_*` restrictions remove a candidate.
- `only_*` restrictions are exclusive in the forward direction and handled specially in reverse search (`REVERSE_WAY_RESTRICTION_ONLY`).
- Via-way restrictions are checked against the parent road (`viaId`).
- When the two searches meet, `checkViaRestrictions()` re-validates the restriction across the meeting point.
- Conditional restrictions and conditional access (`…:conditional`) are evaluated at a configurable time (`routeCalculationTime`, `RouteConditionalHelper.java`). For HH shortcut creation, `ambiguousConditionalTags` treats conditionals as "unlimited".

**Additional graph features:**
- "Direction points" (`RoutingConfiguration.DirectionPoint`) are synthetic points (`osmand_dp`) injected into roads at load time. They add or delete point tags, e.g. to apply user-placed avoid/penalty points.
- "Impassable roads" is a per-request `TLongHashSet` of road ids checked in `GeneralRouter.acceptLine`.
- Area routing for pedestrian areas (`RouteResultPreparation.combineWayPointsForAreaRouting`).

### 4. Algorithms

#### 4.1 BinaryRoutePlanner (BRP): bidirectional A* on the detailed graph

`router/BinaryRoutePlanner.searchRouteInternal()`:
- Two `PriorityQueue<RouteSegmentCost>`s (forward and reverse). The direction to expand is chosen by comparing the queue heads' `f = g + heuristicCoefficient·h`. `planRoadDirection` = 0 (both), 1 (forward only) or −1 (reverse only).
- Heuristic `h()`: straight-line `squareRootDist31 / router.getMaxSpeed()` (admissible if speeds are capped at maxSpeed, and `calcRoutingSegmentTimeOnlyDist` does cap them). If a `PrecalculatedRouteDirection` exists, it is used as a much tighter estimate (see 2-phase below). `heuristicCoefficient` defaults to 1. The bicycle profile uses 1.4 and pedestrian 1.2, which makes those searches deliberately inadmissible and faster. Dijkstra is `heuristicCoefficient = 0`.
- Segment cost = `distance / min(maxSpeed, speed·priority)` + `defineRoutingObstacle` (point obstacles such as traffic signals, barriers and tolls, in seconds) + `defineHeightObstacle` (elevation penalty) + `calculateTurnTime` (transition penalty between road classes, roundabout enter/exit, sharp/slight turn penalties by angle buckets 30°/60°/120°).
- A meeting is detected in `checkIfOppositeSegmentWasVisited`. It creates a `FinalRouteSegment` and puts it back into the queue, so termination is "final segment popped as queue minimum".
- A `boundaries` map makes the search stop at given segments. HH preprocessing and HH last-mile both use it.
- **Multi-target Dijkstra mode** (`dijkstraMode = ±1` when start or end is null) collects all reachable boundary segments into a `MultiFinalRouteSegment`. This is the primitive behind HH shortcut creation and HH start/end connection.
- `altHorizon` keeps both trees growing past the first meeting (alternatives).
- Long ferries: `LONG_DISTANCE_SEGMENTS_SPLIT`.

**2-phase ("COMPLEX") mode for long trips without HH** (`RoutePlannerFrontEnd.searchRoute`, ~460–475). For car profiles, when the distance exceeds `DEVIATION_RADIUS·6` (18 km):
1. Route first on the **basemap** routing tree (`RouteCalculationMode.BASE`, major roads only).
2. Build a `PrecalculatedRouteDirection` (`router/PrecalculatedRouteDirection.java`, a quadtree of route points with cumulative times).
3. Run the detailed A* with that route as the heuristic, giving a corridor search.

This was OsmAnd's long-distance solution before HH. It is fast but not exact, and is now the fallback when HH data is missing or rejected.

#### 4.2 HH routing: what "HH" means in OsmAnd

OsmAnd calls it **"Highway Hierarchy" routing** (`tools:obf-generation/hh-routing/generate-hh-routing.sh`: *"Since 2024, OsmAnd Team has deployed the Highway-Hierarchy routing algorithm"*; reader part name "Highway routing"). Despite the name, **it is not Sanders–Schultes Highway Hierarchies**, and it is not Contraction Hierarchies either. Reading the code, it is a **two-level, partition-based overlay graph**, closest to a single-level CRP / multi-level Dijkstra (MLD) without a separate customization metric layer. On top of that it adds an "optimistic shortcut + verify-and-repair" query that tolerates stale data and runtime parameters.

The design was introduced in 2023 (tools commit "Routing hh base (#677)", 2023-11-30; planner moved into OsmAnd-java 2023-10-07) and shipped as "fast offline routing" in OsmAnd Android 4.7 (2024). The blog post ["OsmAnd's Faster Offline Navigation"](https://docs.osmand.net/blog/fast-routing) (June 2025) describes it the same way: area clusters, a small number of border points per cluster, and shortcuts between them.

**Vocabulary (code ↔ concept):**
- **Network point / border point** (`HHRouteDataStructure.NetworkDBPoint`): a *directed road segment* `(roadId, start, end)` that is a cut edge between two clusters. `clusterId` is the cluster it exits; `dualPoint` is the same segment in the opposite direction, with `dualClusterId`. Using edges rather than nodes as border points keeps turn restrictions inside the detailed graph.
- **Cluster**: a connected region of roughly 1,000–50,000 directed segments (`TOTAL_MIN_POINTS = 1000`, `TOTAL_MAX_POINTS = 50000` in `tools:…/router/HHRoutingSubGraphCreator.java`). Per the blog, the planet car graph (2.07 B points / 2.42 B edges) becomes about 3 M border points in about 541 k clusters, roughly 5–6 border points per cluster, with about 91 M shortcuts.
- **Shortcut** (`NetworkDBSegment`): the exact detailed-graph shortest-path cost between two border points of the same cluster, computed for one (profile, params) combination. Stored as `segmentsIn` / `segmentsOut` byte arrays per point.

**Preprocessing pipeline** (`tools:…/router/`, driven by `generate-hh-routing.sh` and `MainUtilities` commands):

1. **`hh-routing-prepare` → `HHRoutingSubGraphCreator`** (border points, per profile) writes the network-points DB `*.hhdb` (SQLite, `HHRoutingPreparationDB`).
   - `buildRoadNetworkIsland()` grows a region from an unassigned seed segment, BFS by intersection depth, until about `TOTAL_MAX_POINTS` segments are reached. It records the depth distribution.
   - `findMincutUsingMaxFlow()` then solves a **unit-capacity max-flow / min-cut** problem (Ford–Fulkerson with BFS augmenting paths). The super-source is attached to the frontier (`toVisitVertices`); sinks are the core vertices with depth < `minDepth`, where the core holds at least `TOTAL_MIN_POINTS`. The min cut is a minimal set of road segments separating the core from the outside world, and those become the cluster's **border points**.
   - The blog explains why: naive geometric partitions gave 50–80 border points per cluster and a "shortcut explosion". Min-cut gives about 5.
   - Regions are processed across overlapping OBFs (`OVERLAP_FOR_ROUTING = 2°`) so clusters straddle file borders. Long ferries are split by `LONG_DISTANCE_SEGMENTS_SPLIT` (100 km). Adjacent points are merged for continuity.
   - Border points depend only on passability, not speeds. They are computed once per base profile (car/bicycle/pedestrian) and shared by all parameter variants.
2. **`hh-routing-shortcuts` → `HHRoutingShortcutCreator`** (shortcuts, per profile × param set) writes `*.chdb`.
   - Multithreaded (`THREADS`). For every border point it runs `BinaryRoutePlanner.searchRouteInternal(ctx, s, null, segments)`, a one-to-many Dijkstra with `boundaries` = all border points, with U-turns forbidden (`penaltyForReverseDirection = -1`).
   - Every border point reached without crossing another one becomes a shortcut, so each cluster becomes a near-clique over its border points.
   - Parameter variants come from `--routing_params`. Defaults: car `""`; bicycle and pedestrian `""` and `height_obstacles`.
3. **`hh-routing-obf-write` → `HHRoutingOBFWriter`** splits the global network into each region's OBF (`--update-existing-files`). Points whose cluster is only partially inside the region are written with `partialInd` / `incomplete`, and `globalId` stitches them across files.
4. **Verify**: `inspector.sh -vhhrouting`.
5. **Cost**: the blog says 2–3 days for all profiles over the planet. The script suggests `-Xmx32g` and 16 threads. Monthly releases moved from the 2nd to the 5th of the month because of it.

**Query** (`router/HHRoutePlanner.runRouting()`, mirrored in `legacy:native/src/hhRoutePlanner.cpp` and `OsmAnd-shared/…/HHRoutePlanner.kt`):

1. **Select files and profile variant** (`selectBestRoutingFiles`). Among all `HHRouteRegion`s of the base profile intersecting the start/end bbox, it groups by `(edition, profileParams)` and ranks:
   - groups containing both start and end (polygon or bbox) first;
   - then newer edition;
   - then fewer extra params and more matched params;
   - "high-cost" params (`prefer_unpaved`, `avoid_motorway`, `driving_style_prefer_unpaved`) weigh more;
   - unsupported non-default params set `hhHasUnsupportedParameters`.

   Mixing editions is impossible because cluster ids are global per generation.
2. **Load points** (`initHCtx` → `loadNetworkPoints`). All border points of the chosen files are loaded into memory: a few hundred thousand per country, millions for a continent. Shortcut edges are **not** preloaded. They are read lazily per point (`loadNetworkSegmentPoint`), with an LRU-ish budget `MAX_LOADED_EDGES = 250_000` (2026-09, "an edge budget instead of keeping the whole hub graph").
3. **Filter by parameters** (`filterPointsBasedOnConfiguration`). Each point stores the tag-values of its road (`tagValueIds`), so `router.acceptLine()` can mark points `rtExclude` when the current request's avoid parameters forbid that road (e.g. avoid tolls or motorways). This is how avoid options work without regenerating shortcuts.
4. **Last mile** (`findFirstLastSegments` → `initStart`). From the start segment, a detailed **Dijkstra** (BRP `dijkstraMode`, `heuristicCoefficient = 0`, `MAX_VISITED = 150000`) runs with `boundaries = all border points` and records the exact cost to every border point of the start cluster. The same happens in reverse from the end. If start and end are in the same cluster, a direct `PNT_SHORT_ROUTE_START_END` entry is added. If nothing is reached, the next candidate snap points (`RouteSegmentPoint.others`) are tried, up to `MAX_START_END_REITERATIONS = 50`.
5. **Overlay search** (`runRoutingWithInitQueue`). Bidirectional A* (`HHRoutingConfig.astar(0)`, the default from `RoutePlannerFrontEnd.defaultHHConfig()`) runs over border points and shortcuts. The heuristic is straight-line / maxSpeed, and it stops when the meeting cost ≤ the popped key. If a shortcut is shorter than the straight-line lower bound, it is corrected on the fly (`ASSERT_AND_CORRECT_DIST_SMALLER`). Dijkstra, CH (`USE_CH`) and midpoint variants also exist in the config. They are leftovers from experiments: the TODO list in `HHRoutingSubGraphCreator` says *"4.2 Implement CH algorithm – HARD to wait to finish CH"*.
6. **Unpack and verify** (`retrieveSegmentsGeometry` → `runDetailedRouting`). Each shortcut on the found path is recomputed with **bounded bidirectional A* on the detailed graph**, with all other border points as `boundaries`, with the initial/target direction fixed, and with a 4× reverse penalty. Then:
   - If the detailed cost exceeds the shortcut cost × `MAX_INC_COST_CF = 1.25` (1.15 "vigilant"), that shortcut's cost is updated in memory (`editDist`), all underestimated shortcuts on the route are corrected, and the overlay search is rerun (`MAX_COUNT_REITERATION = 30`; *"3 is enough for 90%, 30 is for 10% (100-750km with 1.5m months live updates)"*).
   - If a shortcut is impassable, it is disabled (`dist = -1`) and the cluster may be recomputed (`recalculateNetworkCluster`, `FULL_DIJKSTRA_NETWORK_RECALC = 10`).

   This step makes HH robust to OsmAnd Live hourly updates, runtime avoids, impassable-road lists and parameter mismatches, all without a customization phase.
7. **Alternatives** (`HHAlternativeRoutes.java`, 1048 LOC, 2026). A plateau / via-node method on the two overlay trees, with explicit stretch (`ALT_STRETCH = 0.4`, `+180 s`), plateau (≥10%), distinctness (≥20% / 300 m) and no-loop (500 m) rules, and detailed-geometry rejection. This is explicitly not ported to the Kotlin copy ("Drop the HH alternative routes from OsmAnd-shared").
8. **Prepare results**: `RouteResultPreparation.prepareResult()` produces turns, lanes and timing.

**Integration with BRP** (`router/RoutePlannerFrontEnd.searchRoute`, lines ~407–515):

```
if hhRoutingConfig != null && mode != BASE:
    if nativeLib == null or hhRoutingType == JAVA:  r = runHHRoute(...)   // Java HH
    else:                                           r = nativeLib.runNativeRouting(ctx, hhConfig)  // C++ HH
    if r correct or missing maps or useOnlyHHRouting: return r
# fallback:
if COMPLEX and dist > 18km: BASE route → PrecalculatedRouteDirection
if no intermediates and native: C++ BRP   else: Java BRP per leg (searchRouteImpl)
```

- In the app (`OsmAnd:OsmAnd/src/net/osmand/plus/routing/RouteProvider.java` ~285–301), the user setting `RouteCalculationMethod` = `AUTO | STANDARD_ONLY | FAST_ONLY` controls this. HH is only possible for profiles derived from **car or bicycle** (`RouteCalculationMethod.canProfileUseFastRouting`). `setHHRouteCpp(!SAFE_MODE)` means C++ HH is the default on Android and Java HH runs in safe mode. Pedestrian HH data can be generated with the script, but the app does not use it for HH today.
- The server (`tools:java-tools/OsmAndServer/.../RoutingController.java` `/routing/route`) forces HH beyond `hh-only-limit = 100 km` (`application.yml`). Boats always use BRP because they have no HH data.
- **Intermediate points**: HH routes each leg separately and halves the reverse penalty between legs. BRP routes per leg too, in Java; the native BRP path is only used without intermediates.

#### 4.3 Native (C++) vs Java: who runs what

| situation | engine |
|---|---|
| Android, default | C++ (`legacy:`) via JNI, for both HH and BRP. The memory limit is `nativeMemoryLimitation` / the guard. `NativeOsmandLibrary.getLoadedLibrary()` is used in `RouteProvider`. |
| Android "safe mode" | pure Java (`router/`) |
| Android, routes with intermediates via BRP | Java front-end; each leg goes through `searchRouteInternalPrepare`, which calls native if present |
| iOS | historically C++, now migrating to the Kotlin copy in OsmAnd-shared (no native path, no missing-maps check) |
| Server, tools, tests | Java (optionally native in tests: `RouteTestingNativeTest`, `ApproximationNativeTest`) |

C++ exists for speed and to keep GC pressure off the Android heap (the 30 MB Java limit vs 256 MB+ native). The cost is triple maintenance, with drift caught only by compatibility tests, e.g. the 2026-10-21 commit "Align ARM64 BRP cost() with Kotlin/ARM/x86".

#### 4.4 Precise start/end and recalculation

- **Snapping**: `findRouteSegment()` finds the nearest road segments (with alternates in `others`) via `RoutingContext.loadTileData` around the point, filtering out roads the profile cannot access. `makeStartEndPointsPrecise()` / `makeSegmentPointPrecise()` then cut the first and last result segments at the exact projection, and `calculatePreciseStartTime()` adjusts costs.
- **Recalculation strategy** (`OsmAnd:OsmAnd/src/net/osmand/plus/routing/RouteRecalculationHelper.java`, `RoutingHelper.java`):
  - Off-route detection uses a profile-dependent allowed deviation (`RoutingHelperUtils.getDefaultAllowedDeviation`).
  - When only the start moved (`onlyStartPointChanged`), the previous route is passed as `ctx.previouslyCalculatedRoute`. `RoutePlannerFrontEnd.getRecalculationEnd()` takes the part of the old route beyond `recalculateDistance` (routing.xml `recalculateDistanceHelp = 10000` m), sets its first segment as the target, chained through `parentRoute`, and routes only from the current position to that rejoin point. The old tail is appended (`addPrecalculatedToResult`). The Java multi-leg path does the same through `useSmartRouteRecalculation` (first/rest split when the next via point is within 50 m of the old route).
  - After `RECALCULATE_THRESHOLD_COUNT_CAUSING_FULL_RECALCULATE = 3` recalcs within 2 minutes, it does a full recalculation.
  - `initialDirection` plus `penaltyForReverseDirection` (car 300 s, bike 60 s, foot 10 s, ski 900 s) discourage an immediate U-turn based on the current heading.

#### 4.5 GPX approximation / map matching

`router/GpxRouteApproximation.java` (649), `GpxMultiSegmentsApproximation.java` (402), `GpxPointsMatchApproximation.java` (186), with C++ equivalents (`legacy:native/src/gpx*Approximation.cpp`).

- **Routing-based** (default, `searchGpxRouteByRouting`). It walks the track and routes between successive GPX points spaced `minStepApproximation`…`maxStepApproximation` apart (100–3000 m; profile attributes in routing.xml). It accepts a route if it stays within `minPointApproximation` (50 m car) of the track, otherwise it steps back or shortens and retries. Unmatched parts become straight lines (`generateStraightLineSegment`). `PrecalculatedRouteDirection.build(points)` is used as the heuristic so the router follows the track.
- **Geometry-based** (`setUseGeometryBasedApproximation`, user setting `APPROXIMATION_TYPE`). `GpxMultiSegmentsApproximation` is a best-first search over chains of road segments, scored by distance to the GPX polyline, with branching and a 500 m rollback (`MAX_DEPTH_ROLLBACK`). `GpxPointsMatchApproximation` is a simpler point-to-segment matcher. The comments reference test.osmand.net map URLs for regression cases.
- Native approximation runs via `nativeLib.runNativeSearchGpxRoute` unless disabled.
- None of this is HMM-based map matching (as in Valhalla's Meili or OSRM's `match`). It is pragmatic and geared to "turn my drawn/recorded track into a navigable route with turns".

#### 4.6 Public transport (`TransportRoutePlanner`)

`router/TransportRoutePlanner.java` (857), `TransportRoutingContext`, `TransportStopsRouteReader`, `TransportRoutingConfiguration`. C++ port: `legacy:native/src/transportRoutePlanner.cpp`.

- Data comes from the OBF `transportIndex`, built from **OSM PT route relations** (stops, platforms, route geometry, and `TransportSchedule` when present). **No GTFS.**
- The algorithm is a Dijkstra over `TransportRouteSegment`s, i.e. (route, boarding stop index):
  - Start/end stops within `walkRadius = 3000` m; transfers within `walkChangeRadius = 300` m.
  - Travel time per stop = `stopTime + dist / speedByRouteType`. Boarding cost = half the interval if known (`calcIntervalInSeconds`), else a per-type boarding time. `useSchedule` uses average stop intervals.
  - Limits: `maxNumberOfChanges = 2`, `maxRouteTime = 5400` s (extended for long distance), plus alternatives (`increaseForAlternativesRoutes`, combine-similar-routes thresholds).
  - All of this is configured in the `public_transport` profile of routing.xml (lines ~2990–3015).
- It is frequency-based and timetable-agnostic. It answers "which lines and where to change", not "next departure". Walking legs are then computed with the pedestrian router.

#### 4.7 Multi-region and long distance; missing maps

- Many OBF files are opened at once. BRP does not care about file boundaries, because tiles query all files. HH requires a consistent edition across files.
- `MissingMapsCalculator` (`router/MissingMapsCalculator.java`) uses the world region polygons (`regions.ocbf`) to detect which maps along the straight line or the HH network are missing or outdated. It can continue (`CONTINUE_ON_MISSING_MAPS`) or return an error with the list of maps to download, and HH reduces its re-iteration counts when maps are mixed.
- Before HH, long distance meant the 2-phase basemap + corridor approach (§4.1).
- New (2026-10): `router/sea/` adds boat routing. `BoatRoutePlanner` mixes the water network (fairways, canals, locks via BRP) with open-water legs. `SeaRoutePlanner` computes a visibility graph on convex shore corners from basemap land polygons, planned coarse and verified fine.

### 5. Profiles: `routing.xml` rule engine

**File**: `resources:routing/routing.xml` (3409 lines). It is synced into `OsmAnd-java/src/main/resources/net/osmand/router/` by the Gradle task `collectRoutingResources`, and users can drop a custom `routing.xml` (or `*.xml` with extra profiles) into the app's `routing/` folder. The file header (lines 1–330) is a good tutorial.

**Profiles** (`<routingProfile name=… baseProfile=… minSpeed defaultSpeed maxSpeed sharpTurn slightTurn roundaboutTurn restrictionsAware onewayAware heuristicCoefficient>`): `car` (derived: default, truck, motorcycle), `bicycle`, `pedestrian`, `wheelchair`, `horsebackriding`, `boat`, `ski`, `geocoding`, `public_transport`. Profile `<attribute>`s set penalties and approximation parameters. Derived profiles are switched by `profile_truck`, `profile_motorcycle` params.

**Attribute blocks** (`GeneralRouter.RouteDataObjectAttribute`): `access`, `speed`, `priority`, `destination_priority`, `oneway`, `obstacle` (routing cost, s), `obstacle_time` (ETA, s), `penalty_transition`, `obstacle_srtm_alt_speed` (s per metre climbed by incline bucket), `area`. Each block is `<way attribute="…">` or `<point attribute="…">` containing an ordered list of rules where **the first match wins**:

```xml
<if param="avoid_unpaved">
    <select value="-1" t="surface" v="gravel"/>
</if>
<select value="-1" t="maxweight">
    <gt value1=":weight" value2="$maxweight" type="weight"/>
</select>
```

Expressions are `select`, `if`/`ifnot` (by `param` or by tag), `gt`/`ge`/`lt`/`le`/`eq`, `min`/`max`. `:param` refers to a request parameter value, `$tag` to a road tag value, with unit types such as weight, length and speed (`RoutingConfiguration` parser ~459–520; `GeneralRouter.RouteAttributeContext`, `RouteAttributeExpression`, `RouteAttributeEvalRule`).

**Evaluation is cached** per `(RouteRegion, sorted rule-id array)` in `GeneralRouter.evalCache`. Roads share tag combinations heavily, so each unique tag set is evaluated once per region per request. That makes interpreting XML rules cheap enough for mobile.

**Parameters** (`<parameter id type=boolean|numeric values valueDescriptions default group profiles>`). Car examples: `short_way`, `avoid_toll`, `avoid_unpaved`, `prefer_unpaved`, `avoid_ferries`, `avoid_motorway`, `avoid_borders`, `avoid_low_emission_zone`, `allow_private`, `goods_restrictions`, `hazmat_*`, and numeric `weight`, `height`, `length`, `width`, `maxaxleload`, `weightrating`, `motor_type`. Bicycle adds `driving_style_*` groups, `relief_smoothness_factor_*`, `height_obstacles`, `avoid_sett`/`avoid_cobblestone`. Ski adds piste difficulty and off-piste policy.

The app passes all of these per request (`RouteProvider.initOsmAndRoutingConfig`), and `RoutingConfiguration.Builder.build(profile, memoryLimits, paramMap)` instantiates a `GeneralRouter` with them. Also per request: `impassableRoads`, direction/avoid points, `initialDirection`, `minSpeed`/`defaultSpeed` overrides, and conditional-time evaluation.

**Runtime flexibility.** This is very high for BRP: any parameter combination works, because there is no preprocessing on the detailed graph. HH is the constraint. Shortcut costs exist only for the generated `(profile, params)` sets (car: default only; bicycle: default and `height_obstacles`). Other parameters work in two ways. Avoids act as point filters (`rtExclude` via tag-values on border points). Cost-changing params rely on verify-and-repair, which can degrade into many re-iterations and then a BRP fallback (the blog cites about 50 heavily recalculated shortcuts as the trigger). `getFilteredTags` ignores `short_way`, `height_obstacles` and relief params when matching.

**Elevation-aware routing** (`GeneralRouter.defineHeightObstacle`). If `height_obstacles` is on, per-point heights (from baked `osmand_ele_*` tags) give the incline per sub-segment. The incline is bucketed to 1, 4, 7, 10 …%, and `obstacle_srtm_alt_speed` returns seconds of penalty per metre of climb or descent for that bucket, so uphill and downhill are asymmetric. A sign bug in this was fixed on 2026-09-25 (#26288). Elevation also drives route statistics and slope colouring (`RouteStatisticsHelper`, `RouteColorize`).

### 6. Guidance

**`router/RouteResultPreparation.java`** (2861 LOC, the largest file) turns raw `RouteSegmentResult`s into navigable output:
- `splitRoadsAndAttachRoadSegments` / `attachRoadSegments` record, at every junction, the other roads present (`getAttachedRoutes`), which are needed for turn decisions and "keep left/right".
- `calculateTimeSpeed` computes ETAs with `obstacle_time` (the human-time variant), slow-down at sharp turns and traffic-signal intersection clustering (`TRAFFIC_SIGNALS_INTERSECTION_SIZE = 60` m).
- `prepareTurnResults` / `getTurnInfo` handle turn types (`router/TurnType.java`: C, TL, TSLL, TSHL, TR, TSLR, TSHR, KL, KR, TU, TRU, OFFR, RNDB/RNLB with exit numbers) by angle and road topology, with roundabouts in `RoundaboutTurn.java`.
- Lanes: `turn:lanes` parsing (`parseTurnLanes`, `calculateRawTurnLanes`), active-lane inference (`inferActiveTurnLanesFromTurn`, `setActiveLanesRange`, `findActiveIndex*`), merging of consecutive close turns (`determineTurnsToMerge`, `mergeTurnLanes`), `none` lane handling (`convertNoneLanes`, 2026), and "attach turn:lanes to the junction they belong to" (#26394, 2026-10-12).
- Also: U-turn justification, muting minor turns, `highwaySpeakPriority`, exit refs and names (`ExitInfo`), destination signs (`destination`, `destination:ref`).

The Kotlin copy splits this into `TurnPreparation.kt` and `TurnLanes.kt`.

**Voice**: `OsmAnd:OsmAnd/src/net/osmand/plus/routing/VoiceRouter.java` (~1000 LOC) is a state machine per upcoming turn (`STATUS_LONG_PREPARE → PREPARE → TURN_IN → TURN → TOLD`). Distances and times are speed-dependent (`routing/data/AnnounceTimeDistances`). It also handles speeding, alarms (speed cameras, railway crossings, etc. via `AlarmInfo`) and street-name/ref/destination announcements. Phrases are generated by per-language **JavaScript** TTS scripts (`resources:voice/<lang>/en_tts.js` style) run in the app, or by recorded voice packs. The voice router is app code, not part of OsmAnd-java.

### 7. Can it be used standalone?

Yes, with caveats. There is no packaged "OsmAnd routing server", but every piece needed exists:

- **Library**: `OsmAnd-java` is a plain Java 17 Gradle module (`apply plugin: 'java'`, publishes `osmand-base` via ivy/maven to `OSMAND_BINARIES_IVY_ROOT`).
  - Dependencies: `OsmAnd-shared` (KMP, now required), commons-logging, gson, org.json, commons-compress, junidecode, jts-core, openlocationcode, scribejava, kxml2, a patched Trove jar and a patched protobuf runtime (in-tree).
  - Build quirks: Gradle tasks copy resources from a sibling checkout `../../resources` (OsmAnd-resources) and download `regions.ocbf` from builder.osmand.net.
  - A minimal embed is: `RoutingConfiguration.getDefault().build("car", memLimits, params)`, then `new RoutePlannerFrontEnd().buildRoutingContext(config, null /*no native*/, readers, NORMAL)` (with `setDefaultHHRoutingConfig()` for HH), then `searchRoute(ctx, start, end, intermediates)` → `RouteCalcResult.detailed : List<RouteSegmentResult>`. See `HHRoutePlanner.prepareContext()` and `router/TestRouting.java` for minimal examples.
- **CLI harnesses**:
  - `router/TestRouting.java` `main` runs a route or a JSON test suite against a directory of OBFs.
  - In OsmAndMapCreatorUtilities (`tools:…/MainUtilities.java`): `test-routing`, `hh-routing-run` (`TestHHRouting`), `random-route-tester` (`router/tester/RandomRouteTester`), and the HH generation commands.
  - The OsmAndMapCreator Swing GUI has a `MapRouterLayer` for interactive route debugging.
  - Unit tests: `OsmAnd-java/src/test/java/net/osmand/router/*` (RouteTestingTest uses `src/test/resources/routing/*.obf` plus `test_routing.json`).
- **Server**: `tools:java-tools/OsmAndServer` is a Spring Boot app. `RoutingController` exposes `/routing/route?points=lat,lon,…&routeMode=car,param1,…&avoidRoads=&alternatives=`, `/routing/gpx-approximate`, `/routing/approximate`, `/routing/routing-modes`, `/routing/geocoding`, `/routing/get-route`. It is configured by `application.yml` (`tile-server.routeObf.location`, `osmand.routing.hh-only-limit`). It is the backend of osmand.net/map, so it is real and maintained, but it is part of a monolith (OAuth, purchases, tiles, wiki, MCP…). Running only the routing part means trimming it or writing a small wrapper around OsmAnd-java.
- **Data**: you can use OsmAnd's prebuilt OBFs (monthly, with HH for car and bicycle), or generate your own: `OsmAndMapCreator generate-obf file.pbf`, then `generate-hh-routing.sh` (needs about 32 GB heap for big regions).
- **Native core**: `legacy:` builds with CMake (`android-configure.sh`, `targets/`). It is JNI-oriented (Java object marshalling in `java_wrap.cpp`), so using the C++ router without Java means writing your own front end around `routePlannerFrontEnd.cpp`. That is possible but undocumented.

### 8. Assessment

**Strengths**
- **Truly offline, mobile-first, memory-bounded.** The graph is never fully in RAM. Z16 tiles are loaded lazily from mmap-friendly, seekable files and evicted under a memory budget, and HH edges are loaded lazily within a 250k-edge budget. A whole country routes on a phone with tens of MB of Java heap.
- **Fully runtime-configurable costs.** Any routing.xml rule or parameter applies per request, with no rebuild, and evaluation is cached by tag set. There are rich vehicle constraints (weight, height, hazmat, axle load), conditional tags, elevation penalties and point obstacles.
- **The HH design itself.** The overlay is small (about 1–2% of map size), one partition is shared by all parameter variants, and there is no global preprocessing per user metric. It is regional (it ships inside per-country files), tolerates incremental (hourly) data updates via verify-and-repair, and supports avoids via point filtering. It is an unusually pragmatic answer to "CH-class speed without CH's rigidity and storage" (the blog cites OSRM planet car at about 200 GB of RAM/disk vs about 800 MB for OsmAnd HH car).
- **Edge-based search** that handles via-way restrictions and turn costs.
- **A complete navigation stack**: turn and lane guidance, voice, recalculation with route reuse, GPX approximation, alternatives, PT, boat routing.
- **Battle-tested** on millions of devices, with active development in 2026.

**Weaknesses**
- **GPLv3 plus a company-controlled codebase**: unusable as a dependency for non-GPL work, with a bus factor of about 2 (vshcherb, RZR-UA).
- **Three hand-synced implementations** (Java, C++, Kotlin). Behaviour drift is a constant risk, and API stability is not a goal.
- **Code quality is research-grade in places**: static mutable globals (`RoutePlannerFrontEnd.CALCULATE_MISSING_MAPS`, `HHRoutePlanner.DEBUG_VERBOSE_LEVEL`), `System.out.printf` logging, config objects mutated mid-search and restored (`ctx.config.heuristicCoefficient = 0; … restore`), and commented-out experiments. Documentation is the code, comments, routing.xml's header and a few blog posts.
- **HH only for car and bicycle** in shipped data. Non-standard cost parameters reduce HH to repeated recalculation or BRP fallback. Mixed map editions disable HH.
- **No live traffic** in routing. Speeds come from OSM `maxspeed` and profile defaults. The default heuristic weights for bike and foot (1.4 / 1.2) trade optimality for speed.
- **PT routing is frequency-agnostic** and OSM-only (no GTFS or RAPTOR-style timetables).
- **Standalone use needs glue.** Server mode is buried in a large Spring monolith, and the build requires sibling repos plus a downloaded region file.

**Suitability**

(a) **As a dependency.** Only if the new project is GPLv3 and on the JVM (or Kotlin/KMP). Even then you would be coupling to an internal, unversioned API (`osmand-base` isn't on Maven Central) and to the OBF data pipeline. For a Rust project, which the name "routet" hints at, it is not realistic: you would be embedding a JVM or porting.

(b) **As a reference to learn from or port ideas.** This is where it shines. The design ideas, which are not copyrightable as such, are well worth studying and re-implementing clean-room:
1. **Seekable, region-split, tag-dictionary graph files** with a quadtree of route blocks, lazy tile decoding and eviction. A good template for a mobile-first or mmap-based Rust graph store.
2. **Implicit nodes, explicit directed sub-segments** as search state, with id packing `(roadId << 11) | (pointIdx << 1) | dir`.
3. **Rule-based profiles evaluated on raw tags at query time, cached per unique tag-set.** This combination works well: more flexible than OSRM's baked Lua weights and cheaper than evaluating per edge.
4. **HH = min-cut partition + border-point overlay + optimistic shortcuts with verify-and-repair.** Partitions are profile-agnostic (passability only). Shortcuts are per (profile, param-set), and parameters are matched at query time. Point tag-values support avoid filtering. Detailed bounded A* unpacking doubles as a correctness check, and shortcut costs are corrected in place and the search re-run. This gives near-CH query speed on phones while remaining correct under data drift. A Rust implementation could improve on it with a proper multi-level partition (e.g. Inertial Flow/PUNCH/KaHIP-style, as in RoutingKit/CRP) and real customization, while keeping OsmAnd's "verify against detailed graph" safety net and its per-region file split with global edition ids.
5. **2-phase corridor A*** (basemap route as heuristic) as a cheap fallback when no overlay exists.
6. **Recalculation by rejoining the old route** beyond N km, and edge-based HH border points so restrictions never need modelling in the overlay.

---

## Part II: Navit

### Overview
- **What**: an open-source, modular car/bike/pedestrian navigation system in C (glib-based), with plugins for GUIs (internal/GTK/Qt), graphics backends, map drivers and vehicle (GPS) sources. It runs on Linux, Android, Windows, macOS, Sailfish and embedded devices. The routing engine is part of the core (`navit:navit/route.c`), not a standalone library.
- **License**: GPLv2 (core) and LGPLv2 (parts; see `navit:COPYING`, `GPL-2`, `LGPL-2`).
- **Activity**: there was a long lull (11 commits in 2023), then a revival: 144 commits in 2025 and 394 in 2026. These are mostly "fix(…)" and "chore" commits by **Christof Schulze** (321 in the last year), plus jkoan and renovate. v0.5.7 was released 2026-01-30 (`navit:CHANGELOG.md`: "Routing: Consider one-way property of circular junctions", maptool fixes). `route.c` itself sees only maintenance fixes (overflow guards, leaks, cyclic-chain detection). No algorithmic development.

### Routing implementation (`navit:navit/route.c`, 4364 LOC)
The file's header comment (lines 20–81) is an excellent description:
- **Graph construction per request.** `route_graph_build()` pulls street items from all active maps inside a **map selection** made of rectangles at different map "orders" (zoom levels). The vehicle profile's `route_depth` defines them, default `"4:25%,8:40000,18:10000"` (`route_calc_selection`, ~1011): major roads (order 4) in the start/destination bbox +25%, order-8 roads within 40 km of each point, and all roads (order 18) within 10 km. This is Navit's **hierarchy, done by map-data selection** rather than by preprocessing, and it is a classic heuristic that can miss routes needing minor roads far from both endpoints. Points are road junctions (`route_graph_point`) and segments are items or parts of items (`route_graph_segment`, with `offset` for segmented items).
- **Algorithm: LPA\* without heuristic** (effectively Dijkstra with incremental repair), using a Fibonacci heap (`fib.h`, `fh_insertkey`).
  - It runs **backwards from the destination** and floods the *entire* selected graph (`route_graph_is_path_computed` only checks for an empty heap). Every point therefore knows its cost-to-destination and next segment. After a deviation, the new route is just a walk of `seg` pointers from the new position, with no search, as long as the position stays inside the graph.
  - Multiple destination candidates are seeded with non-zero costs, because the destination may be off-road.
  - LPA* was added with the **traffic module** (`navit:navit/traffic.c`, ~6000 LOC, TraFF-based traffic messages). `route_graph_point_update()` plus `route_graph_compute_shortest_path()` repair only the affected parts when "traffic distortions" (delays, speed limits, closures; also used internally for avoids) change segment costs.
  - The authors note that A* heuristics "cost more than saved" given the limited selection.
- **Costs** (`route_value_seg`, ~2044): access is checked by **bitmasks** (`profile->flags_forward_mask`/`flags_reverse_mask` vs segment `AF_*` flags such as `AF_CAR`, `AF_BIKE`, `AF_ONEWAY`, `AF_TOLL`, `AF_UNPAVED`, `AF_SIZE_OR_WEIGHT_LIMIT`, `AF_DANGEROUS_GOODS` in `navit:navit/attr.h`). Time is `route_time_seg` (length / speed from the roadprofile for the item type, respecting maxspeed per `maxspeed_handling`), plus `through_traffic_penalty`, turn-around penalties, and traffic distortions.
- **Turn restrictions** are separate items (`type_street_turn_restriction_no/only`) added to the graph (`route_graph_add_turn_restriction`, ~2453). Junction points get the `RP_TURN_RESTRICTION` flag and are handled by special-casing in path evaluation. This is node-based with patches, less clean than edge-based.
- **Guidance**: `navit:navit/navigation.c` (4412 LOC) generates maneuvers and announcements. `announcement.c` and speech plugins handle output.

### Map formats
- **binfile** (`navit:navit/map/binfile/binfile.c`): Navit's native format. A **ZIP archive in which each member is a tile** of a quadtree, holding items (type + coords + attrs), plus index files. It is built from OSM by **maptool** (`navit:navit/maptool/`, about 17k LOC of C). Access flags, item types (street_1_city, highway_land …) and order levels are **decided at conversion time**, so profiles can only choose among baked flags and types.
- Other map drivers include `textfile`, `csv`, `shapefile`, `garmin`/`garmin_img`, `mg` (Map&Guide), and `filter`. Routing works on any driver that provides street items with the right attributes.

### Profile configuration
`navit.xml` (`navit:navit/navit_shipped.xml` ~238+). Each `<vehicleprofile name="car" route_depth="…" flags="0x4000000" flags_forward_mask="…" flags_reverse_mask="…" route_mode="0" static_speed static_distance>` contains `<roadprofile item_types="street_2_city,track_paved" speed="30"/>` entries, plus attributes such as `vehicle_width/height/weight/axle_weight/dangerous_goods`, `through_traffic_penalty`, `turn_around_penalty`, `maxspeed_handling`. The parser is `navit:navit/vehicleprofile.c`. This is far less expressive than OsmAnd's routing.xml: there are no arbitrary tag rules, because the tags are gone after maptool.

### Notable design points / assessment
- **Backward full flood + cost-to-go field**: rerouting is nearly free and pairs naturally with LPA* for traffic changes. This idea is worth borrowing for on-device navigation even with a faster primary algorithm (e.g. a bounded backward search around the corridor).
- **Hierarchy by selection** (zoom-level map selection) is simple and memory-cheap but neither exact nor scalable to continental routes. Every route rebuilds the graph from map items.
- **Flag-based access** is fast but rigid, since everything is decided in maptool.
- **Suitability**: as a dependency it is not attractive (GPLv2, C/glib, tied to Navit's object and attribute system). As a learning reference it offers the clear LPA*-in-navigation write-up and the traffic-distortion model. For everything else (data model, profiles, speedups), OsmAnd is the far richer reference.

---

## Quick comparison

| | OsmAnd | Navit |
|---|---|---|
| License | GPLv3 (PRs MIT), UI CC-BY-NC-ND | GPLv2 / LGPLv2 |
| Languages | Java (ref), C++ (Android), Kotlin MP (iOS) | C |
| Data | OBF (protobuf-like, seekable, per-region, tag dictionaries, elevation baked) | binfile (ZIP of quadtree tiles, flags baked) |
| Search state | directed road sub-segments (edge-based) | junction points (node-based) + restriction items |
| Core algorithm | bidirectional A* (BRP); HH overlay (min-cut clusters + shortcuts) + detailed verify; 2-phase basemap corridor fallback | LPA* (h = 0), backward from destination, full flood, Fibonacci heap |
| Long distance | HH (car, bike), else basemap 2-phase | zoom-level map selection (`route_depth`) |
| Profiles | routing.xml rule engine, runtime params, conditionals, elevation | vehicleprofile/roadprofile speeds + bitmasks |
| Live data | OsmAnd Live hourly map diffs; no traffic | TraFF traffic distortions (incremental LPA*) |
| Extras | lanes, voice, GPX approximation, PT (OSM-only), alternatives, boat | maneuvers/speech, traffic |
| Standalone | Java lib + TestRouting CLI + OsmAndServer (Spring) | no |
| Activity 2026 | high (company-driven) | maintenance revival |

Sources (web): [OsmAnd's Faster Offline Navigation (blog, 2025-06-11)](https://docs.osmand.net/blog/fast-routing), [OsmAnd 4.7 Android release notes](https://docs.osmand.net/blog/osmand-android-4-7-released), [SotM US 2025: Multimodal HH routing in OsmAnd](https://openstreetmap.us/events/state-of-the-map-us/2025/multimodal-hh-routing-highway-hierarchy-in-osmand/). Everything else is from source code.
