# Valhalla — deep-dive

*Source studied: `~/g/valhalla` at `618cad5` (2026-10-19), plus `~/g/valhalla-rs` (Rust bindings, 2026-10-21) and `~/g/ferrostar`. Paths are relative to the Valhalla repo root unless noted. Researched 2026-10-23.*

---

## TL;DR

Valhalla is a mature, MIT-licensed C++20 routing engine. It runs on OSM data that it cuts into a **fixed 3-level geographic tile hierarchy**. Costs are computed **at query time** by pluggable C++ costing models, so one tileset serves every mode and every request-time tweak. Speed comes from a **mild hierarchy plus shortcuts and heuristic search-pruning**, not from full contraction hierarchies (CH). Expect 10–100× slower point-to-point queries than OSRM on long routes and much slower large matrices. In exchange you get modest build RAM (planet on 16 GB is possible), mmap-friendly zero-copy tiles that work offline and on mobile, partial/regional loading, and a very wide feature set: turn-by-turn in 35 locales, time-dependent and live/historic traffic, isochrones, map matching, TSP, elevation, transit, truck attributes, avoid polygons and per-request cost factors.

For a Rust project there are two takeaways. First, `valhalla-rs` (cxx-based bindings on crates.io) exists and is actively maintained, so using Valhalla as a dependency is realistic. Second, Valhalla's tile format and its costing/guidance design are good references. Its search-speedup approach is the weakest part to copy. A modern CCH/CRP design gives both dynamic costs and much faster queries.

---

## 1. Overview

| Item | Detail |
|---|---|
| Language | C++20 (`CLAUDE.md`: "C++20, CMake, single `libvalhalla` shared library"). ~43k lines of headers in `valhalla/`, ~101k lines of `.cc`/`.h` in `src/`. `src/mjolnir` (the tile builder) is the largest module at ~34k lines. ~264 test files, ~103k lines. |
| License | MIT (`COPYING`: "Copyright (c) 2018 Valhalla contributors, (c) 2015-2017 Mapillary AB, Mapzen"). Test data is ODbL. |
| Origin | Built by the Mapzen routing team in 2015 (`docs/docs/start/history.md`: THOR = "Tiled, Hierarchical Open Routing", ODIN, TYR). Mapzen shut down, Mapbox hired the team in 2018, and the project is now community/organization run under `github.com/valhalla`. |
| Maintainers | Over the past 12 months, 53 distinct authors. Top committers: Christian Beiwinkel (89), Nils Nolde (85), Stefan Kizim (50), Siarhei Fedartsou (20); Kevin Kreiser (original author) still contributes occasionally. `pyproject.toml` lists Nils Nolde and Kevin Kreiser as authors. There is no formal governance document; the maintainers review and merge PRs (`CONTRIBUTING.md`). |
| Activity | 337 commits from 2025-10-19 to 2026-10-19, roughly 12–43 per month, steady. GitHub: ~6.4k stars, ~1k forks, ~945 open issues. |
| Releases | Semver since 3.x. 3.6.0 (2025-10-23), 3.6.1, 3.6.2, 3.6.3 (2026-02), 3.7.0 (2026-04), 3.8.0–3.8.3 (2026-07), 3.9.0 (2026-09-19), 3.9.1 (2026-10-06). That is roughly one minor release per quarter plus patches. `CHANGELOG.md` is detailed. The python wheel is `pyvalhalla` on PyPI. |
| Code quality signals | Codecov reports **85.1% line coverage** (codecov API, 2026-10-23). CI runs on Linux, macOS, Windows and MinGW, plus format, lint, Node and Python wheel jobs (`.github/workflows/`). The repo has an explicit AI-contribution policy (`README.md`) and agent guide files (`CLAUDE.md`, `AGENTS.md`). These guides contain unusually useful planet-scale facts, quoted below. |
| Production users | The **FOSSGIS public server** at valhalla.openstreetmap.de runs a full planet on current master; its `/status` reported `3.9.1-618cad5`. Others: **Stadia Maps** (hosted routing APIs built on Valhalla; it also maintains Ferrostar), **Mapbox** (hired the team in 2018 and used Valhalla in its navigation stack and for offline/in-car; its current internal status is unknown), and reportedly BMW in-car navigation and Hammerhead (second-hand blog claims). openstreetmap.org offers Valhalla as one of its routing engines. Mobile: `Rallista/valhalla-mobile` (Android AAR on Maven Central and a Swift package; offline route and trace actions). |

## 2. Deployment model

Valhalla is a **library first, with servers wrapped around it**:

- **Embedded C++ library.** `libvalhalla`, with a pkg-config file (`libvalhalla.pc.in`). The main entry point is `valhalla::tyr::actor_t` (`valhalla/tyr/actor.h`). It is built from a `boost::property_tree` config, optionally sharing a `GraphReader`, and exposes `route()`, `locate()`, `matrix()`, `optimized_route()`, `isochrone()`, `trace_route()`, `trace_attributes()`, `height()`, `transit_available()`, `expansion()`, `centroid()`, `status()` and `tile()`. Each takes a JSON string or a protobuf `Api` object and returns JSON/PBF. There is also `act(Api&)` for pure protobuf in and out. An optional `interrupt` functor makes requests cancellable.
- **HTTP server.** `valhalla_service` (`src/valhalla_service.cc`) uses Kevin Kreiser's `prime_server` (ZeroMQ). It runs either multi-threaded in one process or as separate `valhalla_loki_worker`, `valhalla_thor_worker` and `valhalla_odin_worker` processes chained over ZMQ (`src/valhalla_*_worker.cc`). The services build is optional (`ENABLE_SERVICES`; the vcpkg feature `services` notes it is "only needed for the standalone valhalla_service binary").
- **Bindings in-tree**
  - **Python**: `src/bindings/python` (nanobind). This is `pyvalhalla` on PyPI. It bundles the CLI tools (`valhalla_build_tiles`, etc.) as console scripts (`pyproject.toml`) and has graph-utility bindings (`src/bindings/python/src/baldr/graph_id.cc`, `graph_tile_header.cc`).
  - **Node.js**: `src/bindings/nodejs` (N-API addon with TypeScript `index.d.ts`), off by default (`ENABLE_NODE_BINDINGS OFF`), with its own CI workflow.
- **Bindings out of tree**
  - **Rust: `kinkard/valhalla-rs`** (`~/g/valhalla-rs`, crate `valhalla` 0.7.2, edition 2024, cxx-based). It vendors Valhalla as a submodule and builds it via cmake from `build.rs` with `ENABLE_DATA_TOOLS/SERVICES/HTTP/PYTHON=OFF`. It exposes `Actor` (protobuf `Options` in, via prost types), a typed `ConfigBuilder` (no Python or JSON config needed), read-only tile access (`DirectedEdge`, `EdgeInfo`, `NodeInfo`), and **read/write of memory-mapped live-traffic tiles**. It has criterion benches (`benches/actor_bench.rs`). Limitations from its README: `tile_extract` (tar) only, no `tile_dir`, and no logging redirection. It is about 2.4k lines of Rust/C++ glue.
  - **Android/iOS: `Rallista/valhalla-mobile`.** Static iOS / shared Android builds; exposes route, trace_route and trace_attributes against on-device tiles.
  - **Ferrostar** (`~/g/ferrostar/common/ferrostar/src/routing_adapters/valhalla.rs`) is a navigation SDK that only builds HTTP requests for Valhalla. It also parses Valhalla's OSRM-compatible output. On-device routing would go through its `CustomRouteProvider`.
- **Offline/on-device** is a core design goal (`README.md`: "small memory footprint on memory constrained devices, enable offline routing, provide a means for regional extracts and partial updates"). Tiles can come from:
  1. a directory of tile files (`tile_dir`), optionally gzipped;
  2. a single **mmapped tar** (`tile_extract`, `src/baldr/graphreader.cc` `tile_extract_t`);
  3. **HTTP** `tile_url` with an on-disk cache, including a *remote tar with `index.bin` read via HTTP range requests* (`GraphReader::load_remote_tar_offsets`).

  `src/baldr/CMakeLists.txt` has iOS-specific tzdb handling (`third_party/date/src/ios.mm`), and the CHANGELOG mentions Android logging.
- **Thread model.** `GraphReader` and most tile caches are *not* thread-safe (`valhalla/baldr/graphreader.h` comments). Use one reader/actor per thread, optionally sharing a `SynchronizedTileCache`. With `ENABLE_THREAD_SAFE_TILE_REF_COUNT`, tile pointers become `shared_ptr`. valhalla-rs builds with it OFF and treats `GraphReader` as stateless/shared.

## 3. Data pipeline

### Inputs
| Input | Tool / code | Notes |
|---|---|---|
| OSM PBF (one or more files) | `valhalla_build_tiles` → `src/mjolnir/pbfgraphparser.cc` (libosmium + protozero) | Tag → attribute mapping is in **Lua** (`lua/graph.lua`, run by `src/mjolnir/luatagtransform.cc`), so tag-handling changes need no recompile (`docs/docs/contributing/architecture/mjolnir/tag-parsing.md`). |
| Admin boundaries | `valhalla_build_admins` (`src/mjolnir/valhalla_build_admins.cc`, `adminbuilder.cc`, `lua/admin.lua`) → SpatiaLite `admins.sqlite` | Used for country/state codes, driving side, country-specific default access (`countryaccess.cc`), and border-crossing costs. |
| Timezones | `scripts/valhalla_build_timezones` downloads timezone-boundary-builder shapefiles (release 2026e) → `tz.sqlite` | The tz database itself is vendored (`third_party/tz`, `date`), with a CI job to track tz releases. |
| Elevation | `scripts/valhalla_build_elevation` fetches 1-arc-second SRTM-style `.hgt` tiles from `elevation-tiles-prod` on AWS; `src/skadi/sample.cc` reads raw, gz or lz4 HGT (3601×3601 int16). | Applied at build time (`elevationbuilder.cc`, stage `elevation`) to fill per-edge `weighted_grade`, `max_up/down_slope` and encoded EdgeInfo elevation. Also queried at runtime by the `/height` endpoint. |
| Historic traffic | `valhalla_add_predicted_traffic` (`src/mjolnir/add_predicted_speeds.cc`) from per-tile CSVs | Each edge gets a week of 5-minute buckets (2016), **compressed with a DCT-II to 200 int16 coefficients** (`valhalla/baldr/predictedspeeds.h`, `src/baldr/predictedspeeds.cc`), plus free-flow and constrained-flow speeds stored inline on `DirectedEdge`. |
| Live traffic | separate `traffic.tar` with the same tile layout; `valhalla/baldr/traffictile.h` | One 8-byte `TrafficSpeed` per directed edge: overall speed plus 3 sub-segment speeds with breakpoints (7 bits, 2 kph resolution), congestion values and an incident flag. It is **memory-mapped writable and updated in place** by an external process. The header is C-only-compatible for bindings. Incidents use separate incident tiles (`docs/docs/concepts/incidents.md`). |
| Transit GTFS | `valhalla_ingest_transit` (`src/mjolnir/ingest_transit.cc` using `third_party/just_gtfs`) → protobuf intermediates → `valhalla_convert_transit` (`convert_transit.cc`) → transit level-3 tiles | Stops, routes, departures and schedules are stored in graph tiles (`TransitDeparture`, `TransitStop`, …). |
| Landmarks | `valhalla_build_landmarks` / `valhalla_add_landmarks` | Used in guidance ("turn left after X"). |

### Build stages
`BuildStage` in `valhalla/mjolnir/util.h`, orchestrated by `build_tile_set()` in `src/mjolnir/util.cc`. You can start or stop at any stage:

`initialize → parseways → parserelations → parseareaways → parsenodes → buildareas → constructedges → build → enhance → filter → transit → bss → hierarchy → shortcuts → restrictions → elevation → validate → cleanup`

- **parse\***: three passes over the PBF. Fixed-size structs go into mmapped `.bin` sequence files (`midgard/sequence.h`, with its own mmap and Windows shim and a parallel external sort). The design deliberately supports "very large datasets (the planet) with very modest hardware" (tag-parsing doc).
- **constructedges / build**: ways are split at intersections into nodes and directed edges, then written as level-2-style tiles (`graphbuilder.cc`).
- **enhance**: density, speeds, turn types, stop impact, name consistency, not-thru/dead-end detection, country access (`graphenhancer.cc`).
- **filter**: optionally drops driving, bike or pedestrian edges and re-aggregates nodes (`graphfilter.cc`).
- **bss**: bike-share stations; **transit**: connects transit stops to the road graph.
- **hierarchy**: moves edges to level 0/1/2 by road class and duplicates nodes across levels with `NodeTransition` up/down links (`hierarchybuilder.cc`).
- **shortcuts**: `shortcutbuilder.cc`, see §5.
- **restrictions**: complex (via-way) turn restrictions (`restrictionbuilder.cc`).
- **validate**: opposing-edge indices, statistics, tile checksums (`graphvalidator.cc`).
- **Pedestrian areas** (3.9.0, behind `mjolnir.pedestrian_areas`): generated medial-axis traversals through plazas (`docs/docs/concepts/pedestrian-areas.md`).
- **Packaging**: `scripts/valhalla_build_extract` packs tiles into a tar with `index.bin` for mmap or remote range-reading.

### On-disk format
- **Tile hierarchy** (`src/baldr/tilehierarchy.cc`, `docs/docs/concepts/tiles.md`). Each tile file is `{level}/{tileid-path}.gph`.

  | Level | Name | Tile size | Road classes |
  |---|---|---|---|
  | 0 | highway | 4° | motorway, trunk, primary |
  | 1 | arterial | 1° | secondary, tertiary |
  | 2 | local | 0.25° | unclassified, residential, service and other |
  | 3 | transit | 0.25° | transit |
- **GraphId** (`valhalla/baldr/graphid.h`) is 64 bits but uses 46: 3 bits level | 22 bits tile index | 21 bits index within tile. So a tile holds at most ~2M nodes/edges, and up to 8 levels are possible.
- **A tile is a single blob of packed POD structs**, cast in place with no deserialization (`src/baldr/graphtile.cc` ~L325–400, `reinterpret_cast` pointer walk):
  1. `GraphTileHeader`: 272 bytes, `static_assert`ed (`graphtileheader.h`);
  2. `NodeInfo[]`: 32 bytes each;
  3. `NodeTransition[]`;
  4. `DirectedEdge[]`: 48 bytes each, six 8-byte bit-packed words (`directededge.h` L1208–1316);
  5. optional `DirectedEdgeExt[]`;
  6. access restrictions;
  7. transit departures, stops, routes, schedules, transfers;
  8. signs, turn lanes, admins;
  9. 5×5 spatial **edge bins** for snapping (`kBinsDim = 5`);
  10. complex restrictions (fwd/rev);
  11. **EdgeInfo** (shared by both directions: varint-encoded shape, names, OSM way id, `TaggedValue` extensions, encoded elevation);
  12. text list;
  13. lane connectivity;
  14. predicted speeds;
  15. bounding circles.

  Everything is native little-endian and bitfield-layout-dependent.
- **The format is frozen.** `CLAUDE.md`: "The last tile version bump was ~7 years ago… only 4 spare bits in `DirectedEdge` and 1 in `NodeInfo`". New per-edge data goes into variable-length `TaggedValue`s in EdgeInfo. This gives great stability for consumers, but the struct design is ossified (e.g. `spare*` fields marked "TODO: DELETE ME IN V4").

### Planet scale (from the repo's own `CLAUDE.md`, "Performance at Planet Scale")
| Metric | Value |
|---|---|
| Planet PBF | ~85 GiB |
| Output tileset | ~PBF + 10%, **~95 GiB** |
| Tiles | ~205,000 |
| DirectedEdges / NodeInfos | ~1 billion / ~500 million |
| `way_node.bin` intermediate | ~110 GiB |
| Build time | **~7 h on m8gd.8xlarge** (32 vCPU, 128 GiB, NVMe) |

Older docs (`docs/docs/contributing/architecture/mjolnir/index.md`) say a planet build works on a quad-core with **16 GB RAM** and an SSD in about 15 h, including admins and timezones but not elevation. The gis-ops overview says ~12 h on a 16 GB laptop, for *all* profiles at once. That is the big operational contrast with OSRM (>250 GB RAM per profile for planet CH preprocessing) and GraphHopper (64–128 GB per profile). At runtime the tar is mmapped and the `GraphReader` LRU cache defaults to about 1 GB, so the OS page cache does the rest.

## 4. Graph model

- **Nodes = intersections** (`NodeInfo`: lat/lon as 7-digit offsets from the tile base, access mask, first-edge index and count, admin, timezone, intersection type, traffic signal, local driveability, packed headings of up to 8 local edges, elevation, transition index and count).
- **Directed edges**, stored as an outgoing adjacency array per node (CSR-style, `edge_index`/`edge_count`). Each has `endnode` (a GraphId, possibly in another tile/level), `opp_index` (index of the reverse edge at the end node), `edgeinfo_offset`, per-mode forward/reverse access masks (12 modes), speeds (default, free-flow, constrained, truck), use (~40 kinds), classification, surface, slopes, curvature, lanes, density, toll/tunnel/bridge/roundabout/link/internal flags, `turntype` per local edge (24 bits = 8×3), `stopimpact`/`lineid` union, and `shortcut`/`superseded` masks.
- **Search is edge-based.** Labels (`sif::EdgeLabel`, `BDEdgeLabel`) and `EdgeStatus` are keyed by directed-edge GraphId. `TransitionCost(edge, node, pred)` is evaluated with the predecessor edge, which gives turn costs, U-turn detection and turn restrictions without an explicit turn graph.
- **Turn restrictions**
  - *Simple* (via-node) restrictions are an 8-bit mask on the inbound edge of forbidden local outgoing indexes (`restrictions_`), checked in costing `Allowed()`.
  - *Complex* (via-way, conditional, probabilistic) restrictions are stored per tile and flagged by `start_restriction_`/`end_restriction_` per-mode masks. At query time `DynamicCost::Restricted()` walks the predecessor label chain to match the via list (`valhalla/sif/dynamiccost.h` ~L595–720). When a restriction fires, it resets already-permanent edges to `kUnreachedOrReset` so they can be reached by other paths. It supports timed restrictions and `restriction_probability`.
- **Hierarchy.** Each edge lives on exactly one level, by road class. Nodes that touch multiple levels are duplicated, one copy per level, and linked by **`NodeTransition`** (up/down) records instead of edges (`valhalla/baldr/nodetransition.h`, `hierarchybuilder.cc`). Some edges are flagged across tile boundaries (`leaves_tile_`).
- **Shortcuts** exist on levels 0 and 1 only. They are stored as ordinary `DirectedEdge`s with `is_shortcut()`, placed *before* the regular edges of a node. The base edges they replace carry a `superseded` bit-mask. Up to 7 shortcuts per node (`kMaxShortcutsFromNode`). They are expanded back for output by `GraphReader::RecoverShortcut()` (`src/baldr/shortcut_recovery.h`).
- **Tile loading**: `GraphReader::GetGraphTile()` (`src/baldr/graphreader.cc` ~L597) checks the in-memory cache first, then the mmapped tar extract (zero copy; a `TarballGraphMemory` wraps an mmap slice), then `tile_dir` (file read, gz supported), then `tile_url` (curl, with retry/backoff fixes in the UNRELEASED changelog). Cache implementations: `FlatTileCache` (vector indexed by tile id, fastest), `SimpleTileCache`, `TileCacheLRU` (memory-limit control), and `SynchronizedTileCache` for sharing across threads. Traffic tiles are attached to the graph tile at load time.

## 5. Algorithms (thor/, meili/)

Files in `src/thor/` with line counts: `bidirectional_astar.cc` (1548), `unidirectional_astar.cc` (932, provides `TimeDepForward`/`TimeDepReverse`), `multimodal_transit.cc` (838), `multimodal_astar.cc` (620), `costmatrix.cc` (1463), `timedistancematrix.cc` (629), `timedistancebssmatrix.cc` (558), `dijkstras.cc` (1009, the base for isochrones and expansion), `isochrone.cc` (346), `alternates.cc` (199), `optimizer.cc` (196), `centroid.cc` (261), `map_matcher.cc`/`route_matcher.cc`, `triplegbuilder.cc` (2339, path → `TripLeg` protobuf with all attributes).

**Algorithm selection** (`thor_worker_t::get_path_algorithm`, `src/thor/route_action.cc` L227–299):
- `multimodal`/`transit` → `MultiModalPathAlgorithm`; `auto_pedestrian` and `bikeshare` → `MultimodalAStar`, a unidirectional search with two costings (generalized in 2026-02, #5780).
- `depart_at` within `max_timedep_distance` (default **500 km**, `src/thor/worker.cc` L25) → `TimeDepForward`; `arrive_by` → `TimeDepReverse`. Beyond that distance it falls back to bidirectional with warnings 402 or 214. Since the bidirectional search does not evaluate time-varying speeds along the path (`docs/.../thor/path-algorithm.md`), long time-dependent routes are approximated.
- Origin and destination edges the same or adjacent → unidirectional.
- Otherwise → **`BidirectionalAStar`**, the workhorse.

**Bidirectional A\*** (`src/thor/bidirectional_astar.cc`)
- Forward and reverse edge-label sets, each with an **A\*** heuristic (`AStarHeuristic`: great-circle distance × costing's `AStarCostFactor()`).
- Priority queue: a **double bucket queue** (`valhalla/baldr/double_bucket_queue.h`), i.e. bucket sort plus an overflow bucket.
- **Connection & termination**: when the two searches meet (`SetForwardConnection`/`SetReverseConnection`), it sets `cost_threshold_ = c + threshold_delta_` (default **420** cost units, `kThresholdDelta`) and keeps expanding until both queues exceed it. It is therefore not a provably-optimal stopping rule but a tuned slack. For alternates the threshold becomes `1.2 × c + 420` plus an iterations budget (`kAlternativeCostExtend`, `kAlternativeIterationsDelta = 100000`).
- The direction to expand is chosen by comparing sort costs, adjusted by `cost_diff_`. It is also forced when one side has exhausted its hierarchy level.
- **Alternates** (`src/thor/alternates.cc`) use the meeting-point/"via" candidates from the extended search, filtered by Kobitzsch-style viability tests: limited sharing (0.6–0.75 depending on distance), bounded stretch (2.0 for <10 min down to 1.25 for long routes, a fitted hyperbolic curve), and a detour bound.

**How the hierarchy gives speed without full CH**
1. **Levels**: the search starts on local roads and moves up via `NodeTransition`s. Each level has **hierarchy limits** (`valhalla/sif/hierarchylimits.h`): `max_up_transitions` (defaults `{0, 400, 100}` for levels 0/1/2) and `expand_within_dist` (unidirectional `{∞, 100 km, 5 km}`; bidirectional `{∞, 20 km, 5 km}`). Once a level's up-transition count exceeds the limit and the label is farther than `expand_within_dist` from the origin/destination, `StopExpanding()` stops expanding that level and the search proceeds only on higher levels. It is the old commercial-nav idea: "use local roads near the endpoints, highways in between". The pruning is **heuristic**. It can miss the optimal route and is relaxed on failure (`RelaxHierarchyLimits`). The unidirectional search also scales the limits by road density near the destination (`unidirectional_astar.cc` ~L660–691).
2. **Shortcuts** (`src/mjolnir/shortcutbuilder.cc`): on levels 0 and 1, chains of edges through *degree-2-on-this-level* nodes are merged into one edge, provided the edges have matching attributes (`EdgesMatch`) and the node has no upward transition. Contraction across a node is refused when it has an exit sign, a link at a traffic signal or with >3 edges, a turn restriction between the pair, a country border, or a sharp turn (>60°) with other drivable roads present (`CanContract`, L184–294). This is **very conservative contraction**. Only "boring" chains are collapsed and no witness searches are done. That is why any costing still works on shortcuts: they keep enough attributes for `EdgeCost` to be evaluated dynamically.
   - The bidirectional search uses a shortcut only once the next-lower level has stopped expanding. It then sets the `superseded` mask so the replaced base edges are skipped (`ExpandInner`, L179–215).
   - **Time-dependent unidirectional search skips shortcuts entirely** (`unidirectional_astar.cc` L177).
   - **Pedestrian and bicycle never use the hierarchy**: their limits are unlimited, so they expand only the local level (`path-algorithm.md` L21; `ignore_hierarchy_limits_`). That is why their `service_limits` are smaller: bicycle 500 km, pedestrian 250 km, vs auto 5,000 km (`scripts/valhalla_build_config`).
3. **Effect**, from the repo's own measurements (`CLAUDE.md`): edges touched per route are London→Birmingham (190 km) ~485k bidirectional vs ~550k unidirectional; Berlin→Prague (350 km) ~420k vs ~3M; Paris→Berlin (1000 km) ~800k; SF→NYC (4679 km) ~1M. Compare CH engines, which settle on the order of 10³ nodes for continental queries. Valhalla's hierarchy keeps work roughly sublinear in distance, but at about 10⁵–10⁶ edge relaxations, each with a virtual costing call.

**Matrix**
- **`CostMatrix`** (`src/thor/costmatrix.cc`): many-to-many *bidirectional* search. One reverse tree per target and one forward tree per source are expanded in lock-step, and connections are found when a forward search reaches edges already settled by reverse trees (`ReachedMap`). Each tree has hierarchy limits (`hierarchy_limits_config_costmatrix`). An optional second pass catches pairs found in only one direction. Since 3.x it supports time-dependent use when sources share a departure instant. Cost thresholds come from distance (`kCostThreshold*Divisor`). Default limit: 2,500 location pairs.
- **`TimeDistanceMatrix`** (`timedistancematrix.cc`): one-to-many Dijkstra-style expansion per source (or per target). It is exact and time-dependent and is preferred when the request has a time and doesn't `prioritize_bidirectional` (`get_matrix_algorithm`, `src/thor/matrix_action.cc` L33–80). For pedestrian/bicycle it is chosen when either side is small (`kCostMatrixThreshold`).
- **`TimeDistanceBSSMatrix`** for bike share.
- Matrices are Valhalla's known weak spot vs OSRM/CH; see §8.

**Isochrones** (`src/thor/isochrone.cc`, `dijkstras.cc`): multi-origin forward/reverse Dijkstra (supports time, multimodal and BSS) writes costs into a `GriddedData` raster (`valhalla/midgard/gridded_data.h`). Contours are generated as lines or polygons with generalization; there is also GeoTIFF raster output (`ENABLE_GEOTIFF`). Also supported: distance isolines and reverse ("arrive by") isochrones.

**Expansion API** (`src/thor/expansion_action.cc`): streams every edge the route, isochrone or matrix search touches, as GeoJSON or PBF, for visualisation and debugging. Properties include cost, duration, status, pred_edge_id and expansion_index.

**Map matching / Meili** (`src/meili/`): HMM after Newson & Krumm (`docs/.../meili/algorithms.md`).
- Candidates come from a spatial grid over tiles (`candidate_search.cc`).
- Emission is Gaussian on GPS distance; transition compares route distance with great-circle distance (`transition_cost_model.cc`).
- Transitions are computed with a bounded label-set Dijkstra (`routing.cc`).
- Lazy Viterbi uses a priority queue (`viterbi_search.cc`), plus Top-K for alternatives (`topk_search.cc`).
- Thor turns matches into paths (`src/thor/map_matcher.cc`). There is also a fast **edge-walk** mode for exact shapes (`route_matcher.cc`).
- Exposed as `/trace_route` (map-match plus guidance) and `/trace_attributes` (per-edge attributes along the match). Edge-walk is also reused internally for `linear_cost_factors`.

**Optimized route (TSP)** (`src/thor/optimizer.cc`): **simulated annealing** over a CostMatrix result, with fixed first and last points. It is simple and not a VRP solver.

**Centroid** (`centroid.cc`): finds a convergence "meeting point" reachable from multiple origins, using simultaneous Dijkstras.

**Transit** (`src/thor/multimodal_transit.cc`): time-dependent, schedule-aware A\*/Dijkstra over walk edges plus transit edges (`GetNextDeparture` per line). It applies transfer penalties and in-station transfer time (a 30 s hack; "TODO - let's get the transfers.txt implemented!", L366), with walking-distance limits. This is **not RAPTOR/CSA, and not multi-criteria**. Treat it as a basic feature.

## 6. Costing / profiles (sif/)

- Interface: `sif::DynamicCost` (`valhalla/sif/dynamiccost.h`). The hot virtuals are `Allowed(edge, pred, tile, edgeid, time, tz, restriction_idx)`, `AllowedReverse`, `Allowed(node)`, `EdgeCost(edge, tile, time_info, flow_sources)`, `TransitionCost(edge, node, pred)`, `TransitionCostReverse`, `AStarCostFactor`, plus `IsClosed`, `Restricted` and exclusion lists. `Cost` = `{cost, secs}`, separating the optimisation weight from actual elapsed time.
- **Models** (`valhalla/sif/costfactory.h`): `auto`, `bus`, `taxi` (auto variants), `truck`, `motorcycle`, `motor_scooter`, `bicycle` (Road/Hybrid/Cross/Mountain types), `pedestrian` (incl. wheelchair / blind types), `transit`, plus the meta-costings `multimodal`, `bikeshare`, `auto_pedestrian` and `none`. Each lives in its own `src/sif/*cost.cc` file, about 600–1500 lines.
- **Changeable per request, no rebuild needed.** Almost everything:
  - General: penalties and factors for maneuvers, gates, tolls, ferries, country crossings, private/destination-only, alleys, service roads, living streets, tracks, lit; `use_highways`, `use_tolls`, `use_ferry`, `use_distance` (time ↔ distance blend), `shortest`, `top_speed`, `fixed_speed`.
  - Excludes: `exclude_unpaved`, `exclude_tolls`, `exclude_bridges`, `exclude_tunnels`, `exclude_highways`, `exclude_ferries`, `exclude_cash_only_tolls`; ignore-restrictions, oneways, access, closures.
  - HOV/HOT; vehicle `height`, `width`, `length`, `weight`; truck `axle_load`, `axle_count`, `hazmat`, `use_truck_route`, `low_class_penalty`.
  - Bicycle: `use_roads`, `use_hills`, `avoid_bad_surfaces`, `cycling_speed`, `steps_factor`.
  - Pedestrian: `walking_speed`, `step_penalty`, `max_hiking_difficulty`, `use_hills`, `walkway`/`sidewalk`/`driveway` factors, elevator penalty.
  - Traffic: `speed_types` (which of live, predicted, constrained, free-flow to use).
  - Request level: `exclude_locations`, `exclude_polygons` (avoid areas), and the new **`linear_cost_factors`** (user-supplied polylines matched by edge-walk with a cost factor or forced allow; resolved in loki since 3.9.1, `src/loki/linear_cost_factors.cc`).
  - Even **hierarchy limits** can be overridden, if the operator config allows it (`allow_hierarchy_limits_modifications`, `check_hierarchy_limits` in `matrix_action.cc`).

  See the option parsers in `src/sif/dynamiccost.cc` (`ParseBaseCostOptions`) and each model's `Parse*CostOptions`, and the doc `docs/docs/concepts/costing/dynamic-costing.md`.
- **Needs a rebuild**: changes to what is *in* the graph. That covers the tag → attribute interpretation (`lua/graph.lua` or `pbfgraphparser.cc`), default speeds per road class/density (`docs/docs/concepts/speeds.md`, `valhalla_assign_speeds`), access defaults per country, hierarchy/level assignment, shortcut formation, which modes are included (`graphfilter.cc`), elevation and predicted traffic attachment. Live traffic needs no rebuild; you write into `traffic.tar`.
- **Writing a new profile** means a new C++ class plus a factory registration and protobuf options. There is no scripting/DSL for costing. The gis-ops overview rates Valhalla "flexibility 4/5, customizability 2/5" for this reason.

## 7. Features

- **Turn-by-turn** (`src/odin/`):
  - `maneuversbuilder.cc` collapses edges into maneuvers (turn types, ramps, roundabouts, forks, merges, transit legs, ferry, elevators/steps, building entrances, signs and exit numbers, junction names, guidance views, landmarks).
  - `narrativebuilder.cc` produces text, verbal pre-transition / transition-alert / post-transition and succinct instructions. Language-specific subclasses (`NarrativeBuilder_csCZ`, `_hiIN`, `_itIT`, `_ruRU`) handle grammar.
  - **35 locales** (`locales/*.json` generated from gettext `.po` files since 3.9.0, compiled into the binary via `cmake/ValhallaBin2Header.cmake`), including a pirate one.
  - Output: Valhalla JSON, **OSRM-compatible JSON with Mapbox-style `voiceInstructions` and `bannerInstructions`** (`src/tyr/route_serializer_osrm.cc`; this is what Ferrostar and Mapbox nav SDKs consume), GPX, and PBF.
- **Elevation**: `/height` endpoint (heights or range profile along a polyline), elevation-aware bike/ped costing (`use_hills`, grades), and per-route elevation (`docs/docs/concepts/costing/elevation-costing.md`).
- **Traffic**: live speeds (mmapped tar overlay), predicted speeds (DCT-compressed weekly profiles), and constrained/free-flow speeds. Speed resolution order is live → predicted → constrained → free-flow → default (`GraphTile::GetSpeed`, `docs/docs/concepts/speeds.md`). Incidents, closures (`ignore_closures`) and OpenLR encoding of edges (`valhalla/baldr/openlr.h`).
- **Time-dependent routing**: `date_time` types current, depart_at, arrive_by and invariant. Conditional access/restrictions (`*:conditional`) are evaluated with timezone-aware time (`DateTime::is_conditional_active`).
- **Avoid**: points, polygons, linear cost factors, and per-mode excludes.
- **Location semantics** (loki): heading and tolerance, `preferred_side`, `radius`, `search_cutoff`, `minimum_reachability` (avoid snapping to islands; `src/loki/reach.cc`), `search_filter` (exclude tunnels/bridges/ramps/closures/steps, min/max road class), through/via/break waypoints, and `street_side_tolerance`.
- **Other endpoints**:
  - `/locate`: edge and node metadata under a point (verbose).
  - `/trace_attributes`: per-edge attributes along a match; a "make your own segment analytics" tool.
  - `/expansion`.
  - `/transit_available`.
  - `/status`.
  - **`/tile`** (BETA): the routing graph rendered as Mapbox Vector Tiles with edges, nodes, shortcuts, access restrictions and incident layers, built with `vtzero`. Good for debugging (`docs/docs/api/tile.md`).
- Bike share (BSS) routing, multimodal auto+walk (`auto_pedestrian`, park-and-walk), truck routing, HOV, wheelchair/blind pedestrians, ferries and rail ferries, and **pedestrian areas** (medial axis through plazas, 3.9.0).

## 8. Performance characteristics

No rigorous, recent, planet-scale head-to-head benchmark exists publicly. What is known:

- **Work per query** (repo's own numbers, §5): about 0.4–1M edges touched for 200–4700 km auto routes. Each relaxation calls virtual costing (`EdgeCost`/`TransitionCost`/`Allowed`), checks restrictions and may fetch a tile. The repo calls costing "the hottest path… called millions of times per request" (`CLAUDE.md`).
- **Reported latencies**:
  - gis-ops' routing-engine overview (archived 2024) says: regional routes take milliseconds, **Madrid→Moscow 1–2 s**, matrices "comparatively slow". It rates performance OSRM 5/5, GraphHopper 4, ORS 4, Valhalla 3. For memory it rates Valhalla best (4/5): planet built on a 16 GB laptop in ~12 h for all profiles, and regions run on a Raspberry Pi or a low-end phone. OSRM needs >250 GB RAM per profile to preprocess the planet (35–40 GB at runtime); GraphHopper needs 64–128 GB.
  - Valhalla issue #2604 (2020, Sri Lanka, 4 vCPU): single routes were roughly comparable to OSRM on a small country (the results were inconsistent across runs). A **33×1 matrix averaged 2–5 s in Valhalla vs 10–23 ms in OSRM**. Maintainers' explanation (kevinkreiser): OSRM "fixes edge weights through aggressive contraction" while Valhalla keeps contraction conservative for dynamic costing; nilsnolde: CostMatrix expands from all locations at once, so RAM is high. 2022 reports in the same thread: Germany 175×175 >1 min, while OSRM did 1–2.5k² in 12–46 s. CostMatrix has been reworked substantially since (2023–2026: second pass, time-dependence, `filtered_edges`, reservation/`Clear()` changes, etc.), so these numbers are pessimistic. Large matrices are still the domain of CH or matrix-specialised engines.
  - A small-region third-party comparison (Aachen walking, 400 trips) put OSRM at ~0.2 ms median locally. Its Valhalla figure was a public-server round trip (~92 ms incl. network), so it is not comparable.
- **Rule of thumb**: short urban routes take single-digit ms. Long auto routes take ~100 ms to ~1 s+ on the planet. Bike and pedestrian long routes are much slower because there is no hierarchy, hence the 250/500 km service limits. A CH engine is 1–3 orders of magnitude faster per query and per matrix cell, but needs per-profile preprocessing and fixed weights. CCH/CRP engines sit in between: dynamic weights with a customization step.
- **Memory**: a server needs the ~95 GiB tileset on fast disk (mmapped; the hot working set is much smaller), plus a ~1 GB tile cache and per-thread label arrays. A regional extract on mobile can be tens to hundreds of MB.
- **Build**: one planet build serves every mode and every per-request variant, in ~7 h on 32 cores / 128 GiB, or ~12–15 h on 16 GB. Partial and regional rebuilds come naturally from the tiling (`scripts/incremental_build_tiles`).

## 9. Architecture & code quality

### Module map
Norse-themed; headers in `valhalla/<mod>/`, sources in `src/<mod>/`.

| Module | Role | Approx. size |
|---|---|---|
| **midgard** | geometry (PointLL, AABB, polyline encode/decode, tiles math, `sequence<T>` mmap containers, logging, tar reader) | 3.8k |
| **baldr** | graph data structures, tile I/O, GraphReader and caches, datetime/tz, traffic tiles, streetnames, verbal formatting | 9.3k (+ most of `valhalla/baldr` headers) |
| **sif** | costing models | 8.2k |
| **skadi** | DEM sampling | 0.7k |
| **mjolnir** | tile builder and data tools (PBF parse, admins, transit, elevation, traffic ingestion, validation) | 33.9k |
| **loki** | location correlation/snapping, reachability, polygon/linear exclusion resolution, `/locate`, `/tile` | 5.0k |
| **thor** | path algorithms, matrices, isochrones, map-match → path, TripLeg building | 14.4k |
| **meili** | HMM map matching | 3.3k |
| **odin** | maneuvers and narrative | 14.8k |
| **tyr** | actor orchestration and serializers (Valhalla JSON, OSRM JSON, GPX, PBF) | 6.8k |

Request flow: `tyr → loki → thor → odin → tyr`. The modules pass a **protobuf `Api` object** (`proto/descriptors/api.proto`, `options.proto`, `trip.proto`, `directions.proto`, …), which also serves as the public request/response schema. That makes the pipeline stages separable: they can run in different processes over ZMQ.

### Build and dependencies
- CMake with options `ENABLE_TOOLS`, `ENABLE_DATA_TOOLS`, `ENABLE_SERVICES`, `ENABLE_HTTP`, `ENABLE_PYTHON_BINDINGS`, `ENABLE_NODE_BINDINGS`, `ENABLE_STATIC_LIBRARY_MODULES`, `ENABLE_GEOTIFF`, `ENABLE_LZ4`, `ENABLE_THREAD_SAFE_TILE_REF_COUNT`, sanitizers and coverage (`CMakeLists.txt` L38–59). vcpkg manifest (`vcpkg.json`) for Windows.
- **Core runtime deps**: Boost (headers; property_tree is the config type throughout the API), Protobuf (full, not lite), zlib, optional curl and lz4.
- **Vendored submodules**: rapidjson, date/tz, unordered_dense, cpp-statsd-client, microtar, protozero, vtzero, flatbush.
- **Data tools add**: libosmium, LuaJIT, SQLite3 + SpatiaLite, GEOS, (GDAL/GeoTIFF), just_gtfs.
- **Services add**: prime_server + ZeroMQ.

A routing-only build (as valhalla-rs does it) needs just Boost headers, Protobuf and zlib, plus a C++20 compiler.

### Tests
- Unit tests: `test/*.cc`.
- **Gurka**: `test/gurka/`, 139 files. An integration framework that builds tiles from *ASCII-art maps* plus OSM tag dictionaries, runs real API actions and asserts paths and maneuvers (`test/gurka/test_simple_restrictions.cc`, `docs/docs/contributing/gurka.md`). Excellent idea to copy.
- Python and Node binding tests.
- 85% line coverage.
- Perf and regression harness: `run_route_scripts/` (request generators, `wrk` bench, visual diff).

### Embeddability and portability of pieces
- *Embedding the whole engine* is straightforward through `actor_t` or valhalla-rs. The pain points are the heavyweight C++ toolchain, protobuf and Boost, `ptree` configs, a global/singleton tz DB, and a non-thread-safe `GraphReader`.
- *Porting pieces*:
  - **Tile reading (baldr) is easy to port.** The structs are POD with documented bit layouts, so a Rust `#[repr(C)]` / bit-extraction reader over an mmapped tar is a few thousand lines. valhalla-rs already exposes these through FFI as a template.
  - **Costing** depends on many graph attributes, but each model is self-contained.
  - **Odin** is large and stringly-typed, a big porting effort but valuable.
  - **Mjolnir** is the biggest (34k lines) and most entangled with OSM-tag semantics and Lua.
- Code style: older C++ (raw pointers, large functions such as `pbfgraphparser.cc` with 5400+ lines, macros like `JSON_PBF_RANGED_DEFAULT`), gradually modernised (C++20, `std::format` replacing boost::format in 3.9.1, `std::span`). Heavy comments and good docs under `docs/docs/` (mkdocs site at valhalla.github.io/valhalla).

## 10. Strengths, weaknesses, suitability

### Strengths
- **One graph, all modes, per-request costing.** There is no per-profile preprocessing. The set of runtime knobs (truck dimensions, hazmat, avoid polygons, linear cost factors, hills, surfaces, traffic source selection) is among the richest in OSS.
- **Feature breadth in one engine**: high-quality turn-by-turn in 35 languages with OSRM/Mapbox-compatible voice and banner output, isochrones, matrices, HMM map matching, trace attributes, TSP, elevation, live and predicted traffic, time-dependent routing, conditional restrictions, transit, BSS, park-and-walk, expansion and graph-tile debugging endpoints.
- **Operationally light build** (planet on 16 GB RAM) and **tiles as a deployment unit**: regional extracts, mmap zero-copy loading, remote/lazy tile fetch over HTTP range requests, offline mobile use, and a writable live-traffic overlay.
- **Stable on-disk format** (unchanged for about 7 years). Active, responsive maintainers with quarterly releases and good test infrastructure (gurka).
- MIT license, and Rust bindings exist.

### Weaknesses
- **Query speed**: heuristic hierarchy pruning and an approximate termination slack mean slower queries than CH/CCH and occasional suboptimal routes. Very slow for long bike/ped routes. Large matrices are slow and RAM-hungry.
- **Time-dependent routing beyond 500 km falls back to an approximation**, and shortcuts are unused in time-dependent search.
- **Transit is basic**: no RAPTOR, no transfers.txt, single criterion. Use MOTIS, OTP or R5 for serious transit.
- **Customization requires C++**: no profile DSL. Tag interpretation is in Lua at build time, but costing is compiled.
- **Frozen bit-packed format**: little room for new per-edge fields (only `TaggedValue` extensions). There are hard-coded limits (21-bit per-tile ids, 7 shortcuts per node, 3 fixed levels, fixed lat/lon grid, so tiles near the poles are tiny and equator tiles dense).
- **Heavy C++ dependency stack** (Boost, Protobuf, plus Lua, SpatiaLite and GEOS for building). The JSON → protobuf → JSON plumbing adds overhead and complexity.

### (a) As a dependency for `routet`
**Viable if the product needs Valhalla's feature set** (guidance, isochrones, map matching, truck, traffic) more than raw query throughput.
- From Rust, **use `valhalla-rs`** (`cargo add valhalla`; Actor with prost `Options`, typed `ConfigBuilder`, tile and traffic access). Tile building would still happen with upstream `valhalla_build_tiles`, via Docker or pyvalhalla.
- Expect a C++ build in `build.rs` (cmake, Boost headers, protobuf) and a non-trivial binary size. Cross-compiling to Android/iOS is proven (valhalla-mobile) but needs work. WASM is not realistic.
- A middle path: **read Valhalla tilesets natively in Rust**, reusing the planet-scale build pipeline and the FOSSGIS-style data, while writing your own search and costing on top. The POD format makes this feasible, and the format's stability lowers the risk.

### (b) As a reference to learn from or port ideas
High value for:
1. **Tile design**: geographic 3-level tiling, 64-bit GraphId, zero-copy POD structs in an mmapped tar, a shared EdgeInfo for both directions, per-tile spatial bins for snapping, remote tar index with range requests, and a separate writable traffic overlay of 8 bytes per edge.
2. **Costing API shape**: a `Cost{cost, secs}` split; `Allowed`, `EdgeCost` and `TransitionCost` with the predecessor; how penalties vs factors are modelled (`docs/docs/start/terminology.md`); complex-restriction evaluation by walking the predecessor chain.
3. **Odin's maneuver and narrative pipeline**, the OSRM/Mapbox-compatible serializer, and the locale workflow.
4. **Meili HMM map matching**, compact and well documented.
5. **DCT compression of weekly speed profiles** (7 days × 288 buckets → 200 int16 coefficients).
6. **Gurka-style ASCII-map integration tests.**
7. Loki's snapping semantics (heading, side of street, reachability).

Lower value: the hierarchy-limits and shortcut speed-up. It is an old pragmatic heuristic. A new Rust engine wanting dynamic per-request costs with fast queries would do better with **CCH/CRP** (see RoutingKit, `rust_road_router`, `routingkit-cch` in `~/g`) or a hybrid, keeping Valhalla-style rich attributes for the customization and guidance layers. Also don't copy the bit-packed-forever struct layout without a versioning and extension plan.

---

### Key file index
- Graph: `valhalla/baldr/graphid.h`, `directededge.h`, `nodeinfo.h`, `nodetransition.h`, `edgeinfo.h`, `graphtileheader.h`, `src/baldr/graphtile.cc`, `src/baldr/graphreader.cc`, `src/baldr/tilehierarchy.cc`, `src/baldr/shortcut_recovery.h`
- Traffic: `valhalla/baldr/traffictile.h`, `valhalla/baldr/predictedspeeds.h`
- Build: `valhalla/mjolnir/util.h` (stages), `src/mjolnir/util.cc`, `pbfgraphparser.cc`, `graphbuilder.cc`, `graphenhancer.cc`, `hierarchybuilder.cc`, `shortcutbuilder.cc`, `restrictionbuilder.cc`, `elevationbuilder.cc`, `ingest_transit.cc`, `convert_transit.cc`, `lua/graph.lua`, `scripts/valhalla_build_*`
- Search: `src/thor/route_action.cc` (selection), `bidirectional_astar.cc`, `unidirectional_astar.cc`, `multimodal_transit.cc`, `multimodal_astar.cc`, `costmatrix.cc`, `timedistancematrix.cc`, `matrix_action.cc`, `isochrone.cc`, `dijkstras.cc`, `alternates.cc`, `optimizer.cc`, `valhalla/baldr/double_bucket_queue.h`, `valhalla/sif/hierarchylimits.h`
- Costing: `valhalla/sif/dynamiccost.h`, `src/sif/dynamiccost.cc`, `src/sif/autocost.cc`, `truckcost.cc`, `bicyclecost.cc`, `pedestriancost.cc`, `valhalla/sif/costfactory.h`
- Map matching: `src/meili/*.cc`, `docs/docs/contributing/architecture/meili/`
- Guidance: `src/odin/maneuversbuilder.cc`, `narrativebuilder.cc`, `locales/`, `src/tyr/route_serializer_osrm.cc`
- API: `valhalla/tyr/actor.h`, `proto/descriptors/*.proto`, `docs/docs/api/`
- Bindings: `src/bindings/python`, `src/bindings/nodejs`, `~/g/valhalla-rs/src/{actor.rs,libvalhalla.cpp,traffic.rs,config.rs}`

### External sources
- gis-ops FOSS routing engines overview (archived 2024): https://github.com/gis-ops/tutorials/blob/master/general/foss_routing_engines_overview.md
- Valhalla issue #2604 (matrix performance vs OSRM): https://github.com/valhalla/valhalla/issues/2604
- Mapbox ❤️ Valhalla (2018): https://blog.mapbox.com/mapbox-%EF%B8%8F-valhalla-1fdda95ac4f
- valhalla-mobile: https://github.com/Rallista/valhalla-mobile
- valhalla-rs: https://github.com/kinkard/valhalla-rs
- Ferrostar route providers: `~/g/ferrostar/guide/src/route-providers.md`
- Codecov API (85.11% coverage, 2026-10-23); FOSSGIS `/status` (version 3.9.1-618cad5).
