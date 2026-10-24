# Open-source routing engines: overview

*Snapshot 2026-10-23. Summarises the seven detailed reports in this folder; each claim below is backed (with source paths) in the report named in the right-hand column. Repos are cloned in `~/g`.*

| Report | Covers |
|---|---|
| [valhalla.md](valhalla.md) | Valhalla, valhalla-rs, Ferrostar's use of it |
| [osrm.md](osrm.md) | OSRM (osrm-backend) |
| [jvm-graphhopper-ors-brouter.md](jvm-graphhopper-ors-brouter.md) | GraphHopper, openrouteservice, BRouter |
| [organicmaps.md](organicmaps.md) | Organic Maps' embedded router, CoMaps fork, MAPS.ME lineage |
| [osmand-navit.md](osmand-navit.md) | OsmAnd's Java/C++ routers and "HH" routing, Navit |
| [libraries-and-rust.md](libraries-and-rust.md) | fast_paths, rust_road_router, `cch`, RoutingKit, Itinero, Routino, RouteE Compass, pgRouting, … + algorithm survey |
| [transit-multimodal.md](transit-multimodal.md) | MOTIS 2 (nigiri, osr), OpenTripPlanner 2, R5, Navitia, Rust RAPTOR projects |

---

## TL;DR

- **Rust has no mature OSM-to-route engine.** It has good pieces (CCH crates, `fast_paths`, `routx`, RouteE Compass) and good wrappers around C++ engines (`valhalla` crate, `osr-ffi`). A Rust engine fills a gap.
- **The three big servers trade speed against flexibility:**
  - **OSRM:** fastest queries and matrices, almost no per-request customisation, most RAM.
  - **GraphHopper:** lets each request choose between speed and flexibility.
  - **Valhalla:** fully per-request costing, tiled data, works offline and on mobile, but its hierarchy is heuristic and slower.
- **The embedded mobile routers** (Organic Maps, OsmAnd) independently converged on the same design: per-region offline files, lazy loading, a cheap partition-based overlay for long routes (not CH), and A\* for local search. Both are excellent references, and both are poor dependencies.
- **Algorithm choice for a new engine:** Customizable CH (CCH) on a turn-expanded graph is the best fit when weights must change (traffic, user profiles). It gives roughly CH query speed (~0.1 ms, Europe) with second-scale re-weighting.
- **Transit:** MOTIS 2 (nigiri + osr) is the most modern open stack to learn from or deploy. OTP2's RAPTOR is the best-documented multi-criteria implementation. No Rust transit engine is production-grade.

---

## Comparison

| | Lang | License | Form | Speed-up technique | Per-request costing | Turn restrictions | Offline / mobile | Transit | From Rust | Status (2026) |
|---|---|---|---|---|---|---|---|---|---|---|
| **OSRM** | C++20 + Lua | BSD-2 | server / libosrm / Node, Python | CH or MLD (CRP) | Almost none (≤7 predeclared `exclude` combos; weights via CSV + re-customise) | Exact, incl. multi-via (edge-based graph) | No (~123 GB RAM planet car) | No | HTTP; fragile FFI crates | Revived by original author, monthly releases, datasets not forward-compatible |
| **Valhalla** | C++20 | MIT | library-first (`tyr::actor_t`), Python/Node | 3-level tile hierarchy + heuristic limits + degree-2 shortcuts (not exact) | Nearly everything (C++ costing classes) | Yes, edge-based search | Yes (valhalla-mobile) | Basic schedule-aware | **`valhalla` crate (cxx), active** | Very active, quarterly releases |
| **GraphHopper** | Java 25 | Apache-2.0 | server / JVM library | CH (node + edge-based), Landmarks, or flexible A\* | Full in flexible mode; LM allows "only more expensive"; CH fixed | Yes, edge-based CH | No (Android dropped; Janino + Java 25 block it) | Yes (GTFS, trip-based) | HTTP only | Very active; matrix is commercial-only |
| **openrouteservice** | Java | GPL-3.0 (NOTICE says Apache, stale) | server | GH CH + "core" ALT, RPHAST matrix | Avoid/HGV/borders via core | Yes | No | No | HTTP | Active; on a 2021 GH 4 fork |
| **BRouter** | Java | MIT | Android service / server | None (two-pass A\* with pruning) | Fully scripted `.brf` profiles | Yes | **Yes** (AIDL service used by OsmAnd, Locus) | No | HTTP | Maintained |
| **Organic Maps** | C++23 | Apache-2.0 | embedded in app | Bidirectional A\* + cross-mwm "leaps" overlay (precomputed border cliques, car only) | Compiled-in vehicle models; separate weight vs ETA speeds | Yes, incl. conditional access | **Yes**, mobile-first | Frequency-based | Not extractable | Active (2026: alternatives) |
| **CoMaps** | C++23 | Apache-2.0 | embedded (OM fork, 2025) | Same as OM | Adds turn-angle and road-penalty costs | Yes | Yes | Same as OM | Not extractable | Active; mwm format diverging |
| **OsmAnd** | Java / C++ / Kotlin | GPL-3 | embedded; OsmAndServer | Bidirectional A\* + "HH" (partition, min-cut borders, per-profile shortcuts with verify-and-repair) | XML rule engine evaluated at query time, cached per tag set | Yes, incl. via-way | **Yes** | OSM route relations only | No (GPL, JVM) | Very active |
| **Navit** | C | GPLv2/LGPLv2 | embedded | None (backward LPA\* over selected road depth) | Rigid vehicle profiles | Yes | Yes | No | No | Maintenance |
| **MOTIS 2 / osr** | C++23 | MIT (geocoder adr: AGPL) | server; nigiri/osr libraries | osr: none (Dijkstra / bidirectional A\*); nigiri: RAPTOR, PONG, Trip-Based | osr profiles are compile-time types | Yes | Server (planet + ~2,900 feeds in 190 GB) | **Best-in-class** | `osr-ffi` (Catenary) | Very active |
| **OpenTripPlanner 2** | Java | LGPL-3 | server | Range / multi-criteria RAPTOR; A\* streets | Many request params | Yes | No | Yes, most deployed | HTTP (GraphQL) | Very active |
| **RoutingKit** | C++ | BSD-2 | library | CH, CCH | CCH re-customisation | Extracted, but needs manual graph expansion | n/a | No | `routingkit-cch` bindings | Active again 2026 |
| **`cch` crate** | Rust | MIT | library | CCH (RoutingKit port, bit-identical) | Re-customise | Via turn-expanded graph | n/a | No | Native | New (Jun 2026), one author |
| **rust_road_router** | Rust (nightly) | BSD-3 | research library | CCH, CATCHUp (time-dependent), ALT, hub labels | Yes | Turn expansion | n/a | No | Native but likely bit-rotted | Inactive since 2022 |
| **fast_paths** | Rust | MIT/Apache | library | CH (node-based) | Rebuild (≈5× faster with saved order) | No (expand yourself, as A/B Street does) | n/a | No | Native | Stable, frozen |
| **RouteE Compass** | Rust | BSD-3 | engine | None (search with pluggable cost models) | Fully pluggable | Turn delays | n/a | No | Native | Active (NREL) |
| **Itinero 2** | C# | Apache-2.0 | library | None (turn-aware Dijkstra/A\*), editable tiles | Per query | Yes | Possible | No | No | Very active |

Others, briefly: **Routino** (C, AGPL, alive), **pgRouting** (GIS work, not serving), **OSMnx / Pandana** (analysis only), **R5** (accessibility analysis), **Navitia** (legacy, avoid). **Ferrostar** and **MapLibre Navigation** are navigation clients, not routers: they consume OSRM/Valhalla-format responses. CycleStreets, Kurviger and Magic Lane are closed source; gosmore, YOURS and Graphast are dead.

---

## Recurring design patterns

1. **Turns live on edges.** Every serious engine routes on an edge-based / turn-expanded graph (or an equivalent edge-keyed search state). OSRM's restriction graph handles multi-via restrictions exactly. The CCH survey says turn expansion is the only competitive way to combine turns with CCH.
2. **Precompute vs. flexibility is the central trade-off:**
   - CH: fastest, but locked to one fixed weighting per profile.
   - CCH / MLD (CRP): structure computed once, weights in about 1 s.
   - Partition overlays (OM leaps, OsmAnd HH): cheap and offline-friendly, but approximate. OsmAnd adds a verify-and-repair step to correct them.
   - No preprocessing (Valhalla, BRouter, Itinero, osr): anything goes per request, and queries are slower.
3. **Zero-copy, memory-mapped data:**
   - Valhalla tiles are packed structs.
   - OSRM serves tar blocks through mmap or shared memory, with one type acting as both builder and view.
   - nigiri and osr use cista mmaps.
   - Organic Maps routes over the same feature records it renders.
4. **Two speeds per road** (Organic Maps): a preference "weight" speed for search and an ETA speed for times. This lets you tune route choice without wrecking ETAs.
5. **Profile evaluation cached per unique tag combination** (BRouter, OsmAnd). This makes a scripted, flexible profile language cheap at query time.
6. **Offline packaging:**
   - Data is split into per-region files that load lazily.
   - A world-level border graph detects which regions a route needs but you haven't downloaded (OM).
   - Rerouting splices the new route back onto the old one a short way ahead (OM, OsmAnd).
   - Navit's backward search makes rerouting after a deviation nearly free.
7. **Interoperable output:** emitting OSRM/Mapbox-compatible route JSON (as Valhalla does) gets you Ferrostar and MapLibre Navigation clients for free. For transit, MOTIS's OpenAPI contract plays the same role.

---

## Algorithm landscape (road)

Figures for Europe, from the CCH survey (arXiv 2502.10519) cited in `libraries-and-rust.md`:

| Technique | Preprocessing | Weight change | Query | Open implementations |
|---|---|---|---|---|
| Dijkstra / A\* / bidirectional A\* | none | free | 100 ms – seconds | everywhere |
| ALT (landmarks) | minutes | must stay admissible | tens of ms | GraphHopper LM, ORS core, rust_road_router |
| CH | minutes – hours per profile | full rebuild | ~0.1 ms | OSRM, GraphHopper, RoutingKit, fast_paths |
| CCH | ~6 min ordering (InertialFlowCutter) | ~1.25 s on 16 threads | ~0.14 ms | RoutingKit, `cch`, cch-rs, rust_road_router |
| MLD / CRP | partition once | ~1 s | ~1.6 ms | OSRM MLD; OM and OsmAnd use one-level variants |
| Hub labels, Transit Node Routing, arc flags | heavy | rebuild | µs | no maintained open implementation worth depending on |
| Time-dependent (TD-CCH, CATCHUp) | | | | rust_road_router (research) |

Transit: RAPTOR and its variants (nigiri, OTP2, vulture), Trip-Based (nigiri, GraphHopper), CSA (ULTRA, trRouting), ULTRA's transfer shortcuts (KIT). There is no maintained open Transfer Patterns implementation.

---

## What to do, by goal

routet's goals aren't stated yet, so here is a recommendation for each plausible direction.

| If routet should be… | Recommendation |
|---|---|
| **A working router, soon, from Rust** | Embed Valhalla through the `valhalla` crate (per-request costing, guidance, map matching, isochrones, offline tiles). Or run OSRM/GraphHopper over HTTP if throughput matters more than flexibility. |
| **A new Rust road engine (server)** | Use a turn-expanded graph with CCH (start from the `cch` crate; pin or fork it). Use OSRM's extractor and restriction graph and GraphHopper's encoded values as models. Store data mmap-first. Emit OSRM-compatible JSON. Use OSRM's ~1.5k Cucumber scenarios as a conformance suite and GraphHopper/OSRM as correctness baselines. |
| **Offline / on-device (phone)** | Study Organic Maps (mwm sections, cross-mwm leaps, two speeds per road, rerouting) and OsmAnd (OBF lazy loading, HH verify-and-repair, rule profiles cached per tag set). Consider CCH per region plus a border overlay between regions. Valhalla tiles are a ready-made alternative. |
| **Bike / foot with rich preferences** | Learn from BRouter's profile language and per-tag-set caching, OsmAnd's `routing.xml`, Organic Maps' elevation penalties and RouteE Compass's cost-model API. Little or no preprocessing may be enough at city/regional scale. |
| **Public transit / multimodal** | Deploy MOTIS if you just need it. To build: port nigiri's timetable layout and RAPTOR/PONG, take multi-criteria ideas from OTP's McRaptor, and reuse osr (via `osr-ffi`) or its design for walking and transfers. |

---

## Suggested reading order (source)

1. `~/g/osrm-backend`: the edge-based graph factory, restriction graph, CH contractor, MLD partitioner and customiser (10-file list in `osrm.md`).
2. `~/g/RoutingKit` or the `cch` crate: compact CCH.
3. `~/g/valhalla`: `baldr` tile structs, `sif` costing, `thor/bidirectional_astar.cc`, `meili`, `odin` guidance.
4. `~/g/graphhopper`: `EdgeBasedNodeContractor`, `CustomModelParser`, LM admissibility checks, QueryGraph virtual nodes.
5. `~/g/organicmaps/libs/routing`: `index_graph`, `cross_mwm_*`, `edge_estimator.cpp`, `turns_generator`.
6. OsmAnd-java `router/` (HH planner) and OsmAnd-core-legacy `native/src/`. Note that OsmAnd-core's `src/Routing/` is dead code.
7. `~/g/nigiri`, `~/g/osr`: mmap'd timetable and street graph.

---

## Caveats

- **Unverified figures:** the planet build and RAM numbers come from project docs of varying age (OSRM 2021, GraphHopper 2022 data, Valhalla's repo notes). The osr "≤10 GB planet import" claim and any absolute MOTIS latencies were not verified.
- **Young or risky dependencies:** the `cch` crate is new and has a single author. rust_road_router needs nightly Rust and probably no longer builds. OSRM's 2026 revival depends heavily on one person, and its datasets break across monthly releases.
- **License traps:** OsmAnd and openrouteservice are GPL-3. Routino and MOTIS's geocoder (adr) are AGPL. Catenary's backend is AGPL. OM, CoMaps, Valhalla, OSRM, GraphHopper, nigiri and osr are permissive.
- **Gaps:** no benchmarks were run, apart from a quick `fast_paths` run on a 40k-node Bremen test graph (~300 ms build, ~12 µs per query).
