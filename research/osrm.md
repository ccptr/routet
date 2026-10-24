# OSRM (Open Source Routing Machine): research notes

- Repo: <https://github.com/Project-OSRM/osrm-backend>, cloned at `~/g/osrm-backend`
- Snapshot: commit `afd7d0619` (2026-10-21), latest tag `v26.10.0` (2026-10-01)
- File paths below are relative to the repo root.

---

## TL;DR

- **What it is.** A C++20 road router for OSM data. It is the reference open-source implementation of **Contraction Hierarchies (CH)** and **Multi-Level Dijkstra (MLD, a CRP variant)**, both running on an **edge-based (turn-expanded) graph**. Its defining traits:
  - Very low query latency and very fast many-to-many tables (CH).
  - **Everything about cost is decided at preprocessing time by a Lua profile.**
  - Query-time customisation is close to nil: up to 7 pre-baked "exclude" class combinations, plus snapping controls (bearings, radiuses, approaches).
- **Licence:** BSD-2-Clause (`LICENSE.TXT`).
- **Stars and forks:** about 8.2k stars and 4k forks.
- **Governance arc:**
  - Academic project, 2010–2014 (Dennis Luxen at KIT).
  - Mapbox-funded heyday, about 2014–2018: v5 API, guidance, MLD, traffic updates.
  - Long trickle, 2019–2025: 29–142 commits a year, and only v5.27 (2022) and v6.0.0 (April 2025) released.
  - Sharp revival in 2026 led almost single-handedly by original author **Dennis Luxen**:
    - 331 commits so far in 2026, with heavy AI-assisted authorship (Copilot, Claude, Codex co-author trailers).
    - Automated **monthly releases** with `YY.M.patch` versioning.
    - **No dataset compatibility between monthly versions**: every upgrade means re-processing the data.
- **As a dependency:**
  - Excellent for a server-side car, bike or foot routing/matrix/map-matching microservice when you can afford to re-preprocess on profile changes.
  - Poor for mobile/offline: monolithic RAM-resident datasets of about 120 GiB for the planet, a heavy Boost/TBB/Lua/libosmium stack, and no tiling.
  - Poor where costing must change per request.
- **As a reference:**
  - One of the best places to learn production-quality CH (parallel contraction, stall-on-demand, bucket many-to-many) and MLD/CRP (inertial-flow partitioning, cell cliques, customisation).
  - Also good for: turn-expanded graph construction with complex/via-way restrictions, an HMM map matcher, and a large Cucumber-based behavioural test corpus that is reusable as a spec.

---

## 1. Overview

### Language, licence, size
- **C++20** (`AGENTS.md`: "We target C++20 but need to deal with older compilers").
- Plus Lua 5.4 profiles, JS (Node bindings, Cucumber tests), and Python (nanobind bindings).
- **BSD-2-Clause** (`LICENSE.TXT`, "Copyright (c) 2017, Project OSRM contributors").
- About 93k lines of C++ across 571 files in `src/` and `include/`. Largest modules:

| Module | Lines (src + include) |
|---|---|
| extractor | 13.2k + 8.8k |
| engine | 10.4k + 14.1k |
| util | 1.8k + 11.2k |
| guidance | 6.2k + 2.6k |

### History and governance

| Period | Notes |
|---|---|
| 2010–2013 | Started by Dennis Luxen (with Christian Vetter, whose parallel-CH thesis is cited in `src/contractor/graph_contractor.cpp`) at KIT. 700–1000+ commits a year. |
| 2014–2018 ("Mapbox era") | Mapbox employed the core team: Dennis Luxen, Patrick Niklaus, Daniel J. Hofmann, Michael Krasnyk, Moritz Kobitzsch (guidance), Daniel Patterson (traffic/updater), Emil Tin (profiles), Dane Springmeyer. Mapbox ran OSRM behind its Directions/Matrix APIs. Peak of 1713 commits in 2016. Milestones: v5.0 (2016-04, new HTTP API v1); v5.7 (2017-04) introduced MLD: "fast metric updates in below a minute on continental sized networks" (CHANGELOG at `v5.7.0`); v5.8 added `approaches` and conditional restrictions. |
| 2018 | Mapbox absorbed the Mapzen Valhalla team. Mapbox's own blog framed Valhalla as complementary to its OSRM investment, but OSRM commit volume collapsed afterwards: 384 in 2018, then 125/111/61/142/29 in 2019–2023. |
| 2019–2025 ("community trickle") | Volunteers kept it alive: Denis Chaplygin, Michael Bell, Siarhei Fedartsou (2022 and 2024 modernisation, benchmarks CI, Conan 2), Dennis Luxen (returning part-time). 2020 osrm-talk threads called it "essentially abandonware". Releases: v5.27.0 (2022-10), v5.27.1 (2022-10), **v6.0.0 (2025-04-14)**. v6.0 removed core-CH, moved Node bindings to N-API and added generic obstacles. |
| 2026 ("revival") | 331 commits Jan–Oct 2026; Dennis Luxen authored 170 non-merge commits. Other contributors: Alexander Farber, Patrick Niklaus, Marcello Perathoner, Nils Nolde (Python bindings), with Copilot (87 co-author trailers), Claude and Codex all appearing. `CHANGELOG.md` retired in favour of Conventional Commits (`eec4d768e`). Monthly automated releases (`docs/releasing.md`, `.github/workflows/release-monthly.yml`), switch to vcpkg, Python bindings merged in-tree, `AGENTS.md` added. |

**Bus-factor caveat.** The 2026 burst rests heavily on one person working with AI agents. The pace is impressive, but long-term maintenance risk is real. Commit messages look heavily agent-generated, e.g. "fix(engine): let a path point say where it is".

### Activity
- **Releases:** v26.4.0 through v26.10.0, one per month.
- **Open items:** 357 issues and 15 PRs (GitHub API, 2026-10-23).
- **Dataset compatibility:** `docs/releasing.md` says patch releases keep datasets compatible, but a month change gives "**No compatibility between OSRM datasets (needs new processing)**".
- **Packaging:** Debian unstable picked up 26.4.1 in May 2026.

### Production users (evidence-graded)

**Firm:**
- **FOSSGIS servers** behind openstreetmap.org's car, bike and foot directions (`fossgis-routing-server/osrm-backend` fork, actively pushed in 2026).
- **The public demo server**, `router.project-osrm.org`.
- **Mapbox**, historically, for Directions and Matrix.
- **Geofabrik's** commercial routing API (OSRM + GraphHopper).
- **VROOM**, the dominant OSS vehicle-routing optimiser, which uses OSRM as its default backend.

**Visible forks hinting at production use:**
- `woltapp/osrm-backend` (Wolt).
- `nammayatri/osrm-backend` (Namma Yatri, a ride-hailing app).
- `Telenav/osrm-backend`.

**Rumoured, unverified:** Lyft and Uber.

---

## 2. Deployment model

| Surface | Where | Notes |
|---|---|---|
| `osrm-routed` HTTP server | `src/tools/routed.cpp`, `src/server/` | HTTP now on **Boost.Beast** (v6+). POST JSON bodies added in 2026 (`568bb3595`). Output is JSON or FlatBuffers (`format=flatbuffers`). Flags include `--algorithm ch|mld`, `--max-table-size`, `--max-viaroute-size`, `--max-alternatives` (default 3, `include/engine/engine_config.hpp:83`) and `--disable-feature-dataset ROUTE_STEPS|ROUTE_GEOMETRY` (drops guidance/geometry files to save RAM; `include/osrm/datasets.hpp`, `include/storage/storage_config.hpp`). |
| `libosrm` C++ | `include/osrm/osrm.hpp`, `docs/libosrm.md`, `example/example.cpp` | `class OSRM` with `Route/Table/Nearest/Trip/Match/Tile(params, json::Object&)`. A `ResultT` variant overload also exists, for JSON or FlatBuffers. CMake targets `osrm::osrm`, `osrm::extract`, `osrm::contract`, `osrm::partition`, `osrm::customize` (`CMakeLists.txt:195-210`). FetchContent-compatible since 2025. |
| Node bindings | `src/nodejs/node_osrm.cpp`, `docs/nodejs/api.md`, npm `@project-osrm/osrm` | N-API v8 (`src/nodejs/CMakeLists.txt`), prebuilt binaries via node-pre-gyp. The old `node-osrm` repo is archived (2018). |
| Python bindings | `src/python/`, `pyproject.toml` (nanobind + scikit-build-core, package `osrm-bindings`) | Merged from `nilsnolde/osrm-bindings` in May 2026 (`4d04b385c`). Wheels via cibuildwheel. |
| Shared-memory datastore | `src/tools/store.cpp`, `include/storage/shared_memory*.hpp`, `include/engine/data_watchdog.hpp` | `osrm-datastore` loads a dataset into SysV shared memory. `osrm-routed -s` attaches and hot-swaps when a new dataset is published. `--only-metric` swaps just the new weights after a re-customise: the traffic-update path with zero downtime. `--dataset-name` allows several datasets side by side. |

### Embeddability
- The public headers in `include/osrm/` drag in internal headers such as `engine/api/*_parameters.hpp`, `util/coordinate.hpp`, `util/json_container.hpp` and `storage/storage_config.hpp`, plus Boost asserts.
- So there is no clean C ABI. To embed from Rust you would write a C++ shim:
  - Community crates `osrm-binding` and `rsc_osrm` do exactly this.
  - Both need a locally built libosrm plus Boost and TBB, and their docs.rs builds fail.
- The runtime needs TBB threads and loads a dozen-plus `.osrm.*` files. Three allocation modes:
  - Process memory: `include/engine/datafacade/process_memory_allocator.hpp`.
  - mmap: `mmap_memory_allocator.hpp`, `EngineConfig::use_mmap`, default true.
  - Shared memory: `shared_memory_allocator.hpp`.

### Mobile/offline viability
- **Poor**, and never a design goal.
- **The whole graph is one monolithic dataset.** There are no geographic tiles or partial loading. mmap lets the OS page lazily, but any long query still touches much of the hierarchy.
- **Size:**
  - Planet car: about 123 GiB RAM for `osrm-routed` (official wiki, v5.26, 2021 planet).
  - Even a country is hundreds of MB to several GB.
- **Build footprint is large** (Boost, TBB, Lua, libosmium, expat, bzip2, zlib, libarchive, flatbuffers, sol2, protozero, vtzero), although CI does build arm64 Linux and macOS.
- **Datasets break every month**, so they cannot be shipped long-term to devices.
- **Mapbox's own offline navigation used Valhalla**, not OSRM.

---

## 3. Data pipeline

```
planet.osm.pbf ──osrm-extract -p car.lua──► *.osrm.*  (edge-based graph + geometry + guidance data)
                        │
     ┌──────────────────┴────────────────────┐
  CH: osrm-contract                         MLD: osrm-partition ─► osrm-customize
     (.osrm.hsgr)                              (.partition .cells)   (.cell_metrics .mldgr)
     └──────────────► osrm-routed / osrm-datastore / libosrm ◄───────┘
```

- `AGENTS.md` notes a dataset may be both CH and MLD: run partition, contract and customize.
- Running partition before contract also improves CH cache locality through node renumbering.

### osrm-extract
`src/extractor/extractor.cpp` (`Extractor::run` / `ParseOSMData`):

1. **Read the PBF with libosmium** (or XML). Lua `process_node/process_way/process_relation` run in parallel: a TBB pipeline with one independent Lua state per thread (`src/extractor/scripting_environment_lua.cpp`). `setup()` runs once per state (`docs/profiles.md`).
2. **Parse restrictions and maneuver overrides** (`restriction_parser.cpp`, `maneuver_override_relation_parser.cpp`). Collect obstacles: barriers, traffic signals, give-way, stop (`obstacles.cpp`). Optionally mesh pedestrian plazas (2026, experimental, `src/extractor/area/`, `docs/areas.md`).
3. **Build the node-based graph** and compress degree-2 chains (`graph_compressor.cpp`; details in §4).
4. **Edge expansion** (`edge_based_graph_factory.cpp`): for every intersection and every allowed turn, call Lua `process_turn` and create an edge-based edge. Runs as a parallel TBB pipeline (`GenerateEdgeExpandedEdges`, about line 507).
5. **Guidance pre-annotation** (`ProcessGuidanceTurns` → `src/guidance/*`).
6. **Spatial indexes and connectivity:** R-tree over edge-based node segments (`.ramIndex` holds inner nodes in RAM; `.fileIndex` holds leaves, mmapped; `include/util/static_rtree.hpp`). Then strongly connected components, which mark "tiny components" for snapping.

**Extract outputs** (`include/extractor/extractor_config.hpp`):
- **Graphs:** `.osrm.ebg` (edge-based graph), `.ebg_nodes`, `.enw` (node weights, durations, distances), `.cnbg` and `.cnbg_to_ebg` (compressed NBG plus mapping, for the partitioner).
- **Geometry:** `.geometry` (compressed segment geometry, weights, durations, datasources), `.nbg_nodes` (coordinates and OSM ids).
- **Turns:** `.turn_weight_penalties`, `.turn_duration_penalties`, `.turn_penalties_index` (`from,via,to` OSM node triples for each turn, used for turn-penalty CSV updates).
- **Restrictions:** `.restrictions` (conditional restrictions kept for later evaluation), `.maneuver_overrides`.
- **Guidance and names:** `.names`, `.tls`/`.tld` (turn lanes), `.icd` (intersection classes for guidance), `.edges`.
- **Spatial and metadata:** `.ramIndex`, `.fileIndex`, `.properties`, `.timestamp`, optional `.openareas*`.

All files are **tar archives** with named blocks (`include/storage/tar.hpp`, `include/util/mmap_tar.hpp`), which is what makes `osrm-datastore` able to place blocks into shared memory generically.

### osrm-partition (MLD, optional for CH)
`src/partitioner/partitioner.cpp`:
- Works on the **compressed node-based graph** with coordinates.
- Runs **recursive bisection by inertial flow** (`inertial_flow.cpp`, `dinic_max_flow.cpp`, `recursive_bisection.cpp`).
- Projects the node-based partition onto the edge-based graph via `.cnbg_to_ebg`, then fixes nodes left unconnected (comment: "For details see #3205").
- Renumbers the graph for locality (`renumber.cpp`).
- Builds `MultiLevelPartition` and `CellStorage`.
- **Outputs:** `.partition`, `.cells`, and rewritten `.ebg` etc. (`include/partitioner/partitioner_config.hpp`).
- **Defaults** (`partitioner_config.hpp:23-25`): `balance=1.2`, `boundary_factor=0.25`, `num_optimizing_cuts=10`, `small_component_size=1000`, `max_cell_sizes={128, 4096, 65536, 2097152}`. That is 4 levels above the base graph.

### osrm-customize (MLD)
`src/customize/customizer.cpp`, `include/customizer/cell_customizer.hpp`:
- Optionally applies traffic CSVs through the updater.
- Bottom-up per level, computes for each cell the full clique (all boundary-source × boundary-destination shortest weight, duration and distance) using a Dijkstra restricted to the cell. On levels above 1 it uses sub-cell cliques. Parallel over cells with TBB.
- Repeats this for **each exclude-class combination**, giving one `CellMetric` per filter (`customizeFilteredMetrics`).
- **Outputs:** `.cell_metrics`, `.mldgr`.

### osrm-contract (CH)
`src/contractor/contractor.cpp`, `graph_contractor.cpp`:
- Applies updates.
- Contracts the **edge-based graph**.
- **Output:** `.hsgr`, holding a single query graph plus per-exclude-filter edge masks.

### Traffic and turn-penalty updates
`src/updater/updater.cpp`, `src/updater/csv_source.cpp`. Accepted by both `osrm-customize` and `osrm-contract` (`src/tools/customize.cpp:40-80`).
- **`--segment-speed-file`:** CSV `from_osm_node,to_osm_node,speed_kmh[,rate]`. Directional, and the node pair must be adjacent. A blank rate keeps the weight while updating duration (Boost.Spirit X3 grammar, `csv_source.cpp:33-37`). Each file becomes a "datasource"; up to 255 are tracked (`include/extractor/datasources.hpp`) and surfaced through `annotations=datasources`.
- **`--turn-penalty-file`:** CSV `from,via,to,penalty_s[,weight]`, mapped through `.turn_penalties_index`.
- **`--parse-conditionals-from-now <utc>` + `--time-zone-file <geojson>`:** evaluates `restriction:conditional` opening hours (`include/util/opening_hours.hpp`, `include/util/timezones.hpp`) **once, at customise/contract time**. Conditional restrictions are not time-dependent at query time.
- **Typical live-traffic loop:** `osrm-customize --segment-speed-file …`, then `osrm-datastore --only-metric`. `osrm-routed -s` hot-swaps with no restart.
  - MLD makes this feasible: v5.7 claimed "below a minute on continental networks".
  - CH re-contraction takes hours on large extracts.

### Planet-scale requirements

**Official wiki "Disk and Memory Requirements"** (v5.26, Nov 2021 planet, 61 GiB PBF):

| Stage output (car) | Size |
|---|---|
| extract | 415 GiB |
| partition | 220 GiB |
| customize | 174 GiB |
| contract (CH) | 230 GiB |

- The foot profile is a bit larger.
- **`osrm-routed` RAM for the planet, car:** about **123 GiB**. That is roughly equal to the loaded file sizes; `.fileIndex` is mmapped and excluded.

**Preprocessing RAM and time** are not published officially. Anecdotes:
- osrm-talk 2017: planet extract about 232 min.
- CH contract for car "about 12 hours", much longer for bike, foot and custom profiles, because flatter speed hierarchies contract worse.
- MLD partition + customize is "significantly quicker".
- Plan for a 128–256 GiB RAM machine (or swap) for planet extract.

**Small-scale measured numbers** (2024 CI benchmark, Berlin extract on a Raspberry Pi, PR #6943):

| Stage | Time | Peak RAM |
|---|---|---|
| `osrm-extract` | 24.3 s | 395 MB |
| `osrm-partition` | 5.8 s | 122 MB |
| `osrm-customize` | 2.5 s | 112 MB |
| `osrm-contract` | 183 s | 184 MB |

CH contraction costs about 70× customisation.

---

## 4. Graph model

### Node-based graph to edge-based graph
- **Node-based:** OSM nodes become vertices and way segments become edges. It is then **compressed**: degree-2 nodes are merged when annotations are compatible (`src/extractor/graph_compressor.cpp`). Compression is skipped at:
  - Obstacle nodes, unless the obstacle can be expressed as an edge penalty.
  - Entry/exit via-nodes of turn restrictions (lines 35-91).
  - Differing names or annotations.
- **Obstacle penalties** such as traffic signals and barriers are folded into the compressed edge, or emitted as an artificial turn (lines 259-270).
- **Edge-based graph:** each **directed compressed edge becomes a node** (`EdgeBasedGraphFactory::InsertEdgeBasedNode`), and each **allowed turn becomes an edge** (`GenerateEdgeExpandedEdges`).
  - The edge weight is the source segment's weight plus the turn penalty (`edge_based_graph_factory.cpp:672-676`).
  - Node weights carry the segment's weight; the MSB of the node weight flags one-way streets (lines 115-121).
- **All query algorithms run on this edge-based graph.** CH contracts it and MLD partitions it, so turn costs and restrictions are exact with no special handling at query time.

### Turn restrictions
`include/extractor/restriction_graph.hpp` (long design comment), `src/extractor/restriction_graph.cpp`, `way_restriction_map.cpp`:
- Node-via restrictions simply suppress or force turns during expansion.
- Via-way restrictions, including multi-via since 5.27, are compiled into a **restriction graph**: a prefix-tree automaton with "transfer" edges between overlapping restriction paths.
- The edge-based nodes along a via-way path are **duplicated**, so the "inside a restriction" state is encoded in the graph (`GenerateEdgeExpandedNodes`, "Expanding via-way turn restrictions", lines 344-450).
- This is the classic state-expansion approach and is fully exact.
- Conditional restrictions share the structure but carry conditions (`include/extractor/restriction.hpp:27-42`).
- `type=manoeuvre` relations (maneuver overrides) change guidance output, not routing.

### Turn penalties
- Lua `process_turn(profile, turn)` gets a rich `ExtractionTurn` and writes `turn.weight` and `turn.duration`. Inputs include:
  - Angle, number of roads, u-turn, traffic light, driving side.
  - Source and target mode, class, lanes, speed and priority.
  - Roads on the left and right, and the from/via/to OSM node ids.
- See `edge_based_graph_factory.cpp:623-670`, `docs/profiles.md` "process_turn".
- The car profile uses a sigmoid on angle with left/right asymmetry by driving side, plus u-turn and signal penalties, and a huge penalty for entering restricted-access roads under `routability` (`profiles/car.lua` ~560-630).
- Penalties are stored as `int16` deciseconds (`TurnPenalty`, `include/util/typedefs.hpp:109`), so they are capped at about ±54 min.
- 2026 addition (`214ba8f9f`): driving side is resolved per way from coordinates using the **gauche** classifier (§9), so a planet extract no longer has one global `left_hand_driving`.

### Compressed geometry and segment speeds
- Per-segment geometry (node ids) and **per-direction weights and durations bit-packed in 22 bits each** (`SEGMENT_WEIGHT_BITS = SEGMENT_DURATION_BITS = 22`, `include/util/typedefs.hpp:135-141`) live in `SegmentDataContainer` (`include/extractor/segment_data_container.hpp`), plus a datasource id per segment.
- Way-level speed comes from `process_way`; Lua `process_segment` can adjust per-segment weight and duration, e.g. from raster elevation (`profiles/rasterbot.lua`, `src/extractor/raster_source.cpp`).
- Weight model: `weight = length / rate`, with `speed` driving duration separately (`docs/profiles.md` "Understanding speed, weight and rate").
- So a profile can optimise a non-time metric (e.g. `routability`, `cyclability`, `distance`) while still reporting true durations.
- **Only one weight per dataset** (`properties.weight_name`). Storage keys metrics by that name (`src/storage/storage.cpp:590-630`), but no tooling produces several.

### Phantom nodes / snapping
- An input coordinate snaps to a point on an edge-based node segment via the R-tree (`include/engine/geospatial_query.hpp`), giving a "phantom node" with forward and reverse partial weights (offsets).
- Snapping prefers big components and respects `is_startpoint`, bearings, radiuses and approach.

---

## 5. Algorithms

### Contraction Hierarchies (`src/contractor/`)
- `graph_contractor.cpp` has a very readable design comment (lines 1-110) citing Geisberger 2008 and Vetter 2009.
- **Parallel contraction:** repeatedly picks *independent node sets* (no lower-priority node within 2 hops; `IsNodeIndependent`, line ~539) and contracts them in parallel with TBB.
- **Priority:** `2·(edges added/deleted) + 4·(original edges added/deleted) + depth` (`EvaluateNodePriority`, lines 375-391).
- **Witness search:** bounded Dijkstra (`src/contractor/contractor_search.cpp`).
- **Self-loops** are preserved so a source downstream of a target on the same node can "go around".
- **Exclude classes** (`contractExcludableGraph`, line 821):
  - Contract a shared core of nodes allowed under every filter up to 90% (`BASE_CORE = 0.9`).
  - Then finish contraction separately per filter.
  - The result is one `.hsgr` with per-filter edge masks.
- **Query:** bidirectional upward Dijkstra with **stall-on-demand** (`include/engine/routing_algorithms/routing_base_ch.hpp:17-109`).
- Shortcuts store the middle "turn_id", so unpacking is cheap.
- `QueryEdge` stores weight, duration **and** distance (`include/contractor/query_edge.hpp`), so tables can return distances without unpacking.
- Core-CH (partial contraction) was removed in v6.0.

### Multi-Level Dijkstra / CRP (`src/partitioner/`, `include/customizer/`, `routing_base_mld.hpp`)
- **Partitioning: inertial flow** (`src/partitioner/inertial_flow.cpp`):
  - For `n=10` slopes, sort nodes by projected coordinate.
  - Take the first and last 25% (`boundary_factor`) as super-source and super-sink.
  - Compute a min cut with **Dinic** max-flow (`dinic_max_flow.cpp`).
  - Keep the best-balanced cut within `balance=1.2`.
  - Recurse (`recursive_bisection.cpp`), then cut the bisection tree into levels by `max_cell_sizes` (`bisection_to_partition.cpp`).
  - Partitioning is done on the node-based graph and then projected to the edge-based graph.
- **Customisation:** per cell, clique matrices of boundary-in to boundary-out weight, duration and distance (`cell_customizer.hpp`, `include/partitioner/cell_storage.hpp`). Level 1 uses base edges; higher levels use sub-cell cliques. TBB-parallel per cell, with one metric per exclude filter.
- **Query** (`include/engine/routing_algorithms/routing_base_mld.hpp`):
  - Bidirectional Dijkstra in which each node is relaxed at its *query level*: the highest level at which it differs from both source and target cells (`getNodeQueryLevel`, lines 25-178).
  - It uses clique "shortcut" edges at that level plus boundary edges.
  - Sept 2026 optimised this for many phantom candidates (`bd1d32406`).
- **Path unpacking** recursively re-runs the search inside each cell, because cliques store no middle nodes (lines 490-590).
  - This is the main per-query cost over CH.
  - Hence the 2026 "two-tier browse-resistant" LRU unpack cache (`include/util/browse_resistant_cache.hpp`, `538c23830`).

### Many-to-many (table)
- **CH:** classic **bucket-based** algorithm (Knopp et al.). Backward searches from each target deposit `NodeBucket`s; forward searches scan them (`src/engine/routing_algorithms/many_to_many_ch.cpp:103-135`). This is why CH is recommended for "very large distance matrices" (README).
- **MLD:** unidirectional multi-level Dijkstra for 1:N and N:1, and bidirectional bucket-style for M:N (`many_to_many_mld.cpp:220-480`). Benchmarks show it noticeably slower than CH tables (§8).
- **Table options:** `fallback_speed` (crow-fly fallback), `fallback_coordinate`, `scale_factor`, `annotations=duration,distance` (`docs/http.md:331-338`).

### Alternatives
- **CH:** the via-node method with T-test ("Alternative Routes in Road Networks", Abraham et al.). Constants (`src/engine/routing_algorithms/alternative_path_ch.cpp:22-24`): at most 15% longer, at most 75% shared, locally optimal on 25% sub-paths. It **returns at most one alternative**: the `number_of_alternatives` argument is ignored (line 549).
- **MLD:** via-node plus **plateau** detection on overlapping forward and backward search trees (`alternative_path_mld.cpp:366-420`).
  - Parameters (lines 21-41): ≤25% longer, ≤75% sharing, ≥10% local optimality, and a cell-based sharing pre-filter.
  - The stretch allowance is scaled by a fitted hyperbolic function of route duration (lines 69-110).
  - Supports `alternatives=n` up to `--max-alternatives`.
- **Both:** alternatives only work for exactly 2 waypoints (`src/engine/plugins/viaroute.cpp:111`).

### Map matching (HMM)
- Newson & Krumm-style HMM with Viterbi (`src/engine/routing_algorithms/map_matching.cpp`, `include/engine/map_matching/hidden_markov_model.hpp`).
  - **Emission:** Gaussian on snap distance (`sigma_z`, set per point via `radiuses`).
  - **Transition:** exponential on |network distance − great-circle distance| with `beta = 10` (`MATCHING_BETA`).
  - Network distances come from many small shortest-path searches.
- Breaks the trace into sub-matchings on gaps or timestamps (`split_points`, `MAX_BROKEN_STATES=10`, `MAX_DISTANCE_DELTA=2000`).
- Confidence from a Bayes classifier (`include/engine/map_matching/bayes_classifier.hpp`, `matching_confidence.hpp`).
- Options: `timestamps`, `gaps=split|ignore`, `tidy`, `waypoints`.

### Trip (TSP)
`src/engine/plugins/trip.cpp:243-277`:
- Computes a duration table first, then orders the stops.
- **≤15 locations:** exact Held-Karp **dynamic programming** (`include/engine/trip/trip_dynamic_programming.hpp`, new in 2026; replaced brute force ≤10).
- **More than 15:** **farthest insertion + 2-opt** (`trip_farthest_insertion.hpp`).
- Supports `roundtrip`, `source=first`, `destination=last`.
- Tarjan SCC handles unreachable subsets.

### Tile service
`src/engine/plugins/tile.cpp`:
- Mapbox Vector Tiles (via **vtzero**) of the routing graph.
- `speeds` layer: speed, weight, duration, rate, datasource, name, `is_startpoint`, `is_small`.
- `turns` layer: angle, turn weight and duration, from zoom 15 (`MIN_ZOOM_FOR_TURNS = 15`).
- A debugging and visualisation tool, used by osrm-frontend's debug view.

### Nearest
- R-tree k-nearest over segments (`include/util/static_rtree.hpp`, `include/engine/geospatial_query.hpp`).
- Multi-coordinate GET and POST since Aug 2026 (`e9436d329`).

### Pedestrian areas (2026, experimental)
`docs/areas.md`, `include/engine/area_*.hpp`, `src/extractor/area/`:
- Meshes plazas into a pruned visibility graph at extract time: shortest-path trees from entry points plus ring edges.
- Solves plaza-to-plaza at query time.
- Opt-in through `profiles/foot_area.lua`.

---

## 6. Profiles (Lua) — extract time vs query time

- **Files:** `profiles/car.lua`, `bicycle.lua`, `foot.lua` (`api_version = 4`), helpers in `profiles/lib/*.lua` (way handlers, access, maxspeed, obstacles, relations, guidance). About 4.6k lines of Lua in total.
- More contributed profiles live in `Project-OSRM/osrm-profiles-contrib`.
- **Hooks** (`docs/profiles.md`):
  - `setup()` returns config and `properties`: `weight_name`, `weight_precision`, `use_turn_restrictions`, `continue_straight_at_waypoint`, `max_speed_for_map_matching`, `classes`, `excludable`, `restrictions`, `relation_types`, `suffix_list`, `left_hand_driving`, `force_split_edges`.
  - `process_node`: barriers and obstacles, traffic signals.
  - `process_way`: speeds, rates, modes, classes, names, refs, lanes, restricted, roundabout, `is_startpoint`, road classification for guidance.
  - `process_relation`: route relations and similar.
  - `process_segment`: per-segment weight and duration from external data such as rasters.
  - `process_turn`: turn costs.
  - Location-dependent data (GeoJSON polygons injected into `way` lookups; `src/extractor/location_dependent_data.cpp`) and raster sources are also extract-time.
- **Built-in weights:** car `routability` (duration plus strong avoidance of restricted/private access); bike `duration` (`cyclability` optional); foot `duration`.

### What is decided at extract time (essentially everything)
- Accessibility per direction and mode, speeds, rates, weights.
- Turn penalties and restrictions (`use_turn_restrictions`).
- Obstacle penalties.
- Class membership.
- The single optimisation metric.
- Vehicle dimension restrictions such as maxheight and maxweight, applied in `car.lua` as hard filters at extract time.

`docs/profiles.md`: "profiles are used when preprocessing the OSM data, NOT at query time … after modifying a profile **you will need to extract, contract and reload the data again**."

### What can change without re-extracting
- **Speeds and turn penalties via CSV, followed by customise or contract.** This is the only weight-change path. Granularity is per directed OSM node pair and per turn triple. For MLD it is fast (minutes), but it is still global per dataset, not per request.
- **Conditional restrictions,** evaluated against a fixed timestamp at customise/contract time.

### What can change per request
Full list (`docs/http.md:32-51`, `286-294`, `331-338`):
- **`exclude=`** one of at most **7 pre-declared class combinations** (`MAX_EXCLUDABLE_CLASSES = 7`, at most 7 classes per dataset; `include/extractor/class_data.hpp:17-20`).
  - Each combination is pre-computed: a separate MLD cell metric, or a separate CH filter/partial hierarchy.
  - So each one costs memory, and you cannot exclude arbitrary roads, areas or tags.
  - Car ships with `toll`, `motorway`, `ferry` (`profiles/car.lua:153-162`).
- **Snapping controls:** `bearings`, `radiuses`, `approaches` (`curb`, `opposite`, `unrestricted`), `snapping=any`. `hints` are now deprecated and ignored (`include/engine/hint.hpp`, `c8c548ee3`).
- **Route shape:** `continue_straight` (forbid u-turns at via points), `waypoints` (which coordinates create legs), `alternatives`.
- **Output only:** `steps`, `annotations`, `overview`, `geometries`.
- **Table only:** `fallback_speed`, `scale_factor`.

### What is impossible per request
- Different speed or weight models (e.g. "truck 3.5 t today, van tomorrow").
- Avoiding arbitrary polygons or edges.
- Per-request turn-cost tweaks.
- Departure-time-dependent routing (no time-dependent weights).
- Shortest vs fastest switching unless you run two datasets.

The standard workaround is **one dataset per profile variant** (`osrm-extract … --output /data/car` etc., `docs/profiles.md:15-24`), each with full RAM cost. Compare GraphHopper's per-request "custom models" or Valhalla's dynamic costing, which do this at query time.

---

## 7. Features

- **Guidance / turn-by-turn:**
  - Extract time: `src/guidance/*` builds intersection analysis and turn classification. Handlers cover motorways, roundabouts, sliproads, segregated intersections, driveways and suppressed modes, plus turn-lane matching (`turn_lane_*.cpp`).
  - Query time: `src/engine/guidance/*` collapses turns, reduces verbosity and processes lanes (`collapse_turns.cpp`, `verbosity_reduction.cpp`, `lane_processing.cpp`).
  - Output: `RouteStep`s with `maneuver{type, modifier, bearing_before/after, exit}`, `intersections[]` (bearings, entry, lanes, classes), `rotary_name`, `destinations`, `exits`, `ref`, `pronunciation`, `driving_side`.
  - **Text** is not generated by the backend. It comes from the separate JS library **osrm-text-instructions** (`Project-OSRM/osrm-text-instructions`, about 40 languages, still pushed in 2026).
  - Mapbox's "banner/voice instructions" were proprietary additions, not part of OSRM.
- **Annotations** (`annotations=`): per-segment `nodes` (OSM ids), `distance`, `duration`, `weight`, `speed`, `datasources`. `datasources` tells which traffic CSV supplied a segment speed.
- **Traffic updates:** via customise/contract plus datastore hot-swap (§3). Static snapshots only; there are no time-dependent profiles.
- **Exclude classes:** see §6.
- **Approaches:** `curb` and `opposite` use the per-way driving side (now geographic via gauche).
- **Waypoints and snapping:** `radiuses` (default unlimited, or `--default-radius`), `bearings` (value, range), `snapping`, `skip_waypoints`, `generate_hints` (deprecated). Waypoint objects return snapped location, name and distance.
- **Other:**
  - `overview=simplified|full|false|by_legs`, `geometries=polyline|polyline6|geojson`.
  - FlatBuffers responses.
  - `--max-*` size limits on every service.
- **Not supported:** isochrones, elevation output, time-dependent routing, public transit, multimodal routing. Out of scope by design.

---

## 8. Performance

### Query latency (order of magnitude)
- **Typical figure:** sub-millisecond to a few milliseconds per route for CH on continental graphs. The OSM wiki claims "usually below 1 ms for Europe" for the search itself; JSON assembly and guidance often dominate.
- **MLD vs CH:** MLD route queries are roughly **2–5× slower** than CH. A 2017 maintainer said "about 5x"; current e2e numbers show a smaller gap for full requests.

**Reproducible numbers:** the 2024 CI benchmarks on a **Raspberry Pi**, Berlin extract (PR #6943, "Benchmark Results"):

| Benchmark (Node bindings, ops/s) | CH | MLD |
|---|---|---|
| route | 1017 | 568 |
| table (25 coords) | 193 | 42 |
| trip | 190 | 69 |
| match | 210 | 291 |
| nearest | ~10k | ~10k |

- **HTTP e2e route** (overview=full, steps=true): mean 4.6 ms (CH) vs 5.5 ms (MLD) on the Pi.
- **Map matching** of 82 coordinates: 3–7 ms.
- **Takeaway:** CH wins clearly on tables (about 4.6× here). MLD is competitive for single routes and wins on matching.

### Preprocessing
- **CH** is costly: hours to a day for the planet; 183 s vs 2.5 s customise on Berlin/Pi.
- **MLD** customise is cheap enough for frequent traffic updates.
- **Extract** dominates the MLD pipeline.

### Memory
- Everything that is needed is RAM-resident (or mmap/shm): about 123 GiB for planet car.
- `--disable-feature-dataset ROUTE_STEPS,ROUTE_GEOMETRY` trims guidance/geometry memory for table-only servers.
- Each extra exclude combination adds a full metric (MLD) or filter (CH).

### Comparison with GraphHopper and Valhalla
No rigorous, current, apples-to-apples public benchmark was found. Community consensus and architecture-based reasoning:

| Dimension | OSRM | GraphHopper | Valhalla |
|---|---|---|---|
| Raw route latency | fastest (CH) | close with CH; slower with LM/flexible | slowest (tens of ms typical, A*-based on tiles) |
| Matrices | best-in-class (CH buckets) | good (CH) | OK (time-distance matrix, slower) |
| Query-time costing | minimal (exclude ≤7 combos) | custom models per request (LM/flex mode) | full dynamic costing per request |
| Live traffic | MLD customise + hot swap | limited | historical and live speeds in tiles |
| Memory | high, monolithic | moderate | low; tiled, loads on demand |
| Mobile/offline | not viable | feasible (Java/Android) | designed for it (tiles) |
| Transit/multimodal | no | PT module (GTFS) | multimodal (transit via transitland) |

- Blog posts such as sumguy.com 2026 give unsourced ranges like "OSRM single-digit ms, Valhalla/GraphHopper 15–80 ms" and USA RAM estimates. These are not methodical; their 8–12 GB USA RAM for OSRM seems low against the official 123 GiB planet figure.
- Sources: <https://sumguy.com/osrm-vs-valhalla-vs-graphhopper/>, Geofabrik 2017 slides <https://www.geofabrik.de/media/2017-03-23-Routing-Engines_fuer_OSM.pdf>.

---

## 9. Architecture and code quality

### Layout
Each stage is a library and a thin CLI in `src/tools/*.cpp`.

```
src/ & include/
  extractor/   OSM parsing, Lua scripting env, NBG build/compress, restrictions, EBG expansion, R-tree, areas
  guidance/    extract-time intersection/turn classification, lanes (merged into osrm_extract lib in 2025)
  partitioner/ inertial flow, Dinic, recursive bisection, MultiLevelPartition, CellStorage
  customizer/  cell clique computation, metrics per exclude filter
  contractor/  CH contraction, witness search, exclude-aware contraction
  updater/     CSV traffic/turn updates, conditional restriction evaluation
  storage/     tar-based file IO, shared memory layout, osrm-datastore
  engine/      datafacades (CH/MLD views over mmap/shm), routing_algorithms/, plugins/ (viaroute, table,
               nearest, trip, match, tile), guidance post-processing, api/ (JSON+FlatBuffers builders),
               map_matching/, trip/
  server/      Boost.Beast HTTP, URL/POST parsing (Boost.Spirit), service dispatch
  util/        static graph, dynamic graph, packed vectors, query heaps, static R-tree, cheap_ruler,
               opening_hours, timezones, JSON container
  osrm/        public facades (libosrm, osrm_extract, …)
  nodejs/ python/  bindings
  tools/       CLIs: osrm-extract/partition/customize/contract/routed/datastore/components
  benchmarks/  micro & e2e benchmarks
profiles/      Lua profiles + lib/
features/      Cucumber behavioural tests (232 .feature files, ~1571 scenarios)
unit_tests/    Boost.Test (124 files)
fuzz/          libFuzzer targets for URL/parameter parsers
```

### Design patterns worth noting
- **Data-oriented, compile-time polymorphism:**
  - Algorithms are templated on `Algorithm` tags (`ch::Algorithm`, `mld::Algorithm`) and a `DataFacade<Algorithm>` (`include/engine/datafacade/contiguous_internalmem_datafacade.hpp:606-709`).
  - Containers are templated on `Ownership::{Container, View}`, so the same struct is both the builder and a zero-copy view over mmapped or shared-memory blocks. See `include/extractor/segment_data_container.hpp`; this is a nice pattern to copy.
- **Strong typedefs** for weights, durations and distances (`osrm::Alias`, `include/util/alias.hpp`, `typedefs.hpp:98-141`).
- **Code-quality signals:**
  - Coding standards in `AGENTS.md`: no exceptions or RTTI outside IO, warnings as errors, `BOOST_ASSERT` liberally, clang-format-22, clang-tidy.
  - Sanitizer CI configs (`scripts/ci/*sanitizer.conf`), fuzzers, a benchmark CI job.
- **Rough edges:**
  - Huge files such as `extractor.cpp`, with a "TODO: Refactor this function" on `Extractor::run`.
  - Duplicated doc tables (the `process_way` table appears twice in `docs/profiles.md`).
  - The wiki is stale (still describes `--core` contraction and STXXL).
  - Lots of historical complexity in guidance.

### Dependencies
`vcpkg.json`, `CMakeLists.txt:315-478`:
- **Boost ≥1.83:** asio, beast, geometry, interprocess, iostreams, program_options, spirit, test, and more.
- **oneTBB, Lua 5.4 + sol2, libosmium + protozero + expat + bzip2 + zlib.**
- **Other:** rapidjson, flatbuffers, libarchive, vtzero, fmt (fallback for `std::format`).
- **gauche:** a Rust crate (`Project-OSRM/gauche-rs`, BSD-2) for left/right-hand-traffic classification. It is compiled **Rust → wasm32 → C via w2c2 → C++ wrapper** (`vcpkg-overlay-ports/gauche/portfile.cmake`, used by `src/extractor/driving_side_index.cpp`). This is the project's first Rust component, and an interesting precedent for mixing Rust in without a Rust toolchain at build time.

### Build and CI
- CMake ≥3.18 with presets (`CMakePresets.json`).
- vcpkg builds all dependencies from source on first configure; there are overlay ports for sol2, vtzero and gauche.
- Docker images on GHCR are Debian-based (moved from Alpine, June 2026).
- CI matrix (`.github/workflows/osrm-backend.yml`): many Ubuntu 26.04 GCC/Clang variants, ubuntu-arm, macOS arm64, Windows 2025.

### Tests
- **Cucumber behavioural suite** (`features/`; runner in `features/support`, `features/step_definitions`, `cucumber.mjs`).
  - Scenarios draw ASCII node maps, declare ways, relations and tags in tables, then assert routes, turns, speeds, guidance, etc. (e.g. `features/car/restrictions.feature`).
  - About 1.5k scenarios covering car, bike and foot tagging semantics, restrictions, guidance, options and nearest.
  - **Extremely valuable as an executable spec for any new router.** The format is engine-agnostic enough to port: an OSM fixture generator plus expected route names.
- **Unit tests:** Boost.Test in `unit_tests/` per module.

### Ease of embedding or porting pieces
- **Embedding the whole engine:** OK from C++, painful from other languages (C++ shim + Boost/TBB linking). Node and Python are first-class.
- **Porting algorithms:** the CH contractor, inertial-flow partitioner, cell customiser and MLD query are each a few hundred to about 1500 LOC and fairly self-contained (`src/contractor/graph_contractor.cpp`, `src/partitioner/*.cpp`, `include/customizer/cell_customizer.hpp`, `include/engine/routing_algorithms/routing_base_mld.hpp`).
- **Harder to port:** extraction semantics (restrictions, obstacles, the compression rules, the guidance pipeline). Lots of OSM-specific edge cases accumulated over 15 years, but well covered by the Cucumber tests.

---

## 10. Strengths, weaknesses, suitability

### Strengths
- **Speed:** top-tier query latency; best open-source distance matrices (CH buckets).
- **Exact turn handling:** edge-based graph gives exact turn costs and complex (multi-via-way, conditional) restrictions with no query-time special cases.
- **Two mature speedup techniques** in one codebase. MLD gives fast metric customisation and zero-downtime traffic updates (`osrm-datastore --only-metric`).
- **Proven services:** guidance and turn-by-turn, HMM map matching, trip, vector tiles.
- **Scale:** proven at planet scale and in high-QPS production (Mapbox historically, osm.org, VROOM users).
- **Permissive licence** (BSD-2).
- **Very large behavioural test corpus.**
- **Active again in 2026,** with monthly releases and Python bindings.

### Weaknesses
- **Rigid costing:** any profile change needs re-extract (hours for the planet). Per-request flexibility is limited to ≤7 pre-baked exclude combinations plus snapping parameters. There are no vehicle parameters, avoid-areas or time-dependence at query time.
- **One metric per dataset;** multiple profiles means multiple full datasets in RAM.
- **Heavy and monolithic:** about 123 GiB RAM for planet car, no tiling, unsuitable for mobile/offline.
- **Heavy C++ dependency stack;** no C ABI.
- **No dataset compatibility across monthly releases.**
- **Governance:** single-maintainer risk, a high share of AI-generated changes, and a stale wiki.
- **Alternatives:** CH gives only one alternative; MLD queries and unpacking are slower than CH.
- **Scope:** no transit or multimodal, no isochrones, no elevation output.

### (a) As a dependency for `routet`
- **Good fit if:**
  - The new project is a server-side car, bike or foot router/matrix service.
  - It can treat profiles as deploy-time configuration.
  - It is fine running a sidecar `osrm-routed` (HTTP) or Node/Python bindings.
- **Pin a monthly version and re-process data on each upgrade.**
- **From Rust:** either talk HTTP or FlatBuffers to `osrm-routed` (simplest and robust), or maintain a C++ shim over libosrm (fragile builds).
- **Poor fit if:** you need per-request costing, an embedded/mobile/offline router, small memory, multimodal/transit, or a pure-Rust build.

### (b) As a reference to learn from or port
**Highly recommended reading order:**
1. `src/contractor/graph_contractor.cpp` (design comment + parallel independent-set contraction).
2. `include/engine/routing_algorithms/routing_base_ch.hpp` (stall-on-demand) and `many_to_many_ch.cpp` (buckets).
3. `src/partitioner/inertial_flow.cpp` + `dinic_max_flow.cpp` + `recursive_bisection.cpp`.
4. `include/customizer/cell_customizer.hpp` + `include/partitioner/cell_storage.hpp`.
5. `include/engine/routing_algorithms/routing_base_mld.hpp` (query level, clique relaxation, unpacking).
6. `include/extractor/restriction_graph.hpp` + `src/extractor/edge_based_graph_factory.cpp` (turn expansion with duplicated via-way nodes).
7. `src/extractor/graph_compressor.cpp` (what must not be compressed).
8. `src/engine/routing_algorithms/alternative_path_mld.cpp` (plateau alternatives with tuned constants).
9. `src/engine/routing_algorithms/map_matching.cpp` + `include/engine/map_matching/*` (compact HMM matcher).
10. `features/**/*.feature` as a conformance suite.

**Ideas worth porting to a Rust design:**
- **The `Ownership::View/Container` pattern:** one type is both builder and zero-copy mmap view. It maps naturally onto Rust (`rkyv`, `zerocopy`, or custom `&[T]` views).
- **Tar-of-named-blocks dataset format** for generic mmap and shared-memory loading.
- **CSV speed and turn updates keyed by OSM node pairs and triples**, with a `datasource` annotation per segment.
- **Exclude-classes-as-bitmask** with per-combination metrics (cheap in MLD).
- **Splitting extract-time profile logic from query-time cost.** But note that OSRM's biggest limitation comes from putting *all* cost into extract time. A new design might keep OSRM's edge-based graph and MLD/CRP, but customise at query time (CCH) or on demand per profile.

### Notable forks and derived projects
- **`fossgis-routing-server/osrm-backend`** (+ scripts): runs openstreetmap.org's routing.
- **`Telenav/osrm-backend`:** large fork (2019–2023) that adds about 0.8 MB of Go services on top of the C++ core; now stale.
- **`woltapp/osrm-backend`, `nammayatri/osrm-backend`:** company forks (delivery, ride-hailing).
- **Ecosystem:** VROOM (VRP solver using OSRM tables/routes), `Project-OSRM/osrm-frontend` (Leaflet UI, active), `osrm-text-instructions`, `osrm-profiles-contrib`.
- **Bindings:** R packages `osrm` and `osrm.backend`; Rust crates `osrm-binding`, `rsc_osrm` (FFI) and `osrm_client` (HTTP).
- **Rust:** `Project-OSRM/gauche-rs`, a Rust component consumed via wasm→C.
- **Mapbox** historically ran an internal fork behind its Directions API.

---

## Sources
- Repo source (paths above), git history and tags of `~/g/osrm-backend`.
- GitHub API repo stats (2026-10-23).
- OSRM wiki: [Disk and Memory Requirements](https://github.com/Project-OSRM/osrm-backend/wiki/Disk-and-Memory-Requirements), [Running OSRM](https://github.com/Project-OSRM/osrm-backend/wiki/Running-OSRM), [Traffic](https://github.com/Project-OSRM/osrm-backend/wiki/Traffic). The last two are partly stale.
- PR #6943 benchmark results (Raspberry Pi, Berlin): <https://github.com/Project-OSRM/osrm-backend/pull/6943>.
- osrm-talk 2017, planet contract times: <https://lists.openstreetmap.org/pipermail/osrm-talk/2017-September/001533.html>.
- Mapbox ❤️ Valhalla (2018): <https://blog.mapbox.com/mapbox-%EF%B8%8F-valhalla-1fdda95ac4f>.
- Mapzen OSRM sunset: <https://www.mapzen.com/blog/osrm-sunset>.
- Geofabrik routing: <https://www.geofabrik.de/en/data/routing.html>.
- Debian tracker: <https://tracker.debian.org/osrm-backend>.
- gauche-rs: <https://github.com/Project-OSRM/gauche-rs>.
- Rust crates: <https://docs.rs/crate/osrm-binding/1.0.0>, <https://lib.rs/crates/rsc_osrm>.
- Comparison blog (low rigour): <https://sumguy.com/osrm-vs-valhalla-vs-graphhopper/>.
