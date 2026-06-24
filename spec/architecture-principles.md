# Architecture Principles

Status: Draft v0.1
Updated: 2026-06-24

## 1. Purpose

This document defines normative architecture principles for a four-party model:

- User
- Staff
- Cartographer
- Library

The design goal is practical decision support with explicit accountability boundaries and minimal information leakage across trust boundaries.

## 2. Core Principles

1. Separation of responsibilities
- User owns intent.
- Staff owns semantic interpretation.
- Cartographer owns rendering.
- Library owns data publication and discoverability.

2. Human accountability handoff
- Handoff from Staff output to Cartographer input MUST remain human-mediated in baseline operations.

3. Convention over custom protocol
- Widely adopted conventions MUST be preferred over project-unique protocol definitions.

4. Reproducibility over convenience
- Decisions that affect map output SHOULD be represented in Map Intent.

5. Least disclosure
- Content transferred to internet-side components MUST be minimized to what is required for rendering and explanation.

## 3. Canonical Terminology

The following terms are normative across all core specifications.

- `Map Intent`: the YAML payload shared across parties.
- `catalog_entry`: one configured catalog item under `catalog_context.active_catalogs`.
- `catalog_type`: one of `martin`, `layers_txt`, `stac`.
- `source_id`: the canonical layer identifier emitted into `required_layers` or `optional_layers`.
- `resolution_policy.precedence`: ordered list of `catalog_type` values.

## 4. Faceless Cartographer Pattern

Cartographer instances are internet-facing and MUST be designed as faceless endpoints.

### 4.1 Endpoint shape

- Public interactive endpoint MUST be `/`.
- Semantic URL paths MUST NOT be used for map state.
- Query parameters and URL hash MUST NOT carry map state.

### 4.2 Request behavior

- `GET /` MUST return an HTML page for posting Map Intent.
- `POST /` MUST accept Map Intent and render a web map.
- Rendered map URLs MUST remain clean (`/`) during interaction.

### 4.3 Share model

- Primary share artifact MUST be Map Intent text.
- URL sharing MUST NOT be the recommended sharing channel.
- UI SHOULD provide a visible `Copy Map Intent` action.

### 4.4 Data minimization and logging

- Server implementations SHOULD avoid persistent storage of posted Map Intent.
- Access/error logs MUST NOT include raw Map Intent payloads.
- Client-side automatic persistence of Map Intent SHOULD be disabled by default.

## 5. Party Characterization

## 5.1 User

Allowed:
- Submit mission-oriented questions.
- Review and transfer Map Intent.

Forbidden:
- Treating URL state as official share artifact.

Accountable for:
- Final transfer decision across trust boundary.

## 5.2 Staff

Allowed:
- Interpret user requests using startup-specified catalogs.
- Produce Map Intent with catalog provenance.

Forbidden:
- Using unspecified catalogs as hidden fallback.

Accountable for:
- Semantic correctness and minimization of sensitive context.

## 5.3 Cartographer

Allowed:
- Parse Map Intent and render map + explanation.

Forbidden:
- Inferring identity or enterprise context beyond provided intent.
- Encoding state into shareable URL.

Accountable for:
- Deterministic rendering behavior under declared inputs.

## 5.4 Library

Allowed:
- Publish catalog metadata and geospatial resources.

Forbidden:
- Opaque or undocumented catalog mutation.

Accountable for:
- Discoverability, source traceability, and schema continuity.

## 6. Relationship to Other Specs

- Map Intent schema and fields: see [map-intent-vnext.md](map-intent-vnext.md)
- Catalog model and resolution behavior: see [catalog-integration.md](catalog-integration.md)
