# Sekka — Architecture

## Flutter App Layer Structure

```
lib/
├── data/
│   ├── routely_repository.dart       # Main data loader + Dijkstra routing engine
│   ├── remote_data_service.dart      # Firestore fetch + version-gated cache update
│   ├── local_cache.dart              # Read/write local JSON cache on device
│   └── intent_logger.dart            # Anonymous route-view logging
├── models/
│   └── transit_models.dart           # All data models: ExternalRoute, MicrobusStop, etc.
├── screens/
│   ├── home_screen.dart              # Trip Planner (origin/destination + routing results)
│   ├── metro_navigator_screen.dart   # Standalone metro navigator
│   ├── line_viewer_screen.dart       # Routes Hub — hub-and-spoke browser
│   ├── search_by_number_screen.dart  # Route lookup by number
│   ├── search_by_destination_screen.dart  # Route lookup by from/to stops
│   └── microbus_stations_screen.dart # Microbus مواقف hubs — fully dynamic
├── widgets/
│   └── external_route_card.dart      # Shared expand-in-place route card
└── utils/
    └── device_id.dart                # Anonymous install-scoped device UUID
```

---

## Graph Schema

### graph_nodes
Each node represents a transit stop, metro station, or microbus hub.

```json
{
  "id": "string",
  "name_ar": "string",
  "name_en": "string",
  "type": "metro_station | bus_stop | microbus_hub",
  "lat": 30.0617745,
  "lng": 31.2465478,
  "line": "optional — metro line identifier"
}
```

### graph_edges
Each edge represents either a ride segment or a transfer walk.

```json
{
  "from": "node_id",
  "to": "node_id",
  "type": "ride | transfer | walk",
  "route_id": "optional — for ride edges",
  "duration_min": 5,
  "fare_egp": 5.0,
  "network": "metro | bus | microbus"
}
```

---

## Metro ↔ Bus Transfer Injection

Transfer edges between metro stations and nearby bus stops are injected at data-build time, not hardcoded in the app.

The injection process:
1. For every metro station node, scan all bus/microbus nodes within 800 m (Haversine distance)
2. For each qualifying pair, create a bidirectional transfer edge:
   - `type: "transfer"`
   - `duration_min`: `distance_m / 83.3` (walking at 5 km/h)
   - `fare_egp: 0` (walking is free)
3. Transfer edges are stored in `edges[]` in `routely_data.json` alongside ride edges

Key transfer stations (manually verified):
- Al-Shohadaa (Lines 1 + 2) — major bus interchange
- El-Sadat (Lines 1 + 2) — Tahrir Square hub
- Attaba / Nasser (Lines 1 + 3 adjacency zone)
- Cairo University (Line 2) — Giza bus hub

---

## Dijkstra Cost Functions

Three separate Dijkstra runs on the same graph, each using a different cost function:

### Mode 1 — Fastest
```dart
cost = duration_min + (transfer_count * 3.0)
```
Transfer penalty of 3 minutes per vehicle change accounts for waiting time.
Network wait times added at boarding: metro = 2 min, bus = 5 min, microbus = 3 min.

### Mode 2 — Cheapest
```dart
cost = fare_egp
```
Pure fare minimization. Walk edges cost 0. Transfer edges cost 0.
May produce slower journeys that avoid paid transit segments.

### Mode 3 — Fewest Transfers
```dart
cost = vehicle_change_count * 1000 + duration_min
```
A weight of 1000 per transfer makes minimizing transfers the primary objective,
with duration as the tiebreaker. Produces the most direct (fewest-vehicle) journey.

---

## Firestore Remote Config + Version-Gating

```
Firestore structure:
├── app_config/
│   └── data_manifest          { data_version: "1.0.0", last_updated: timestamp }
└── transit_data/
    └── v1_0_0/
        └── sections/
            ├── stops           JSON doc — corridor stops
            ├── routes          JSON doc — corridor routes
            ├── edges           JSON doc — Dijkstra edges
            ├── metro           JSON doc — 84 stations, 3 lines
            ├── external_routes JSON doc — 666 bus/minibus routes
            └── microbus_stops  JSON doc — 27 مواقف hubs
```

### Version-gating flow
1. App launches → load bundled `routely_data.json` asset (cold-start baseline)
2. `remote_data_service.dart` reads `app_config/data_manifest`
3. Compare remote `data_version` against cached `data_version` in SharedPreferences
4. If remote version is newer: fetch all sections, merge into local cache, update stored version
5. If versions match: use existing local cache
6. If offline: use local cache if present, else fall back to bundled asset

This pattern ensures:
- Zero cold-start latency (bundled data always available)
- Automatic background updates without user action
- Controlled rollout — only clients that fetch the manifest get new data