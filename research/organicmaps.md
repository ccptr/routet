# Organic Maps / CoMaps routing engine (MAPS.ME lineage)

*Research snapshot: 2026-10-23. Organic Maps at `~/g/organicmaps` (HEAD `408e7945`, 2026-10-23). CoMaps at `~/g/comaps` (HEAD `5bd62c92`, 2026-10-23, cloned from codeberg.org/comaps/comaps). Paths below are relative to the Organic Maps root unless they start with `comaps:`.*

---

## TL;DR

- **What it is:** an offline, mobile-first, multi-profile router (car, bicycle, pedestrian, frequency-based public transit). It runs on the phone over per-region `.mwm` files. It is **not a standalone product**: it lives in `libs/routing` (~31k LOC C++23, not counting tests and tools) plus `libs/routing_common` (~2.4k LOC of vehicle models). It sits on OM's own `indexer`/`coding`/`geometry`/`platform`/`base` stack. License: **Apache-2.0** (code). Map data is ODbL.
- **Core idea:** the graph is **not** a separate road network. The `routing` section of each mwm stores only **topology**: "joints", meaning which (feature, point-index) pairs are the same intersection. That is compressed with Elias-gamma/delta coding. Geometry, road class, oneway, surface and so on are read lazily at query time from the **same feature records used for rendering**, through a FIFO cache. Speeds come from a compiled-in `VehicleModel`, so per-edge weights are computed at query time. There is **no contraction hierarchy**. Search is **bidirectional A\*** with a plain straight-line/max-speed heuristic, run on segments or on joint-contracted "JointSegments".
- **Cross-region routing:** each mwm has a `cross_mwm` section. It lists the border "transition" segments (border-crossing ways are duplicated as "twins" in both mwms and matched by OSM way id) and a precomputed **enter×exit weight matrix**, for car only. Long car routes first run A\* on this tiny "leaps" overlay (`LeapsOnly` mode). That yields up to about 15 candidate corridors. Each corridor is refined by real A\* inside each mwm (`JointSingleMwm`). This is a two-level, partition-based scheme: think "CRP-lite with a precomputed clique per cell and no customization".
- **Mobile-first details worth stealing:** routing data is about 4–5% of an mwm on top of geometry it already has. Graphs are loaded on demand per mwm. A sparse world-level border graph (`routing_world` in World.mwm, 270 KB) works out which **undownloaded** regions a route would need. Off-route "adjust" rebuilds splice the old route back in within 5 minutes of travel time. Dual "weight vs ETA" speeds keep preference separate from time estimates. Time-dependent `access:conditional` and `maxspeed:conditional` are evaluated at the predicted arrival time.
- **2026 news (OM):** distance-biased **alternative route** (one, from a second strategy), multi-candidate snapping, alternatives for transit, warnings for steps/gates, and transit extended to bus/tram/light rail. The experimental GTFS-in-mwm code was removed in Sep 2026.
- **CoMaps** forked from OM on 2025-04-02 (squashed import, no shared git history). Its routing changes include a tag-based `roadpenalty` section (traffic calming, signals, gates), a turn-angle penalty matrix with left-hand-traffic awareness, avoid-paved and avoid-steps options, routable `highway=construction`, CarPlay guidance and roundabout/TTS improvements. It has **not** adopted OM's 2026 alternative routes. Its mwm format now differs from OM's (an extra `roadpenalty` section, among others).
- **As a dependency:** poor fit. It is welded to the mwm container, the classificator types and the C++ support libraries, and it needs OM's generator to produce data. **As a reference:** excellent, especially for offline packaging, cross-region decomposition, lazy loading, snapping, rerouting, turn generation and lanes.

---

## 1. Overview

### Language, license, layout

| Item | Value |
|---|---|
| Language | C++23 (`CLAUDE.md`, CMake). Python 3 for the generation pipeline (`tools/python/maps_generator`) |
| License | Apache-2.0 (`LICENSE`). Data is ODbL (`DATA_LICENSE.txt`) |
| Runtime router | `libs/routing/` (≈180 files, 29.2k lines in top-level `.cpp/.hpp`, ≈31.4k including `base/` and `lanes/`) |
| Vehicle models | `libs/routing_common/` (car/bicycle/pedestrian models, maxspeed conversion, `NumMwmId`) |
| Transit data types | `libs/transit/` (stops, gates, edges, lines, transfers; JSON→binary serdes) |
| Traffic plumbing | `libs/traffic/` (speed groups; the server URL is empty in `private.h`, so there is **no live traffic** in OM) |
| Opening hours | `libs/opening_hours/`, a **C++23 port of the Rust crate `opening-hours-rs`** (2026-08), used for `*:conditional` |
| Data generation | `generator/` (C++ `generator_tool`) plus `tools/python/maps_generator` (stage orchestration) |
| Tools | `libs/routing/routes_builder/`, `libs/routing/routing_quality/`, `routing_benchmarks/`, `routing_integration_tests/`, `routing_consistency_tests/`, `routing_tests/` (47 files of unit tests) |

Library layering, per `CLAUDE.md`, is strict: `base` → `coding`/`geometry` → `platform` → `indexer`/`transit`/`routing_common` → `routing`/`traffic`/`storage`/`search` → `map` (Framework) → UI (`android/`, `iphone/`, `qt/`).

### How routing is integrated into the app

- `libs/map/routing_manager.cpp` (1.9k lines) owns a `routing::RoutingSession` and builds an `IndexRouter` for the selected `VehicleType` (Pedestrian=0, Bicycle=1, Car=2, Transit=3; see `libs/routing/vehicle_mask.hpp`). It also builds a `RulerRouter` (straight-line "ruler" measuring mode) and an `AbsentRegionsFinder`.
- `libs/routing/async_router.{hpp,cpp}` runs the `IRouter` on a worker thread. It handles cancellation, progress callbacks and timeouts. In parallel, it runs the `AbsentRegionsFinder` thread, which can return `RouterResultCode::NeedMoreMaps` with a list of countries to download.
- `libs/routing/routing_session.cpp` covers following, off-route detection, rerouting, turn/lane/speed-camera notifications and TTS.
- The platforms call the core through JNI (`android/sdk/src/main/cpp/`) and ObjC++ (`iphone/CoreApi/`). The Qt desktop app (`qt/`) can also route.

### Activity in the routing code

Commits touching `routing/`, `routing_common/` (and `libs/…` after the July 2025 move to `libs/`):

| Year | Commits |
|---|---|
| 2016 | 294 |
| 2018 | 311 |
| 2020 | 207 |
| 2021 | 58 |
| 2022 | 156 |
| 2023 | 69 |
| 2024 | 43 |
| 2025 | 50 |
| 2026 (to Oct) | 95 |

- **Maintainers:** since 2021, 298 commits are by **Viktor Govako** (`vng`, a former MAPS.ME routing dev and the de-facto routing maintainer), 76 by **Alexander Borsuk** (OM founder), then Anton Makouski, Konstantin Pastbin, Andrei Shkrob and others. The original authors from the MAPS.ME era include Vladimir Byko-Ianko (bykoianko), Mikhail Gorbushin (gmoryes), Lev Dragunov, Dobriy Eeh, Yuri Gorshenin and Alex Zolotarev.
- **2026 highlights**, from `git log -- libs/routing`:
  - alternative routes, with `RoutesResult` refactoring, a route "balloon" at the max-difference midpoint, keeping the chosen variant across rebuilds, and a distance-biased strategy
  - transit alternatives (walk factor 3, boarding penalty)
  - experimental multi-candidate start/end snapping
  - steps/gate/lift_gate warnings
  - `maxspeed:conditional`
  - lane info merge within 30 m
  - turn classification fixes and filtering of cross-mwm twin duplicates from turn candidates
  - the `GeneratedMapTest` framework (build an mwm from OSM XML inside a test)
  - removal of the experimental GTFS code
  - a C++ port of opening-hours-rs

### Lineage: MAPS.ME → Organic Maps → CoMaps

- **2014:** MAPS.ME (Minsk; later bought by Mail.ru) shipped offline car routing on **OSRM**. Per-country `.routing` files held contraction hierarchies (see `git log --grep=osrm`: "Added OSRM online engine" 2014-07, "[routing] Fixed OSRM parser").
- **2015:** a pedestrian A\* router was added ("A\* algorithm is moved out", 2015-04).
- **2016-11:** the "indexed astar router" (`index_graph.hpp`) arrived, called "dynamic indexed routing" in `generator_tool.cpp` flags. It replaced OSRM so that car/bike/foot share one graph, restrictions, access and avoid-options can be applied at runtime, and file sizes shrink.
- **2017-12:** OSRM was removed ("Removing osrm"). The last OSRM leftovers were deleted on 2026-02-22.
- **2019-11:** `LeapsOnly` mode was refactored. **2020-11:** `RegionsRouter`/`RegionsSparseGraph` were added (absent-region detection on World.mwm).
- **Dec 2020:** **Organic Maps** forked from the open-sourced MAPS.ME code. Part of the MAPS.ME team, including Viktor Govako and Alexander Borsuk, continued the work there. MAPS.ME itself went in an ad-heavy direction and later faded. Many comments still reference MAPS.ME Jira (`https://jira.mail.ru/browse/MAPSME-10694` in `index_router.cpp`).
- **2025-04/05:** **CoMaps** forked over governance concerns. Organic Maps is run by the for-profit Organic Maps OÜ, with disputes over donations, the Kayak affiliate and proprietary components ([It's FOSS](https://itsfoss.com/news/organic-maps-fork-comaps/), [AlternativeTo](https://alternativeto.net/news/2025/5/organic-maps-contributors-fork-project-as-concerns-over-governance-and-transparency-grow/)). The first CoMaps commit is "Organic Maps sources as of 02.04.2025 (fad26bbf…)". It is a squashed import, so `git merge-base` with OM does not exist. See §11.

---

## 2. Offline data model: routing inside `.mwm`

### Container

An `.mwm` is a simple tagged-section container (`libs/coding/files_container.{hpp,cpp}`): a uint64 offset to a TOC at the start, then the TOC as a vector of `{tag string, varuint offset, varuint size}`. Sections are memory-mapped or read on demand. Section tags are defined in `defines.hpp`.

**Routing-relevant sections:**

| Tag | Built by | Content |
|---|---|---|
| `routing` | `generator/routing_index_generator.cpp::BuildRoutingIndex` → `IndexGraphSerializer` (`libs/routing/index_graph_serialization.hpp`) | Road **joints** only. Sections split by `VehicleMask` (e.g. car+bike+foot roads, foot-only roads), so a car load can skip pedestrian-only sections. Each road lists `(pointId delta (gamma-coded), jointId)` with a "new vs repeat joint id" bit. No geometry and no weights |
| `cross_mwm` | `BuildRoutingCrossMwmSection` (same file) → `CrossMwmConnectorBuilderEx` (`libs/routing/cross_mwm_connector_serialization.hpp`) | Transitions (border-crossing segments: feature id, segment idx, OSM way id, oneway, forward-is-enter, road mask). For **car only**, an enter×exit weight matrix (seconds, rounded **up** to 4 s granularity so it stays ≥ the A\* heuristic). Stored as `MapUint32ToValue` (format v2). Weights load lazily |
| `transit_cross_mwm` | `BuildTransitCrossMwmSection` | Same idea for transit edges crossing borders |
| `restrictions` | `generator/restriction_generator.cpp` | `No`/`Only`/`NoUTurn`/`OnlyUTurn` restrictions as feature-id sequences (multi-way via supported), delta-coded |
| `roadaccess` | `generator/road_access_generator.cpp` | Per-vehicle way and point (barrier) access: `No`/`Private`/`Destination`/`Yes`, plus `*:conditional` with opening-hours expressions |
| `maxspeeds` | `generator/maxspeeds_builder.cpp` | Per-feature forward/backward/conditional maxspeeds (compressed rank bit-vector of feature ids plus a uint8 vector), plus **per-mwm default speeds per highway type, in-city vs out-of-city**, computed as averages of tagged maxspeeds |
| `city_roads` | `generator/city_roads_generator.cpp` | Bit-vector "feature is inside a city boundary", used for in-city/out-of-city speeds |
| `altitudes` | `generator/altitude_generator.cpp` from SRTM (`generator/srtm_parser.cpp`) | Per-point altitudes of road features. Only loaded for non-car profiles (`loadAltitudes = type != Car` in `routes_builder.cpp`) |
| `speedcams` | `generator/camera_info_collector.cpp` | Speed cameras bound to (feature, segment, coef) with max speed |
| `transit` | `generator/transit_generator.cpp` | Subway/light rail/tram/bus graph: stops, gates, edges, transfers, lines with **intervals** (headway), shapes |
| `routing_world` (World.mwm only) | `generator/routing_world_roads_generator.cpp` + `generator/world_roads_builder/` | Sparse **cross-border road graph** for the whole planet (`CrossBorderGraph`, `libs/routing/cross_border_graph.hpp`), used to find which regions a route crosses |

Route **geometry is not duplicated**. `libs/routing/geometry.cpp` loads `FeatureType` from the regular `features`/`geom*` sections with `ParseGeometry(FeatureType::BEST_GEOMETRY)` (coordinates are 30-bit quantized mercator, `libs/coding/point_coding.hpp`). It derives oneway, highway type, routing options (toll, motorway, ferry, dirty, steps), speeds and pass-through flags from the feature's classificator types.

### Measured section sizes (2026-07-14 data, version 260714)

I downloaded these from `cdn.organicmaps.app/maps/260714/` and parsed the TOC with a small script:

| Section | Luxembourg (45.0 MB total) | Germany_Berlin (101.5 MB total) | World.mwm (63.9 MB) |
|---|---|---|---|
| `features` (shared with rendering) | 14.7 MB | 31.5 MB | 18.6 MB |
| `routing` | 549 KB | 2.23 MB | — |
| `cross_mwm` | **696 KB** (many borders for a small country) | 154 KB | — |
| `altitudes` | 884 KB | 1.58 MB | — |
| `transit` | 408 KB | 1.14 MB | — |
| `roadaccess` | 77 KB | 348 KB | — |
| `city_roads` | 77 KB | 285 KB | — |
| `maxspeeds` | 40 KB | 139 KB | — |
| `restrictions` | 4 KB | 4 KB | — |
| `transit_cross_mwm` | 3.6 KB | 10 KB | — |
| `speedcams` | 0.4 KB | 0.6 KB | — |
| `routing_world` | — | — | 270 KB |
| **Routing-specific total** | ≈2.7 MB (6%) | ≈5.9 MB (5.8%) | 0.4% |

`data/countries.json` lists **1,166 mwms totalling about 90 GB**, with a median of about 66 MB. Regions are split so that each file stays phone-sized; the largest is `Canada_Nunavut_South` at 877 MB. For comparison, a CH or OSRM file per region would typically be several times the size of the `routing` section, because CH stores shortcuts and geometry.

### Generation pipeline

`tools/python/maps_generator/generator/stages_declaration.py`:

1. `StageDownloadAndConvertPlanet` / `StageUpdatePlanet` take a planet `.o5m` (osmctools). Then come `StageCoastline` and `StagePreprocess`.
2. `StageFeatures` maps OSM tags to **classificator types** (`data/mapcss-mapping.csv`, `generator/osm2type.cpp`). For example, `highway=primary` becomes `highway-primary` and `oneway=yes` becomes `hwtag-oneway`. Collectors write intermediate files alongside: restrictions (`restriction_collector.cpp`), road access (`road_access_generator.cpp`), maxspeeds (`maxspeeds_collector.cpp`), cameras (`collector_camera.cpp`), city boundaries (`collector_routing_city_boundaries.cpp`), and **cross-mwm OSM ways** (`cross_mwm_osm_ways_collector.cpp`, which records which way segments cross the region polygon).
3. `StageMwm` runs per country, in this order:
   - `StageIndex`
   - `StageSrtm`, which calls `generator_tool --srtm_path` → `BuildRoadAltitudes`
   - `StageIsolinesInfo`
   - `StageDescriptions`
   - `StagePopularity`
   - **`StageRouting`**: `generator_tool --make_city_roads --make_routing_index --generate_maxspeed --make_cross_mwm --generate_cameras`. Order matters: city roads, then the routing graph, then restrictions and road access, then maxspeeds (`generator_tool.cpp` lines ~429-492)
   - `StageRoutingTransit` (needs `SUBWAY_URL`, i.e. JSON from the [organicmaps/subways](https://github.com/organicmaps/subways) validator, processed by `tools/python/transit/transit_graph_generator.py`; see `docs/SUBWAY_GENERATION.md`) → `--transit_path --make_transit_cross_mwm`
   - `StageMwmDiffs`
4. For **World**: `StagePrepareRoutingWorld` filters planet roads (`filter_roads`) and runs `world_roads_builder_tool`. `StageRoutingWorld` writes `routing_world`.

**Inputs:**
- OSM planet
- `borders/*.poly` region polygons. These drive both the mwm split and the cross-mwm transitions
- SRTM tiles
- isolines (for display)
- subway/transit JSON
- cities boundaries
- `countries.json` hierarchy. The parent country selects the per-country `VehicleModel`, so "countries file is needed" is a hard error in `generator_tool.cpp`
- optionally Wikipedia data (unrelated to routing)

**Cross-mwm weights** (`FillWeights` in `generator/routing_index_generator.cpp`) are computed at generation time. For each enter segment, a Dijkstra wave (`AStarAlgorithm::PropagateWave` over `DijkstraWrapperJoints`) runs over the region's own `IndexGraph` with the car model and **a fixed "current time"** (`graph.SetCurrentTimeGetter([time = GetCurrentTimestamp()]…)`). Weights are recorded to every reachable exit. Consequences:
- Weights ignore traffic and avoid-options.
- `*:conditional` is frozen at build time.
- The comment "We use leaps for cars only" means bike and foot never use the leaps overlay.

---

## 3. Graph model

### Vertices: directed segments

- `Segment` (`libs/routing/segment.hpp`) is `{NumMwmId, featureId, segmentIdx, forward}`, a **directed piece between two consecutive points of a road feature**. Comment: "You can imagine the segment as a material arrow… Head of each arrow is connected to fletchings of next arrows". Edges are therefore *segment→segment* transitions. This is effectively an **edge-based graph**, which makes turn restrictions and U-turn penalties natural (the cost is attached to a pair of segments).
- `RoadPoint` = `(featureId, pointId)`. `Joint` (`joint.hpp`) is a set of `RoadPoint`s at the same location (an intersection). `RoadIndex` (`road_index.hpp`) maps featureId → `RoadJointIds` (pointId → jointId). `JointIndex` maps jointId → the list of RoadPoints.
- `IndexGraph` (`index_graph.{hpp,cpp}`) is the per-mwm graph. It holds geometry (`Geometry`, a lazy FIFO cache of `RoadGeometry` per feature, `kRoadsCacheSize = 10000`), the estimator, the road and joint indexes, restrictions (forward and backward maps), no-U-turn restrictions, `RoadAccess` and avoid-options.
- **Edge enumeration** (`IndexGraph::GetNeighboringEdges`): at the head point of a segment, look up the joint (if any) and enumerate the next/previous segments of every feature in that joint, honouring oneway, `SuitableForOptions(avoid)`, access "No for sure", restrictions and U-turn rules. The weight is `CalculateEdgeWeight = estimator segment weight (+ climb) + GetPenalties(u, v)`.

### Joints mode: contracted vertices

- `JointSegment` (`joint_segment.hpp`) represents a run of segments of **one feature** between two joints. `IndexGraphStarterJoints<Graph>` (`index_graph_starter_joints.hpp`) presents a graph where you "jump" between joints. Fake start/end joint segments are built by BFS from the start/finish segments to the nearest joints. This roughly halves to tenths the number of A\* vertices on long roads with many shape points. Paths are re-expanded with `ReconstructJoint`/`ProcessJoints` (`index_router.cpp`).
- Modes are defined in `WorldGraphMode` (`world_graph.hpp`): `LeapsOnly`, `NoLeaps`, `SingleMwm`, `Joints`, `JointSingleMwm`. Selection happens in `IndexRouter::SetupAlgorithmMode`:
  - pedestrian/bicycle use `Joints` (or `NoLeaps` near guide tracks)
  - transit uses `NoLeaps`
  - car uses `Joints` if the mwms are "near" (start mwm rect intersects a finish mwm, or the distance is < 300 km), else `LeapsOnly`

### Fake vertices for start and finish

- **Snapping** (`IndexRouter::PointsOnEdgesSnapping`, `index_router.cpp` ~1343-1615):
  - find candidate roads in widening radii {first, 500 m, 2000 m}
  - keep at least `kMinRoadCandidates = 2` and up to `kMaxRoadCandidates = 10`
  - drop dead-end-only components (`FillDeadEndsCache`/`EraseIfDeadEnd`)
  - drop candidates "fenced off" by another road crossing the straight connector (`IsFencedOff`)
  - prefer codirectional edges given the GPS heading
  - for car, the start segment is "strict forward" when codirectional, to avoid immediate U-turns

  Since 2026, several candidate projections are fed into A\*, which then chooses among them.
- `FakeEnding` (`fake_ending.hpp`) is an origin junction plus a list of `Projection{segment, oneway, front, back, projected junction}`. `IndexGraphStarter` (`index_graph_starter.{hpp,cpp}`) adds fake vertices and edges in a `FakeGraph` (`fake_graph.hpp`). They come in two types (`fake_vertex.hpp`):
  - `PureFake`: the straight jump from the point to the projection, priced at offroad speed. For car the weight speed is 0.01 km/h, so off-road jumps are very expensive in weight but 0 s in ETA (`car_model.cpp` comment)
  - `PartOfReal`: the partial segment from the projection to the segment's ends
- `FakeEdgesContainer` keeps fake edges between subroutes (via points) and for "adjust" rebuilds. Every subroute after the first starts on the last real segment of the previous one, "to prevent jumping from one road to another one using a via point".

### Turn restrictions and U-turns

- Stored as feature-id chains. `RestrictionLoader` (`restriction_loader.cpp`) converts **`Only` to `No`** at load time. For every M, it adds "f1…f(M−1), fK" for each other feature fK at the joint between f(M−1) and fM. `OnlyUTurn` is converted to `No` the same way.
- At search time, `IndexGraph::IsRestricted` checks restriction chains **against the A\* parent chain**, walking back through parents until the feature id changes. This is a label-setting search with parent-dependent edge filtering. It is not an exact state expansion, so with via-way restrictions it can in theory miss optimal paths. It is pragmatic and cheap. Bidirectional search needs `AreWavesConnectible` to check restrictions where the two waves meet.
- `NoUTurn` is a per-feature-end flag (`m_noUTurnRestrictions`). `IsUTurnAndRestricted` applies it. U-turn penalties are 2 min for car, 20 s for bicycle and 0 for foot (`edge_estimator.cpp`).

### Road access, including conditional

- `RoadAccess` (`road_access.hpp`) has types `No`, `Private`, `Destination` and `Yes`. `Permit` and `Locked` exist only in the generator (`Permit` becomes `Yes`; `locked=yes` becomes `Private`). There are maps for way access and point access (barriers on nodes), plus `Conditional` entries made of `{type, osmoh::OpeningHours}`.
- Tag tables live in `generator/road_access_generator.cpp`: `motorcar`/`motor_vehicle`/`vehicle`/`access`, `barrier=bollard→No`, `gate→Private`, `foot`, `bicycle`, `*:conditional`, and implicit conditionals such as `winter_road=yes` meaning "no @ (Feb–Dec)".
- **Semantics:** `No` with `Confidence::Sure` is a hard filter (`IsAccessNoForSure`). `Private` and `Destination` are **soft**: `RouteWeight` counts `m_numAccessChanges`, and each change costs **2 h** in `GetIntegratedWeight`. So you can start or end inside a private area, but cutting through one is heavily discouraged. Conditional access is evaluated at `start time + weight so far` (`GetAccess(featureId, weightToFeature)`). If the result is `Confidence::Maybe` (an unparseable or uncertain rule), it counts as `m_numAccessConditionalPenalties`.
- The **pass-through** concept works the same way. Per country, certain highway classes (`living_street`, `service`, `track`) have `isPassThroughAllowed=false` (`car_model.cpp`: Germany, Russia, Poland, Brazil…). Each change between a pass-through and a non-pass-through zone costs **30 min**. A route may not make more than 2 such changes (`IndexGraphStarter::CheckLength`). One "free" change is banked per ending located inside such a zone.
- `RouteWeight` (`route_weight.{hpp,cpp}`) is a small struct `{weight s, numPassThroughChanges, numAccessChanges, numAccessConditionalPenalties, transitTime}`. It compares lexicographically/integrated. This multi-criteria cost is a neat way to express "avoid unless necessary".

### Building the graph at runtime

`IndexGraphLoader` (`index_graph_loader.cpp`) works per `NumMwmId`, on first touch:
1. Open the mwm via `MwmDataSource`.
2. `DeserializeIndexGraph` (joints for the required vehicle mask).
3. Create `Geometry` with a `GeometryLoader` that reads features, altitudes (optional) and maxspeeds and city_roads.
4. Load restrictions.
5. Load road access for the vehicle.

Graphs are cached in the loader for the duration of the request (`ClearRouteCalculationState`).

---

## 4. Cross-region routing

### Twins and transitions

`CrossMwmGraph` (`cross_mwm_graph.hpp`) carries an ASCII diagram. Every **transition feature** (an OSM way crossing a region border) is present in both neighbouring mwms as a "twin". The **transition segment** is the one whose endpoints lie in different mwms. `CrossMwmConnector<CrossMwmId>` (`cross_mwm_connector.hpp`) indexes transitions by `(featureId, segmentIdx)` and maps `CrossMwmId` (OSM way id, or `TransitId` for transit) → featureId. That lets an exit in mwm A be matched to the enter twin in mwm B. There is a workaround for segment-index skew caused by imprecise packed borders (OM issue #1736).

In `Joints`/`NoLeaps` modes, `SingleVehicleWorldGraph::GetEdgeList` adds **twin edges** whenever the A\* reaches a transition segment. The search then simply continues inside the neighbouring mwm, whose graph loads on demand. Neighbours that are not downloaded produce no twins.

### LeapsOnly: the overlay for long car routes

`IndexRouter::CalculateSubrouteLeapsOnlyMode` (`index_router.cpp` ~1012-1215) and `LeapsGraph` (`leaps_graph.cpp`) work in stages.

1. **Overlay graph:**
   - start → **every exit** of each start mwm, priced with `CalcLeapWeight` (straight line at a fixed 16 m/s ≈ 57.5 km/h; `ComputeDefaultLeapWeightSpeed` has a comment explaining the tuning against specific test routes)
   - enter → exit inside an mwm uses the **precomputed matrix** (`CrossMwmGraph::GetOutgoingEdgeList`)
   - exit → twin enter in the next mwm, with `MwmHierarchyHandler::GetCrossBorderPenalty`. That is **2 h for crossing a country border with controls**, waived inside Schengen, the EAEU, Ireland/Northern Ireland and Israel/Palestine (`mwm_hierarchy_handler.cpp`)
   - every enter of each finish mwm → finish
2. **Candidate enumeration:** `FindPathBidirectionalEx` with an emitter that keeps returning `false`, which makes the bidirectional A\* continue after the first meeting and emit further paths (a new 2026 capability: "Continue multi-route search after queue exhaustion"). Paths are deduplicated by (first leap's end feature, last leap's begin feature). The search stops at 15 distinct keys at each end or after a **30 s timeout**. Candidates are then filtered to at most 2 per `HighwayCategory` at each end (issues #2998 and #3625).
3. **Refinement:** cross-mwm caches are purged ("CrossMwmConnector takes a lot of memory with its weights matrix"). For each candidate, real routes are computed for the start→first-exit and last-enter→finish legs (`RoutesCalculator::Calc2Times`), and the best total of `start leg + middle precomputed weight + finish leg` is picked.
4. **Full expansion:** `ProcessLeapsJoints` rebuilds every intermediate leap enter→exit as a real `JointSingleMwm` A\*. It tries the variants with and without loops back into the start or finish mwm (issues #821, #2085, #5069). If a leg fails, which happens when **avoid options** forbid the roads the precomputed weight assumed, it falls back to a multi-mwm `Joints` search over the neighbouring leaps. The code comments are candid: "'Cross MWMs' together with 'Avoid options' doesn't guarantee a best route, because cross-mwm graph keeps only _clean_ (no any avoids) weights."
5. `LeapsPostProcessor` exists but is disabled as buggy.

### Absent-region detection

`AbsentRegionsFinder` (`absent_regions_finder.cpp`) starts `RegionsRouter` (`regions_router.cpp`) on a separate thread. It runs bidirectional A\* over `RegionsSparseGraph`, which is deserialized from World.mwm's `routing_world` (a planet-wide sparse graph of cross-border road segments with weights stored as rounded meters). From the path it collects the mwm names. Anything not downloaded is reported back as `NeedMoreMaps`, so the UI can offer to download exactly the countries on the route. A relative-error tolerance on reduced weights accounts for rounding (issue #333).

### How it stays fast on a phone

- There is no CH or other heavy preprocessing. The per-mwm graph is just joints, about 2 MB for Berlin. Everything else is computed lazily from features already on disk.
- Graphs and geometry load lazily per mwm, with a 10k-road FIFO geometry cache. One cache is shared by both A\* directions ("faster… than two separate caches", `geometry.hpp`).
- Joints mode shrinks the vertex count.
- The LeapsOnly overlay turns a continental route into searches over a few thousand border vertices plus local searches near the endpoints and per-mwm "connect the dots" searches.
- Cross-mwm weight matrices load only when needed and are purged before the refinement stage.
- `ska::bytell_hash_map` holds distances and parents. `SmallList`/`buffer_vector` provide small-buffer edge lists. Cancellation is polled periodically (`PeriodicPollCancellable`).
- The geometry cache is FIFO, not LRU. There is a `@todo Use LRU cache?` in `geometry.hpp`.

---

## 5. Algorithms

### `AStarAlgorithm<Vertex, Edge, Weight>` (`libs/routing/base/astar_algorithm.hpp`, 819 lines)

- It is generic over a graph concept (`base/astar_graph.hpp`): `GetOutgoingEdgesList`, `GetIngoingEdgesList`, `HeuristicCostEstimate`, `SetAStarParents`/`DropAStarParents`, `AreWavesConnectible`, `GetAStarWeightEpsilon`.
- **APIs:**
  - `PropagateWave` (Dijkstra / one-way A\* with visitor, edge-weight adjuster and state filter)
  - `FindPath` (unidirectional)
  - `FindPathBidirectional` / `FindPathBidirectionalEx(params, emitter)`
  - `AdjustRoute(params, prevRoute)`
- **Bidirectional A\*** uses the **average consistent potentials** `p_f = ½(π_f − π_r)` and `p_r = ½(π_r − π_f)` (Ikeda/Goldberg-style), with a well-commented derivation in `BidirectionalStepContext::ConsistentHeuristic`. The queues alternate every `kQueueSwitchPeriod = 128` steps, not by queue size. Stopping criterion: `top_f + top_r ≥ best − ε`. Reduced weights that come out negative are logged (`m_badReducedWeight`) and clamped to 0.
- **Heuristic:** straight-line distance / max possible weight speed across all loaded country models (`CalcMaxSpeed` in `index_router.cpp`, `EdgeEstimator::CalcHeuristic`). Transit uses 60 km/h, deliberately ignoring high-speed rail. Under the DistanceBiased strategy a "tight" heuristic (distance / cap speed) is used when no leaps or guides are present. There are **no landmarks/ALT, no arc flags and no CH**. Regional pruning comes only from the LeapsOnly hierarchy and the length checkers: `AStarLengthChecker` → `WorldGraph::CheckLength(weight, startToFinishDistance)` limits detours, and pass-through counting applies.
- Generic visitors: `JunctionVisitor` drives progress reporting and debug visualization (`junction_visitor.hpp`, `astar_progress.cpp`).

### Rebuild / "adjust"

`IndexRouter::CalculateRoute(..., adjustToPrevRoute, ...)`. When the user leaves the route, is within 5 km of it (`kAdjustRangeM`) and is at least 10 km from the finish, `AdjustRoute` runs a **one-directional Dijkstra from the new position**, limited to **5 minutes** of weight (`AdjustLengthChecker`). It stops at the best `(dist to v + remaining weight of old route from v)` and splices the rest of the old route. This is very cheap compared with a full reroute. Otherwise a full rebuild runs. Rebuilds are triggered by `RoutingSession` after `kOnRouteMissedCount = 10` "move-away" ticks; slow movement counts 1 and fast movement counts 2.

### Avoid options

`RoutingOptions` (`routing_options.hpp`) has the bits `Toll`, `Motorway`, `Ferry`, `Dirty` (unpaved) and `Steps`. They are saved per vehicle mode (2026-10: "Apply routing options per mode"). They act as **hard filters** in `IndexGraph::GetNeighboringEdges`. The precomputed cross-mwm weights ignore them (see §4).

### Alternative routes (new in 2026)

`IndexRouter::CalculateRoute` (lines ~446-590) works like this:
- The active route uses the user's current strategy (`Normal`).
- The alternative is computed with the **other `EdgeEstimator::Strategy`**. `DistanceBiased` sets weight = `max(normal weight, distance / cap)` with caps of car 40 km/h, bike 12 km/h and foot 4 km/h. In effect this is a "reasonable shortest" route that still avoids terrible roads (issue #12953).
- It is only computed for full builds within 300 km (car) or 100 km (other modes), and it is dropped if its ETA is > 1.5× the active route's.
- For transit, the alternative scales walking ×3 and adds a 5 min boarding penalty, which favours direct buses over subway+walk.
- The UI shows a balloon at the midpoint of the longest differing part (`FindMaxDiffMidpoint`). `SwapAltRouteToActive` makes the alternative the followed route and keeps a separate adjust-cache per route.

So these are not "via-node"/plateau/penalty alternatives in the OSRM/GraphHopper sense: there are **exactly two routes from two cost functions**. LeapsOnly's multi-candidate emitter could support more, but it is not used for that.

### Guidance and turns

After A\*, `RedressRoute` converts segments to junctions, computes per-segment ETAs (`CalculateETA`, time-dependent), builds an `IndexRoadGraph`, and calls `DirectionsEngine::Generate` (`directions_engine.cpp`).
- `FillPathSegmentsAndAdjacentEdgesMap` reloads features around each route junction to collect **turn candidates**: all outgoing edges with angles and highway classes, including twins across mwm borders.
- `CarDirectionsEngine` (`car_directions.cpp`) or `PedestrianDirectionsEngine` (`pedestrian_directions.cpp`) classifies turns into `CarDirection` (`turns.hpp`: straight, slight/normal/sharp left and right, U-turns, enter/leave/stay on roundabout, exit highway left/right, reached destination). The logic includes:
  - discarding turns by highway class when the alternatives are minor roads
  - "go straight" correction
  - rightmost/leftmost correction
  - roundabout exit counting
  - U-turn detection (`CheckUTurnOnRoute`)
  - merging segregated left/right turns (2026)
  - filtering cross-mwm twin duplicates (2026)
  - turn-angle smoothing limits on sparse geometry

  Helpers live in `turns_generator.cpp` and `turns_generator_utils.cpp`.
- **Street names** are read from features (with "ref", destination and junction refs). There is i18n handling for street-name splicing (Armenian full stop, CJK spacing).
- **Lanes** (`lanes/`): `turn:lanes[:forward|:backward]` metadata is parsed by `lanes_parser.cpp` into `LaneWays` bitsets (ReverseLeft…ReverseRight). `lanes_recommendation.cpp` selects the recommended lanes for the maneuver. Lane info within 30 m is merged (2026-08).
- **Speed cameras** (`speed_camera.cpp`, `speed_camera_manager.cpp`) have Auto/Always/Never modes. The country prohibition list is in `speed_camera_prohibition.cpp`: no camera data for Macedonia, Switzerland and Turkey, and "partly prohibited" (no warnings by default) for France and Germany.
- **Warnings** for steps, gates and lift gates on foot/bike routes are produced by `CollectRouteWarnings` in `RedressRoute` (2026).

---

## 6. Profiles (vehicle models)

`libs/routing_common/vehicle_model.{hpp,cpp}`, `car_model.cpp`, `car_model_coefs.hpp`, `bicycle_model.cpp`, `pedestrian_model.cpp`.

- **Compiled-in**, not data-driven. Each model has:
  - `HighwayBasedSpeeds`: per `HighwayType`, `InOutCitySpeedKMpH`
  - `HighwayBasedFactors`
  - a list of allowed road types with `isPassThroughAllowed`
  - surface factors (`psurface-paved_good` 1.0 / `paved_bad` 0.6 / `unpaved_good` 0.4 / `unpaved_bad` 0.2 for car weight)
  - additional road tags (e.g. `hwtag-yescar`, `hwtag-nocar`, `hwtag-yesbicycle`, `hwtag-nofoot`)
  - an offroad speed
- **Two speeds per road:** `SpeedKMpH{m_weight, m_eta}`. The *weight* speed expresses preference and the *eta* speed is the time estimate. Examples:
  - pedestrian trunk is weight 1 km/h but ETA 5 km/h ("possible but disliked")
  - car ferry ETA vs weight
  - car motorway factors 0.95 weight / 0.93 ETA
  - car offroad is weight 0.01 km/h but ETA "not used" (0 s)

  The estimator uses `Purpose::Weight` for A\* and `Purpose::ETA` for timings. This is a clean, cheap way to tune routes without wrecking ETAs.
- **Car speeds** (`car_model_coefs.hpp`, "default car model coefficients for open source developers") run from motorway 118 (in city) / 124 (out of city) km/h down through tertiary 50/50, residential 20–26 and living street 10, multiplied by per-class factors. In `VehicleModel::GetTypeSpeedImpl`:
  - the OSM **maxspeed** tag, when present, replaces the class speed for car, and the surface factor is clamped
  - otherwise the **per-mwm default speed per highway class** (average of tagged maxspeeds, from the `maxspeeds` section) overrides the global default if they differ by less than 2×
  - the `city_roads` bit chooses in-city or out-of-city speeds
  - conditional maxspeeds are applied by time

  The comment on the coefficients suggests MAPS.ME had internal, possibly traffic-trained coefficients that were never published.
- **Per-country variants** come from the `countries.json` parent name:
  - car: `CarModelFactory` (no pass-through living_street in Austria/Belarus/Hungary/Romania/Slovakia, no pass-through service in Poland, no tracks in Denmark, …)
  - pedestrian: about 30 countries (`NoTrunk`, `YesCycleway`, `YesBridleway`, `AllAllowed`, `IncreasePrimary`)
  - bicycle: similar, plus `DismountPathSpeed`, `PreferFootwaysToRoads` and bidirectional living streets

  These encode the [OSM access-restrictions wiki table](https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing/Access-Restrictions).
- **The models key off classificator types** (`indexer/classificator`), not raw OSM tags. Tag→type mapping happens at generation time, so changing what a tag means requires regenerating maps.
- **Elevation-aware bike and foot** (`edge_estimator.cpp`): `GetPedestrianClimbPenalty` (with extra penalty above the 2,500 m "mountain sickness" altitude) and `GetBicycleClimbPenalty` (`1 + 50·|tan|·impact` uphill, different downhill) divide the speed per segment using SRTM altitudes. The bicycle estimator also lowers speeds on bad roads. Car climb penalty is 1. Route ascent and descent are shown in the UI (`map` `CalculateAscentDescent`).
- **Penalties** (in `EdgeEstimator` subclasses): ferry landing 10 min weight / 8 min ETA for foot and bike. Car has U-turn 2 min and its own ferry landing penalty. **OM has no turn-angle or traffic-signal costs**; CoMaps added them (§11).

---

## 7. Public transit

- **Data:** OSM route relations validated by the `organicmaps/subways` project (subway, light rail, tram, bus as of the 2026-06 release; see the [OM news](https://organicmaps.app/news/2026-06-29/satellite-imagery-public-transport-routes-alternate-routes-new-search-and-route-planning-interface-for-android-large-accessibility-fonts/)). The JSON is converted by `tools/python/transit/transit_graph_generator.py` and serialized into the `transit` section by `generator/transit_generator.cpp`. The format is in `libs/transit/transit_types.hpp`: `Stop`, `Gate`, `Edge`, `Transfer`, `Line{interval, type, color, title}`, `Shape`, `Network`.
- **Routing model** (`transit_graph.cpp`, `transit_world_graph.cpp`, `transit_graph_loader.cpp`):
  - Transit edges become fake features (`FakeFeatureIds::kTransitGraphFeaturesStart`).
  - Gates connect the transit graph to the **pedestrian** road graph; transit mode loads the pedestrian `IndexGraph`.
  - **Wait time = line interval / 2**, applied as the transfer/boarding penalty ("We assume arrival time has uniform distribution…").
  - This is **frequency-based, with no timetables**. The experimental GTFS-in-mwm path (`transit/experimental/*`, `transit_schedule.*`) was removed in Sep 2026, "superseded by the schedule service and CDN architecture tracked in #5331", i.e. planned online schedules.
- **Search:** plain bidirectional A\* in `NoLeaps` mode over road plus transit segments. The heuristic is capped at 60 km/h. The start and finish ending candidates include nearby gates within 150 m (2026). Cross-region transit uses `transit_cross_mwm`.
- The subway/transit *display* layer is in `libs/map/transit/` and `libs/drape_frontend/transit_scheme_builder.cpp`.
- **Guides:** `guides_graph.cpp`/`guides_connections.cpp` let you route *along a user-selected GPX/KML track* (once MAPS.ME "Guides", now "select a guide track for routing", 2026-03). The track becomes a fake graph connected to OSM roads near the checkpoints, and `NoLeaps` mode is forced. It is unrelated to transit.

---

## 8. Turn-by-turn, voice, following and rerouting

- **`RoutingSession`** (`routing_session.cpp`) has states `RouteNotStarted`, `OnRoute`, `RouteNeedRebuild`, `RouteRebuilding`, `RouteFinished`, `RouteBuilding`, ….
  - It follows the route with `FollowedPolyline` (`base/followed_polyline.cpp`), which matches the position to the nearest point ahead within a window.
  - It counts move-away events and triggers rebuilds, using adjust mode first.
  - It reannounces "route recalculated".
  - It computes `FollowingInfo` (`following_info.hpp`): distance and time to target, next turn, next street, lanes, speed limit, and completion percent.
  - It shows lanes within 500 m (`kShowLanesMinDistInMeters`) and pedestrian turns within 20 m.
  - It rebuilds on traffic updates; this hook remains even though no traffic feed exists.
- **`turns::sound::NotificationManager`** (`turns_notification_manager.cpp`, `turns_sound_settings.cpp`) decides *when* to speak. Thresholds depend on speed and units. It covers the "then …" second turn, roundabout exit numbers, street names and speed-camera warnings.
- **`TurnsTtsText`** (`turns_tts_text.cpp`, `turns_tts_text_i18n.cpp`) builds the strings from `data/sound-strings/*.json` (≈51 languages). Positional placeholders are expanded without printf (2026). Platforms speak the text with the system TTS; there are no bundled voices.
- **Speed limits** on route segments come from `maxspeeds`, and conditional maxspeeds apply by current time (2026). `SpeedCameraManager` beeps or speaks when a camera is ahead and the user risks exceeding the limit.
- **Router interaction:**
  - `AsyncRouter` serializes requests.
  - `IndexRouter` keeps `m_lastRoute` (a `SegmentedRoute`) plus `m_lastFakeEdges` for adjust. These are invalidated if routing options change.
  - Passed intermediate points are skipped (`Checkpoints::GetPassedIdx`).
  - `CheckpointPredictor` decides where to insert a newly added intermediate point (the shortest broken line). As of 2026-09 there is also "explicit route point optimization", which reorders stops.

---

## 9. Can it be extracted and used standalone?

### Dependencies

From `#include` counts across `libs/routing`:

| Dependency | Include count |
|---|---|
| `routing/` (internal) | 469 |
| `base/` | 129 |
| `geometry/` | 102 |
| `coding/` | 59 |
| `routing_common/` | 43 |
| `indexer/` | 37 |
| `platform/` | 25 |
| `traffic/` | 8 |
| `transit/` | 7 |
| `kml/` | 3 |
| `opening_hours/` | 2 |

CMake (`libs/routing/CMakeLists.txt`) links `base routing_common platform traffic transit opening_hours`, and `routing_common` pulls in `indexer`. Transitively:
- `indexer` → `platform geometry coding om::timezone`
- `coding` → `glaze`, `succinct`, `expat`, **ICU**, `minizip`, `zlib`, `blake3`
- `platform` → Qt6 Core/Network on desktop

Roughly 100k+ LOC of OM core comes along (base 15k, coding 12k, geometry 8.7k, platform 10.7k, indexer 22.6k, routing 31k).

**Hard couplings:**
- **`DataSource`/`MwmSet`/`FeatureType`/classificator**: all road attributes come from OM features.
- **The mwm container and section formats**: there is no import path from raw OSM at runtime.
- **The generator** (`generator/`, itself depending on most of the tree) to build mwms from planet extracts plus `borders/*.poly` and `countries.json`.
- **Country lookup callbacks** (`CountryInfoGetter` from `storage`, `CountryParentGetter`) passed into the `IndexRouter` constructor (`index_router.hpp:73`).
- `platform::settings` for options. The `kml` types for guides.

### Tools that run it outside the app

- **`routes_builder_tool`** (`libs/routing/routes_builder/`): CLI with flags `--routes_file`, `--dump_path`, `--data_path`, `--resources_path`, `--vehicle_type car|pedestrian|bicycle|transit`, `--threads`, `--launches_number` (benchmarking) and `--timeout`. It builds routes in a thread pool against a directory of mwms. It can also call **Google/Mapbox APIs** (`--api_name`, `--api_token`) to dump comparison routes. `routes_builder.cpp` shows the minimal wiring: `FrozenDataSource` + `RegisterMap(localFile)` for each mwm, `NumMwmIds`, `CountryInfoGetter`/`CountryParentGetter` from `storage`, a `traffic::TrafficCache`, then `IndexRouter(...)` and `CalculateRoute(checkpoints, dir, adjust=false, needAlternatives=true, delegate, result)`.
- **`routing_quality_tool`** (`routing_quality/routing_quality_tool/`) compares OM routes with the dumped API routes and with each other (error-type counters, benchmark stats, similarity). `routing_quality_tests` hold waypoint-based quality checks.
- **`routing_integration_tests`** (turns, roundabouts, cross-country, bicycle, pedestrian, transit, speed cameras, street names, absent regions, …) need real downloaded mwms. The new **`GeneratedMapTest`** (`generated_map_test.hpp`) builds an mwm from an OSM XML file inside the test, which is the closest thing to "route over arbitrary OSM without the whole pipeline".
- `routing_benchmarks`, `routing_consistency_tests`, and the **Qt desktop app** (`qt/`, with debug commands in `docs/DEBUG_COMMANDS.md`) are also available.

**Verdict:** you can run it headless on Linux using `routes_builder_tool` and generated mwms. Extracting it as a library means extracting OM's core and generator too. There is no C API, no server, and no stable data contract. MWM formats change with app versions, and CoMaps has already diverged.

---

## 10. Strengths, weaknesses, suitability

### Strengths

- **A proven offline mobile design at planet scale**, with millions of users. The data is compact, adds about 5% to mwm files already used for rendering and search, and installs per region.
- **Flexibility without preprocessing:** profiles, avoid-options, restrictions, conditional access/maxspeed, altitude penalties and (latent) traffic factors are all evaluated at query time, because there is no CH. The price is search time, which the hierarchy mitigates.
- **A clean two-level cross-region decomposition:** twins at borders, precomputed enter→exit cliques per region, overlay search, then per-region refinement, plus absent-region detection from a world-level sparse graph.
- **Pragmatic multi-criteria costs** (`RouteWeight` counters for pass-through/access/conditional), the **dual weight/ETA speeds**, and per-country legal models.
- **Careful snapping** (dead-end and fenced-off filtering, heading, multiple candidates).
- **Cheap "adjust" rerouting.**
- **Rich guidance:** turns, roundabouts, lanes, street names and refs, cameras, 50+ TTS languages. Years of integration tests encode real-world edge cases, with issue numbers in comments.
- **An active maintainer** (Viktor Govako) and real feature work in 2026.

### Weaknesses

- **Performance ceiling:** plain A\* with a weak (max-speed) heuristic. Long car routes depend on the leaps overlay, which uses a heuristic candidate search (15 keys, 30 s timeout, at most 2 per road class) and is not exact. The code has many `@todo`/"crutch" comments. Long routes on phones can take seconds to tens of seconds.
- **Precomputed cross-mwm weights are car-only and static.** They ignore avoid-options and time-dependence, so they are suboptimal with avoids. Bike and foot cross-region routes run plain Joints-mode A\* across mwms.
- **The restriction handling** (parent-chain checks) is not exact for via-way restrictions. No-pass-through counting is also a heuristic.
- **No turn/intersection costs** in OM, and no traffic data. Transit is frequency-only (no timetables or realtime).
- **Only one alternative**, from a different cost function, and no multi-criteria/Pareto routing. No isochrones, matrix or map-matching APIs (there is no server).
- **Tight coupling** to OM's feature model, classificator and mwm format. Profiles are compiled C++, so there is no runtime profile language.
- **Two diverging forks** with incompatible mwm sections.

### Suitability

- **(a) As a dependency for a new (Rust?) project: not recommended.** You would need OM's generator to produce data and OM's C++ core to read it, behind FFI. You also inherit ICU, Qt (desktop platform) and other dependencies. Licensing is fine (Apache-2.0), but engineering cost and format churn are high. It is only viable if you are building *on* OM or CoMaps themselves, e.g. a server that reuses their mwms. Even then, `routes_builder` is the only real entry point.
- **(b) As a reference: highly recommended**, particularly for offline and mobile ideas:
  1. **Store topology only, reuse geometry**: a varint/gamma-coded joint index keyed by (feature, point), with attributes computed lazily via a cache.
  2. **Per-region files with border twins** matched by OSM way id, plus a **precomputed boundary clique** per region. This is a simple CRP-style overlay; a Rust port could add proper customization per profile and per avoid-set, which fixes OM's biggest weakness.
  3. **A world-level sparse border graph** to tell the user which regions to download.
  4. **Dual weight/ETA speeds**, and `RouteWeight` with lexicographic "soft" counters for private/destination/pass-through zones.
  5. **Bidirectional A\* with average potentials**, including the multi-emit variant for candidate generation.
  6. **The adjust-reroute trick** (5-min Dijkstra from the current position to the old route).
  7. **Snapping heuristics** (dead ends, fences, heading, multiple candidates).
  8. **Per-country legal access tables** and in-city/out-of-city speed defaults derived from tagged maxspeeds per region.
  9. **Turn-candidate collection** and the lane recommendation pipeline. The integration tests double as a corpus of tricky intersections.
  10. **`opening-hours-rs`**: OM itself ported this Rust crate to C++, and it is directly reusable in Rust for conditional restrictions.

  Treat LeapsOnly's candidate heuristics and the restriction parent checks as cautionary examples. Do them exactly (edge-based states, proper overlay customization) in a new design.

---

## 11. CoMaps (codeberg.org/comaps/comaps)

- **Fork point:** OM `fad26bbf` (2025-04-02), imported as one squashed commit on 2025-04-13 ("Organic Maps sources as of 02.04.2025"). There are 3,201 commits since. Same Apache-2.0 license and same `libs/` layout. CoMaps runs its own map generation.
- **Routing activity:** 44 commits in 2025 and 53 in 2026 (`libs/routing`, `libs/routing_common`, `libs/traffic`). Contributors include Henry Sternberg (turns/TTS/penalties), x7z4w, Mike Gelfand (dependency hygiene and include-what-you-use; i18n broken out of base/coding/indexer into `libs/i18n`), Konstantin Pastbin, eisa01 (CarPlay), Nils Hasler, mvglasow, Gonzalo Pesquero, and occasional cherry-picks of Viktor Govako's OM commits (e.g. "[routing] Enable access conditional", which was reverted and then fixed for 32-bit crashes).
- **Notable routing differences from OM:**
  - **`roadpenalty` section** (`comaps:libs/routing/road_penalty.{hpp,cpp}`, `ROAD_PENALTY_FILE_TAG` in `comaps:defines.hpp`): point time penalties for traffic calming (small/medium/large), gates, and uncontrolled or controlled junctions (traffic signals), per vehicle, from OSM tags (2025-07, "OSM tag based time and turn penalties").
  - **Turn-angle penalty matrix** in `comaps:libs/routing/edge_estimator.cpp` (`TurnPenaltyMatrix`, ≈288 entries by from/to `HighwayType` and vehicle), applied in `IndexGraph::GetPenalties` with `m_isLeftHandTraffic`. Bicycle turn penalties were added in 2025-08.
  - **Extra avoid options:** `Paved` and `Steps` (`comaps:libs/routing/routing_options.hpp`, with per-mode masks). Options are per transport type.
  - **`DecoderModel`** (`comaps:libs/routing_common/decoder_model.*`) and **routable `highway=construction` types**, part of mvglasow's work toward TraFF traffic-message support (the `origin/traffic` branch is not merged).
  - **Guidance:** chained roundabout instructions, "at the end of the road", wide U-turn detection, no U-turn instruction on hairpins, dropping "stay on same road" instructions, CarPlay rich guidance, lane guidance using unrestricted lanes, an option to disable rerouting, a 10 min ferry wait, and the speed camera list adjusted (Germany removed, etc.).
  - **Missing compared with OM:** 2026 alternative routes / DistanceBiased strategy, multi-candidate snapping, transit alternatives, and the opening-hours-rs port.
- **Code-size drift since the fork:** `git diff --shortstat` of OM@fork vs each HEAD for `routing/` is about 10.3k+/8.9k− lines in CoMaps vs about 11.4k+/10.8k− in OM. Part of that is reformatting, but the two are clearly diverging, and the mwm formats are no longer interchangeable.
- **For your purposes:** CoMaps is the more community-governed codebase and has better intersection costing (penalties and turn costs). OM has the better algorithmic changes in 2026 (alternatives, snapping) and the more active core routing maintainer. Architecturally they are identical, and everything in §§2–10 applies to both.

---

## Sources

- Source trees: `~/g/organicmaps` (HEAD 408e7945, 2026-10-23), `~/g/comaps` (HEAD 5bd62c92, 2026-10-23).
- mwm samples: `https://cdn.organicmaps.app/maps/260714/{Luxembourg,Germany_Berlin}.mwm` and the repo's `data/World.mwm`, parsed with a TOC reader.
- [Organic Maps news, 2026-06-29](https://organicmaps.app/news/2026-06-29/satellite-imagery-public-transport-routes-alternate-routes-new-search-and-route-planning-interface-for-android-large-accessibility-fonts/) (transit modes, alternate routes).
- [AlternativeTo, Oct 2026](https://alternativeto.net/news/2026/10/organic-maps-brings-improved-alternative-routes-intermittent-water-areas-and-more/) (reworked alternatives, stop reordering).
- [It's FOSS: CoMaps fork](https://itsfoss.com/news/organic-maps-fork-comaps/) and [AlternativeTo: CoMaps fork](https://alternativeto.net/news/2025/5/organic-maps-contributors-fork-project-as-concerns-over-governance-and-transparency-grow/) (governance background; partly reprinted on CoMaps' site, so not neutral).
