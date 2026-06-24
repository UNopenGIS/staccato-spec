# Catalog Integration

Status: Draft v0.1
Updated: 2026-06-24

## 1. Purpose

This document defines how the system handles multiple catalog ecosystems:

- Martin catalog
- `layers.txt` (gsi-cyberjapan/layers-txt-spec)
- STAC

The objective is predictable layer discovery and reproducible source selection.

## 2. Canonical Term Alignment

This document uses the canonical terms defined in [architecture-principles.md](architecture-principles.md).

- Configured catalog items are `catalog_entry` values.
- Catalog discriminator values are `catalog_type` values.
- Emitted layer identifiers are canonical `source_id` values.

## 3. Catalog Types

## 3.1 `martin`

- Optimized for tile service publication metadata.
- Commonly aligned with PMTiles or server-managed vector/raster sources.

## 3.2 `layers_txt`

- Interoperable with layer list conventions used in Japanese geospatial contexts.
- Useful as an externally curated layer index.

## 3.3 `stac`

- Asset-centric geospatial cataloging model.
- Supports collection/item style discovery beyond tile-centric catalogs.

## 4. Staff Startup Contract

1. Staff MUST receive active catalog configuration at startup.
2. Staff MUST restrict interpretation to configured catalogs.
3. Staff MUST record used catalogs in Map Intent `catalog_context`.
4. Hidden fallback catalogs MUST NOT be used.

## 5. Canonical Layer Resolution Model (Draft)

Staff resolves user intent to source IDs using this order:

1. Build candidate set per active catalog.
2. Normalize candidate IDs into canonical `source_id` values.
3. Apply precedence order from `resolution_policy.precedence`.
4. Apply conflict rule (`first_match` by default).
5. Emit selected `source_id` values into `required_layers` and `optional_layers`.

## 6. Conflict Handling

When equivalent semantic layers exist in multiple catalogs:

1. Resolution MUST follow declared precedence.
2. The losing candidates SHOULD be omitted from rendering fields.
3. Provenance SHOULD retain enough metadata for audit/replay.

## 7. Minimal Interop Mapping

Each chosen layer SHOULD preserve mapping references:

- internal `source_id`
- catalog `type`
- catalog `id`
- original identifier in that catalog

This mapping MAY be emitted in a future extension field (for example `catalog_bindings`).

## 8. Operational Guidance

1. Prefer stable catalog URIs and explicit version tags.
2. Track schema and endpoint changes as ADR entries.
3. Keep catalog precedence small and explicit to reduce ambiguity.

## 9. Relationship to Other Specs

- Architectural rationale: see [architecture-principles.md](architecture-principles.md)
- Payload schema: see [map-intent-vnext.md](map-intent-vnext.md)

## 10. Preferred Information Model (Recommendation)

Proposal:
- Use TileJSON 3.0 (collection model) as the canonical information model that `Cartographer` consumes when resolving catalog entries from `martin`, `layers_txt`, and other tile-centric catalogs.

Rationale:
- TileJSON 3.0 is tile-centric and aligns naturally with PMTiles/Tile servers and MapLibre rendering workflows.
- A TileJSON collection can enumerate individual tile/asset descriptors and metadata (bounds, zooms, attribution) in a way that is directly useful to a Cartographer implementation.

STAC note:
- STAC is an asset-centric catalog model and is a valuable source model; however, Cartographer should treat STAC as a source of datasets rather than the primary tile-collection model. A practical approach is:
	1. If a STAC Item/Collection contains PMTiles or tile assets, convert or map them to the TileJSON collection representation at ingestion time.
 2. Support a shallow STAC subset interpretation for discovery, and then materialize TileJSON descriptors for rendering.

This approach allows immediate, deterministic consumption for Cartographer while leaving STAC support as a complementary discovery pathway.
