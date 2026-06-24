# staccato-spec

Practical architecture specifications for the Staccato model.

This repository stores normative specifications under `spec/`.
The `docs/` path is intentionally reserved for GitHub Pages and non-normative content.

## Reading Order

1. [Architecture Principles](spec/architecture-principles.md)
2. [Map Intent vNext](spec/map-intent-vnext.md)
3. [Catalog Integration](spec/catalog-integration.md)
4. [Background](spec/background.md)
5. [Use Cases](spec/usecase.md)

## Repository Structure

- `spec/`: normative specifications (MUST/SHOULD/MAY)
- `spec/adr/`: architecture decision records
- `docs/`: reserved for GitHub Pages and explanatory materials

## Scope

The current baseline defines a four-party model:

- User
- Staff
- Cartographer
- Library

It also defines a faceless Cartographer pattern where URL sharing is intentionally de-emphasized and Map Intent is the primary shareable artifact.

## Staccato Architecture

The Staccato architecture separates responsibilities between an internal `Staff` component (enterprise-side) and an external `Cartographer` component (internet-side).

- `Staff` (internal): runs inside an organization's secure environment, interprets natural-language queries from users, and produces a structured `Map Intent` (YAML). `Staff` resolves layers only from catalogs configured at startup and records `catalog_context` and lightweight provenance in the intent.
- `Cartographer` (external): runs in an internet-accessible environment and consumes posted `Map Intent` to render browser maps (MapLibre GL JS). Cartographer is designed to be faceless: a single `/` endpoint where `GET /` presents an input form and `POST /` accepts Map Intent. It intentionally avoids encoding map state in URLs and minimizes payload persistence and logging to reduce leakage risk.

Key principles:
- Human-mediated handoff: a human copies the `Map Intent` from `Staff` to `Cartographer`, which clarifies accountability.
- Data model: TileJSON 3.0 (collection) is the recommended canonical consumption model for tile-centric catalogs; STAC is supported as a discovery/source model and materialized into TileJSON when PMTiles or tile assets are present.
- Minimal provenance: `Map Intent` should include `catalog_context` and a `generated_at`/`generated_by` provenance snapshot to aid reproducibility.

Purpose: the Staccato architecture aims to provide reproducible, auditable map generation while reducing the risk of accidental leakage from URL sharing or server-side persistence.

## Repository Guide — Quick Start

- What to read first: `spec/architecture-principles.md`, `spec/map-intent-vnext.md`, `spec/catalog-integration.md`, then `spec/background.md` and `spec/usecase.md` for context and examples.
- Normative vs informative: `spec/` contains normative specifications (MUST/SHOULD language), ADRs (`spec/adr/`) record decisions; other files provide background and examples.
- Cartographer consumption model: TileJSON 3.0 (collection) is the recommended canonical model; STAC is supported for discovery and should be materialized to TileJSON for rendering where PMTiles/tile assets exist.
- Contribution flow: propose changes via PR; record important architecture or policy changes as an ADR in `spec/adr/000N-*.md` and reference it from the PR.

If you plan to commit changes, use a clear message such as: "spec: add repository guide; standardize ingestion model to TileJSON 3.0".
