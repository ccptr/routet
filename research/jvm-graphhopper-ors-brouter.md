# JVM routing engines: GraphHopper, openrouteservice, BRouter

Research snapshot: 2026-10-23. Sources read:

| repo | path | HEAD |
|---|---|---|
| graphhopper/graphhopper | `~/g/graphhopper` | `5d19929` (2026-10-23), version `12.0-SNAPSHOT` |
| GIScience/openrouteservice | `~/g/openrouteservice` | `52fee39` (2026-10-21), version `10.2.0-SNAPSHOT` |
| GIScience/graphhopper (the fork ORS depends on) | `~/g/graphhopper-ors-fork` (shallow, tag `v4.16.0`) | 2026-08-31 |
| abrensch/brouter | `~/g/brouter` | `2989810` (2026-09-28), latest tag `v1.7.10` |

File paths are relative to each repo's root unless stated otherwise. Statements about the code come from reading the source. Web-sourced claims are marked and linked.

---

## TL;DR

- **GraphHopper** is the most complete JVM engine and among the best-engineered OSM routers in any language. It is Apache-2.0 and run by GraphHopper GmbH, with steady releases (about one major per year, v11.0 in Oct 2025, v12 in pre-release). It offers three modes: **CH** ("speed"), **Landmarks/ALT** ("hybrid") and **Dijkstra/A\*** ("flexible"). It has node- and edge-based CH (so turn restrictions work in CH), and **custom models** (a JSON/YAML expression language compiled to JVM bytecode with Janino). It also includes isochrones, map matching (HMM), GTFS transit (including a newer trip-based router) and a `/navigate` endpoint. The matrix API is **not** open source; it belongs to the commercial Directions API. Master now needs **Java 25** and uses the FFM API (`java.lang.foreign`). Android/offline use was dropped in 2020, and Janino bytecode generation makes modern versions effectively unusable on Android ART.
- **openrouteservice (ORS)** is HeiGIT's (Heidelberg) service layer on a **heavily patched fork of GraphHopper 4.0 (2021)**. It is still on that base: the fork is tagged `v4.16.0`, has 709 `ORS-GH MOD` markers in 134 files, and in 2026 is still merging upstream commits on the way to GH 5. ORS adds a matrix (RPHAST on CH, a core-based matrix), "core" routing (partial CH with an uncontracted core plus ALT, for dynamic avoid-features and time-dependent traffic), fast isochrones (Inertial Flow partitioning), wheelchair/HGV profiles, many extra edge storages (green, noise, shadow, heat stress, CSV), and a PostgreSQL dynamic-data feature store. It is actively maintained (about 770 commits in the last year). Licence: the LICENSE file is GPL-3.0, while NOTICE.md still says Apache-2.0. ORS is a product to deploy, not a library to depend on.
- **BRouter** is a compact (about 32k LOC), MIT-licensed, **offline-first bike router**. It reads precompiled 5°×5° `.rd5` tiles: entropy-coded, with elevation embedded, downloaded weekly from brouter.de. It uses **no preprocessing per profile**. Instead it runs a 2-pass A\*→Dijkstra search with an adaptive cost cutoff, over an **edge-based** graph (OSM links are the search nodes), driven by a **scripting profile language** (`.brf`, prefix notation) with CRC-keyed result caching over the tag bitstream. It has elevation hysteresis, a kinematic/energy model and environmental pseudo-tags (noise, forest, river, traffic). On Android it runs as an **AIDL service** (`IBRouterService.getTrackFromParams(Bundle)`) that Locus, OsmAnd and others call. It is maintained by a small team (mostly one active maintainer). Search effort grows roughly quadratically with distance, so it is suited to day-trip ranges rather than continental car routing.

---

# Part 1 — GraphHopper

## 1. Overview

### Licence, ownership, people
- **Apache License 2.0** (`LICENSE.txt`). The README explicitly encourages embedding in closed-source products.
- Copyright headers say "Licensed to GraphHopper GmbH under one or more contributor license agreements" (every source file). Contributions need a CLA (`CONTRIBUTING.md`).
- **GraphHopper GmbH** (Germany) is the company. It was founded by Peter Karich (`karussell`), still the main author.
- Commits in the last year (2025-10-23 → 2026-10-23, 316 commits): Peter (171), Michael Zilske (62, transit and core), easbar (40, CH and core algorithms; author of the edge-based CH and the Rust `fast_paths` crate), ratrun (10), Andi (8), otbutz, Nakaner, plus contributors from Bosch eBike and others. `CONTRIBUTORS.md` lists about 100 people over time. The core team is small (2–4 people) but has been very consistent.

### Release cadence
From `git for-each-ref refs/tags` and `CHANGELOG.md`: 6.0 Sep 2022 → 7.0 Mar 2023 → 8.0 Oct 2023 → 9.0 Apr 2024 → 10.0 Nov 2024 → 11.0 14 Oct 2025 (11.1 Sep 2026) → **12.0-pre1..pre3** (Sep 2026). That is about one major per year, sometimes two. Majors often break the Java API and sometimes the config or graph format: the CHANGELOG is full of "breaking:" entries, and `docs/migration/config-migration-11-12.md` exists. Graphs must be **re-imported** on most upgrades, because `Constants.VERSION_*` is checked in `GHUtility.checkDAVersion`.

### Production users (web-sourced)
Wikipedia and GraphHopper's blog list Komoot (historically), GNOME Maps (via the Directions API), Rome2rio, Deutsche Bahn, Flixbus, Falk/route.nl, Ride with GPS and Volo. GraphHopper has been one of the routers on openstreetmap.org since 2015. OPTITOOL sponsored features in 11.0 (private-road avoidance). Bosch eBike engineers commit upstream. ORS (below) is the largest open derivative. ([Wikipedia](https://en.wikipedia.org/wiki/GraphHopper), [GH blog: Falk, Ride with GPS, Volo](https://www.graphhopper.com/blog/2015/04/21/falk-ride-with-gps-and-volo-uses-graphhopper/), [GH 11.0 release post](https://www.graphhopper.com/blog/2025/10/14/graphhopper-routing-engine-11-0-released/))

### Open-core split
README §"GraphHopper Directions API": the commercial offering provides the Routing, **Matrix**, Isochrone and Map Matching APIs, plus **Route Optimization** (based on their OSS `jsprit`, using the fast Matrix API) and geocoding (based on OSS `photon`).

What is **missing from open source**:
- **Matrix (many-to-many)**. There is no `/matrix` resource in `web-bundle/src/main/java/com/graphhopper/resources/`. Only the *client* exists: `client-hc/.../GraphHopperMatrixWeb.java`, `GHMatrixBatchRequester.java`. In core, the nearest things are `routing/DijkstraOneToMany.java` (used for CH witness searches) and the SPT/isochrone code. A maintainer's forum reply confirms matrix is commercial-only ([discuss.graphhopper.com](https://discuss.graphhopper.com/t/matrix-requests-on-local-install/7279)).
- **Route optimization service** (VRP; jsprit itself is OSS, the hosted service is not).
- **Geocoding service** (photon is OSS; the hosted service is not).
- Hosted infrastructure: planet graphs, traffic integration, the "Profiles API" and similar. Some features ship in OSS but are disabled in the hosted API, e.g. custom-model `turn_penalty` per the 11.0 post.

Everything else is open: CH/LM/flex routing, custom models, isochrones, SPT, map matching, GTFS/PT, `/navigate`, MVT debug tiles. The open-source engine is the same engine the company runs.

## 2. Deployment options

### 2.1 Standalone server (Dropwizard)
- `web/src/main/java/com/graphhopper/application/GraphHopperApplication.java` is the Dropwizard app (Dropwizard 5.0.2, Jetty 12.1.8 per root `pom.xml`). CLI commands: `server`, `import` (`web/.../cli/ImportCommand.java`), `match` (`MatchCommand.java`).
- `web-bundle/` holds the reusable Dropwizard bundle (`http/GraphHopperBundle.java`, `GraphHopperManaged.java`) and JAX-RS resources: `RouteResource` (`/route` GET+POST), `IsochroneResource` (`/isochrone`), `SPTResource` (`/spt`, CSV shortest-path-tree), `MapMatchingResource` (`/match`), `NearestResource`, `InfoResource`, `MVTResource` (`/mvt/{z}/{x}/{y}.mvt` vector tiles of the graph), `PtRouteResource` (`/route-pt`), `PtIsochroneResource`, `I18NResource`, `HealthCheckResource`.
- `navigation/src/main/java/com/graphhopper/navigation/NavigateResource.java` provides `/navigate`, which outputs a Mapbox-Directions-compatible response for the MapLibre Navigation SDK and ferrostar (README §Mobile Apps).
- The web UI is "GraphHopper Maps" (separate repo `graphhopper/graphhopper-maps`), bundled into the jar.
- Docker: only community images (`israelhikingmap/graphhopper`), per the README.
- Requirements: **JVM ≥ 25** (README §Installation; `pom.xml` sets `<release>25</release>`).
- Config: `config-example.yml` (profiles, `profiles_ch`, `profiles_lm`, `graph.encoded_values`, `graph.dataaccess.*`, elevation, routing limits).

### 2.2 Java library embedding
- Maven `com.graphhopper:graphhopper-core`. Entry point `core/src/main/java/com/graphhopper/GraphHopper.java` (1812 lines): `importOrLoad()`, `route(GHRequest)`, `getBaseGraph()`, `getLocationIndex()`.
- Examples in `example/src/main/java/com/graphhopper/example/`: `RoutingExample.java`, `RoutingExampleTC.java` (turn costs), `HeadingExample.java`, `IsochroneExample.java`, `LocationIndexExample.java`, `LowLevelAPIExample.java` (building a `BaseGraph` by hand and running algorithms directly).
- `web-api` holds the request/response model (`GHRequest`, `GHResponse`, `ResponsePath`, `CustomModel`, `Instruction*`), shared by the server and `client-hc`.
- Core runtime dependencies (`core/pom.xml`): HPPC (primitive collections), Janino (custom-model compiler), JTS, Jackson (+ JTS, XML), xmlgraphics-commons and webp-imageio (elevation tiles), `osm-legal-default-speeds-jvm` (Kotlin; default speed limits per country), `osmpbf`, slf4j/logback. This is a modest dependency set; the web layer pulls in the Dropwizard stack.

### 2.3 Android / offline
- Historically GH shipped an Android demo APK (up to 1.0, May 2020; links in the README). iOS was a community fork (`graphhopper-ios`, via J2ObjC).
- **Issue #1940 "Future of 'Offline' Routing" (Feb–Jul 2020)**: the maintainer dropped explicit Android support for 2.0 because of the cost of Java 7 compatibility, the Android per-process memory limits, the need for graph tiling/merging, and the lack of a maintainer. The README now says: "Offline routing is no longer officially supported but should still work as Android supports most of Java." Community workarounds are Termux or UserLAnd running a full OpenJDK and pointing apps at `localhost:8989` (e.g. OsmAnd's online router). ([issue #1940](https://github.com/graphhopper/graphhopper/issues/1940))
- **My assessment from the current code: "should still work" is no longer true for in-process ART use.**
  1. Bytecode level 25 and `java.lang.foreign` usage (`storage/ForeignMemoryDataAccess.java`, `MMapForeignMemoryDataAccess.java`, `MMapForeignReadOnlyDataAccess.java`). ART has no FFM API. The legacy `MMAP`/`RAM` DATypes remain (`storage/DAType.java`), so this part could be patched around.
  2. **Custom models are the only weighting**: `fastest`/`shortest` were removed in 8.0/9.0 (CHANGELOG). They are compiled at runtime by Janino into JVM `.class` bytecode (`routing/weighting/custom/CustomModelParser.java`, `createCompiler` → `SimpleCompiler`, loaded via `getClassLoader().loadClass(...)`). ART cannot load JVM bytecode without dexing, so this is a fundamental blocker short of writing an interpreter for the expression AST.
  3. There is no tiling: one monolithic graph per region, and no cross-graph routing (issue #293).

  So GH is a poor fit for an offline mobile engine today. The 0.x/1.0 Android design (MMAP plus CH on a pre-built graph) is still a useful historical reference.

## 3. Data pipeline

### 3.1 OSM import
Flow in `GraphHopper.process()` (`core/.../GraphHopper.java` ≈ L814–855):
`prepareImport()` → build `BaseGraph` → `importOSM()` → `postImportOSM()` → `cleanUp()` (subnetwork removal) → `baseGraph.freeze()` → write EncodingManager into properties → `calcChecksums()` → `flush()` → `postProcessing()` (location index, CH, LM).

- **`reader/osm/WaySegmentParser.java`** reads the PBF (`reader/osm/pbf/*`, multi-threaded blob decoding) **twice**. Pass 1 skips nodes and records which OSM node IDs are junctions of accepted ways. Pass 2 splits ways into segments at junctions, adds artificial segments (barriers, loops) and calls an `EdgeHandler`. It assumes the file order nodes → ways → relations.
- **`reader/osm/OSMNodeData.java`** maps 64-bit OSM node IDs to compact IDs. Negative values are tower nodes (junctions, stored directly in the final graph's node storage to save memory). Positive values are *pillar* nodes (geometry-only), stored as packed lat/lon in the map. Node tags are kept only for the few nodes that matter (barrier, highway, crossing, ford; `INCLUDE_IF_NODE_TAGS`).
- **`reader/osm/OSMReader.java`** (691 lines) creates edges, runs the tag parsers, handles relations (route networks, restrictions) and node tags.
- **Turn restrictions**: `OSMRestrictionConverter.java`, `RestrictionTagParser.java`, `RestrictionTopology.java`, `WayToEdgeConverter.java`, `routing/util/parsers/RestrictionSetter.java`. Since 10.0 these support multiple and overlapping via-ways. Via-way restrictions are implemented by **cloning edges** ("artificial edges", #2689; see the comment in `postImportOSM`). Conditional restrictions and `except=` are honoured (`docs/core/turn-restrictions.md`).
- **Post-import** (`postImportOSM`): urban density (`routing/util/UrbanDensityCalculator.java`, road-junction based), "softblocks" (private/destination entry detection, `IsSoftblockedAtEntry`), max-speed estimation from country legal defaults (`routing/util/MaxSpeedCalculator.java` + `osm-legal-default-speeds`), elevation interpolation for bridges, tunnels and ferries, slope (`SlopeCalculator`), curvature, and then **Hilbert-curve sorting of nodes and edges** (`sortGraphAlongHilbertCurve`, L1068) for cache locality ("improved performance by sorting graph during import, #3177", v11).
- **Subnetwork removal**: `routing/subnetwork/PrepareRoutingSubnetworks.java` runs edge-based Tarjan per profile and marks small islands in a per-profile `Subnetwork` boolean EV (`prepare.min_network_size`), so snapping avoids them.
- **Country/state assignment** from bundled polygons (`CountryParser`, `StateParser`; custom areas `docs/core/custom-areas-and-country-rules.md`). Country rules (toll, access defaults) are enabled by default in 12.

### 3.2 Encoded values and EncodingManager
- Every edge has a fixed-width bit field (`bytesForFlags`). **Encoded values (EVs)** are typed views into it: `routing/ev/BooleanEncodedValue`, `IntEncodedValueImpl`, `DecimalEncodedValueImpl` (int with a factor), `EnumEncodedValue`, `StringEncodedValue` (small dictionary), plus `ExternalBooleanEncodedValue`. Most EVs store **two directions** (forward/backward bits) where needed.
- About 70 built-in EVs live in `routing/ev/`: `RoadClass`, `RoadEnvironment`, `RoadAccess`, `Surface`, `Smoothness`, `Toll`, `MaxSpeed`, `MaxWeight`, `Hazmat*`, `BikeNetwork`, `Country`, `UrbanDensity`, `AverageSlope`, `MaxSlope`, `Curvature`, `Lanes`, `Sidewalk`, `Cycleway`, `Crossing`, `MtbRating`/`HikeRating`, `VehicleAccess`/`VehicleSpeed`/`VehiclePriority` (per vehicle: `car_access`, `bike_average_speed`, …), temporal access EVs (`CarTemporalAccess`…), `TurnCost`/`TurnRestriction` (turn EVs), `Subnetwork`, `OSMWayID`, and more.
- **`routing/ev/DefaultImportRegistry.java` / `ImportRegistry` / `ImportUnit`** map each EV name to its parser(s) and required EVs. In `prepareImport()` (L857+), GH scans the profiles' custom models for referenced variables (`CustomModelParser.findVariablesForEncodedValuesString`), resolves dependencies, and builds the `EncodingManager` plus `OSMParsers`. **The set of EVs is fixed at import time.** Adding an EV, or a profile that needs one, means re-importing (`docs/core/deploy.md`: "Adding or modifying profiles is not possible and you need to run a new import").
- `routing/util/EncodingManager.java` serialises the EV layout into `properties` (`graph.encoded_values`, `graph.turn_encoded_values`, `graph.em.bytes_for_flags`) so that a load can reconstruct it exactly.
- Tag parsers live in `routing/util/parsers/`: one `OSM*Parser` per EV (e.g. `OSMRoadClassParser`, `OSMMaxSpeedParser`, `OSMSurfaceParser`) plus vehicle parsers (`CarAccessParser`, `BikeAverageSpeedParser`, `BikePriorityParser`, `FootAccessParser`, `MountainBike*`, `RacingBike*`, `ModeAccessParser`).

### 3.3 Storage: `DataAccess` and friends
- `storage/DataAccess.java` is the abstraction for a growable, segmented array of bytes/ints with a small header, persisted as one file per DataAccess in the graph folder.
- **`storage/DAType.java` (refactored in 12, #3382)** is a registry of factories:
  - `RAM` (`RAMDataAccess`, segmented `byte[][]` on heap; loads/flushes the whole file explicitly), `RAM_INT`, `RAM_INT_1SEG`, `RAM_1SEG`, `RAM_LONG` (single contiguous heap arrays, up to ~16 GB for `long[]`).
  - `FOREIGN_ANON` (`ForeignMemoryDataAccess`, off-heap anonymous memory via the FFM API).
  - `FOREIGN_MMAP` (`MMapForeignMemoryDataAccess`; read-only fast path `MMapForeignReadOnlyDataAccess` with all-final fields).
  - Legacy `MMAP` (`MMapDataAccess`, ByteBuffer-based).
  - There is a system-wide `graph.read_only` mode. Types can be chosen **per DataAccess name** (`dir.getDefaultType("nodes", true)` etc., configurable through `graph.dataaccess.*`), e.g. CH in RAM and geometry mmapped. In `BaseGraph.Builder.build()` (L736) geometry defaults to `FOREIGN_MMAP` when file-backed.
- **Memory model consequences** (`docs/core/deploy.md`): the advice is `-Xmx ≈ graph-cache size minus geometry file`. Avoid G1 humongous-allocation waste (align `graph.dataaccess.segment_size`), prefer ParallelGC for import and ZGC/Shenandoah for serving. MMAP planet import is possible in 31 GB of RAM but takes about 3 days, versus about 3 h with about 60 GB.

### 3.4 Graph folder format (`graph-cache/`)
One file per DataAccess (names from `dir.create(...)` calls):

| file | content | source |
|---|---|---|
| `nodes` | per node: first-edge ref, lat, lon, [ele], [turn-cost ref] | `storage/BaseGraphNodesAndEdges.java` |
| `edges` | per edge: nodeA, nodeB, linkA, linkB, dist (mm, int32), kv ref, flags[bytesForFlags], geo ref (5 bytes) | same |
| `geometry` | pillar points of edges | `storage/BaseGraph.java` |
| `edgekv_keys`, `edgekv_vals` | street names, refs, conditional tags… (key/value per edge) | `search/KVStorage.java` |
| `turn_costs` | turn-cost entries | `storage/TurnCostStorage.java` |
| `location_index` | spatial index | `storage/index/LocationIndexTree.java` → `LineIntIndex` |
| `properties` | versions, EV layout, profiles, checksums, prep status | `storage/StorableProperties.java` |
| `nodes_ch_<profile>`, `shortcuts_<profile>` | CH levels and shortcuts | `storage/CHStorage.java` |
| `landmarks_<profile>`, `landmarks_subnetwork_<profile>` | LM tables | `routing/lm/LandmarkStorage.java` |

Every file header stores a version constant checked on load. Each file can be loaded as RAM or mmapped independently.

### 3.5 Elevation
`reader/dem/` contains providers for SRTM (`SRTMProvider`, `SRTMGL1Provider`), CGIAR, GMTED, Skadi (AWS terrain tiles), Sonny (European LiDAR DTMs), HGT, multi-source combiners (`MultiSourceElevationProvider`, `MultiSource3ElevationProvider`), and **PMTiles** (`PMTilesElevationProvider`, `PMTilesReader`, `PMTilesExtract`, new; see `docs/core/deploy.md` and #3391). Edge sampling, smoothing (`EdgeElevationSmoothingMovingAverage`, `EdgeElevationSmoothingRamer`) and bridge/tunnel interpolation (`EdgeElevationInterpolator`, `BridgeTunnelTowerCorrection`) are applied. Elevation is stored per node (`N_ELE`) and per pillar point, and feeds `average_slope`/`max_slope` EVs used by e.g. `custom_models/bike_elevation.json`. Guide: `docs/core/elevation.md`.

### 3.6 Planet resource figures (from `docs/core/deploy.md`, data as of about May 2022)
- Base graph: about **415 M edges** (Europe 150 M, North America 86 M). Import needs about **60 GB RAM and about 3 h**, or 31 GB with MMAP and about 3 days.
- **CH** for car *with turn costs* (edge-based): about **120 GB RAM, about 25 h**. Node-based CH (enough for bike) takes about 5 h.
- **LM** for car: about **80 GB RAM, about 4 h**. Earlier blog figures put 16 landmarks at about 5 GB per profile worldwide and about 550 MB for Germany.
- Germany import: "roughly 10 minutes" (`docs/core/technical.md`).
- CH and LM can be added to existing profiles after import. Changing profiles' EVs needs a re-import.

## 4. Graph model

- **`storage/BaseGraph.java`** (1218 lines) and **`BaseGraphNodesAndEdges.java`**. Node and edge IDs are dense ints 0..N-1. Each **undirected edge is stored once** with nodeA/nodeB, and adjacency is an **intrusive linked list**: a node points to its first edge, and each edge has `linkA`/`linkB` next-pointers for its two endpoints (`docs/core/technical.md`, diagram `docs/core/images/wiki-graph.png`). Direction is encoded via **edge keys** (`edgeKey = 2*edgeId + reverse`). Access and speed per direction come from the EVs' forward/backward bits.
- **Tower vs pillar nodes**: only junctions are graph nodes. Intermediate geometry is stored in `geometry`, which saves enormous memory compared to node-per-OSM-node graphs.
- Distances are stored as int32 **millimetres** (max about 2147 km per edge). Accumulation uses exact `long` arithmetic since 12.0 (#3286, `getDistance_mm`). Loop edges are disallowed since 8.0.
- **`EdgeIteratorState` / `EdgeExplorer`**: cursor-style, allocation-free iteration. Filters: `routing/util/EdgeFilter`, `AccessFilter`, `DirectedEdgeFilter`.
- **Turn costs/restrictions storage**: `storage/TurnCostStorage.java` is a key/value store keyed by `(fromEdge, viaNode, toEdge)`. Each 16-byte entry is `|from|to|flags|next|`, chained as a linked list from the via node's `N_TC` slot. Turn EVs (`TurnRestriction` boolean, `TurnCost` decimal) are bit fields within `flags`. Turn costs are evaluated by `routing/weighting/DefaultTurnCostProvider.java`, and from 11.0 also by custom-model `turn_penalty` statements with `prev_*` and `change_angle` variables.
- **Node-based vs edge-based traversal**: `routing/util/TraversalMode.java` has `NODE_BASED` and `EDGE_BASED` (search state = directed edge key). A profile with `turn_costs:` configured is routed edge-based, which turn restrictions require. The cost is roughly 2–3× more search states and much heavier CH preparation. `curbside` and finite `u_turn_costs` require edge-based profiles (`Router.java` L397–405).

## 5. Algorithms (all in `core/src/main/java/com/graphhopper/routing/`)

### 5.1 Flexible mode (no preprocessing)
- `Dijkstra.java`, `DijkstraBidirectionRef.java`, `AStar.java`, `AStarBidirection.java` share bases `AbstractRoutingAlgorithm`, `AbstractBidirAlgo`, `AbstractNonCHBidirAlgo` and path extraction in `Path.java`, `PathExtractor.java`, `DefaultBidirPathExtractor.java`.
- Heuristics: `weighting/BeelineWeightApproximator.java`, `BalancedWeightApproximator.java` (for bidirectional A\*).
- Default flex algorithm: bidirectional A\* (`RoutingAlgorithmFactorySimple.java`). Limits: `routing.max_visited_nodes`, `routing.non_ch.max_waypoint_distance`, and request timeouts (#2795). Without these, long flex queries can use lots of RAM per request (deploy guide, #734).
- `FlexiblePathCalculator.java` is the glue between solver and algorithm.

### 5.2 Contraction Hierarchies — "speed mode" (`routing/ch/`, about 5.8k LOC with LM)
- **Preparation**: `PrepareContractionHierarchies.java` handles node ordering with a priority queue, periodic, lazy and neighbour priority updates (`CHParameters.java`: `prepare.ch.updates.periodic|lazy|neighbor|neighbor_max`), and parallel preparation of several profiles (`CHPreparationHandler.java`). It works on a compact `CHPreparationGraph.java` and writes `CHStorage`.
- **Node-based**: `NodeBasedNodeContractor.java` plus `NodeBasedWitnessPathSearcher.java`. Priority uses edge difference, original edges and hierarchy depth.
- **Edge-based** (turn costs inside CH): `EdgeBasedNodeContractor.java` plus `EdgeBasedWitnessPathSearcher.java` and `BridgePathFinder.java`, using an "aggressive" witness search over original in/out edges of neighbours. Shortcuts store the first and last *original* edge keys (`S_ORIG_KEY_FIRST/LAST` in `storage/CHStorage.java`), so turn costs at shortcut endpoints can be looked up in the base turn table. Loop shortcuts are handled, and witness searches are bounded by settled-edge counts tuned on the fly (`OnFlyStatisticsCalculator.java`). The design write-up in `docs/core/edge-based-ch.md` is excellent, with literature (Geisberger 2008; Geisberger and Vetter 2011; Nowak 2014).
- **Storage**: `storage/CHStorage.java` stores per shortcut `NODEA|NODEB|WEIGHT|SKIP1|SKIP2|[ORIG_FIRST|ORIG_LAST]` (20 or 28 bytes) and per node `LEVEL|LAST_SC`. Weights are stored as integers (12.0: "weightings are now expected to return whole numbers… x10", #3297). The CH overlay sits on the base graph: `RoutingCHGraphImpl`, `RoutingCHEdgeIterator*`.
- **Query**: `CHRoutingAlgorithmFactory.java` picks the algorithm. Node-based CH defaults to `DijkstraBidirectionCH` (with stall-on-demand) or `DijkstraBidirectionCHNoSOD`. Edge-based CH defaults to `AStarBidirectionEdgeCHNoSOD` (A\* helps there; no SOD, per the docs). Unpacking uses `ch/ShortcutUnpacker.java`, `NodeBasedCHBidirPathExtractor.java`, `EdgeBasedCHBidirPathExtractor.java`. Query graph integration uses `querygraph/QueryRoutingCHGraph.java`, which overlays virtual snap nodes on the CH graph without re-contracting.
- **Restrictions in speed mode** (`Router.java` `CHSolver.checkRequest`, L452–465): no `heading`, no `pass_through`, **no per-request `custom_model`**, no `round_trip`. The weighting is frozen at preparation time.

### 5.3 Landmarks / ALT — "hybrid mode" (`routing/lm/`)
- `PrepareLandmarks.java` and `LMPreparationHandler.java` (default **16 landmarks**, `landmarkCount = 16`; 8 active per query by default, `routing.lm.active_landmarks`). `LandmarkStorage.java` stores, per node and landmark, from/to weights as **unsigned 16-bit** values scaled by a factor (`SHORT_INFINITY`, `factor = maxWeight/PRECISION`). This is why LM must split the world into areas (`SplitArea.java`, `LandmarkSuggestion.java`; Europe/Africa/Asia borders) and can be very slightly suboptimal for tiny differences (`docs/core/landmarks.md`). `LMApproximator.java` is the A\* potential, used by `AStarBidirection`. `epsilon < 1` allows faster, inexact queries.
- `preparation_profile:` lets several profiles share one LM preparation, provided their weights are ≥ the prepared one's (`docs/core/profiles.md`).
- **Per-request custom models are allowed** in LM mode under constraints that keep the potentials admissible: query `multiply_by` must be in [0,1] and `distance_influence` must not be smaller. This is enforced by `routing/weighting/custom/FindMinMax.checkLMConstraints` (called from `Router.LMSolver`).
- Mode switching per request: `ch.disable=true` drops to LM if prepared, otherwise flex. `lm.disable=true` forces flex (`docs/core/routing.md`).

### 5.4 Alternatives
- Flex/LM: `AlternativeRoute.java` uses the plateau method plus a partial penalty method (Camvit choice routing; Abraham et al. 2010; and others cited in the javadoc).
- CH: `AlternativeRouteCH.java` (node-based) and `AlternativeRouteEdgeCH.java` (edge-based), "minimum moving parts" via-node alternatives after Abraham et al., with `maxWeightFactor`, `maxShareFactor`, `localOptimalityFactor` and `maxPaths`.
- Only two points are supported, with no `pass_through` or `curbside` (`Router.routeAlt`).

### 5.5 Round trips
`RoundTripRouting.java` and `routing/util/tour/*` generate tours from a start point, heading and distance (flex mode only).

### 5.6 Isochrones and SPT
- `isochrone/algorithm/ShortestPathTree.java`: a Dijkstra up to time, distance or weight limits, with deleted-flag lazy deletion (javadoc notes why). It is also used by `/spt` (CSV dump of the tree; [blog on high-precision reachability](https://www.graphhopper.com/blog/2018/07/04/high-precision-reachability/)).
- Polygons: `JTSTriangulator.java` builds a Delaunay triangulation of reached points, and `ContourBuilder.java` runs marching triangles over the triangulation. `IsochroneResource` and `SPTResource` force `ch.disable=true`, so isochrones run in flex mode only. Example: `example/.../IsochroneExample.java`; docs: `docs/isochrone/java.md`. The isochrone module has been open source since 0.11 (2018).

### 5.7 Matrix
Not in open source (see §1). `DijkstraOneToMany.java` exists but is an internal CH-witness tool; it does not support `QueryGraph` per the maintainer. You would have to build many-to-many yourself, e.g. bucket-CH/RPHAST on `RoutingCHGraph`, which is what ORS did.

### 5.8 Map matching (separate module `map-matching/`)
`map-matching/src/main/java/com/graphhopper/matching/MapMatching.java` implements an HMM after Newson and Krumm (2009), using Viterbi (`computeViterbiSequence`), `HmmProbabilities.java` (Gaussian emission with σ=10 m, exponential transition with β=2), and `Distributions.java`. Candidates come from `LocationIndexTree`. Transitions are computed by the normal `Router` (flex or LM; CH is disabled in `MapMatchingResource`), and there are u-turn penalties. It is served at `/match`, with a CLI `match` command.

### 5.9 Public transit (`reader-gtfs/`)
- `GtfsReader.java` and `GtfsStorage.java` build a **time-expanded "modulo operating day" graph** (`PtGraph.java`) with ENTER_TEN/LEAVE_TEN/BOARD/TRANSFER… edge types (schema in `reader-gtfs/README.md`, `pt-model.png`). It is connected to the OSM foot network at STOP_ENTER/STOP_EXIT nodes (`PtLocationSnapper.java`). It embeds a fork of Conveyal's GTFS lib (`com/conveyal/gtfs/*`).
- Router 1 is `PtRouterImpl.java` plus **`MultiCriteriaLabelSetting.java`**: multi-criteria label-setting (arrival time, transfers, walking, fares via `fare/*`), with profile queries (time ranges) and walk-only alternatives.
- Router 2 (11.0, #3184, Michael Zilske) is `PtRouterTripBasedImpl.java` plus **`TripBasedRouter.java`** and `Trips.java` (trip transfers). It is **trip-based routing** (Witt 2015 style, rounds = transfers) and "much faster… especially for profile queries" ([11.0 post](https://www.graphhopper.com/blog/2025/10/14/graphhopper-routing-engine-11-0-released/)).
- `PtRouterFreeWalkImpl.java` handles free walking between stops. GTFS-RT is supported via `RealtimeFeed.java`, plus fares and `/isochrone-pt`. The maintainers say it handles national networks on a laptop and continental ones on bigger servers.

## 6. Profiles and custom models

- A **profile** is `name` + `custom_model_files` / `custom_model` + optional `turn_costs` (`vehicle_types`, `u_turn_costs`) (`docs/core/profiles.md`). Built-in models are in `core/src/main/resources/com/graphhopper/custom_models/`: `car.json`, `bike.json`, `bike_elevation.json`, `bike_tc.json`, `mtb.json`, `racingbike.json`, `foot.json`, `hike.json`, `truck.json`, `bus.json`, `motorcycle.json`, `car4wd.json`, `cargo_bike.json`, `curvature.json`, `*_avoid_private*.json`, etc.
- **Weight formula** (`docs/core/custom-models.md`, `CustomWeighting.java`):
  `edge_weight = distance / (speed * priority) + distance * distance_influence + turn_penalty`.
  `speed` changes both time and weight. `priority` (0..1) changes only the weight. `distance_influence` is in seconds per km.
- **Statements**: ordered `if` / `else_if` / `else` blocks with `multiply_by` or `limit_to` over EVs, e.g. `car.json`:
  ```json
  {"distance_influence": 90,
   "priority": [{"if": "!car_access", "multiply_by": "0"}],
   "speed": [{"if": "road_environment == FERRY", "limit_to": "ferry_speed"},
             {"else": "", "limit_to": "car_average_speed"},
             {"if": "true", "limit_to": "max_speed * 0.9"}]}
  ```
  Special variables: `backward_*` (reverse direction), `in_<area>` (GeoJSON `areas` inside the model; replaces the removed `block_area` parameter, see `Router.java` L173), `prev_*` and `change_angle` (in `turn_penalty` only), `country.isRightHandTraffic()`, `edge.getDistance()`.
- **12.0 additions**: a `parameters` section (named numbers and booleans, referenced as `p_xy`, defined server-side with optional ranges). Requests can override values **without recompiling** (see `bike.json`: `"parameters": {"max_mtb_rating": 2, ...}`). The operator whitelist is tightened (`==, !=, <, >, <=, >=, &&, ||, + - * / %`).
- **Compilation** (`routing/weighting/custom/CustomModelParser.java`, 819 lines): the expressions are parsed with Janino's parser and validated by `ConditionalExpressionVisitor` / `ValueExpressionVisitor` (an identifier whitelist against EV names; guards against code injection). A Java source subclass of `CustomWeightingHelper` with `getSpeed` / `getPriority` / `getTurnPenalty` methods is generated, compiled with `SimpleCompiler`, and cached in an LRU (`graphhopper.custom_weighting.cache_size`, default 1000). The code comment says compilation costs 10–40 ms without the cache, which makes LM requests about 20% slower on average. `FindMinMax.java` computes value ranges, for LM admissibility and for max-speed bounds used by A\*.
- **What can change per request**:

  | mode | per-request custom_model | heading | pass_through | alt routes | u-turn costs / curbside |
  |---|---|---|---|---|---|
  | CH (speed) | ✗ (weighting frozen) | ✗ | ✗ | ✓ (CH alt algo) | fixed at prep |
  | LM (hybrid) | ✓ but only "more expensive" (`multiply_by` ∈ [0,1], distance_influence ≥ base) | ✓ | ✓ | ✓ | ✓ (edge-based profile) |
  | flex | ✓ anything | ✓ | ✓ | ✓ | ✓ |

  Per-request models are **merged**: query statements are appended to the server model, and `distance_influence` is overridden if given.

## 7. Features

- **Snapping / QueryGraph**: `storage/index/LocationIndexTree.java` (+ `LineIntIndex`, `PixelGridTraversal`, `InMemConstructionIndex`) is a grid/tree index over edges that returns `Snap` (`TOWER`, `PILLAR` or `EDGE` snaps). `routing/querygraph/QueryGraph.java`, `QueryOverlayBuilder.java`, `VirtualEdgeIterator*.java` create per-request **virtual nodes and edges** that split snapped edges. This is a lightweight overlay, so concurrent queries don't interfere, and it is tested with thousands of points (map matching). Snap filters: `DefaultSnapFilter`, `SnapPreventionEdgeFilter` (`snap_preventions: [tunnel, bridge, ferry]` default in config), `NameSimilarityEdgeFilter` (`point_hints`), `HeadingEdgeFilter`.
- **Heading**: `HeadingResolver.java` and `DirectionResolver.java` (`docs/core/heading.md`) penalise edges whose direction differs from the given azimuth, in flex and LM modes only. **Curbside** (`CurbsideAutoHelper.java`) is for edge-based profiles. `pass_through` avoids u-turns at via points.
- **Via routing**: `ViaRouting.java` and `Router.routeVia`, with path merging into `ResponsePath`.
- **Instructions**: `InstructionsFromEdges.java`, `InstructionsHelper.java`, `InstructionsOutgoingEdges.java` (turn sign classification from angles and outgoing alternatives, roundabouts with optional exit splitting in 12, `instructions_base_mode` car/bike/foot in 12). Translations: 51 locale files in `core/src/main/resources/com/graphhopper/util/*.txt`, via Weblate-like contributions (`docs/core/translations.md`). Voice instructions come with `/navigate` (`navigation/.../VoiceInstructionConfig.java`).
- **Path details** (`util/details/`): per-segment intervals of any EV (`EnumDetails`, `DecimalDetails`, `BooleanDetails`, `IntDetails`, `KVStringDetails` for street_name/ref), plus `TimeDetails`, `DistanceDetails`, `WeightDetails`, `AverageSpeedDetails`, `EdgeIdDetails`, `EdgeKeyDetails`, `IntersectionDetails`, `ChangeAngleDetails`. This is very handy for clients (surface and road-class bars, etc.).
- **Block areas**: done via custom-model `areas` + `in_area` → `multiply_by: 0`. In CH mode that requires a prepared profile with the area baked in. Flex or LM is needed for ad-hoc areas.
- **Conditional/temporal access**: `car_temporal_access` etc. EVs, with `reader/osm/conditional/*` parsers; used in custom models.
- **Output**: JSON with encoded polylines (optionally with elevation), GPX, and Ramer–Douglas–Peucker simplification (`util/DouglasPeucker`), `points_encoded`, `instructions`, `details`, `bbox`.
- **MVT** (`/mvt`) exposes graph vector tiles with EVs, for debugging and visualisation.

## 8. Performance and memory

Benchmarks with numbers (GraphHopper's own; there are no independent rigorous comparisons):
- **2017 blog "flexible routing 15 times faster"** ([link](https://www.graphhopper.com/blog/2017/08/14/flexible-routing-15-times-faster/)), Germany: 8.6 M nodes, 10.9 M edges.
  - Flex: uni-Dijkstra about 500 k visited nodes on about a 130 km route, bi-Dijkstra about 220 k, uni-A\* about 50 k, bi-A\* about 25 k. Cross-country bi-A\* averages about **1.2 s**.
  - LM, 8 active landmarks, cross-country average: 16 landmarks → **72 ms** (17×), 32 → **46 ms** (26×), 64 → **33 ms** (36×). Storage: about 550 MB for 16 landmarks in Germany, about 5 GB per profile worldwide.
  - CH: about 600 visited nodes, "about 10× faster than LM", and **under 100 ms for continent-length routes including instructions and geometry**. (Unverified rule of thumb, not from the sources: pure node-based CH searches on country-sized graphs are in the low single-digit ms; most of the <100 ms is path unpacking, instructions and serialisation.)
- Planet preparation costs: see §3.6 (car edge-based CH about 25 h and about 120 GB; LM about 4 h and about 80 GB).
- Continuous benchmarking: `benchmark/benchmark.sh` runs `tools/.../Measurement.java` (node/edge CH, LM with 4/8/12 active landmarks, flex, map matching) per commit, and `post_benchmark.sh` posts the results. Other tools: `tools/.../CHMeasurement.java`, `GraphSpeedMeasurement.java`, `DataAccessBenchmark*.java`.
- Recent perf work: Hilbert-curve graph sorting (11.0), FFM-based and single-segment DataAccess types (12.0), integer weights (12.0), exact mm distances.
- Memory characteristics: a compact base graph (tower nodes only; edge row about 29 bytes + flags), and per-profile overlays (CH shortcuts are typically about 0.3–1× the edge count, more for edge-based; LM is 4 bytes per node per landmark). Flex-mode queries allocate per-visited-node `SPTEntry` objects on the heap, which is GC-heavy for long queries; hence `max_visited_nodes`.
- Third-party comparisons are mostly anecdotal. The common view is OSRM fastest for raw CH throughput, Valhalla lighter on memory and more dynamic, and GraphHopper in between with the best flexibility/speed trade-off in one engine. ([RouteXL technote](https://www.routexl.com/blog/openstreetmap-router-graphhopper-osrm-gosmore/), [sumguy comparison](https://sumguy.com/osrm-vs-valhalla-vs-graphhopper/))

## 9. Architecture and code quality

Modules (root `pom.xml`): `core`, `reader-gtfs`, `tools`, `map-matching`, `web-bundle`, `web-api`, `web`, `client-hc`, `navigation`, `example`.

- Size: about 98.6 k LOC of main Java, about 63 k in `core`. There are 274 test classes, including heavy randomised and property-style tests for CH and LM (e.g. `EdgeBasedNodeContractorTest`, random graph comparisons of CH against Dijkstra) and many OSM fixture tests. CI runs on GitHub Actions and AppVeyor.
- Style: pragmatic Java, performance-minded (primitive HPPC collections, cursor iterators, no per-edge objects). Some long-lived god classes (`GraphHopper.java` 1.8k lines, `Router.java`), and API churn between majors. The docs in `docs/core/*.md` are good and honest about limitations. Javadoc cites the literature.
- Coupling: algorithms depend on `Graph` / `Weighting` / `EdgeExplorer` interfaces (clean). The import pipeline is tightly bound to OSM and EncodingManager. Custom models are tied to Janino and the JVM.

---

# Part 2 — openrouteservice (ORS)

## Overview
- Maintained by **HeiGIT gGmbH** (Heidelberg Institute for Geoinformation Technology, a spin-off of the Heidelberg University GIScience group). It runs the public API (api.openrouteservice.org / maps.openrouteservice.org) with a planet per profile on 128 GB machines (`docs/run-instance/system-requirements.md`).
- Activity is high: 771 commits in the last year, by Julian Psotta (185), aoles / Andrzej Oleś (224 under both names), takb / Takara Baumbach (145), Jochen Häußler (69), jarinox and others, plus renovate and dependabot. Releases are monthly-ish (v9.4 Sep 2025 … v9.10 Jul 2026, **v10.0.0 16 Sep 2026**, v10.1.0 13 Oct 2026). v10 moved to Java 25, Spring Boot 4 and a distroless slim image, and dropped WAR packaging (`CHANGELOG.md`).
- **Licence**: the repo `LICENSE` is **GPL-3.0** (with `LICENSE.LESSER` alongside), and GitHub reports GPL-3.0. `NOTICE.md` still says "licensed under the Apache license, Version 2.0", which looks stale. Treat it as copyleft (GPL-3.0) unless clarified.
- Stack: Spring Boot app (`ors-api/`, controllers in `ors-api/src/main/java/org/heigit/ors/api/controllers/`: `RoutingAPI`, `IsochronesAPI`, `MatrixAPI`, `SnappingAPI`, `ExportAPI`, `MatchingAPI`, `HealthAPI`, `StatusAPI`). The engine is in `ors-engine/` (443 files, about 49 k LOC). There are also `ors-benchmark/`, `ors-test-scenarios/` and a docs site (`docs/`, VitePress).

## How it extends GraphHopper
- **Dependency**: `pom.xml` → `com.github.GIScience.graphhopper:graphhopper-core|web-api|reader-gtfs|map-matching` at `${graphhopper.version} = v4.16.0`.
- **The fork** (`~/g/graphhopper-ors-fork`, tag `v4.16.0`, internal version `4.16-SNAPSHOT`): its `CHANGELOG.md` top entry is "### 5.0 [not yet released]", so it is **upstream GH 4.0 (Sep 2021) plus cherry-picked commits toward 5.0**. In April 2026 the team was still merging upstream PRs #2455–#2458 (from about Nov 2021) in branch `chore/merge_gh_5`. There are **709 `ORS-GH MOD` markers in 134 files**, touching the core algorithms (`Dijkstra`, `AStar`, `AbstractBidirAlgo`, `AbstractBidirCHAlgo`, `Router`, `ViaRouting`, `CHPathCalculator`, `PathExtractor`, `InstructionsFromEdges`, `AlternativeRoute`, `LMApproximator`, `CHPreparationGraph`, `CHPreparationHandler`), the OSM reader (`OSMReader`, `WaySegmentParser`, `OSMNodeData` — node-ID mapping for tower nodes added in 2026), `EncodingManager`, `GraphHopper.java`, and a custom `ExtendedGHRequest`. The fork itself was moved to Java 25 and Jackson 3 in Aug 2026.
- **Coupling is very tight**: ORS subclasses GH internals (`ORSGraphHopper extends GraphHopperGtfs` in `ors-engine/.../routing/graphhopper/extensions/ORSGraphHopper.java`; `ORSRouter`, `ORSOSMReader`, `ORSGraphHopperStorage`, `ORSWeightingFactory`, `ORSEdgeFilterFactory`, `OrsEncodedValueFactory`, `OrsTagParserFactory`). It also adds classes in GH's own package (`ors-engine/src/main/java/com/graphhopper/routing/ev/`). Upgrading to modern GH (12.x) would be a rewrite of most of the GH-facing layer, and the 2024–2026 changelog shows them gradually converging ("migrate X storage to encoded values" in 9.4–10.1, "full support for encoded values" in 9.4).
- Still pre-GH-5 concepts: **FlagEncoders** (`extensions/flagencoders/`: `CarFlagEncoder`, `HeavyVehicleFlagEncoder`, `EmergencyFlagEncoder`, `FootFlagEncoder`, `HikingFlagEncoder`, `PedestrianFlagEncoder`, `WheelchairFlagEncoder`, `bike/*`), and **GraphExtensions / extra storages** (`extensions/storages/`: `GreenIndexGraphStorage`, `NoiseIndexGraphStorage`, `ShadowIndexGraphStorage`, `TrafficGraphStorage`, `CsvGraphStorage`, `WarningGraphExtension`). Many former storages (WayCategory, WaySurface, HeavyVehicle, Tollways, HillIndex, TrailDifficulty, RoadAccessRestrictions, countries/borders, wheelchair) were migrated to EVs during 2025–2026.

## Extra features over GH
- **Profiles** (`ors-config.yml`): `driving-car`, `driving-hgv` (HGV dimensions, weight, hazmat restrictions via the HeavyVehicle EVs), `cycling-regular|road|mountain|electric`, `foot-walking|hiking`, **`wheelchair`** (kerb heights, incline, surface, width: `WheelchairAttributes.java`, `WheelchairTypesEncoder.java`), and `public-transport` (GH's reader-gtfs).
- **Weightings** (`extensions/weighting/`): `GreenWeighting`, `QuietWeighting` (noise), `ShadowWeighting`, `HeatStressWeighting`, `SteepnessDifficultyWeighting`, `AvoidHillsWeighting`, `FastestSafeWeighting`, `HgvAccessWeighting`, `LimitedAccessWeighting`, `PreferencePriorityWeighting`. GH custom models have been supported for directions since 9.0 (#1950), with ALT for custom models since 9.5.1.
- **"Core" routing** (`extensions/core/`): `PrepareCore extends PrepareContractionHierarchies` contracts everything *except* nodes touched by "restricted" edges (filters for highways, tollways, borders, HGV restrictions, …; up to 99% of nodes are contracted). Those stay as an uncontracted **core** at the top of the hierarchy. Queries use CH outside the core and **ALT/Dijkstra inside the core** (`CoreALT.java`, `CoreDijkstra.java`, `CoreLandmarkStorage.java` with `lmsets`), so **avoid features, avoid countries/borders, avoid polygons and HGV parameters** can be applied per request at near-CH speed. Time-dependent variants `TDCoreALT.java` and `TDCoreDijkstra.java` handle traffic (`routing/traffic/TrafficSpeedCalculator.java`, `TrafficGraphStorage`). This is the most interesting algorithmic addition, essentially a CH/ALT hybrid with a dynamic core (cf. core-ALT from Bauer, Delling et al., "Combining hierarchical and goal-directed speed-up techniques", 2008).
- **Matrix** (`ors-engine/.../matrix/algorithms/`): `rphast/RPHASTMatrixAlgorithm.java` (CH-based RPHAST one-to-many/many-to-many), `core/CoreMatrixAlgorithm.java` (matrix with dynamic restrictions on the core graph), and `dijkstra/DijkstraManyToMany.java` / `DijkstraMatrixAlgorithm.java` for flex profiles. It supports up to thousands of locations depending on config. **This is the open-source matrix that GH lacks**, but it lives on the GH 4 fork.
- **Isochrones** (`isochrones/`, `fastisochrones/`): the default `concaveballs/ConcaveBallsIsochroneMapBuilder.java` (concave hull of SPT points), and **fast isochrones**: preprocessing partitions the graph with **Inertial Flow** (`fastisochrones/partitioning/InertialFlow.java`, `EdmondsKarpAStar.java`, `MaxFlowMinCut.java`) into cells with border nodes, precomputes per-cell eccentricities (`Eccentricity.java`), and queries with `RangeDijkstra` / `ActiveCellDijkstra` / `CoreRangeDijkstra`. The idea follows Baum et al. ("Fast exact computation of isochrones in road networks"). Isochrone **statistics** (population via PostgreSQL, `isochrones/statistics/postgresql/`) are included.
- **Snapping**, **export** (base graph as GeoJSON/TopoJSON), **matching** (map features to edges for dynamic data; `matching/`, 9.4+). A **dynamic data service**: a PostgreSQL-backed feature store with scheduled sync, used to update edge attributes at runtime (9.6.0, #2156). **Extra info** in responses (surface, waytype, steepness, tollways, green, noise, …), like GH path details.
- **Optimization (VROOM)**: *not in this repo*. HeiGIT hosts VROOM separately behind the public API, and VROOM calls ORS matrix (README).
- Ops: a **graph repo client** (download prebuilt graphs from S3/HTTP repositories; `docs/technical-details/graph-repo-client/`), `preparation_mode` (build and pack graphs as `.ghz`), Docker and Kubernetes-ready images, and Prometheus-style status.
- Requirements: Germany about 8 GB, planet bike/foot about 100–105 GB heap per profile (`system-requirements.md`).

## Status assessment
Healthy and very active as a *service*. It is architecturally stuck on a 2021 GH base and is slowly converging toward upstream concepts, while upstream GH has moved through 8 majors (edge-based CH improvements, custom-model-only weighting, turn penalties, trip-based PT, FFM storage). Code quality is mixed: extensive features, with a fair amount of legacy, duplication (`TODO Refactoring` comments in `ORSGraphHopper.java`) and config sprawl.

---

# Part 3 — BRouter

## Overview
- Author **Arndt Brenschede** (`abrensch`), 2013+. The licence is now **MIT** (`LICENSE`, "Copyright (c) 2019 BRouter contributors"). Today the most active maintainer is **afischerdev** (124 of 189 commits in the last year), with Totorrr, Johannes Buehler, Emux and others contributing. Releases: 1.7.0 Apr 2023 … 1.7.8 Jul 2025, 1.7.9 Apr 2026, **1.7.10 Jul 2026** (`docs/revisions.md`). The pace is slow but alive, focused on the Android app, profiles and quality-of-life engine features (round trips, dynamic waypoint catching range, via-point correction, motorway-exit voice hints, pseudo tags from a database).
- About 32 k LOC of main Java. There are 23 test classes. The Gradle build has modules `brouter-util`, `brouter-codec`, `brouter-expressions`, `brouter-mapaccess`, `brouter-core`, `brouter-map-creator`, `brouter-server` and `brouter-routing-app` (Android; `settings.gradle` includes it only when `local.properties` exists).
- Ecosystem: the **brouter-web** frontend (separate repo; the de-facto bike planner for many OSM cyclists), brouter.de public server, weekly planet segments, F-Droid and Play Store app, and integration in **Locus Map** (most featureful), **OsmAnd**, OruxMaps and others (README).

## Data format: rd5 segments
- The planet is cut into **5°×5° files** named by their SW corner (`E5_N45.rd5`, `W50_N35.rd5`), downloaded from `brouter.de/brouter/segments4/` and rebuilt weekly. Some sizes (web, 2026-10-23): `E5_N45.rd5` about 241 MiB, `E10_N50.rd5` about 119 MiB. The planet total is on the order of 10+ GB. There are also `diff/` files for incremental updates (`mapaccess/Rd5DiffManager.java`, `Rd5DiffTool.java`).
- Structure (`brouter-mapaccess/src/main/java/btools/mapaccess/PhysicalFile.java`): a header with a version in the top 16 bits of the first long, a `fileIndex[25]` → **25 1°×1° sub-files**, header CRCs, and `divisor = 80`. Each degree is split into **80×80 "micro caches"** of 1/80° (about 1.4 km). Each micro cache is an independently decodable, CRC-checked, **entropy-coded bitstream** (`brouter-codec/.../MicroCache2.java`, `StatCoderContext.java`, `TagValueCoder.java` (Huffman-like coding of tag-value bitsets), `NoisyDiffCoder.java` (delta coordinates), `IntegerFifo3Pass.java`; 3-pass encoding: counters, statistics, encoding).
- Per node: position, **elevation** (SRTM 90 m / CGIAR plus Sonny LiDAR above 60°N, embedded at map-creation time; `elevationType`), node tag bitset, links with way-tag bitsets and geometry (forward links only, reverse links reconstructed), and **turn restrictions** (from/to as coordinates, plus exceptions for bikes, PSV and so on).
- Tags are not stored as strings. They are stored as indices into the **lookup table `misc/profiles2/lookups.dat`** (tag → value list with aliases), so the data format and profiles share a vocabulary, and adding a tag requires a new lookup version and a re-export.
- `DirectWeaver.java` decodes directly into the in-memory object graph (`OsmNode`, `OsmLink`, `OsmTransferNode`), bypassing the intermediate cache. `NodesCache.java` lazily loads tiles on demand with a memory budget (`maxmem` derived from `RoutingContext.memoryclass`, on Android from `ActivityManager.getMemoryClass()`) and evicts and re-weaves when needed. Only tiles the search touches are decoded, which is why it works on phones with small heaps.
- **Map creation** (`brouter-map-creator/`): `OsmCutter`/`OsmFastCutter` → `NodeCutter`/`WayCutter5`/`RestrictionCutter5` → `PosUnifier` (elevation from `.bef` rasters made by `ElevationRasterTileConverter`) → `WayLinker` (encodes rd5). Pipeline scripts are in `misc/scripts/mapcreation/process_pbf_planet.sh`. **Pseudo-tags** (`noise_class`, `river_class`, `forest_class`, `town_class`, `traffic_class`) are computed by spatial SQL in PostGIS (`brouter.sql`, `docs/developers/environmental_considerations_and_pseudo_tags.md`: 64 m road buffers and the share of the buffer near noise sources, in forest, etc.).

## Algorithm
`docs/developers/algorithm.md` plus `brouter-core/src/main/java/btools/router/RoutingEngine.java` (2489 lines):
- **No CH, no landmarks, no per-profile preprocessing.** Cost function: `g(x) + c·h(x)`, where `h` is the air distance scaled by the profile's minimum cost factor.
- **2-pass search**: pass 1 uses `c = pass1coefficient` (e.g. 1.5, an inadmissible, greedy, roughly linear-time A\*) to quickly find a *maximum cost estimate*. Pass 2 uses `c = pass2coefficient` (0 = Dijkstra) and **prunes any path whose cost plus remaining air-distance bound exceeds the estimate** (an adaptive cost cutoff that is lowered on the fly when pass 2 meets pass-1 nodes with a cheaper combined cost). Dijkstra is chosen over A\* in pass 2 because the open set stays smaller under the cutoff. Both coefficients are profile variables; a negative value disables a pass (`RoutingEngine.java` ≈ L1756–1800, `airDistanceCostFactors`).
- **Recalculation**: off-route recalcs skip pass 1 by matching against the previous track (`guideTrack`, `matchPath`, `mergeTrack`). On timeout it merges a "dirty" track from the old and new results.
- **Edge-based search**: search "nodes" are OSM *links*, and transitions at OSM nodes are the Dijkstra edges, so **turn costs** (`turncost · cos(angle)`), `initialcost` on classifier changes, and turn restrictions are exact (`OsmPath.java`, `StdPath.java`).
- **Elevation hysteresis**: an elevation buffer (`elevationpenaltybuffer`, `elevationmaxbuffer`, `elevationbufferreduce`, `up/downhillcutoff`) filters SRTM noise. This makes path cost history-dependent, so it is not strictly Dijkstra-correct; this is accepted ("don't care" plus a one-step delayed node decision).
- **Kinematic model** (`KinematicModel.java`, `KinematicPath.java`; used by `car-vario.brf`): energy- and physics-based car or bike travel time and cost (mass, drag `S_C_x`, rolling resistance `C_r`, power, braking, curves). The trekking profile's kinematic parameters (`totalMass`, `bikerPower`, `S_C_x`, `C_r`) drive realistic ETAs.
- Open set: `btools/util/SortedHeap.java`. Expression results are cached per tag bitstream (below). The scaling caveat (`docs/features/offline.md`): "processing time that scales quadratic with distance, leading to a limit at about 150 km in air-distance". This is an older statement; the server and newer hardware do more, but long car routes are not its domain.
- Other modes: **alternatives** (`alternativeidx` 0–3 via penalising the previous solution), **round trip** (`engineMode=4`), elevation lookup (`engineMode=2`), segment info (`engineMode=3`), nogo circles/polylines/polygons with optional weights, beeline segments (`d`), dynamic waypoint catching range up to 60 km (`MAX_DYNAMIC_RANGE`), `correctMisplacedViaPoints`.
- Voice hints and turn instructions: `VoiceHint*.java` produces several dialects (`turnInstructionMode`: Locus, OsmAnd, Orux, gpsies, comment). Output formats: GPX, KML, GeoJSON, CSV (`FormatGpx/Kml/Json/Csv.java`).

## Profile scripting language (`.brf`)
- Spec: `docs/developers/profile_developers_guide.md`. Examples are in `misc/profiles2/`: `trekking.brf` (reference), `fastbike.brf`, `mtb.brf`, `gravel.brf`, `hiking-mountain.brf`, `car-vario.brf`, `moped.brf`, `shortest.brf`, `river.brf`, `rail.brf`, `skating.brf`, `vm-forum-velomobil-schnell.brf`, …
- Three contexts: `---context:global` (engine parameters: elevation costs, pass coefficients, turn instruction mode, kinematic parameters, user-tweakable switches annotated `# %name% | description | type` that frontends turn into option dialogs), `---context:way` (must assign `costfactor`, `turncost`, `initialcost`, `uphillcostfactor`, `downhillcostfactor`, `initialclassifier`, `priorityclassifier`, `nodeaccessgranted`), and `---context:node` (`initialcost`; can read way variables via `way:` prefix).
- Syntax: `assign var = expr`, **prefix/Polish notation** for operators (`or a b`, `and …`, `not`, `add`, `multiply`, `max`, `min`, `greater`, `lesser`, `equal`, `xor`, `sub`, `div`), `if … then … else …`, and lookup matches `highway=primary|secondary`, with an empty value meaning "tag absent" (`surface=`).
- Implementation: `brouter-expressions/src/main/java/btools/expressions/BExpression.java` (parser and tree interpreter), `BExpressionContext.java`, `BExpressionContextWay.java`, `BExpressionContextNode.java`. Evaluation is **cached per distinct tag bitstream**: CRC32 of the encoded way/node tags → `LruMap` of result vectors (`BExpressionContext` L63–123, L341–400). Since real data has a few thousand distinct tag combinations per area, the interpreter cost is amortised almost entirely away. This is a neat trick worth porting. There is also unused-tag elimination (`processUnusedTags=false`) to increase cache hits, `IntegrityCheckProfile.java` and `ProfileComparator.java`.
- Profiles can be **uploaded per request** (`remoteProfile`, `ProfileUploadHandler.java`). Parameters can be overridden per request (`profile:avoidsteps=0` on the server, `extraParams` on Android). Everything is "flexible mode" by construction.

## Android: offline service consumed by other apps
- `brouter-routing-app/src/main/aidl/btools/routingapp/IBRouterService.aidl` exposes **one method**: `String getTrackFromParams(in Bundle params)`. The params include `lonlats`, `profile` / `v`+`fast` / `remoteProfile`, `nogos`/`polylines`/`polygons`, `alternativeidx`, `timode` (instruction dialect), `heading`, `direction`, `engineMode`, `roundTripDistance`/`roundTripPoints`, `extraParams`, `maxRunningTime` (default 60 s), `trackFormat` (gpx/kml/json, optionally compressed), and `pathToFileResult`. Since Android Q the track is returned as a string because of scoped storage.
- `BRouterService.java` binds the AIDL stub and `BRouterWorker.java` runs `RoutingEngine.doRun(maxRunningTime)`. Memory is sized from the device's `getMemoryClass()` (`BRouterActivity.java` L124, `BRouterView.java` L557). `serviceconfig.dat` maps `v`/`fast` to profile names. Client apps (Locus, OsmAnd via its "BRouter (offline)" engine, Orux) bind to the service, and users download segments in BRouter's own installer UI (`BInstallerActivity`, `DownloadWorker`). Docs: `docs/developers/android_service.md`, `docs/users/osmand.md`.
- This "routing engine as a separate app/service plus a shared tile data directory" pattern is a good architecture for offline mobile routing. It decouples map UI apps from the router.

## Strengths for bike routing
Elevation-aware with sensible noise filtering; arbitrarily expressive, per-user profiles with no re-preprocessing; turn costs and restrictions; route-network (icn/ncn/rcn/lcn) magnetism; environmental pseudo-tags (quiet, green, river routes); kinematic ETA; tiny footprint (tile-wise lazy decoding; works on decade-old phones); weekly global data with diffs; and a strong community of profile authors (brouter-web, velomobile forums).

Weaknesses: no speed-up technique, so long-distance car routing is slow (quadratic search growth). The heuristic first pass and hysteresis mean results aren't strictly optimal. The tag vocabulary is fixed by `lookups.dat`, so new tags need a format bump and new segments. The custom binary format means you depend on brouter.de or your own PostGIS + map-creator pipeline. There is no matrix, isochrone or transit support. The code is dense and lightly documented (single-letter fields, few tests), and maintenance depends on very few people.

---

# Part 4 — Comparison and suitability

| | GraphHopper | openrouteservice | BRouter |
|---|---|---|---|
| Licence | Apache-2.0 | GPL-3.0 (LICENSE; NOTICE says Apache, stale) | MIT |
| Steward | GraphHopper GmbH (company) | HeiGIT gGmbH (research non-profit) | individuals |
| Activity (last 12 mo) | 316 commits, about 4 core devs | 771 commits, about 6–8 devs | 189 commits, about 1–2 core devs |
| Speed-ups | CH (node and edge), LM/ALT | CH, LM, core-ALT, RPHAST, fast-isochrone partitions | none (2-pass A\*/Dijkstra with cutoff) |
| Per-request flexibility | flex: full; LM: constrained; CH: none | avoid features via core; custom models (flex/ALT) | full (profile scripts, uploadable) |
| Turn restrictions | yes (edge-based, also in CH) | yes | yes (edge-based by design) |
| Matrix | ✗ (commercial) | ✓ | ✗ |
| Isochrones | ✓ (flex) | ✓ (+fast) | ✗ |
| Map matching | ✓ (HMM) | ✓ (via GH fork + feature matching) | ✗ |
| Transit | ✓ GTFS (MCLS + trip-based) | ✓ (GH 4 reader-gtfs) | ✗ |
| Offline mobile | dropped (Janino/FFM/Java 25) | no | **yes**, primary target |
| Data format | monolithic graph folder of DataAccess files | same (GH 4) | 5°×5° tiles, entropy-coded |
| Planet footprint | about 60 GB import; CH car+TC about 120 GB | about 100–128 GB per profile | about 10+ GB of tiles on disk; RAM tiny (on demand) |

## GraphHopper
**Strengths**: the most complete open feature set in one JVM engine. Edge-based CH with a well-documented design. Three modes switchable per request. The custom-model language is the best-designed "user-editable cost function" among the major engines (declarative, safe, compiled, LM-compatible with clear rules). A compact, mmap-able storage abstraction. Strong tests and benchmarks. Permissive licence with a company behind it.
**Weaknesses**: matrix is closed. CH is rigid (no per-request changes, no heading). The EV set is fixed at import (profile changes mean a re-import). Monolithic graph with no tiling or partial updates. Large RAM for planet CH with turn costs. JVM-only and now Java 25+, so no viable Android/iOS embedding. API churn between majors.

**(a) As a dependency**: excellent for a JVM server project (Maven `graphhopper-core`, Apache-2.0). For a non-JVM project (e.g. Rust), it would only be usable as a sidecar HTTP service, which is fine for prototyping or as a correctness oracle, but not as an embedded library. Not usable for on-device mobile.
**(b) As a reference to port**: very high value. Worth studying and porting:
- tower/pillar split and the edge-row layout (`BaseGraphNodesAndEdges.java`)
- EV bit-packing and `ImportRegistry` dependency resolution
- two-pass OSM segment parsing with compact node ID maps (`WaySegmentParser`, `OSMNodeData`)
- via-way restrictions by edge cloning
- `TurnCostStorage` keyed by (from, via, to)
- edge-based CH with original-edge keys on shortcuts (`EdgeBasedNodeContractor`, `docs/core/edge-based-ch.md`)
- LM with 16-bit scaled potentials and the admissibility check for per-request models (`FindMinMax.checkLMConstraints`)
- QueryGraph virtual-node overlay, including over CH (`QueryRoutingCHGraph`)
- custom-model semantics (speed vs priority vs distance_influence vs turn_penalty, `parameters`, areas). In Rust these could be compiled to closures or bytecode rather than Janino.
- path details, instruction generation, the HMM map matcher, and the trip-based PT router.

Easbar's Rust crate `fast_paths` (already in `~/g/fast_paths`) is effectively a Rust port of GH's node-based CH ideas.

## openrouteservice
**Strengths**: the richest *open* service feature set (matrix, fast isochrones, avoid-features at CH-like speed via core-ALT, wheelchair, HGV, green/quiet/shadow/heat weightings, dynamic data store, graph repository, Kubernetes images). Very active institutional maintenance with a public API at scale.
**Weaknesses**: built on a 2021 GraphHopper fork with about 700 invasive patches, so it is hard to upgrade and far from upstream improvements. GPL-3.0. Heavy (Spring Boot, GeoTools, PostgreSQL integrations). Memory-hungry (about 100+ GB per planet profile). The codebase mixes legacy FlagEncoders, GraphExtensions and newer EVs.

**(a) As a dependency**: only as a deployed service (Docker). Not as a library: GPL, tight GH-4 coupling, Spring.
**(b) As a reference**: valuable specifically for:
- the **core-ALT idea** (partial CH with an uncontracted restricted core for dynamic avoid-areas and time-dependent costs, `extensions/core/PrepareCore.java`, `CoreALT.java`, `TDCoreALT.java`)
- **RPHAST matrix** on CH (`matrix/algorithms/rphast/`)
- **partition-based fast isochrones** with Inertial Flow (`fastisochrones/`)
- wheelchair and HGV attribute modelling, and environmental weightings.

## BRouter
**Strengths**: purpose-built, high-quality bike (and hiking) routing. Fully per-request programmable profiles. Excellent elevation handling. A truly offline, low-memory, tiled data format with weekly global builds and diffs. An Android AIDL service architecture adopted by major outdoor apps. Small codebase, MIT licence.
**Weaknesses**: no preprocessing, so search effort grows roughly quadratically with distance and it is unsuitable for long car trips, matrices or high QPS. The format is tied to a fixed tag lookup table. Slightly heuristic results. Small maintainer base. Dense code with few tests or docs. No isochrones, matrix or transit.

**(a) As a dependency**: as an Android companion service (users install BRouter, your app calls `IBRouterService`), or as a self-hosted HTTP server (`brouter-server`, `docs/developers/http_server.md`). Embedding the Java library on Android is also feasible (MIT). From Rust: only via HTTP or IPC.
**(b) As a reference**: high value for an offline or mobile-first design. Worth studying:
- the tiled, entropy-coded, CRC-checked segment format with lazy per-micro-tile decoding and a memory budget (`PhysicalFile`, `MicroCache2`, `DirectWeaver`, `NodesCache`)
- the tag-bitstream lookup vocabulary
- **expression result caching keyed by tag-set hash** (makes an interpreted profile language almost free)
- the 2-pass greedy A\* → Dijkstra with adaptive cutoff and recalculation reuse
- edge-based search for exact turn costs without preprocessing
- elevation hysteresis and the kinematic ETA model
- annotated profile parameters (`# %name% | desc | type`) that UIs render automatically.

## Takeaways for a new (likely Rust) engine at `/home/dev/p/routet`
- None of the three is a natural *library* dependency for a Rust project. GraphHopper is the best **oracle and baseline** to run alongside (HTTP) for correctness and performance comparisons. BRouter's server is the best baseline for bike-routing quality.
- Combining **GH's data model and algorithms** (compact graph, EVs, edge-based CH, LM, custom-model semantics, QueryGraph) with **BRouter's tiling and on-device ideas** (tiled compressed segments, lazy decoding, tag-set-hash cost caching, service-style mobile integration) and **ORS's core-ALT / RPHAST / fast-isochrone techniques** covers server speed, per-request flexibility and offline mobile. This mix is what none of these JVM engines offers alone.
