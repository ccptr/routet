# Routing libraries, algorithm implementations, and smaller engines (Rust focus)

Research date: 2026-10-23. Scope: libraries, algorithm implementations, and small or little-known engines. The Rust ecosystem gets the most attention. Big engines (Valhalla, OSRM, GraphHopper, ORS, BRouter, Organic Maps, OsmAnd, Navit, transit) are covered in other reports and appear here only as context.

Source clones I read are under `~/g/`: `fast_paths`, `rust_road_router`, `RoutingKit`, `routing2` (Itinero 2), `routing` (Itinero 1), `osm4routing` (old C++/Python), `osm4routing2` (Rust), `ferrostar`, `cch-rodeapps`, `cch-rs`, `routingkit-cch`, `routee-compass`, `valhalla-rs`, `routx`, `ch-go` (LdDl/ch), `abstreet`, `routino-dl/routino-3.4.4` (release tarball; upstream uses SVN).

Activity data comes from the GitHub API and crates.io on 2026-10-23. Star counts are approximate.

---

## 1. TL;DR

- **No mature, end-to-end, OSM-to-route Rust routing engine exists** that compares with OSRM, Valhalla or GraphHopper. Rust has good *building blocks*: PBF parsers, CH/CCH cores, and graph libraries. Rust also has good *wrappers* around the C++ engines. A new Rust engine (`routet`) would fill a real gap.
- **Best algorithm cores to build on or learn from:**
  - **RoutingKit** (C++, BSD-2). The canonical, compact and readable CH + CCH implementation, with an OSM importer. It is active again in 2026.
  - **rust_road_router** (Rust, BSD-3, KIT). A research framework with the best engineered CCH, CATCHUp (time-dependent), CH-potentials, ALT, HL, RPHAST and multi-criteria search. It is the reference code for the 2025 *CCH survey*. It is dormant since 2022 and nightly-only, so treat it as a **reference, not a dependency**.
  - **fast_paths** (Rust, MIT/Apache). A simple, stable, node-based CH. It has no turn costs. A/B Street uses it.
  - **`cch` crate (Rodeapps)** (Rust, MIT, 2026). A pure-Rust port of RoutingKit's CCH, differential-tested against RoutingKit to be bit-identical, with mmap bundles. It is very young but the most promising *dependency-grade* CCH in Rust. **cch-rs** (METIS-based, incremental updates) and **routingkit-cch** (cxx bindings) are alternatives.
- **Turn restrictions and turn costs** are the most common gap in small libraries. The two working approaches:
  - Edge-based (turn-expanded, "line graph") graphs. This is what the CCH survey, rust_road_router, osm2ch, A/B Street and Itinero 2 do.
  - Node duplication ("phantom nodes") for restrictions only. This is what routx/pyroutelib3 do.
- **Itinero 2** (C#) is the most interesting *design* reference among smaller engines. It is tile-based, uses plain Dijkstra/A* with query-time profiles and an edge-based search, supports in-place network edits, and does map matching. It is very active in Oct 2026.
- **Routino** (C, AGPL) is a long-lived small engine. It has a simple "super-node/super-segment" hierarchy with A*, released v3.4.4 in Aug 2026. Read it as a lesson in practical OSM tagging, but its licence and design rule it out as a dependency.
- **Ferrostar** contains **no router**. It is a Rust-core turn-by-turn navigation SDK that consumes OSRM-format, Valhalla or GraphHopper responses. A `CustomRouteProvider` lets it take routes from a local engine. If routet emits OSRM-compatible JSON, Ferrostar (and MapLibre Navigation) can drive navigation UIs on top of it.
- **Algorithm landscape, for a new engine:** **CCH** (with a turn-expanded graph and InertialFlowCutter/inertial-flow ordering) is the best match. It handles query-time customization (traffic, per-user weights) and has fast queries (≈0.1–0.3 ms continental), a simple implementation, and open references in both C++ and Rust. MLD/CRP (OSRM) is the main alternative. Hub Labels / TNR / Arc-Flags have no maintained open road implementations worth depending on.

---

## 2. Rust ecosystem

### 2.1 fast_paths — `easbar/fast_paths`
| | |
|---|---|
| Language / licence | Rust, MIT OR Apache-2.0 |
| Activity | v1.0.0 released 2024-05-04; last commit 2024-05-07; ~295★; ~32k downloads. Stable but not evolving. |
| Scope | Library: CH preprocessing and queries on an abstract directed graph |
| Algorithms | Node-based CH (witness search with settled-node limits, priority = edge-quotient + hierarchy depth); bidirectional Dijkstra over the CH; many-sources/many-targets with initial weights; Floyd–Warshall and Dijkstra for tests |
| Input | `InputGraph::add_edge(from, to, weight)` with dense `usize` node ids. It does **not** parse OSM. |
| Turn restrictions | None in the library. Users must build an edge-based graph themselves (A/B Street does this, see below). |
| Customization | None. Changing weights means re-preparation, but `prepare_with_order()` reuses the node ordering, which is ~5× faster. |
| Persistence | `FastGraph` is serde-serializable; `serialize_32`/`deserialize_32` make wasm32-friendly output |
| Performance (README, M1 Max) | DIMACS USA (24M nodes) travel time: prep 5.5 min, query 305 µs. NY: 6 s / 26 µs. My run on the Bremen test map (40k nodes): heuristic prep 300 ms, prep with fixed order 59 ms, avg query 11–13 µs. |

**Key files:**
- `~/g/fast_paths/src/fast_graph_builder.rs` (contraction driver, `Params`)
- `src/node_contractor.rs` (shortcut and relevance computation)
- `src/witness_search.rs`
- `src/path_calculator.rs` (bidirectional query + stall-free unpacking)
- `src/fast_graph.rs` (CSR up/down graphs, `replaced_in_edge`/`replaced_out_edge` for unpacking)

**Assessment:**
- (a) **As a dependency:** OK for prototypes, analytics and small regions. It is the easiest CH in Rust. There is no turn awareness, no customization, `usize` weights only, and a memory layout that is not tuned (it stores `base_node` per edge). It does not grow into a full engine.
- (b) **As a reference:** Excellent for learning CH. It is about 4k LOC, readable, and has good tests (it checks results against Dijkstra on random graphs).

**Real-world user, A/B Street** (`~/g/abstreet/map_model/src/pathfind/vehicles.rs`, `engine.rs`):
- It builds an **edge-based graph**: graph nodes are `DirectedRoad`s plus "uber-turns" (turn sequences through intersection clusters), and edges are allowed movements.
- Then it calls `fast_paths::prepare`.
- On map edits it calls `prepare_with_order_with_params` with the old ordering.
- This is a good, small worked example of turn-aware routing on top of a node-based CH.
- A/B Street itself is in maintenance mode (last commit 2025-09).

### 2.2 rust_road_router — `kit-algo/rust_road_router`
| | |
|---|---|
| Language / licence | Rust (nightly), BSD-3-Clause; InertialFlowCutter submodule (BSD-3) |
| Activity | `master` last commit **2022-11-08**. Only dependabot branches since (the 2026 "pushed" date comes from dependabot). ~44★. It is the research code of Tim Zeitz / Ben Strasser / KIT, behind the CATCHUp, CH-Potentials and CCH-survey papers. |
| Scope | Research framework: engine library, Rocket HTTP server, HERE/Mapbox converters, experiment binaries |
| Algorithms | Dijkstra (many variants, incl. time-dependent and multi-criteria), A*, **ALT**, **CH** (contraction + query; node ordering is not implemented), **CCH** (heavily engineered: separator decomposition, triangle-enumeration customization, parallel customization, perfect customization, directed CCH, elimination-tree queries, nearest-neighbour/POI with lazy RPHAST), **CATCHUp** (exact time-dependent CCH), **TD-Sampling**, **CH-Potentials** (perfect A* potentials from a CH/CCH lower bound, used for live-traffic and turn penalties), **Hub Labels** computed from a CH, **RPHAST**, **Topocore**, traffic-aware/live routing |
| Input | RoutingKit binary vector format (`first_out`, `head`, `travel_time`, `latitude`, ...), produced by RoutingKit's `osm_extract` or by its own HERE/Mapbox converters. It has **no native OSM parser**. |
| Turns | Via line-graph expansion: `datastr::graph::line_graph(&graph, turn_cost_fn)` plus a directed CCH (`engine/src/bin/cch_with_turns.rs`). It also has OSM turn expansion experiments (`cchpp/src/bin/turn_expand_osm.rs`). |
| Customization | Yes. This is the whole point of CCH: re-customize per metric in about 1 s for Europe (parallel). |
| Performance | CCH survey (arXiv 2502.10519, 2025), Europe DIMACS: IFC ordering 341 s, contraction ~20 s, customization 10.8 s sequential / 1.25 s on 16 threads, queries **0.14 ms** (perfect customization, travel time), versus CH 0.11 ms and CRP 1.65 ms |

**Key files:**
- `engine/src/algo/customizable_contraction_hierarchy/{mod.rs, contraction.rs, customization.rs, customization/parallelization.rs, customization/directed.rs, query.rs, query/stepped_elimination_tree.rs, separator_decomposition.rs}`
- `engine/src/algo/catchup/*`
- `engine/src/datastr/graph/floating_time_dependent/*` (piecewise-linear TTF machinery)
- `engine/src/algo/ch_potentials/*`, `engine/src/algo/{alt.rs, hl.rs, rphast.rs, topocore.rs}`
- `engine/src/bin/cch.rs` (minimal end-to-end example)
- Shell glue for ordering: `flow_cutter_cch_order.sh`

**Assessment:**
- (a) **As a dependency:** Not recommended. It needs nightly features (`slice_group_by` was removed from nightly, so it likely no longer builds without patches). It has an experiment-oriented API, does no OSM import, and has had no maintenance for 4 years.
- (b) **As a reference:** **The single best Rust reference** for state-of-the-art customizable and time-dependent road routing. Read the CCH module alongside the CCH survey paper. The cchpp code accompanies the survey.

### 2.3 `cch` (Rodeapps) — `Rodeapps/cch`
| | |
|---|---|
| Language / licence | Rust (stable, MSRV 1.85), MIT (NOTICE credits RoutingKit) |
| Activity | Created **2026-06**, v0.3.0 on 2026-07-14, ~16★, ~5.6k downloads. One main author. It was extracted from a production distance-matrix service. |
| Scope | Library: full CCH pipeline |
| Algorithms | `degree_order` and `inertial_order` (inertial-flow nested dissection, the same quality as RoutingKit), CCH build, (parallel) customization with buffer reuse, distance / one-to-many / many-to-many matrix, node path, elimination-tree query |
| Input | CSR `Graph { first_out, head, weight }`. No OSM parsing. |
| Turns | None built in. A turn-expanded graph would work in principle, since CCH runs unmodified on expanded graphs. |
| Persistence | Memory-mappable `.cch-struct` / `.cch-metric` bundles, **byte-compatible with RoutingKit** |
| Correctness | Differential tests against vendored C++ RoutingKit (dev-only), bit-identical arrays, 100% line coverage enforced in CI |
| Performance | Parity with RoutingKit on small grids. Albania-sized network: customize 367 ms on 1 thread, 81 ms on 18 threads. |

**Key files:**
- `~/g/cch-rodeapps/src/{structure.rs, customize.rs, query.rs, path.rs, bundle.rs, writer.rs}`
- `src/order/nd/{inertial.rs, flow.rs, fragment.rs}`
- `tests/equivalence.rs`

**Assessment:**
- (a) **As a dependency:** The **most attractive pure-Rust CCH core today**. It has a stable toolchain, no FFI and mmap serving. It is very young with a bus factor of 1, so pin the version, or vendor and fork it.
- (b) **As a reference:** A clean, idiomatic Rust transliteration of RoutingKit's CCH, smaller than rust_road_router.

### 2.4 cch-rs — `wmsnp/cch-rs`
- Rust 2024, MIT/Apache, created 2026-03, v0.2.0, ~5★, ~1.7k LOC.
- CCH with **METIS** ordering (via `metis-sys`, a C dependency), rayon parallel customization, **partial re-customization** (`update_weights_partial`), **incremental topology updates**, and an `ArcSwap` snapshot model for wait-free reads during updates. It uses `f32` weights and named profiles.
- Claims (USA-COL, 435k nodes): init 1.3 s; query + path 20 µs, versus 52 µs for RoutingKit; partial single-edge update 6.6 µs.
- **Assessment:** These are interesting ideas for a live-traffic service: partial updates and snapshot swapping. It is tiny and very young. Better as a reference for the update/concurrency design than as a dependency. `cch` (Rodeapps) is the more rigorous core.

### 2.5 routingkit-cch — `HellOwhatAs/routingkit-cch`
- Rust and Python bindings to RoutingKit's CCH via `cxx`. BSD-2, v0.1.4 (2026-07).
- Ordering helpers, sequential/parallel customization, partial update, one-to-many / many-to-one, node and arc path extraction.
- **Assessment:** Use it if you want the C++ original with Rust ergonomics. It brings a C++ build dependency but no large dependency tree.

### 2.6 Other Rust CH crates (new, tiny)
- **faster_paths** (`christianstaib/faster_paths`, 2026-05, Apache): "parallel CH preprocessing". 0★, experimental.
- **ch-router** (`tBuLi12/ch-router`, 2025): CH with path unfolding and "hot groups". No licence on GitHub.
- **osmgraphing** (`dominicparga`, archived 2022): a master-thesis project (OSM parsing, Dijkstra/CH, multi-metric balancing). Dead.

None of these is dependency-grade.

### 2.7 routx — `MKuranowski/routx`
| | |
|---|---|
| Language / licence | Rust, MIT; C and C++20 bindings, Python bindings (`routx-python`) |
| Activity | v1.1.0 released 2026-06-28. Small, by the author of pyroutelib3 (whose Rust successor it is). |
| Scope | Small end-to-end library: OSM (XML/PBF, gz/bz2) → in-memory graph → A* route |
| Algorithms | A* only (plus a variant without turn-arounds), k-d tree nearest node, RDP simplification |
| Turn restrictions | **Yes.** `no_*` and `only_*` restrictions, via phantom/cloned nodes (`src/osm/mod.rs` `find_phantom_node` tests). Profiles define access, oneway and penalties (`src/osm/profile.rs`). Conditional restrictions are not supported. |
| Graph | `BTreeMap<i64, (Node, Vec<Edge>)>` keyed by OSM id. Simple, not cache-efficient. |

**Assessment:**
- (a) **As a dependency:** Fine for small areas, scripts and embedded use from C/Python.
- (b) **As a reference:** A concise, honest example of OSM interpretation: profiles, access tags, oneway handling, restriction relations to graph surgery.

### 2.8 osm4routing2 — `Tristramg/osm4routing2` (now `rust-transit/osm4routing2`)
- Rust 2024, MIT, v0.8.x (repo commits 2026-03), ~95★, ~162k downloads (the most downloaded OSM-to-graph crate).
- Converts PBF into `nodes.csv` / `edges.csv`, or into in-memory `Vec<Node>, Vec<Edge>`. Each edge has per-mode accessibility (car/bike/foot, forward/backward), geometry, an optional `merge_ways()` contraction of degree-2 joints, and tag filters (`reject`/`require`/`read_tag`).
- **No turn restrictions, no routing.**
- Key files: `~/g/osm4routing2/src/osm4routing/{reader.rs, categorize.rs, models.rs}`.
- The old `Tristramg/osm4routing` (C++/Python, last commit 2021) is superseded.
- **Assessment:** A good *ingest* dependency, or a template for your own ingest step. You will outgrow it once you need turn restrictions, barriers, or rich attributes (speeds, conditionals).

### 2.9 RouteE Compass — `NatLabRockies/routee-compass` (NREL, now "National Lab of the Rockies")
| | |
|---|---|
| Language / licence | Rust core + Python API, BSD-3-Clause |
| Activity | Very active (v0.19.5 2026-06; commits 2026-09). Funded lab project, ~29★, ~18k downloads. |
| Scope | **Full Rust routing engine** (library + CLI + Python), energy-aware |
| Algorithms | Label-setting Dijkstra/A* over a **state vector** (distance, time, energy, SOC, …), k-shortest paths (Yen's, single-via-paths), map matching, termination models |
| Customization | Fully query-time: traversal models, cost models (weighted blends of time/distance/energy/cost) and constraint models are pluggable per query. Powertrain energy models predict consumption per edge. |
| Turns | Turn-restriction constraint model and turn-delay traversal model (`rust/routee-compass-core/src/model/constraint/default/turn_restrictions/`, `model/traversal/default/turn_delays/`) |
| Input | Datasets generated in Python from **OSMnx** graphs (`python/nrel/routee/compass/io/generate_dataset.py`). It does not parse PBF natively. |
| Speed-ups | No CH. Plain search, parallel across queries, nation-sized networks held in RAM. |

**Assessment:**
- (a) **As a dependency:** Only if research and energy routing is the goal. Not a production consumer router.
- (b) **As a reference:** The **best Rust example of a pluggable, state-based cost/constraint model architecture** (traits for traversal/cost/constraint/label, builders, config-driven). Worth reading for the API design of query-time profiles. Key directory: `rust/routee-compass-core/src/model/*`, `src/algorithm/search/*`.

### 2.10 graphways — `kyleloving/graphways` (formerly `osm_graph`)
- Rust core with a PyO3 Python API, MIT, v0.4–0.5 (2026). Builds a `SpatialGraph` from a place or bbox (downloads from Overpass).
- Isochrones, reachability, routing. The README says it respects turn restrictions and uses CH for point-to-point routing.
- Aimed at urban analytics, like OSMnx but fast. Small (~19★).
- Reference only.

### 2.11 Wrappers and bindings for the big engines (Rust)
- **`valhalla`** (`kinkard/valhalla-rs`, MIT, v0.7.2 on 2026-10-21, active). `cxx` bindings that vendor and build Valhalla. It offers tile access (`GraphReader`, `DirectedEdge`, `NodeInfo`), **read/write live traffic on mmapped traffic.tar**, the Actor API (route/matrix/…), and a typed `ConfigBuilder` (no Python). It cannot build tiles (mjolnir is disabled), so you need Valhalla's tools for that.
  - **This is the pragmatic "Rust + Valhalla in-process" path.**
- **`valhalla-client`** (jelmer, Apache) is an HTTP client.
- **Rallista/valhalla-mobile** is an iOS/Android build of Valhalla used with Ferrostar for offline routing.
- **OSRM:**
  - `osrm-binding` (blob-map, 2025–26; embedded route/table/trip; no licence file on GitHub, so check before use)
  - `osrm_interface` (FFI + HTTP + mock)
  - `rs_osrm` / `rsc_osrm` (NextBillion fork, updated 2026-10)
  - `osrmreader` (reads `.osrm` files, 2020)
  - `osrm-model-extensions` (OSRM response model types)
- No serious **pure-Rust port** of OSRM or Valhalla exists.

### 2.12 Ferrostar — `stadiamaps/ferrostar`
| | |
|---|---|
| Language / licence | Rust core (`common/ferrostar`), UniFFI → Swift/Kotlin, wasm for web, React Native; BSD-3-Clause |
| Activity | Very active (v0.59.0 on 2026-10-22), ~444★, Stadia Maps |
| Scope | **Navigation SDK, not a router.** It handles route-state tracking, step/waypoint advance, deviation detection and recalculation triggers, simulation, recording, and UI components. |
| Routing | `routing_adapters/` has request generators and response parsers for **Valhalla**, **OSRM-format** (`osrm/`) and **GraphHopper**. `RouteProvider.CustomProvider` / `CustomRouteProvider` (Kotlin/Swift) handle local or offline route generation, for example Valhalla-mobile. |

**Key files:**
- `~/g/ferrostar/common/ferrostar/src/routing_adapters/{mod.rs, valhalla.rs, graphhopper.rs, osrm/}`
- `navigation_controller/`
- `deviation_detection.rs`
- `android/core/src/main/java/com/stadiamaps/ferrostar/core/RouteProvider.kt`

**Assessment:**
- (a) **As a dependency:** If routet ever wants a navigation client, Ferrostar is the obvious pairing. Emit OSRM-compatible JSON (with Valhalla's banner/voice extensions) or write a Ferrostar adapter.
- (b) **As a reference:** Its Rust route model (`models.rs`) documents what a navigation client needs from a router: steps, maneuvers, annotations, spoken/visual instructions.

### 2.13 Headway — `headwaymaps/headway` (was `michaelkirk/headway`)
- Self-hostable maps stack, Apache-2.0, ~3k★, active (2026-10).
- It uses **Valhalla** for driving/walking/cycling, **OpenTripPlanner** for transit, Pelias for geocoding, and planetiler/tileserver for tiles.
- Its Rust service **`travelmux`** (`services/travelmux`) multiplexes Valhalla and OTP behind one API.
- No routing algorithms of its own. It is useful as an integration and deployment example.

### 2.14 General graph crates
- **petgraph** (MIT/Apache, v0.8.3, ~570M downloads): `dijkstra`, `astar`, `bellman_ford`, `floyd_warshall`, `johnson`, `k_shortest_path`, `spfa`, and more. These are generic and not tuned for road graphs (no bidirectional search, no speed-up techniques, heavyweight graph types). Fine for tooling and tests.
- **pathfinding** (evenfurther, MIT/Apache, v4.16.0): generic successor-function API (`astar`, `dijkstra`, `bfs`, `fringe`, `idastar`, `yen`, Kuhn–Munkres, …). Great for prototypes, but closure-driven searches are slow on 10⁷-node graphs.
- **OSM I/O in Rust:**
  - `osmpbf` (MIT/Apache, parallel PBF reading, mmap support; the usual choice)
  - `osmpbfreader` (WTFPL, used by osm4routing2; has convenient relation/dependency fetching)
  - `osm-lump-ways` (topology grouping)
  - `openlr` (OpenLR location referencing, useful for traffic feeds)

### 2.15 Other newly discovered tiny projects (2025–2026)
These are all experiments, not dependencies. Some look partly AI-generated. They are listed for completeness.
- `mengchheanglong/osm-pathfinder` (Rust, Axum, CH/A*/TDSP, created 2026-10)
- `NicolasDevel/roadpack` (CH graph for ESP32 on microSD, 2026-10)
- `sage-ice-cream/wayseeker-rs`
- `bbox-routing-server` (bbox-services, Rust, uses fast_paths 0.2; OGC-style routing service, 2024)
- `stefan855/osmnavi` (C++, MIT; moved to codeberg `stefanchr/osm-router`). An interesting hobby design: edge-based Dijkstra/A*, Louvain clustering for a two-level overlay, labels for via-way restrictions, mmapped graph, Raspberry Pi target.

---

## 3. C++ libraries

### 3.1 RoutingKit — `RoutingKit/RoutingKit`
| | |
|---|---|
| Language / licence | C++17, BSD-2-Clause |
| Activity | ~483★. Revived in 2026: Ben Strasser merged car-speed and area-filter PRs in Apr 2026, plus nested-dissection memory work in Jan 2026. Otherwise slow. |
| Scope | Library: CH, CCH, Dijkstra, OSM importer, geo nearest-neighbour, polyline, DIMACS converters, CLI tools |
| Algorithms | CH (own ordering, contraction, path/pinned/many-to-many queries, "extra weight"), **CCH** (inertial-flow nested dissection ordering, sequential and parallel customization, perfect customization, partial customization), Dijkstra |
| OSM | `osm_simple.h`: `simple_load_osm_car_routing_graph_from_pbf()`, plus pedestrian and bicycle variants. A custom PBF decoder (`src/protobuf.cpp`, `osm_decoder.cpp`, `buffered_asynchronous_reader.cpp`); `osm_profile.cpp` (car speeds, access, oneway); `osm_graph_builder.cpp` (routing-node detection, geometry modelling). |
| Turns | `decode_osm_car_turn_restrictions()` outputs `forbidden_turn_from_arc/to_arc`. CH/CCH do **not** consume these directly. You expand the graph yourself (rust_road_router does this). |
| Customization | Yes (CCH) |
| Performance | CCH paper (Dibbelt, Strasser, Wagner, JEA 2016). Continental queries take sub-millisecond time. |
| API ergonomics | Excellent. A 40-line OSM→CH→query example in the README. SoA vectors (`first_out`, `head`, `weight`) throughout. |

**Key files:**
- `~/g/RoutingKit/src/customizable_contraction_hierarchy.cpp` (1.9k LOC)
- `src/contraction_hierarchy.cpp` (2.2k)
- `src/nested_dissection.cpp`
- `src/osm_graph_builder.cpp`, `src/osm_profile.cpp`
- Headers in `include/routingkit/`
- Docs in `doc/*.md`

**Assessment:**
- (a) **As a dependency:** Solid, if C++ via cxx is acceptable (see `routingkit-cch`, or port, as `cch` did).
- (b) **As a reference:** **The best compact reference** for CH, CCH and a minimal OSM importer. If you read only one codebase before writing routet, read this one.

### 3.2 Partitioning and ordering tools
- **InertialFlowCutter** (`kit-algo/InertialFlowCutter`, C++, BSD-3, last 2023). Computes CCH nested-dissection orders. It gives the best order quality per the CCH survey and takes about 6 min for Europe. It is the rust_road_router submodule.
- **FlowCutter PACE17** (`kit-algo/flow-cutter-pace17`, BSD-2) is its predecessor.
- **KaHIP** (`KaHIP/KaHIP`, C++, MIT, v3.25 on 2026-03, active). High-quality graph partitioning, including nested-dissection node ordering. **Mt-KaHyPar** is its parallel hypergraph sibling.
- **METIS** (Apache-2.0) is classic and used by cch-rs through `metis-sys`. Its orders are lower quality than IFC for road networks, but it is easy to get.
- **Pure Rust ordering:** `cch::inertial_order` (Rodeapps) reimplements RoutingKit's inertial flow. No pure-Rust FlowCutter or IFC exists yet.

### 3.3 Other C++ code
- **KaTCH** (`GVeitBatz/KaTCH`, AGPL-3, last 2021): time-dependent CH (TCH).
- **michaelwegner/CRP** (MIT, 2016): an academic CRP implementation. OSRM's MLD is the production CRP-like implementation (other report).
- **iwiwi/pruned-landmark-labeling** (2014): HL for general and social graphs, not road-tuned.
- **libosmium** (BSL-1.0, active). The standard C++ OSM I/O library, used by OSRM and osm2pgrouting. No routing.
- **osm-oscar/path_finder** (Apache, 2024, 0★): HL/CH experiments from Stuttgart. Negligible.

---

## 4. C# — Itinero

### 4.1 Itinero 1 — `itinero/routing`
- C#, Apache-2.0, ~233★. Last code commit 2023-09; NuGet `Itinero` 1.6.0-pre037. Effectively frozen.
- It had a node- and **edge-based CH** (`src/Itinero/Algorithms/Contracted/{HierarchyBuilder.cs, FastHierarchyBuilder.cs, EdgeBased/, Dual/}`), many-to-many, Lua profiles, OSM import, and `routerdb` files.
- A reasonable reference for "CH inside a small .NET engine", including edge-based CH.

### 4.2 Itinero 2 — `itinero/routing2`
| | |
|---|---|
| Language / licence | C# (.NET), Apache-2.0 |
| Activity | **Very active.** About 50 commits since 2025, many in Sep–Oct 2026: isochrones, alternatives, corridor routing, perf work. One main developer (xivk / Ben Abelshausen, ANYWAYS). Package version 2.0.46-alpha. ~23★. |
| Scope | Library/engine: tiled routing network, OSM import, routing, instructions, map matching, Lua or C# profiles |
| Algorithms | Deliberately **no CH/CCH/HL**. Uses Dijkstra, bidirectional Dijkstra and A*, edge-based when turn costs are enabled, plus one-to-many/many-to-many, isochrones, alternative routes via corridor search, HMM map matching (Newson & Krumm), and reachability/island detection. |
| Data model | **Tiles** (zoom-14-style Mercator tiles) that hold topology, geometry, attributes and turn costs (`src/Itinero/Network/Tiles/NetworkTile*.cs`). They load on demand and support in-place mutation and diffs (`RoutingNetwork.Mutation.cs`, `NetworkTile.Diffs.cs`). |
| Customization | **Full query-time profiles.** `Profile.Factor(attributes)` and `TurnCostFactor(attributes)` (`src/Itinero/Profiles/Profile.cs`) are cached per edge type (`ProfileCached.cs`). |
| Turns | OSM turn restrictions are parsed (`src/Itinero.IO.Osm/Restrictions/Turns/OsmTurnRestrictionParser.cs`) into per-tile turn-cost tables (`NetworkTile.TurnCosts.cs`, `Network/TurnCosts/OrderCoder.cs`). Barriers are handled too. |
| Search core | `src/Itinero/Routing/Flavours/Dijkstra/Dijkstra.cs` (edge-based search, turn costs, u-turn filtering, "main-network" pruning), `Bidirectional/BidirectionalDijkstra.cs` |

**Assessment:**
- (a) **As a dependency:** Only for .NET users.
- (b) **As a reference:** **A very relevant design reference for routet** if flexibility (live edits, per-request profiles, routable tiles) matters more than continental query speed. Its README is an honest essay on that trade-off. The tile-based mutable network, edge-type caching of profile evaluation, and corridor-based alternatives are all worth studying.

---

## 5. Go

- **LdDl/ch** (Go, Apache-2.0, v1.11.0 on 2026-10-03, ~55★, active):
  - Node-based CH with bidirectional queries, one-to-many and many-to-many, isochrones, thread-safe query pools.
  - "Recustomize" support, i.e. lightweight weight updates on a fixed hierarchy (`recustomize.go`).
  - A turn-restricted vanilla Dijkstra (`vanilla_dijkstra_turn_restricted.go`).
  - Key files: `~/g/ch-go/{contraction.go, bidirectional_ch*.go, recustomize.go, isochrones.go}`.
- **LdDl/osm2ch** (Apache, 2026-06):
  - PBF → `osm2gmns` (Go port) macroscopic network with turning movements.
  - **Edge expansion** (links become vertices, movements become edges).
  - CH preparation, then CSV output. Supports `no_*`/`only_*` turn restrictions and filters U-turns.
  - This is the clean "turn restrictions via expanded graph + plain CH" pipeline.
- LdDl also maintains `horizon` (map matching on top of `ch`).
- **Assessment:** A good small reference for an end-to-end OSM → expanded graph → CH pipeline and its CSV formats. Go GC and the `map`-heavy structures limit scale. It is regional-scale software.

---

## 6. SQL / Python / R ecosystem (analytics-oriented)

- **pgRouting** (C++ in PostgreSQL, GPL-2.0, v4.0.2 on 2026-09, ~1.4k★, very active):
  - SQL functions over edge tables: Dijkstra, A*, bidirectional variants, `pgr_withPoints` (snapping to edge fractions), driving distance (catchment), TRSP (turn-restricted shortest path), K-shortest, TSP, flow, and a `pgr_contraction` that is graph simplification, not a query-time CH.
  - Every query builds a Boost graph from SQL, so latency is milliseconds to seconds. Great for GIS workflows, poor for serving.
  - v4.0 cleaned up the API (C++17, signature changes).
  - **osm2pgrouting** (GPL-2.0, libosmium, 2026-01) imports OSM into pgRouting topology tables.
  - **osm4routing2** can also feed PostgreSQL (`create_tables.sql`, `import_postgres.sh`).
- **OSMnx** (Python, MIT, v2.1.x, ~5.9k★):
  - Downloads OSM via Overpass into NetworkX MultiDiGraphs, with simplification, speeds/travel times, and stats.
  - Routing is NetworkX Dijkstra/A*. No turn restrictions. Small or medium cities only.
  - It is the data source for RouteE Compass.
- **pyrosm** (Python, MIT, v0.8 on 2026-08): fast PBF → GeoDataFrames, exporting to networkx, igraph or pandana graphs.
- **Pandana** (UDST, C++/Cython, AGPL-3.0, v0.8 on 2026-08):
  - CH (code derived from OSRM's old CH by Dennis Luxen) for fast many-to-many and accessibility aggregations (nearest-POI, aggregate within distance).
  - Now a "stable compatibility library". New development moves to **Pandarm**.
- **igraph / NetworkX:** generic. NetworkX is pure Python and slow. igraph's C core is fast for Dijkstra but has no road-specific features.
- **r5py** (Python wrapper of Conveyal's Java R5, GPL-3.0/MIT-ish mix, v1.1.7): multimodal accessibility and travel-time matrices, transit focused (other report). Mentioned for completeness.
- **pyroutelib3** (Python, GPL-3.0, v2.0.0 in 2024): simple OSM routing with A* and turn restrictions via phantom nodes. Superseded by the same author's **routx** (Rust).

---

## 7. Other small engines and closed or dead projects

| Project | Status | Notes |
|---|---|---|
| **Routino** (C, AGPL-3.0, routino.org, SVN) | **Alive.** v3.4.4 on 2026-08-22 (3.4.2/3.4.3 in 2025). | `planetsplitter` preprocesses OSM (XML/PBF/O5M) into a custom DB. The router (`src/optimiser.c`) runs Dijkstra from start and end to "super-nodes", **A*** on a super-segment graph (iterated contraction of degree-2/same-property chains; `src/superx.c`), and Dijkstra to expand. Simple (way-node-way) turn restrictions, barriers, weight/height/width/length limits, per-highway preferences and speeds (`xml/routino-profiles.xml`), tagging rules (`xml/routino-tagging.xml`), multilingual instructions. Docs: `doc/ALGORITHM.txt`, `doc/DATA.txt`. ~33k LOC. A good reference for pragmatic OSM tagging and a "poor man's hierarchy". AGPL and its design make it a poor dependency. |
| **gosmore** / **YOURS** | Dead (gosmore ~2014; yournavigation.org gone) | Historical only |
| **Graphast** (Java) | Dead | Skip |
| **pyroutelib3** | Superseded by routx | See above |
| **CycleStreets** | Engine closed (own in-house); public API plus open-source R/Python clients | "Fastest/quietest/balanced" cycle routes. Nothing to reuse. |
| **Kurviger** | Closed. Its own extension of GraphHopper OSS, per the Kurviger forum. | Curvy-road motorcycle routing. GraphHopper is in the other report. |
| **Magic Lane** (Magic Earth SDK) | Proprietary SDK | Note only |
| **Atlas** and similar navigation apps | Closed | Note only |
| **MapLibre Navigation** (Android Java/MIT, iOS Swift; active 2026) | **No router.** It consumes OSRM/Mapbox-Directions-format routes (Valhalla OSRM-compat output works). | Like Ferrostar, it is a client to target with OSRM-compatible output |
| **A/B Street** (Rust, Apache) | Maintenance mode (2025-09) | Its pathfinding (fast_paths over an edge-based graph with uber-turns) is a great small worked example |

---

## 8. State of the art in road-routing algorithms, and open implementations

Rough continental numbers are for Europe (DIMACS, ~18M nodes, travel time) on a 2010s server. They come mainly from the 2025 CCH survey and the classic Bast et al. 2016 "Route Planning in Transportation Networks" survey.

| Technique | Idea | Prepro / customization | Query (Europe) | Flexibility | Open implementations |
|---|---|---|---|---|---|
| **Dijkstra / bidir. Dijkstra** | baseline | none | ~2 s | total | everywhere (petgraph, pathfinding, pgRouting, Itinero 2, RouteE, routx) |
| **A\*** (geo heuristic) | goal direction | none | ~1 s (weak for travel time) | total | Routino, routx, Itinero 2, RouteE, Valhalla (bidir A* + hierarchies) |
| **ALT** (A*, Landmarks, Triangle inequality) | landmark lower bounds | minutes; landmarks are metric-dependent but stay valid as lower bounds if weights only grow | ~10–100 ms | high | GraphHopper "LM" (other report), rust_road_router `algo/alt.rs` |
| **Arc Flags** | per-region edge flags | hours | <1 ms | low | No maintained open road implementation; academic only |
| **Reach / REAL** | prune by reach | hours | ~1 ms | low | Mostly superseded by CH; academic |
| **Highway Hierarchies** | | | | | Superseded by CH; Valhalla's "hierarchies" are a heuristic cousin |
| **CH** (Contraction Hierarchies, Geisberger et al. 2008) | contract nodes in importance order, add shortcuts; bidirectional upward search | ~2–10 min per metric | **~0.1 ms** | low (re-contract per metric; fixed-order re-contraction is faster) | OSRM (CH mode), GraphHopper, RoutingKit, fast_paths, LdDl/ch, rust_road_router (query+contraction), Pandana, Itinero 1, faster_paths/ch-router |
| **Edge-based / turn-aware CH** | CH on turn-expanded graph or with turn tables | ~2–3× node-based | ~0.2–0.5 ms | low | OSRM, GraphHopper (edge-based CH), Itinero 1 EdgeBased, osm2ch + LdDl/ch, A/B Street + fast_paths |
| **CCH** (Customizable CH, Dibbelt/Strasser/Wagner 2014/2016) | metric-independent nested-dissection order and chordal supergraph; per-metric customization via triangle enumeration; elimination-tree or Dijkstra query | order ~6 min (IFC) + contraction ~20 s, once; **customization ~1 s (16 threads)** | **~0.14–0.3 ms** | **high** (weights can change at any time, partial updates) | **RoutingKit**, **rust_road_router** (+ cchpp, the CCH survey reference), **`cch` (Rodeapps, pure Rust)**, **cch-rs**, **routingkit-cch**. Turn costs via expanded graph (the survey calls this the only competitive option). |
| **CRP / MLD** (Customizable Route Planning, Delling et al. 2011; multilevel overlay) | multilevel partition (PUNCH/KaHIP/inertial flow), cell clique matrices, turn-aware natively via turn tables | partition minutes; customization ~1 s | ~1–2 ms | high | **OSRM MLD** (production; other report), michaelwegner/CRP (academic 2016), osmnavi (hobby, Louvain-based two-level variant) |
| **Hub Labels (HL)** (Abraham et al. 2011) | per-node forward/backward label sets; query = sorted merge | hours, from CH | **~1 µs** | none (labels are per metric) | rust_road_router `algo/hl.rs` (labels from CH), iwiwi PLL (general graphs). No maintained production road HL. Memory ~20+ GB continental. |
| **Transit Node Routing (TNR)** | precomputed tables between "access nodes" for long queries | hours | ~1–3 µs (long-range) | none | No maintained open implementation (academic only) |
| **PHAST / RPHAST** | one-to-all / one-to-many sweeps over CH/CCH order | as CH/CCH | one-to-all ~100 ms; batched matrices are very fast | as base | rust_road_router `algo/rphast.rs`, lazy RPHAST in CCH (POI queries, survey); RoutingKit pinned queries; OSRM/GraphHopper matrices (other reports) |
| **CH-Potentials / A\* with CH lower bounds** | use CH/CCH on lower-bound metric as perfect-ish A* heuristic for richer metrics (live traffic, penalties, TD) | as CH/CCH | ms range | high | rust_road_router `algo/ch_potentials/` |
| **Time-dependent:** TCH / TD-CH, **CATCHUp**, TD-CRP, TD-S | piecewise-linear travel-time functions on shortcuts | heavy; CATCHUp customization ~minutes | ~1 ms exact (CATCHUp) | medium | **KaTCH** (TCH, C++, AGPL, 2021), **rust_road_router** (CATCHUp, TD-S, TD-Dijkstra/A*, traffic-aware). TD-CRP has no open code. Valhalla does TD via historical speeds in A* (other report). |
| **Multi-criteria / Pareto** | label-setting with dominance | none | slow (s) | high | rust_road_router (multicriteria Dijkstra), RouteE Compass (state-vector labels) |

**Practical conclusions for a new engine:**
1. **CCH over a turn-expanded graph** gives query-time customizable weights (traffic, vehicle profiles, user preferences) at CH-like query speed.
   - Implementation complexity is moderate. The core (contraction + customization + elimination-tree query) is a few thousand LOC: see RoutingKit, `cch`, and the rust_road_router CCH module.
   - The hard parts are elsewhere:
     - a good nested-dissection order (port inertial flow, or call InertialFlowCutter/KaHIP)
     - turn expansion (memory 2–3×; the survey describes extra preprocessing steps that keep the slowdown near the expansion factor)
     - OSM interpretation (restrictions, barriers, conditionals)
     - snapping/phantom nodes, and instructions
2. **CH** is simpler and has the fastest queries, but needs per-profile re-contraction. Fixed-order re-contraction (fast_paths `prepare_with_order`) eases that.
3. **MLD/CRP** (OSRM) is the main production-proven alternative for customizability with native turn tables. Its queries are ~10× slower than CCH. Its partitioning can double as an offline tiling scheme.
4. **Dijkstra/A\* on a compact, tiled graph** (Itinero 2, Valhalla-style, Routino-style) maximizes flexibility and editability. It is enough for city or regional scale, and can be accelerated later with CH-potentials or ALT.
5. HL, TNR and Arc-Flags are not worth implementing first: they are memory-heavy and inflexible, and lack maintained open references.

---

## 9. Comparison table

| Project | Lang | Licence | Last activity | Kind | Algorithms | Parses OSM | Turn restrictions | Query-time customization | Dependency? | Reference? |
|---|---|---|---|---|---|---|---|---|---|---|
| fast_paths | Rust | MIT/Apache | 2024-05 (v1.0.0) | lib | CH | no | no (DIY edge-based) | no (fixed-order re-prep) | ✔ small/prototype | ✔✔ CH basics |
| rust_road_router | Rust nightly | BSD-3 | 2022-11 | research fw | Dijkstra, A*, ALT, CH, **CCH**, CATCHUp, TD-S, CH-pot, HL, RPHAST | no (RoutingKit format) | line-graph expansion | ✔✔ | ✘ (bitrot) | ✔✔✔ SOTA |
| cch (Rodeapps) | Rust | MIT | 2026-07 (v0.3) | lib | CCH + inertial order | no | DIY expansion | ✔✔ | ✔ (young) | ✔✔ |
| cch-rs | Rust | MIT/Apache | 2026-03 | lib | CCH (METIS), partial/incr. | no | DIY | ✔✔ | ~ | ✔ (update design) |
| routingkit-cch | Rust (cxx) | BSD-2 | 2026-07 | bindings | RoutingKit CCH | no | DIY | ✔✔ | ✔ (C++ build) | – |
| routx | Rust | MIT | 2026-06 | small lib | A* | ✔ | ✔ (phantom nodes) | profiles at load | ✔ small areas | ✔ OSM interp. |
| osm4routing2 | Rust | MIT | 2026-03 | ingest | – | ✔ | ✘ | – | ✔ ingest | ✔ |
| RouteE Compass | Rust+Py | BSD-3 | 2026-09 | engine | Dijkstra/A*, KSP, map matching | via OSMnx | ✔ | ✔✔ (pluggable models) | research use | ✔✔ cost-model API |
| graphways | Rust+Py | MIT | 2026-10 | analytics lib | CH, isochrones | via Overpass | ✔ (claimed) | ~ | ~ | ~ |
| valhalla (crate) | Rust/C++ | MIT (Valhalla MIT) | 2026-10 | bindings | Valhalla | via Valhalla tools | ✔ | ✔ | ✔✔ pragmatic | – |
| Ferrostar | Rust core + Swift/Kotlin/web | BSD-3 | 2026-10 | nav SDK | none (adapters) | – | – | – | ✔ nav client | ✔ route model |
| Headway / travelmux | Rust (+stack) | Apache | 2026-10 | stack | uses Valhalla+OTP | – | – | – | – | deployment |
| petgraph / pathfinding | Rust | MIT/Apache | 2025/2026 | generic | Dijkstra, A*, BF, Yen… | no | no | – | tooling | – |
| RoutingKit | C++17 | BSD-2 | 2026-04 | lib | CH, **CCH**, Dijkstra | ✔ | outputs forbidden turns | ✔✔ (CCH) | ✔ (C++) | ✔✔✔ |
| InertialFlowCutter | C++ | BSD-3 | 2023 | tool | ND ordering | – | – | – | ✔ offline tool | ✔ |
| KaHIP | C++ | MIT | 2026-09 | lib/tool | partitioning, ND | – | – | – | ✔ offline tool | – |
| KaTCH | C++ | AGPL-3 | 2021 | lib | TCH | no | no | TD | ✘ | ✔ TD |
| Itinero 2 | C# | Apache | **2026-10** | engine/lib | Dijkstra, A*, bidir; edge-based; map matching; isochrones; alternatives | ✔ | ✔ | ✔✔ (profiles per query, mutable tiles) | .NET only | ✔✔ design |
| Itinero 1 | C# | Apache | 2023-09 | engine/lib | CH (node+edge-based) | ✔ | ✔ | ✘ | ✘ | ✔ |
| LdDl/ch + osm2ch | Go | Apache | 2026-10 | lib + tool | CH, recustomize, isochrones | ✔ (osm2ch) | ✔ (edge expansion) | partial | Go only | ✔ pipeline |
| pgRouting | C++/SQL | GPL-2 | 2026-10 (v4.0.2) | DB ext | Dijkstra, A*, TRSP, KSP, DD… | osm2pgrouting | ✔ (TRSP) | ✔ (SQL costs) | GIS workflows | ~ |
| OSMnx | Python | MIT | 2026-07 (v2.1) | analytics | NetworkX | via Overpass | ✘ | ✔ (slow) | analytics | – |
| Pandana | C++/Py | AGPL-3 | 2026-08 (v0.8) | analytics | CH (old OSRM) | no | ✘ | ✘ | ✘ | – |
| pyroutelib3 | Python | GPL-3 | 2024 (v2.0) | small lib | A* | ✔ | ✔ | – | ✘ (→routx) | ~ |
| Routino | C | AGPL-3 | 2026-08 (v3.4.4) | engine | A* on super-graph + Dijkstra | ✔ | ✔ (simple) | ✔ (profiles at query) | ✘ (AGPL) | ✔ tagging/hierarchy |
| osmnavi / osm-router | C++ | MIT | 2026-10 (codeberg) | hobby engine | edge-based Dijkstra/A*, Louvain clusters | ✔ | ✔ (incl. via-ways) | ✔ | ✘ | ✔ ideas |

✔✔✔ = top recommendation, ✔✔ = strong, ✔ = useful, ~ = marginal, ✘ = no.

---

## 10. Recommendations for `routet` (Rust)

1. **Read first:**
   - RoutingKit (`src/customizable_contraction_hierarchy.cpp`, `src/osm_graph_builder.cpp`, `src/osm_profile.cpp`)
   - the CCH survey (arXiv 2502.10519)
   - rust_road_router's CCH, CH-potential and line-graph code
   - fast_paths for a minimal Rust CH
   - Itinero 2's README and tile/profile design, for the flexibility trade-off
2. **Likely architecture:**
   - Rust OSM ingest with `osmpbf`, modelled on osm4routing2, routx and RoutingKit's profile logic, and adding barriers and restriction relations (including via-way).
   - A turn-expanded graph (or compact turn tables, if going the MLD route).
   - A CCH core: depend on or fork `cch` (Rodeapps), or write your own using it and RoutingKit as oracles. Ordering via the inertial-flow port first, with InertialFlowCutter/KaHIP as optional offline tools.
   - Per-profile and live-traffic customization.
   - OSRM-compatible JSON output so Ferrostar and MapLibre Navigation work out of the box.
3. **Escape hatch:** if the aim is a product rather than an engine, embed Valhalla via the `valhalla` crate, or OSRM via `osrm-binding`/`rs_osrm`, and spend effort elsewhere.
4. **Test oracles:**
   - plain Dijkstra (fast_paths-style randomized tests)
   - RoutingKit (bit-identical CCH, as `cch` does)
   - DIMACS graphs, plus Geofabrik extracts such as Bremen or Monaco for quick runs

---

## Sources
- Repos (GitHub API metadata and local clones): easbar/fast_paths, kit-algo/rust_road_router, kit-algo/InertialFlowCutter, kit-algo/cchpp, RoutingKit/RoutingKit, Rodeapps/cch, wmsnp/cch-rs, HellOwhatAs/routingkit-cch, MKuranowski/routx, rust-transit/osm4routing2, NatLabRockies/routee-compass, kyleloving/graphways, kinkard/valhalla-rs, stadiamaps/ferrostar, headwaymaps/headway, itinero/routing, itinero/routing2, LdDl/ch, LdDl/osm2ch, a-b-street/abstreet, KaHIP/KaHIP, GVeitBatz/KaTCH, michaelwegner/CRP, iwiwi/pruned-landmark-labeling, pgRouting/pgrouting, pgRouting/osm2pgrouting, gboeing/osmnx, UDST/pandana, pyrosm/pyrosm, MKuranowski/pyroutelib3, r5py/r5py, stefan855/osmnavi, maplibre/maplibre-navigation-android, maplibre/maplibre-navigation-ios, Rallista/valhalla-mobile.
- crates.io API (search and crate stats), NuGet flat-container API (Itinero versions).
- [Customizable Contraction Hierarchies — A Survey (Bläsius, Buchhold, Wagner, Zeitz, Zündorf, arXiv 2502.10519, 2025)](https://arxiv.org/abs/2502.10519)
- [Routino homepage (v3.4.4, licence, features)](https://www.routino.org/), plus the source tarball `routino-3.4.4.tgz` (`doc/ALGORITHM.txt`)
- [pgRouting 4.0 release notes](https://docs.pgrouting.org/4.0/en/release_notes.html), [pgRouting 4.0.2 announcement](https://discourse.osgeo.org/t/release-of-pgrouting-version-4-0-2/155135)
- [Pandana README (status, Pandarm successor)](https://raw.githubusercontent.com/UDST/pandana/main/README.md)
- [Kurviger forum on GraphHopper usage](https://forum.kurviger.com/t/graphhopper/25749)
- [CycleStreets OSM wiki](https://wiki.openstreetmap.org/wiki/CycleStreets)
- [Hub labels (Wikipedia)](https://en.wikipedia.org/wiki/Hub_labels), [Abraham et al., hub-based labeling (MSR)](https://www.microsoft.com/en-us/research/publication/a-hub-based-labeling-algorithm-for-shortest-paths-on-road-networks/)
- Local benchmark: `cargo test --release -- --ignored` on fast_paths Bremen maps (rustc 1.99.1; target dir in the scratchpad, repo untouched).
