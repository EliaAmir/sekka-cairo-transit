# Sekka — Data Pipeline

## Source Data

Cairo transit data does not exist in any standardized machine-readable form. Raw inputs:

- **CTA PDFs**: Official Cairo Transit Authority timetables and route listings (Arabic)
- **Scraped web sources**: Route name/stop data from transit enthusiast sites

---

## 3-Step Arabic Name Normalization

Arabic stop names in raw source data have significant inconsistency across sources.
Three normalization passes are applied before geocoding:

### Step 1 — Split-Word Fixes
Arabic text scraped from web sources is often missing whitespace between words due to encoding artifacts.

Examples:
- `رمسيسشارع` → `رمسيس شارع`
- `الجيزةمحطة` → `الجيزة محطة`

A dictionary of known split-word patterns is applied via regex substitution.

### Step 2 — Canonical Merges
Multiple name variants for the same physical stop are unified to a single canonical form.

Examples:
- `رمسيس`, `محطة رمسيس`, `ميدان رمسيس` → canonical: `رمسيس`
- `العتبة`, `ميدان العتبة` → canonical: `العتبة`

A canonical map file drives this pass. Merging reduces the stop registry size
and prevents the geocoder from treating the same place as multiple nodes.

### Step 3 — Geo-Disambiguation
Some stop names are genuinely ambiguous — the same name exists in multiple Cairo districts.

Examples:
- `مدرسة النصر` exists in Heliopolis, Nasr City, and Giza
- `الجامعة` could be Cairo University or Ain Shams University

Disambiguation uses the route's known geographic corridor (derived from its other stops)
to select the correct geocoding candidate via bounding-box filtering.

---

## Geocoding Strategy

Geocoding uses **Nominatim** (OpenStreetMap's geocoding API):

1. Query each normalized stop name: `https://nominatim.openstreetmap.org/search?q={name}&countrycodes=eg`
2. Filter results to a Cairo bounding box (lat 29.9–30.2, lng 31.0–31.5)
3. If multiple candidates pass the box filter, apply geo-disambiguation (Step 3 above)
4. Record result with trust level:
   - `HIGH`: single unambiguous result in bounding box
   - `MEDIUM`: disambiguation used
   - `LOW`: closest match, manual review recommended
5. Store as `stop_coordinates` in `external_routes[]`:

```json
{
  "name_ar": "رمسيس",
  "lat": 30.0617745,
  "lng": 31.2465478,
  "trust": "HIGH",
  "source": "nominatim",
  "stop_type": "manual_fix | road | city"
}
```

Current geocoding coverage: **9,117 / 13,012 slots (70.1%)**

---

## Microbus Hub Data Structure

Microbus data uses a hub-and-spoke model (موقف = departure hub):

```json
{
  "hub_id": "hub_ramses",
  "name_ar": "رمسيس",
  "name_en": "Ramses",
  "lat": 30.0617,
  "lng": 31.2465,
  "destinations": [
    {
      "name_ar": "الجيزة",
      "name_en": "Giza",
      "lat": 30.0131,
      "lng": 31.2089,
      "fare_egp": 5
    }
  ]
}
```

27 هubs are currently in `microbus_stops[]`. GPS coordinates for hubs and
Cairo-local destinations are being geocoded in Phase 1 of the data roadmap.

---

## Output: routely_data.json

The final output of the pipeline is `routely_data.json` — a single 3.4 MB JSON file:

| Key | Contents |
|-----|---------|
| `stops[]` | 19 corridor stops with GPS |
| `routes[]` | 7 corridor routes |
| `edges[]` | 30 Dijkstra edges |
| `metro{}` | 84 stations, 3 lines, zone pricing |
| `external_routes[]` | 666 bus/minibus routes with stop_coordinates |
| `microbus_stops[]` | 27 مواقف hubs with destinations |