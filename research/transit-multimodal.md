# Public-transit and multimodal routing engines: a survey

*Researched 2026-10-23. I read the source of shallow clones in `~/g/` (the paths below are relative to each repo root unless they start with `~/g/`). Activity figures come from the GitHub API on the research date. Claims marked **(unverified)** come from project marketing or secondary sources and I did not check them against code or benchmarks.*

---

## 0. TL;DR

| | MOTIS 2 (+ nigiri, osr) | OpenTripPlanner 2 | R5 | Navitia | Rust crates (Catenary, vulture, blaise, ferrobus, Solari) |
|---|---|---|---|---|---|
| Language | C++23 | Java 25 | Java | C++ (kraken) + Python (jormungandr) | Rust |
| License | MIT (adr geocoder is **AGPL-3.0**) | LGPL-3.0 | MIT | AGPL-3.0 | MIT / Apache / Blue Oak; Catenary backend is AGPL |
| Activity | very high (commits daily, v2.11.4 on 2026-10-21) | very high (v2.10.0 on 2026-09-09; 2.11-SNAPSHOT) | moderate (v7.6 on 2026-08-01) | maintenance (v15.120.1). Hove said it would close the repos, but the public one is still being pushed to | small hobby/early projects. Catenary is the most active |
| Shape | server (REST/OpenAPI, also MCP and OJP); nigiri and osr are usable as C++ libraries | server (GraphQL: GTFS API and Transmodel API); Raptor is a separable Java module | analysis library/worker; r5py (Python) and r5r (R) wrap it | server cluster (kraken + jormungandr + tyr + ed DB) | libraries/servers |
| Transit inputs | GTFS (+Flex, Fares v2), NeTEx, HRD | GTFS, NeTEx (Nordic/EPIP), GTFS Flex | GTFS | NTFS (via transit_model), GTFS | GTFS only |
| Real time | GTFS-RT, SIRI-ET/SX/FM, VDV-454 AUS, GBFS, elevator status (FaSta) | GTFS-RT TU/VP/alerts, SIRI-ET/SX/VM, GBFS, vehicle parking | none (scenario patches instead) | Kirin (GTFS-RT, COTS) + Chaos disruptions | none |
| Transit algorithm | RAPTOR (range/interval), **PONG** (default), **Trip-Based** with preprocessing, **CUDA GPU RAPTOR** | Range RAPTOR + **McRaptor** (arrival, transfers, duration, generalized cost c1, c2), heuristics pass, transfer optimizer | Range RAPTOR with Monte Carlo frequencies; McRAPTOR for suboptimal paths | RAPTOR (multi-pass) | RAPTOR (vulture, blaise, ferrobus, Solari, Catenary); rRAPTOR in ferrobus |
| Street routing | **osr** (its own engine: mmap'd OSM-as-graph, Dijkstra / A* / bidirectional A*, 10 profiles) | its own object graph + A* (`street`, `astar` modules) | its own column-store graph + Dijkstra | its own (kraken), or external Valhalla, Asgard, HERE or Geovelo | Solari uses Valhalla tiles + fast_paths CH; Catenary uses **osr through osr-ffi** |
| Scale | the whole world on one box (Transitous: planet OSM + ~2,900 GTFS + ~1,060 GTFS-RT + 225 GBFS feeds on a 190 GB server) | region/country (docs: Finland ≈10 GB, Germany ≈45 GB heap) | city/region, distributed analysis workers | country/region coverages | small/regional |

**Main conclusion.** For a new project, MOTIS (nigiri + osr) is the most modern and resource-efficient open stack. It is MIT-licensed and actively developed, and its data-oriented design (cista mmap containers, traffic-day bitfields, profile-defined overlay graphs) is the best reference to learn from. OTP2's `raptor` module is the most thoroughly documented McRaptor implementation (a generalized-cost criterion and a clean SPI). For Rust specifically, there is no mature Rust transit engine yet. Catenary has already bridged to osr from Rust (`catenarytransit/osr-ffi`, via `cxx`), which shows a "Rust core + osr for streets" path is practical.

---

## 1. Algorithm primer (with the open implementations of each)

### 1.1 Timetable models
- **Time-expanded / time-dependent graphs.** These are the classical approaches: one node per event, or one node per stop with time-dependent edges, searched with Dijkstra or multi-criteria label setting. They are flexible but slow. Open examples: GraphHopper's `reader-gtfs` (`MultiCriteriaLabelSetting.java`, time-expanded), Valhalla's `src/thor/multimodal*.cc` (time-dependent), OTP1 (A* over a graph that included transit; removed in OTP2), and MOTIS 1's `routing` module.
- **Array/route-based models** (RAPTOR family, CSA, Trip-Based). The timetable is stored as flat arrays. These dominate modern engines.

### 1.2 RAPTOR family (Delling, Pajor, Werneck 2012)
- **RAPTOR.** The search proceeds in rounds, and round *k* gives the best arrival time using *k* trips. Each round scans every *route* (a group of trips with an identical stop sequence and no overtaking) that serves a stop improved in the previous round, "hopping on" the earliest catchable trip. Footpaths are then relaxed. It needs no preprocessing and no priority queue, and memory access is cache-friendly. It is Pareto-optimal in (arrival, transfers).
- **rRAPTOR / Range RAPTOR.** This answers profile queries over a departure-time window. It runs RAPTOR for each departure time from latest to earliest and reuses labels ("self-pruning"), producing all Pareto-optimal (departure, arrival, transfers) journeys.
- **McRAPTOR.** Bags of labels per stop allow arbitrary extra criteria (fare, walking, cost). It is much slower; OTP reports roughly 80 ms for RR versus 400 ms for McRR with 4 criteria and about 1 s with walking distance added (`raptor/router/src/main/java/org/opentripplanner/raptor/package.md`).
- **ULTRA** (KIT). This preprocesses "shortcut" transfers so that unlimited walking/cycling between stops stays exact without a full transitive closure. Variants exist for RAPTOR, CSA and TB.
- **Open implementations:** nigiri (`include/nigiri/routing/raptor/raptor.h`, 1,571 LoC of templates), OTP (`raptor/` module, ~23k LoC), R5 (`profile/FastRaptorWorker.java`), Navitia (`source/routing/raptor.cpp`), KIT ULTRA (`Algorithms/RAPTOR/*`: RAPTOR, McRAPTOR, MCR, ULTRA-RAPTOR, UP-RAPTOR), and in Rust vulture, blaise, ferrobus (rRAPTOR), Solari and Catenary.

### 1.3 PONG (nigiri-specific, "ping-pong")
`src/routing/raptor/pong.cc` in nigiri works as follows:
1. A forward RAPTOR "ping" finds the earliest arrival for a departure.
2. A backward RAPTOR "pong" from that arrival finds the latest departure that still achieves it.
3. The search then steps to the next departure time ("PLAY" loop).

This yields Pareto (departure, arrival, transfers) journeys over an interval without the full rRAPTOR iteration over every distinct departure. It is the MOTIS default (`openapi.yaml` `algorithm` enum: `RAPTOR | PONG | TB`, default `PONG`). I am not aware of a published paper on it (unverified).

### 1.4 Connection Scan Algorithm (CSA) (Dibbelt, Pajor, Strasser, Wagner 2013)
CSA sorts all elementary connections (dep stop, arr stop, dep time, arr time, trip) by departure time and scans them linearly once, keeping per-stop earliest arrival and per-trip "reachable" flags. It is extremely simple and fast for earliest-arrival queries, and profile CSA handles ranges. Weaknesses: transfers need a transitively closed footpath graph (or ULTRA), and multi-criteria and real-time updates are less natural than in RAPTOR.

**Open implementations:** ULTRA (`Algorithms/CSA/CSA.h`, `ULTRACSA.h`, `HLCSA.h`), chairemobilite/**trRouting** (C++, GPL-3.0, active, used for Transition/Montréal planning), ATFutures/**gtfs-router** (R/C++), planarnetwork/connection-scan-algorithm (TypeScript), linkedconnections/csa.js, trainline-eu/csa-challenge (multi-language comparison), Tristramg/csa-rust (tiny, 2021). MOTIS 1 also had a GPU CSA module; MOTIS 2 dropped it in favour of nigiri.

### 1.5 Trip-Based Routing (TB/TBTR) (Witt 2015)
TB precomputes trip-to-trip transfers *between specific trip segments* and prunes useless ones (U-turn and "no improvement" reductions). A query is then a BFS over trips in rounds, using "reached" bookkeeping per trip and stop index. It is typically several times faster than RAPTOR, at the cost of preprocessing time and memory and fragility under real-time changes.

**Open implementations:**
- nigiri `include/nigiri/routing/tb/*`, `src/routing/tb/preprocess.cc` (597 LoC). Transfers are stored as 16-byte records with traffic-day bitfields and day offsets (`tb_data.h`). This is a notable design for multi-day timetables.
- KIT ULTRA `Algorithms/TripBased/` (with ULTRA-TB and bounded multi-criteria).
- GraphHopper `reader-gtfs/.../TripBasedRouter.java`, `PtRouterTripBasedImpl.java`.

In MOTIS, TB is used only when the query has none of the following, otherwise it falls back to RAPTOR (`src/endpoints/routing.cc` ~L1215):
- a real-time transport in the timetable
- arriveBy
- clasz filters
- time-dependent offsets
- custom transfer times
- vias
- bike/car-transport requirements

### 1.6 Transfer Patterns (Bast et al. 2010; Scalable TP 2016)
TP precomputes, for each stop pair, the DAG of "transfer patterns" (stop sequences where transfers occur) of all optimal journeys. A query then evaluates only those patterns using direct-connection lookups. It is extremely fast at query time (this is the basis of Google Transit), but precomputation is huge: Scalable TP partitions the network to tame it. **There is no maintained open implementation.** Catenary's `catenarytransit/routing` README records that they tried TP and Scalable TP and abandoned them ("the memory overhead for precomputation was much too high"), switching to RAPTOR. Their attempt is in `~/g/catenary-routing/src/old files/transfer_patterns.rs`.

### 1.7 Multimodal glue concepts that matter in practice
- **Access/egress:** one-to-many street searches from the origin and destination to nearby stops, producing time offsets that feed the transit search. Every engine does this. MOTIS does it in `src/endpoints/routing.cc` → `osr::route_one_to_many`.
- **Transfers/footpaths:** precomputed stop↔stop walking times. They come from Euclidean distance, a street router (MOTIS `src/compute_footpaths.cc`, OTP `graph_builder/module/transfer/DirectTransferGenerator.java`, R5 `transit/TransferFinder.java`), or ULTRA shortcuts.
- **Time-dependent footpaths:** MOTIS supports these for elevator outages (`include/nigiri/td_footpath.h`, `src/update_rtt_td_footpaths.cc`).
- **Sharing (GBFS) and on-demand:** extra street-side state machines (walk → rent → ride → return → walk). MOTIS implements these as osr profiles (`bike_sharing.h`, `car_sharing.h`); OTP implements them as street-graph State modes.

---

## 2. MOTIS 2 (motis-project/motis)

**Repo:** `~/g/motis` (HEAD 6bf3da0, 2026-10-23). MIT. About 618★. C++23. About 29.5k LoC of glue around the components. Release v2.11.4 on 2026-10-21. Powers **Transitous** (api.transitous.org), which is used by KDE Itinerary, GNOME Maps, Träwelling, Öffi NG and about 20 other apps.

### 2.1 Architecture
MOTIS 2 is a rewrite that replaced the old MOTIS 1 module system (with its routing, csa, tripbased, raptor and ppr modules) with a single binary that composes libraries pinned in `.pkg` (its own `pkg` dependency manager, git submodule-like):

| Component | Repo | Role |
|---|---|---|
| **nigiri** | motis-project/nigiri (MIT) | timetable + transit routing core |
| **osr** | motis-project/osr (MIT) | OSM street routing (walk, bike, car, sharing, indoor) |
| **adr** | triptix-tech/adr (**AGPL-3.0**) | geocoding/typeahead/reverse geocoding (n-grams, sift4 fuzzy scoring: `include/adr/ngram.h`, `sift4.h`, `typeahead.h`) |
| **tiles** | motis-project/tiles (MIT) | vector tile server built from OSM (Lua profiles, `tiles-profiles/full.lua`) |
| cista | felixguendling/cista (MIT) | zero-copy serialization and mmap containers (the foundation for everything) |
| utl, geo, net, ctx | motis-project/* | utilities, geometry, HTTP (Boost.Beast), coroutine scheduler |

> **License caution:** a default MOTIS build links adr (AGPL-3.0, `~/g/adr/LICENSE`). Using nigiri + osr alone avoids this.

### 2.2 Inputs (from README.md and `docs/setup.md`)
- One OSM `.osm.pbf` (streets, addresses, indoor levels, platforms).
- Multiple **GTFS** (including GTFS-Flex, GTFS Fares v2, Google ticketing extension), **NeTEx** (`nigiri/src/loader/netex`) and **HRD/HAFAS** (`nigiri/src/loader/hrd`) static feeds. Lua scripting can rewrite agencies, routes and stops during import (`docs/scripting.md`; nigiri pulls in LuaJIT + sol2).
- Real time: **GTFS-RT** (TripUpdates, alerts), **SIRI-ET, SIRI-SX, SIRI-FM** (elevators), **VDV-454 AUS** (`nigiri/src/rt/vdv_aus.cc`), and DB **FaSta** elevator status (`src/elevators/parse_fasta.cc`).
- **GBFS** feeds with geofencing (`src/gbfs/*`).
- Elevation: SRTM HGT or other DEMs (`osr/src/preprocessing/elevation/hgt_driver.cc`, `dem_driver.cc`). The full SRTMGL1 set is about 350 GB (`docs/elevation-setup.md`).
- On-demand "PRIMA" ride-sharing/taxi integration (`src/odm/*`: `meta_router.cc`, `prima.cc`).

### 2.3 Pipeline
`motis config my.osm.pbf gtfs.zip` → `motis import` (builds the osr graph, nigiri timetable, footpaths via osr, adr index, tiles DB and shapes, all as mmap/cista binary files in `data/`) → `motis server`.

`src/compute_footpaths.cc` routes stop-to-stop transfers with osr for several profiles (foot, wheelchair, car). Results are stored in `tt.locations_.footpaths_out_[profile]`, with `kNProfiles = 5` (`nigiri/include/nigiri/types.h:213`). Stops are matched to OSM platforms (`src/match_platforms.cc`), which makes "routed transfers" platform-accurate, including elevators.

### 2.4 Query flow (`src/endpoints/routing.cc`, 1,358 LoC)
1. Parse the `/api/v6/plan` params.
2. Compute a direct street route (osr) for comparison and filtering.
3. Run one-to-many osr searches for pre-transit and post-transit offsets (`osr::route_one_to_many`, ~L354). These include sharing modes (GBFS profiles), flex, ODM, and *time-dependent* offsets (`td_start_`/`td_dest_`, e.g. a rental station's opening times).
4. Build a `nigiri::routing::query`: interval, max transfers, clasz mask, transfer-time settings, up to 2 vias (`kMaxVias = 2`), profile index for routed footpaths, `fastest_direct_` for pruning.
5. Choose an algorithm (~L1170–1240): GPU (if built with `NIGIRI_CUDA` and the query is supported), **PONG** (default when applicable), **TB** (only for plain queries without RT), otherwise **RAPTOR**.
6. Reconstruct journeys, then `journey_to_response` runs osr again for detailed leg geometry, including indoor levels, elevators and elevation profiles. Fares and alternatives are added.

### 2.5 Real time (`src/rt_update.cc`)
Every `update_interval` (default 60 s), MOTIS builds a new `nigiri::rt_timetable` and publishes it with `std::atomic_store(&d.rt_, new_rt)` (an RCU-style swap). Readers keep their `shared_ptr` snapshot. The new timetable is either a clean slate from `create_rt_timetable(tt, today)` or, when `incremental_rt_update: true`, a carried-over copy of the previous one. RT lower bounds are then refreshed (`rtt->update_lbs`), and the copy is uploaded to the GPU if one is used.

### 2.6 API
REST/JSON with an OpenAPI 3 spec (`openapi.yaml`, API v6). Endpoints include:
- `/plan`, `/one-to-many`, `/experimental/one-to-many-intermodal`, `/one-to-all` (isochrones)
- `/trip`, `/stoptimes`, `/refresh-itinerary`
- `/map/trips` (railviz live vehicles), `/map/stops`, `/map/levels`
- `/geocode`, `/reverse-geocode`, `/rentals`, `/health`, `/metrics` (Prometheus), plus OpenTelemetry tracing

Two less common endpoints exist:
- **OJP** (Open Journey Planner, the CEN/VDV XML standard): `src/endpoints/ojp.cc`, 1,186 LoC.
- A built-in **MCP server** (Model Context Protocol, stateless Streamable HTTP): `src/endpoints/mcp.cc`.

A generated JS client is on npm (`@motis-project/motis-client`), and the web UI is in `ui/` (Svelte + MapLibre).

### 2.7 Scaling and performance
- **Transitous production:** planet OSM (daily `pyosmium-up-to-date`, `ansible/roles/motis/templates/motis-update-feeds-secondary.sh`). From `~/g/transitous/feeds/*.json`: 2,901 GTFS, 1,061 GTFS-RT, 225 GBFS, 28 SIRI and 1 NeTEx source definitions. Per the Transitous wiki "Hardware Resources" page, MOTIS runs on one root server with **190 GB RAM and ~1 TB NVMe**, with a 70 GB staging VM. Import of the full planet OSM is done on university servers with 400 GB.
- **osr README:** a planet extract has about 12.7 GB of "routing data" that should be resident; geometries (about 19 GB + 19 GB) are only memory-mapped. The import is claimed to need ≤10 GB RAM **(unverified)**.
- **README claims:** "Loading one year of timetable doesn't take much more RAM than loading one month". This follows from deduplicated 512-day bitfields (`bitfield = bitset<kMaxDays=512>`, `types.h:62-63`).
- Query timeouts and limits are configurable (`routing_max_timeout_seconds: 90`, `street_routing_max_direct_seconds: 21600` "high values can lead to long-running, RAM-hungry queries").
- I found no published absolute latency numbers. nigiri PR #436 (2026-10) benchmarks "full Transitous Europe" with 1,000 queries but reports only relative deltas (~4–6% mean improvements).

### 2.8 Strengths and weaknesses
**As a dependency**
- Strengths: one static binary, planet-scale on one box, best-in-class RT format coverage for Europe (SIRI, VDV), indoor/elevator-aware transfers, GBFS, Flex, ODM, fares, MIT core, releases for Linux/macOS/Windows/ARM.
- Weaknesses: C++23 with the custom `pkg` dependency system, so it is heavy to build and embed (osr-ffi notes "native dependencies are compiled from source… substantial time"). API churn (v6 now). Small core team (largely TU Darmstadt / triptix). The adr component is AGPL.

**As a reference**
Outstanding. It is modern data-oriented C++, with compact typed index containers (`vecvec`, `paged_vecvec`, `mm_vecvec`), zero-copy cista images, UTC "transports" versus "trips" (DST-correct), route grouping, and four algorithm implementations sharing one timetable. It is dense and template-heavy, and documentation is light outside the READMEs.

---

## 3. nigiri (motis-project/nigiri), the transit core

`~/g/nigiri` (HEAD 2026-10-23). MIT. About 70k LoC (including loaders and the GPU code). 45★, but development is very active.

### 3.1 Timetable data structures (`include/nigiri/timetable.h`)
- **Trips versus transports.** External trips (GTFS trip_id) are converted to UTC. Because of DST, one trip can produce several `transport_idx_t`, each with a `bitfield_idx_t` (traffic days) plus a contiguous series of dep/arr `delta`s relative to the day (`README.md` "Key Concepts").
- **Stay-seated (block_id) chains** are concatenated into a single transport so they do not count as transfers (README "Stay seated").
- **Routes** group transports with the same stop sequence and attributes such that no later departure can be better (FIFO, no overtaking). Stop times are stored **route-major, then stop/event-major, then transport**: `route_stop_times_` is a flat `vector<delta>`. For each route there are runs of `[stop0-dep: t1..tN][stop1-arr: t1..tN][stop1-dep: …]`, addressed by `route_stop_time_ranges_[r]` (`event_times_at_stop()`, ~L140). This layout lets RAPTOR binary-search one contiguous span per (route, stop).
- **Traffic days** are deduplicated `bitset<512>` values (`vector_map<bitfield_idx_t, bitfield> bitfields_`). A one-year timetable costs little more than a one-month one.
- **Locations** carry parent/child relations and equivalences, plus `footpaths_out_/in_` per profile. Each footpath is **32 bits: a 23-bit target and a 9-bit duration in minutes** (`include/nigiri/footpath.h`). There are also an r-tree and per-location transfer times.
- **Lower-bound graphs** (`fwd/bwd_search_lb_graph_`) are used by a Dijkstra (`src/routing/dijkstra.cc`) to compute per-stop lower bounds for A*-like RAPTOR pruning (`kUseLowerBounds = true` in `raptor.h`).
- Everything is serialized with cista (`timetable::write/read` returns `cista::wrapped<timetable>`), so loading is a near-zero-copy mmap.

### 3.2 Algorithms (`include/nigiri/routing/`)
- `raptor/raptor.h`: templated on direction, real-time on/off, number of vias and one-to-one versus one-to-all. Up to 14 transfers (`limits.h: kMaxTransfers = 14`). Features: clasz masks, bike/car-carriage, wheelchair, no-reservation filters, transfer-time factor, td-footpaths, vias with minimum stay.
- `search.h`: the interval search driver. It runs the algorithm over a departure interval, then **extends the interval** earlier or later (via `interval_estimator`) until `min_connection_count_` results are found or a timeout is reached. This is how "give me 5 connections" works. Pareto criteria: departure, arrival, transfers (`journey.h:81-86`).
- `raptor/pong.cc`: PONG (see §1.3).
- `tb/*`: Trip-Based, with preprocessing in `src/routing/tb/preprocess.cc`. Transfers carry bitfields to stay correct over multi-day bitfield timetables.
- `gpu/raptor_impl.cuh` (1,297 LoC): **CUDA RAPTOR**. The timetable and RT timetable are uploaded to the device (`device_timetable.cuh`).
- `one_to_all.h`, `direct.h`, `leg_alternatives.h`, `ontrip_train.h` (start onboard a train), and `get_fastest_direct.h`.

### 3.3 Real time (`include/nigiri/rt/rt_timetable.h`)
The RT timetable is an overlay. Updated static transports get a modified bitfield copy (flagged `kRtBitfieldFlag`) so that the static version is masked out on that day. RT transports, both delayed copies and added trips, are stored separately and are "not grouped into routes to simplify the code" (header comment at L35). Loaders exist for GTFS-RT (`gtfsrt_update.cc`), alerts, VDV AUS and SIRI. RT lower-bound graph deltas are maintained for pruning.

### 3.4 Assessment
nigiri is the best open transit-core reference for a compact, multi-day, multi-feed, DST-correct timetable with several algorithms. It can be used as a C++ library without MOTIS (its own `.pkg`). Weaknesses: there is no stable API, the C++ toolchain is demanding, and documentation is in code.

---

## 4. osr (motis-project/osr), Open Street Router, in depth

`~/g/osr` (HEAD 2026-10-21). MIT. About 22k LoC. 59★. Has its own binaries, `osr-extract` and `osr-backend` (an HTTP debug UI in `exe/backend`).

### 4.1 Graph model: "OSM as the graph" (no explicit edge list)
Per the README "Data Model" and `include/osr/ways.h`:
- Only OSM nodes shared by more than one way become routing nodes (`node_idx_t`, versus `osm_node_idx_t`). Ways relevant for routing get compact `way_idx_t`.
- Adjacency is **implicit**. The structure stores `way_nodes_` (way → its routing nodes in order), `way_node_dist_` (uint16 metres between consecutive routing nodes, with `long_way_node_dist_` for overflow), `node_ways_` (node → ways) and `node_in_way_idx_` (the position of that node in each way). A node's neighbours are positions *i−1 / i+1* in each way it belongs to (README "Node Neighborhood").
- Attributes are bit-packed per way (`way_properties`, ~L170: car/bike/foot/bus/rail/ferry access, oneway variants, steps, elevator, big street, destination, penalties, speed, level bits) and per node (`node_properties`: barriers, elevators, levels).
- Other stored data:
  - turn restrictions (`node_restrictions_`, `node_is_restricted_` bitvec) and turn bearings
  - multi-level elevators
  - HGV attributes (`way_hgv_info_`)
  - **conditional restrictions and opening hours**: a full parsed `opening_hours` model, with `src/extract/conditional_parser.cc` at 1,105 LoC
  - way names and polylines
  - `class_components`: per-class connected components, used to avoid snapping to islands
- Everything lives in `cista::basic_mmap_vec`-backed containers (`mm_vec`, `mm_vecvec`, `mm_paged_vecvec` in `include/osr/types.h:49-75`). On a planet extract, 12.7 GB of "routing" files should be locked in RAM, while OSM-ID mappings and polylines (19 GB + 19 GB) can stay on disk and be paged in on demand (README file listing).
- Snapping uses an r-tree over way bounding boxes. A query is matched by perpendicular distance to candidate ways and initialised at the "left" and "right" routing nodes on that way (`src/lookup.cc`, `include/osr/lookup.h`). `platforms.h` provides OSM public-transport platform lookup for MOTIS stop matching.

### 4.2 Profiles: a compile-time overlay graph per profile
`include/osr/routing/profiles/`: `foot.h` (+wheelchair, indoor levels), `bike.h` (elevation-cost aware), `car.h`, `car_parking.h` (drive → park → walk), `bike_sharing.h`, `car_sharing.h` (GBFS, with an extra-nodes overlay in `sharing_data.h`), `hgv.h` (920 LoC: weights, heights, hazmat and similar), `railway.h`, `ferry.h`.

A profile is a C++ type that provides the following (README "Routing Profiles"):
- `node`: the search state. Foot uses `{node, level}`; car uses `{node, way, direction}`, so u-turns and turn restrictions become part of the state.
- `key` / `entry`: several states share one hash-map entry to shrink the cost map. For car, one entry stores costs for all (way, dir) variants of a node.
- `label`, `resolve_endpoint`, `resolve_all`, `adjacent<SearchDir, WithBlocked>(…)`: neighbour enumeration that computes costs from packed attributes at query time.

So **adding a profile costs zero extra memory or preprocessing**: costs are computed on the fly. This is the same philosophy as Valhalla's dynamic costing, applied at a finer grain (the state space itself is per profile). Profiles are compile-time templates, with runtime `profile_parameters` (speeds and similar, `include/osr/routing/parameters.h`). Sharing profiles chain walk → vehicle → walk inside a single search.

### 4.3 Algorithms (`include/osr/routing/`)
- `dijkstra.h`: Dijkstra with a **Dial bucket queue** (`dial.h`, integer seconds as buckets) and an `ankerl::unordered_dense` hash map for costs (no dense per-node arrays, so memory is proportional to the explored area). Max-duration cutoff, early termination, **one-to-many**, and backward search.
- `astar.h`: A* with a profile-supplied `lower_bound_heuristic` based on great-circle distance.
- `bidirectional.h`: **bidirectional A*** (`routing_algorithm::kAStarBi`, selectable as `"bidirectional"` in `src/route.cc:52`) with stopping-slack handling for rounded costs.
- An optional `routing_time_t` enables **time-dependent** evaluation of conditional restrictions and opening hours.
- `bitvec<node_idx_t>* blocked` allows per-query blocked nodes, used by MOTIS for out-of-service elevators.
- `src/map_matching.cc` (814 LoC): matching a polyline, e.g. a GTFS shape, to the network. MOTIS uses this to route bus/rail shapes (`src/route_shapes.cc`), and `railway.h` makes it work for rail.
- Elevation (`elevation_storage.h`) is used for bike costs and returned elevation profiles.
- **No speed-up preprocessing:** no CH, MLD, ALT or arc flags. The README's "future work" still lists A* and bidirectional A* as todo, but both are now implemented.

### 4.4 Preprocessing (`src/extract.cc`, 1,107 LoC)
libosmium multi-pass:
1. Load OSM pass 1 (node usage counts and relations).
2. Ways (tag parsing via `include/osr/extract/tags.h` into `way_properties`).
3. Areas (pedestrian areas are triangulated or connected, `src/extract/osm_areas.cc`) and area annotation.
4. Node properties, restrictions and elevators.
5. Connect ways, build class components, compute big-street neighbours.
6. Build the r-tree.

Elevation is attached in a separate step. Output is a directory of mmap files.

### 4.5 Compared with OSRM and Valhalla
| | osr | OSRM | Valhalla |
|---|---|---|---|
| Graph | implicit: OSM ways + shared nodes, mmap | explicit edge-expanded graph | tiled hierarchical graph (3 levels), mmap tiles |
| Profiles | many on one dataset, zero extra cost (compile-time C++ templates + runtime params) | **one profile per dataset** (Lua at extract time) | many on one dataset (dynamic costing at runtime) |
| Speed-up | none (Dijkstra/A*/bi-A*, Dial queue) | CH or MLD (ms queries continent-wide) | hierarchies + bidirectional A* (time-dependent) |
| Long car routes | slow and RAM-hungry (MOTIS caps direct routing at 6 h) | very fast | fast |
| Short one-to-many (≤15–60 min) | very good fit: cheap, exact, indoor-aware | table service is fast but per-profile | matrix and isochrones OK |
| Indoor/levels/elevators | first-class (`foot.h` level in state, multi-level elevators) | no | limited |
| Planet RAM | ~13 GB resident (claim) | car CH/MLD planet: tens of GB, preprocessing ~100+ GB (other agents cover this) | low (tile cache) |
| Navigation guidance | basic (path segments, names, levels) | full turn-by-turn | full turn-by-turn (+Ferrostar ecosystem) |

**Verdict on osr.** It is a modern *first/last-mile and transfer* router: compact, multi-profile, multimodal-state-aware, and easy to extend with a new profile. It is **not** a replacement for OSRM or Valhalla for long-distance car routing or turn-by-turn guidance. The "OSM-as-graph + compile-time profile overlay + mmap" design is very well worth copying in Rust. It is also bindable today: `catenarytransit/osr-ffi` wraps it for Rust with `cxx`, with the graph, r-tree and elevation staying C++-owned and mmap'd.

---

## 5. OpenTripPlanner 2 (opentripplanner/OpenTripPlanner)

`~/g/OpenTripPlanner` (HEAD 2026-10-22, 2.11.0-SNAPSHOT). **LGPL-3.0**. About 2.8k★. About 284k LoC Java (main + sandbox). Maven multi-module: `raptor/`, `astar/`, `street/`, `transit/`, `core/`, `application/`. Governed as an OSGeo-style community project; Entur, Digitransit (HSL/Fintraffic/Waltti), IBI/Arcadis, Leonard and others are core contributors.

### 5.1 Deployments
From `doc/user/Deployments.md`:
- **Entur** (Norway, national, since 2017; Ruter uses it through Entur). The docs mention peaks above 20 req/s.
- **Digitransit** (Finland, national), VR (Finnish rail), Skånetrafiken (NeTEx Nordic + SIRI).
- In the US: TriMet, MTA NYC, NYSDOT, LA Metro, MARTA, MBTA, Sound Transit, SEPTA, Houston.
- In Europe: VBN, Leipzig, Strætó (Iceland), Piemonte, Valencia, Grenoble, Rennes, Poznań, Aubin (GB), and InfoPlaza/Transdev NL (since 2025, with flex and shared mobility).
- **Headway/maps.earth** runs per-zone OTP instances (`builds/*/k8s/*opentripplanner-*`).

### 5.2 Inputs
OSM (pbf), GTFS, **NeTEx** (Nordic and EU profiles; `application/.../netex`), GTFS-Flex (sandbox `ext/flex`), and elevation GeoTIFF (`doc/user/Preparing-DEM.md`). The graph is built once and serialized with Kryo (`routing/graph/SerializedGraphObject.java`, `graph.obj`).

### 5.3 Transit routing (`raptor/`, ~23k LoC)
- A self-contained library with **no dependencies on the rest of OTP**, enforced by `RaptorArchitectureTest`. Transit data is supplied through an SPI: `raptor/spi/RaptorTransitDataProvider.java`, `RaptorRoute`, `RaptorTripSchedule`, `RaptorTimeTable` and related types.
- Profiles: single-iteration RAPTOR, Range RAPTOR (`rangeraptor/standard`, forward and reverse) and **multi-criteria Range RAPTOR** (`rangeraptor/multicriteria`). McRR criteria are arrival time, transfers, travel duration, **generalized cost c1**, and an optional **c2** with relaxed dominance (`package.md`).
- Heuristics: a fast RR pass (forward and/or reverse) computes bounds and destination pruning before McRR (`service/HeuristicSearchTask.java`). A dynamic search window is computed from the heuristics (`RangeRaptorDynamicSearch.java`).
- Via-location search via chained segments (`RangeRaptorWorkerComposite`). Constrained transfers (guaranteed, stay-seated, priority) are supported.
- Debugging: accepted, rejected and dropped path event listeners.
- Post-processing: a **transfer-optimization** step (`routing/algorithm/transferoptimization/`) and an itinerary filter chain.
- Transit layer: `routing/algorithm/raptoradapter/transit/RaptorTransitData.java` replicates the transit model as **TripPatternForDate** per local service date ("Patterns and trips are split out by days"). Per-request filtering builds `TripPatternForDates`. This is less compact than nigiri's bitfield model, and memory grows with the number of days loaded.
- Reported numbers: about 80 ms for RR versus 400 ms for McRR and about 1 s with walking as a criterion, on the SpeedTest (`package.md`). These are examples, not a benchmark.

### 5.4 Street routing (`street/`, `astar/`)
- A classic **object graph**: `street/model/edge/*` (StreetEdge, ElevatorHop, Escalator, Pathway, Area edges, vehicle-parking links and more) and `vertex/*`. A generic **A*** (`astar/AStar.java`) runs over `State` objects. Multimodal street state covers walk, bike, car, rental and park-and-ride, with GBFS geofencing (`service/vehiclerental/street/geofencing`). Elevation extensions and intersection traversal models (`street/search/intersection_model`) are included.
- **Integration:**
  - Access/egress run as A* searches to stops at query time (`raptoradapter/router/street/AccessEgressRouter.java`).
  - Stop-to-stop transfers are precomputed at build time (`graph_builder/module/transfer/DirectTransferGenerator.java`).
  - Direct street routes are separate A* searches.
- Memory: "Finland a bit over 10 GB, Germany 45 GB" (`doc/user/System-Requirements.md`). Single-thread performance and cache size matter. Entur runs a fleet of `c4-standard-8` instances.

### 5.5 Real time (`application/.../updater/`)
GTFS-RT TripUpdates (`updater/trip/gtfs`), **SIRI-ET/SX** (`updater/trip/siri`, sandbox `ext/siri`), vehicle positions, alerts, GBFS vehicle rental, vehicle parking, and Azure Service Bus SIRI. Updates are applied to a `TimetableSnapshot`. A `GraphWriterRunnable` serializes writes and `RealTimeRaptorTransitDataUpdater` swaps in changed dates. Updates can be incremental or full-dataset.

### 5.6 API
- **GTFS GraphQL API** (`apis/gtfs`, the Digitransit lineage)
- **Transmodel GraphQL API** (`apis/transmodel`, the Entur lineage)
- vector tiles (`apis/vectortiles`), and OJP (sandbox `ext/ojp`)
- the old REST API was removed in OTP2

There are many sandbox extensions (`application/src/ext/.../ext/`): carpooling, emissions, fares, flex, ridehailing, taxi, stop consolidation, empirical delay and others.

### 5.7 Strengths and weaknesses
**As a dependency.** It is the most battle-tested and the most deployed, with a large community, rich configuration, Transmodel/NeTEx support, and LGPL (fine as a separate server). On the downside, it is a JVM service with high RAM per region, so it is not planet-scale on one box. Graph builds are slow. Its large codebase carries legacy complexity.

**As a reference.** The `raptor` module is the cleanest *readable* McRaptor in the open: well documented, with a clear SPI/API split and explicit heuristics. The way generalized cost is used as a criterion is good to study. The street side is a conventional object graph and less interesting to copy.

---

## 6. R5 (conveyal/r5)

`~/g/r5` (HEAD 2026-08-01, v7.6). **MIT**. About 405★. About 78k LoC Java. Wrappers: **r5py** (Python, 194★, active) and **r5r** (R, IPEA Brazil, 245★, active).

- **Purpose:** accessibility analysis. It computes many-to-many travel-time *distributions* over a departure window ("realistic routing") rather than serving single A→B itineraries. The README says Conveyal "does not provide technical support for third-party deployments" and there is "no stable programming interface".
- **Inputs:** OSM + GTFS. Frequency-based (headway) routes are supported. There is no RT. Instead, **scenarios** apply lightweight patches to the network (`analyst/scenario/*`: AddTrips, RemoveTrips, AdjustFrequency, AdjustSpeed, Reroute, AddStreets, ModifyStreets, RoadCongestion, PickupDelay and others).
- **Transit algorithm:** `profile/FastRaptorWorker.java` runs Range RAPTOR over every minute of the window. For frequency routes it uses **Monte Carlo randomized schedules** (`FrequencyRandomOffsets.java`; Conway, Byrd & van der Linden 2017) to produce travel-time percentiles. `McRaptorSuboptimalPathProfileRouter.java` handles suboptimal paths, and fare-constrained routing (`FareDominatingList.java`) is supported.
- **Street:** `streets/EdgeStore.java` is a column store of parallel arrays, with paired forward/back edges (even/odd). The header comment explains why it avoids an object graph. `StreetRouter.java` is Dijkstra on `PriorityQueue<State>`. LTS (level of traffic stress) bike costs and LA DOT cost tags are supported. `PerTargetPropagater` propagates from stops to grid cells (`LinkedPointSet`).
- **Strengths:** fast one-to-many accessibility, uncertainty-aware results, a scenario system, and an academic pedigree. MIT.
- **Weaknesses:** not a journey-planner server, no RT, an unstable API, and it is Conveyal-roadmap driven.
- **As a reference:** the best open code for travel-time matrices, isochrones and percentiles, and for the column-store street graph in Java.

---

## 7. Navitia (hove-io/navitia)

`~/g/navitia`. **AGPL-3.0**. About 457★. About 116k LoC C++ + 130k LoC Python. Latest release v15.120.1. Commits continue (2026-10-21).

**Status:** `readme.rst` carries a French notice from Hove saying a new version is coming and that GitHub repos would be closed progressively "by the end of September" (that wording points to 2023). The historical version stays available. Despite this, the public repo still receives maintenance commits in 2026, such as SQLAlchemy 2.x and mimir upgrades. Treat it as **legacy and in maintenance mode, with no commitment to openness**. navitia.io is a freemium service.

**Architecture:**
- **kraken** (C++ RAPTOR core; `source/routing/raptor.cpp`, `dataraptor.cpp`, `isochrone.cpp`, `heat_map.cpp`)
- **jormungandr** (Python/Flask API orchestration, "distributed" scenario)
- **tyr** (data administration), **ed** (PostGIS import), **eitri** (data binarization)
- data in **NTFS** format, produced by `hove-io/transit_model` (Rust, AGPL, actively maintained; GTFS/NeTEx/NTFS conversions)

**Street network:** pluggable backends in `source/jormungandr/jormungandr/street_network/`: kraken (built-in), `valhalla.py`, `asgard.py` (Hove's Valhalla-based service), `geovelo.py`, `here.py`, `handimap.py`, `andyamo.py`, plus taxi, ridesharing and car-with-park.

**RT:** Kirin (a separate service for GTFS-RT and SNCF COTS) and Chaos (disruption management).

**Verdict:** not recommended as a dependency (AGPL, an uncertain future, and a heavy multi-service deployment). Mildly interesting as a reference for an API design (the HATEOAS-style Navitia API) and for how pluggable street backends are orchestrated.

---

## 8. Smaller and Rust projects

| Project | Lang / License | Activity | What it is |
|---|---|---|---|
| **catenarytransit/catenary-backend** | Rust, AGPL-3.0, 48★ | very active (2026-10-23) | Catenary Maps backend. Routing pieces: **Gentian** (timetable compiler, `src/gentian/*`, compiles a 366-day window and footpaths via an **osr worker**), **Edelweiss** (RAPTOR query server, `src/edelweiss/main.rs`, emits MOTIS-compatible responses via `motis_response.rs`), and `src/routing_common/*` (timetable, `raptor_query`). Uses **`osr-ffi`** (cxx bridge to motis osr). Regional builds via `routing_regions.toml`. |
| **catenarytransit/routing** | Rust, MIT | active | Algorithm playground. Earlier Transfer Patterns / Scalable TP attempts were abandoned for memory reasons; now RAPTOR plus a simple road Dijkstra. |
| **catenarytransit/osr-ffi** | Rust/C++ | 2026-10 | `cxx` bindings to osr (graph and r-tree stay C++-owned and mmap'd). **Directly relevant to a Rust project.** |
| **ellenhp/solari** | Rust, Apache-2.0, 151★ | **dormant since 2025-06** | Headway/maps.earth experimental transit router. RAPTOR over **memory-mapped timetables** (memmap2), redb, and transfers via **Valhalla graph tiles (stadiamaps `valinor`) + `fast_paths` CH** (`solari-transfers`). No RT and no rRAPTOR. Planet-scale claims are unverified. |
| **urschrei/vulture** | Rust, Apache-2.0 / Blue Oak, 12★ | 2026-08 | Clean RAPTOR library with a `Timetable` trait, a GTFS adapter (`gtfs-structures`), and a WASM build (~290 KB gz). Earliest-arrival and transfers Pareto. Good for learning. |
| **vincbro/blaise** (+ gtfs-bin) | Rust, MIT | 2026-05 | RAPTOR library + REST server, fuzzy stop search, mmap'd precomputed GTFS binaries. Early. |
| **chingiztob/ferrobus** | Rust core + Python (pyo3), Apache-2.0 | 2026-04 | R5-like analysis library: RAPTOR/**rRAPTOR**, isochrones, matrices, rayon. Streets from OSM via `osm4routing`. "No Java needed" alternative to r5py. |
| keogami/raptor-rs, s3n-w6i/drino, HoriaSav/Rustline, FrankBStack/Timeshed, princefr/transit, iter-maps/server | Rust | various | Small or WIP RAPTOR implementations, including offline on-device (Rustline) and door-to-door GTFS + OSM (Timeshed). |
| hove-io/transit_model | Rust, AGPL | active | Not a router: a GTFS/NeTEx/NTFS data model and converters. |
| gtfs-structures (crate) | Rust, MIT | active | De-facto GTFS parser used by most of the above. |
| **kit-algo/ULTRA** | C++, MIT, 85★ | 2026-01 | Research framework: CSA, RAPTOR, McRAPTOR, MCR, Trip-Based, ULTRA variants, CH. Reference-grade algorithm code, not production. |
| chairemobilite/trRouting | C++, GPL-3.0 | active | CSA-based transit routing server (Transition platform). |
| ATFutures/gtfs-router | R/C++, GPL-3 | 2026-09 | CSA routing and analysis in R. |
| headwaymaps/headway | Rust/TS, Apache-2.0, 3k★ | active | A self-hosted maps stack, not a router. Uses **Valhalla** for streets and **per-zone OTP** instances for transit. |

**Takeaway for Rust:** nothing in Rust approaches MOTIS or OTP in feature completeness (no RT, no NeTEx, no range queries in most). The most serious Rust effort (Catenary) chose a nigiri-style RAPTOR plus *osr through FFI* for streets, and emits MOTIS-format responses.

---

## 9. Built-in transit in road engines (brief; covered in depth elsewhere)

- **Valhalla:** GTFS ingest (`src/mjolnir/ingest_transit.cc`, `convert_transit.cc`, `valhalla_ingest_transit`) builds transit tiles. Routing is a time-dependent multimodal Dijkstra (`src/thor/multimodal_transit.cc`, which notes "Disable A* for multimodal"), single-criterion, with no RAPTOR and no meaningful RT. Adequate for demos and small regions only.
- **GraphHopper:** the `reader-gtfs` module builds a time-expanded `PtGraph`. It has two routers: `MultiCriteriaLabelSetting.java` (earliest arrival and transfers, MLS) and a newer **`TripBasedRouter.java` / `PtRouterTripBasedImpl.java`**. GTFS-RT is supported via `RealtimeFeed.java`. Apache-2.0. Last touched 2026-08. A secondary feature of GraphHopper.

---

## 10. Comparison table

| Criterion | MOTIS 2 / nigiri / osr | OTP 2 | R5 | Navitia | Solari | Catenary |
|---|---|---|---|---|---|---|
| Language | C++23 | Java | Java | C++/Python | Rust | Rust (+C++ osr) |
| License | MIT (adr AGPL) | LGPL-3 | MIT | AGPL-3 | Apache-2 | AGPL (backend) |
| Static formats | GTFS, GTFS-Flex, Fares v2, NeTEx, HRD | GTFS, NeTEx, Flex | GTFS | NTFS, GTFS | GTFS | GTFS |
| RT formats | GTFS-RT, SIRI-ET/SX/FM, VDV-454, GBFS | GTFS-RT, SIRI-ET/SX/VM, GBFS | — | GTFS-RT/COTS via Kirin | — | GTFS-RT (display side) |
| Timetable model | UTC transports, 512-day bitfields, route-major flat stop-time arrays, cista mmap | TripPatternForDate per service date, Kryo graph | patterns + TripSchedule arrays, Kryo | dataraptor arrays | mmap'd arrays | compiled binary timetable |
| Algorithms | RAPTOR, interval search, PONG, Trip-Based, GPU RAPTOR, one-to-all | RR, McRR (c1, c2), heuristics, via, transfer optimizer | RR + Monte Carlo, McRAPTOR | RAPTOR | RAPTOR | RAPTOR |
| Criteria | dep, arr, transfers (+ filters) | arr, transfers, duration, gen. cost, c2 | time percentiles, fares | arr, transfers, walking | arr, transfers | arr, transfers |
| Street router | osr (own; Dijkstra/A*/bi-A*; mmap) | own object graph + A* | own column store + Dijkstra | own or Valhalla/HERE/… | Valhalla tiles + CH | osr via FFI |
| Indoor/elevators | yes (levels, elevator status → td-footpaths) | yes (pathways, elevators) | no | limited | no | via osr |
| Sharing / on-demand | GBFS (+geofencing), Flex, ODM | GBFS, Flex, carpool, taxi (sandbox) | on-demand access (Conveyal) | bss, ridesharing, taxi | no | no |
| Geocoding | adr (built in) | basic (sandbox) | no | mimir (separate) | no | separate |
| Tiles | built in (tiles) | vector tiles API | no | no | no | separate |
| API | REST/OpenAPI v6, OJP, MCP | GraphQL (GTFS, Transmodel) | Java API / r5py / r5r | REST (Navitia API) | REST /v1/plan | REST |
| Planet on one box | **yes** (Transitous, 190 GB server) | no (country ≈ 10–45 GB) | no | no | claimed | no (regional) |
| Maturity | high, fast-moving | very high | high (niche) | legacy | experimental, dormant | early/active |

---

## 11. Recommendations for `routet`

1. **To build on a server today:** deploy **MOTIS** (or use Transitous's API, respecting its usage policy). It is the best capability-to-hardware ratio by far.
2. **To learn transit-core design:** read, in this order:
   - the nigiri `README.md`
   - `timetable.h` (data layout)
   - `routing/raptor/raptor.h`
   - `routing/search.h` (interval extension)
   - `raptor/pong.cc`
   - `tb/preprocess.cc`
   - `rt/rt_timetable.h` (RT overlay)
   - then OTP's `raptor/package.md` and `rangeraptor/multicriteria` for McRaptor and generalized cost
   - then ULTRA for CSA, TB and ULTRA shortcuts
3. **To learn street-side design for multimodal:** read osr's README, `ways.h`, `routing/profiles/foot.h`, `car.h`, `bike_sharing.h`, `dijkstra.h`, `dial.h` and `bidirectional.h`. The "OSM-as-graph + per-profile state overlay + mmap" idea maps naturally to Rust (`memmap2` + `bytemuck`/`rkyv` zero-copy, const-generic or trait-based profiles).
4. **For a Rust implementation:**
   - Transit core: RAPTOR over a nigiri-style layout (UTC transports, deduplicated day bitsets, route-major stop-time arrays, 32-bit packed footpaths), mmap'd and zero-copy. Add interval/range search, then PONG or TB.
   - Streets: either bind osr through **osr-ffi** (proven by Catenary; C++23 build pain) or port its model.
   - Use `gtfs-structures` for parsing. NeTEx and SIRI support would have to be written, since no Rust NeTEx router exists (transit_model has NeTEx parsing but is AGPL).
   - Copy **MOTIS's OpenAPI** (`openapi.yaml`) as the API contract to get client compatibility for free (Catenary already does this).
5. **Avoid as a base:** Navitia (AGPL, legacy), the transit modes of Valhalla and GraphHopper (not competitive), and Solari (dormant).

---

## 12. Sources
- Local source: `~/g/motis`, `~/g/nigiri`, `~/g/osr`, `~/g/adr`, `~/g/motis-tiles`, `~/g/OpenTripPlanner`, `~/g/r5`, `~/g/navitia`, `~/g/transitous`, `~/g/solari`, `~/g/catenary-routing`, `~/g/vulture`, `~/g/blaise`, `~/g/ferrobus`, `~/g/ULTRA`; existing clones `~/g/valhalla`, `~/g/graphhopper`.
- GitHub API (stars, pushed_at, releases), and catenarytransit/catenary-backend and osr-ffi read via the API.
- [Transitous](https://transitous.org/); [Transitous wiki: Hardware Resources](https://github.com/public-transport/transitous/wiki/Hardware-Resources); [nigiri PR #436](https://github.com/motis-project/nigiri/pull/436)
- [OTP Deployments](https://docs.opentripplanner.org/en/latest/Deployments/); OTP `doc/user/System-Requirements.md`
- Navitia notice: `~/g/navitia/readme.rst`; [CanalTP/navitia](https://github.com/CanalTP/navitia)
- Papers: Delling et al., *Round-Based Public Transit Routing* (ALENEX 2012); Dibbelt et al., *Intriguingly Simple and Fast Transit Routing* (CSA, SEA 2013); Witt, *Trip-Based Public Transit Routing* (ESA 2015); Bast et al., *Transfer Patterns* (ESA 2010) and *Scalable Transfer Patterns* (ALENEX 2016); Baum et al., *ULTRA* (ESA 2019 / Transportation Science 2023); Conway, Byrd & van der Linden (TRR 2017).
