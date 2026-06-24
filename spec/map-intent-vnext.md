# Map Intent vNext

Status: Draft v0.1
Updated: 2026-06-24

## 1. Purpose

Map Intent is the primary share artifact between parties.
It is a structured payload that captures user goal, layer requirements, and provenance needed for reproducible rendering.

## 2. Normative Requirements

1. Map Intent MUST be serializable as YAML.
2. Map Intent MUST include catalog provenance.
3. Map Intent MUST be shareable as plain text independent of URL state.
4. Optional rendering state MAY be included and updated for re-use.

## 3. Canonical Term Alignment

This document uses the canonical terms defined in [architecture-principles.md](architecture-principles.md).

- `catalog_context.active_catalogs[*]` is a `catalog_entry`.
- `catalog_context.active_catalogs[*].type` is `catalog_type`.
- `required_layers[*].source_id` and `optional_layers[*].source_id` are canonical `source_id` values.

## 4. Schema (Draft)

```yaml
spec_version: "map-intent/v2"

goal: "Understand evacuation zones and land use around target volcano"

area:
  name: "target area name"
  bbox: [lon_w, lat_s, lon_e, lat_n]  # optional

catalog_context:
  active_catalogs:
    - id: "martin-catalog"
      type: "martin"
      uri: "https://library.example/catalog"
      version: "2026-06-24"
    - id: "gsi-layers"
      type: "layers_txt"
      uri: "https://example.org/layers.txt"
      version: "2026-06-20"
    - id: "stac-main"
      type: "stac"
      uri: "https://example.org/stac"
      version: "1.0.0"
  resolution_policy:
    precedence: ["martin", "layers_txt", "stac"]
    on_conflict: "first_match"

required_layers:
  - source_id: "hazard.evacuation_zone"
    label: "Evacuation Zone"
  - source_id: "landuse.current"
    label: "Land Use"

optional_layers:
  - source_id: "forest.admin"
    label: "Forest Boundary"

relationships_to_highlight:
  - "overlap between evacuation zones and populated land use"

render_hints:
  center: [lng, lat]   # optional
  zoom: 10.5           # optional
  bearing: 0           # optional
  pitch: 0             # optional

sharing_policy:
  url_share: false
  intent_share: true

provenance:
  generated_by: "staff-agent-name"
  generated_at: "2026-06-24T12:34:56Z"
  intent_id: "uuid-or-ulid"
```

## 5. Field Semantics

- `catalog_context.active_catalogs`: catalogs used by Staff during interpretation.
- `catalog_context.resolution_policy`: deterministic behavior when equivalent candidates overlap.
- `render_hints`: optional camera state; included for practical re-opening.
- `sharing_policy`: explicit declaration that sharing target is the intent payload, not URL.
- `provenance`: minimum audit metadata.

## 6. Validation Rules (Draft)

1. `spec_version`, `goal`, `catalog_context`, `required_layers`, and `provenance` are REQUIRED.
2. `required_layers` MUST contain at least one layer entry.
3. Each `catalog_entry` MUST have `id`, `type`, and `uri`.
4. `resolution_policy.precedence` MUST only contain `catalog_type` values that exist in `active_catalogs`.
5. `sharing_policy.url_share` SHOULD be `false` in faceless Cartographer deployments.

## 7. Compatibility

- v1 to v2 migration SHOULD preserve existing fields and append `catalog_context`, `sharing_policy`, and `provenance` when absent.
- Consumers that do not support new fields SHOULD ignore unknown keys and continue rendering required layers.
