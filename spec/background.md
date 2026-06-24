# Background (Concept Preparation)

Created: 2026-06-24

Purpose
- Extract and organize background information from the attached CONOPS document (FOIL4G Hokkaido, CONOPS v5.2) to serve as a basis for subsequent specification discussions.
- This document focuses on factual extraction and issue framing; it does not make design decisions or normative prescriptions.

1. Problem statement (current challenges)

Facts
- Multiple public agencies produce geospatial information (volcano maps, land use, evacuation zones, etc.), but these resources are maintained separately and it is difficult to combine them on demand into a single, immediately usable map.
- Many organizations operate in networked environments where direct access to external web map services is restricted.

Interpretation candidates (to be verified)
- Some practitioners likely require "on-demand situation awareness" via combined data layers. CONOPS notes that web maps often assume interactive operation; whether this mismatch affects all user groups requires further validation.

2. Fundamental constraints (network, responsibility separation, operational constraints)

Facts
- The CONOPS assumes generation AI running within enterprise environments will not directly access Internet map services.
- Security, network separation, and information governance commonly restrict external access in many operational organizations.

Interpretation candidates (to be verified)
- Given these constraints, separating "intent interpretation within the enterprise" from "map rendering on the Internet side" is a pragmatic architectural choice.

3. Why a two-agent split is proposed

Facts
- The CONOPS proposes a Staff Agent (enterprise-side AI) that generates a Map Intent (YAML), and a Cartographer Agent (Internet-side) that renders maps based on that intent; a human mediates the handoff via copy-and-paste.
- The Cartographer output is expected to be a browser map (MapLibre GL JS).

Interpretation candidates (to be verified)
- The split clarifies accountability because a human performs the transfer; this fits governance constraints in enterprise contexts.

4. Information-management concerns (leakage, misperception, sharing unit)

Facts
- The CONOPS highlights the risk that URL sharing can leak intent or context, and it favors designs that make Map Intent (not URLs) the primary share artifact.
- Server-side logs or persistent storage can accumulate sensitive information and therefore present governance concerns.

Interpretation candidates (to be verified)
- Limiting sharing to Map Intent text may improve reproducibility, but it introduces trade-offs in collaborative workflows and convenience; the operational burden is not yet quantified.

5. Local context (extractions from CONOPS)

Facts (CONOPS excerpts)
- Document: CONOPS FOIL4G Hokkaido ("Dodo Information Base" third-layer experiment) v5.2
- Created: 2026-06-24
- Operated by: UN Smart Maps Group
- Participants include: Geospatial Agency (GSI) Hokkaido Surveying Division
- Example PMTiles referenced in the document:
  - `gsi.kazan_tochijoken.pmtiles` (volcano land-condition map)
  - `gsi.kazan_kihonzu.pmtiles` (volcano base map)
  - `jma.funka_keikai.pmtiles` (volcanic warning area)
  - `rfc.kokuyurin.pmtiles` (national forest)
  - `hpg.hinan_kuiki.pmtiles` (evacuation zones)

Stakeholder agencies noted in CONOPS:
- Japan Meteorological Agency (volcanic warning areas)
- Forestry Agency (national forest data)
- Hokkaido Prefecture (evacuation zones, disaster information)

Interpretation candidates (to be verified)
- These agency-specific datasets are treated in the CONOPS as an initial, region-specific corpus; in a general specification it is preferable to abstract them into catalog metadata and catalog types so the same architecture can apply elsewhere.

6. Generalized issues (abstracted from local context)

- Catalog precedence and conflict resolution when multiple catalogs contain equivalent layers
- Clear responsibility boundaries between enterprise-generated intent and Internet-side rendering
- Operational rules and training for sharing artifacts (Intent vs URL)
- Logging, persistence, and audit practices that balance observability and confidentiality

7. Unresolved questions (for the next discussions)

- Minimal provenance fields to retain during layer resolution (example: `catalog_id`, `catalog_type`, `version`, `resolved_at`) — to be finalized
- Standard error response shape when resolution fails (example: `error_code`, `missing_layers`, `suggested_action`) — to be finalized
- Trade-offs between collaboration convenience and confidentiality (permalinks, encrypted tokens, etc.)
- Map Intent versioning strategy and backward compatibility rules (policy for `spec_version`) — to be finalized

Note: The above items are discussion starters; priorities and implementation details require further agreement.

8. Terminology memo

- Map Intent: a structured text payload expressing user goal, target area, and required layers (CONOPS assumes YAML).
- Staff: the enterprise-side actor (agent or person) that interprets user requests and generates Map Intent.
- Cartographer: the Internet-side actor/service that renders maps from Map Intent.
- Library: the set of data catalogs and geospatial resources (tiles, PMTiles, STAC, layers.txt, etc.).

9. Source excerpts (direct CONOPS examples)

- "Staff AI (enterprise AI): interprets staff-entered questions and generates Map Intent"
- "Cartographer AI (map-generation AI): generates browser-ready maps from the design document"
- "PMTiles files arranged in a single folder" / "implemented and served via tile server (Martin)"
- Map Intent example: `required_layers: - source_id: jma.funka_keikai`

10. Non-normative proposals (discussion material)

The following are non-normative proposals derived from the review, intended for discussion and not yet part of the specification:

- Candidate minimal provenance fields to include in Map Intent:
  - `catalog_context`: configured catalog list (each entry: `id`, `type`, `uri`, `version`)
  - `resolved_layers`: mapping of each `source_id` to the `catalog_id` and `catalog_type` used to resolve it
  - `generated_by`: identifier of the Staff component that generated the intent (e.g. `staff:source_name`)
  - `generated_at`: ISO8601 timestamp

- Candidate minimal error response (Cartographer → client) for missing layers:
  - `error_code` (string)
  - `message` (human-readable string)
  - `missing_layers` (array of `source_id`)
  - `provenance_snapshot` (subset of `catalog_context` and attempted resolution info)
  - `suggested_action` (optional guidance)

These are design candidates; field names and types should be harmonized with the overall Map Intent schema in a later step.

Quality note
- This document is a factual extraction and interpretation summary based on the CONOPS document. It intentionally avoids prescriptive design statements; unresolved items are left for later decision processes.