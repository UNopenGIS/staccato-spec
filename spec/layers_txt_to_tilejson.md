# Mapping: layers.txt → TileJSON 3.0 (collection)

Purpose
- Provide a practical mapping guide for transforming `layers.txt` style layer indices into TileJSON 3.0 collection descriptors that `Cartographer` can consume deterministically.

Scope
- This guidance focuses on tile-centric layers (PMTiles, tile endpoints). STAC interoperability is discussed separately in `catalog-integration.md`.

Assumptions
- `layers.txt` contains lines describing available layers; conventions vary but typically include an identifier, a title, a URL (or tile endpoint), and optional metadata (bounds, zooms, attribution).
- TileJSON 3.0 collection is chosen as the canonical consumption model for Cartographer.

Mapping rules (high level)

1. Parse `layers.txt` into layer entries
- Each non-comment line becomes a candidate `layer_entry`.
- Try to extract: `id`, `title/label`, `url` (tile template or asset), `bounds` (if present), `minzoom`/`maxzoom`, and `attribution`.

2. For each layer_entry produce a TileJSON descriptor
- TileJSON minimal properties to populate:
  - `type`: "Collection" (if using collection envelope) or per-layer TileJSON
  - `id` / `name`: use `id` from layers.txt
  - `tiles`: array of tile URL templates (if `url` is a tile template or PMTiles HTTP endpoint)
  - `bounds`: [west, south, east, north] if available
  - `minzoom` / `maxzoom` if available
  - `attribution` / `attribution_url` if available
  - `version` (set to ingestion timestamp or catalog version)

3. PMTiles handling
- If `url` points to a PMTiles resource (directly or via a hosting pattern), produce a TileJSON entry whose `tiles` array points to a tile endpoint that serves PMTiles (e.g., `/tiles/{z}/{x}/{y}@pbf` behind a PMTiles server) and include PMTiles metadata (size, contents) as `meta`.

4. Nested / grouped layers
- `layers.txt` may imply groups or nested categories. Maintain a `group` property in the TileJSON collection metadata and expose a flat list for resolution while preserving group labels for UI.

5. Conflict and precedence hints
- Attach `catalog_id` and `catalog_type` metadata to each TileJSON entry so `resolution_policy` can prefer entries deterministically.

6. Ingestion flow (recommended)
  1. Read `layers.txt` and parse entries.
  2. Normalize IDs and detect PMTiles assets.
  3. For PMTiles assets, probe the PMTiles metadata (if reachable) to populate bounds/zoom.
  4. Materialize a TileJSON collection document and register it as the canonical descriptor in the `catalog_context`.

STAC interoperability note
- If `layers.txt` references STAC-hosted assets or a STAC catalog contains PMTiles, perform STAC discovery and materialize equivalent TileJSON descriptors for the discovered PMTiles assets. This keeps Cartographer ingestion deterministic.

Example mapping (pseudo)

layers.txt line:
  `gsi.kazan_kihonzu | Volcano base map | https://example.org/tiles/gsi.kazan_kihonzu/{z}/{x}/{y}.pbf | bounds=... | minzoom=6 | maxzoom=14`

Resulting TileJSON (fragment):
```json
{
  "id": "gsi.kazan_kihonzu",
  "name": "Volcano base map",
  "tiles": ["https://example.org/tiles/gsi.kazan_kihonzu/{z}/{x}/{y}.pbf"],
  "bounds": [lon_w, lat_s, lon_e, lat_n],
  "minzoom": 6,
  "maxzoom": 14,
  "catalog_id": "gsi-catalog",
  "catalog_type": "layers_txt"
}
```

Notes and caveats
- `layers.txt` has variations; implement robust parsing and conservative defaults.
- Where metadata is missing, probe services (if allowed) to infer bounds/zoom from tile metadata or PMTiles headers.
- For disconnected enterprise environments, ingestion must be done during catalog staging (offline materialization).
