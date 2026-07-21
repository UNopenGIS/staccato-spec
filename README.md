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

## Reference Implementations

Working, publicly deployed reference implementations exist for two of the four roles:

- **Library**: [`hfu/layers-martin`](https://github.com/hfu/layers-martin) — converts GSI's `layers.txt` into a Martin-compatible static TileJSON catalog (`/catalog` + `/{source_id}`), published via GitHub Pages. Curates ~12,600 raw candidate layers down to a usable ~1,861-entry catalog (suppressing satellite-snapshot noise, deduplicating repeated tile URLs), and extends TileJSON with a `legend_image_url` field. Design decisions are recorded as ADRs in its own `DECISIONS.md`.
- **Cartographer**: [`hfu/faceless-cartographer`](https://github.com/hfu/faceless-cartographer) — a static single-page app (no server, no LLM this generation) deployed at <https://hfu.github.io/faceless-cartographer/>. Implements the faceless pattern (ADR 0001) as a client-side view transition rather than a literal `GET /` / `POST /` HTTP split, since no page ever actually reloads or changes URL. This was originally a deliberate, recorded deviation from the ADR's literal wording (see its `DECISIONS.md`); [ADR 0003](spec/adr/0003-spa-endpoint-model.md) now proposes clarifying ADR 0001 to formally accommodate this interpretation, using this implementation as evidence.

Both are wired together end-to-end: a `Staff` system-prompt addendum lives in `layers-martin`'s `STAFF_PROMPT.md` (iterated against real enterprise-AI role-play), and `faceless-cartographer` fetches and displays it so a compatible `Staff` agent can be set up without leaving the page.

A notable finding from combining a second, independently-operated Library (a live Martin server at `stars.optgeo.org`, publishing GSI's optimized vector-tile basemap) alongside `layers-martin` in one Map Intent: no catalog-aggregator component was needed. `catalog_context.active_catalogs` being an array already supports listing multiple, unrelated Library catalogs side by side, and `Cartographer` resolves and renders both without any merging step. See `UNopenGIS/7#936` and `#938` for the fuller writeup.

That same `stars.optgeo.org` Martin server also publishes complete MapLibre styles (`GET /style/{style_id}`), not just tile sources — `faceless-cartographer` added `required_styles`/`optional_styles` (Map Intent fields referencing a whole published style rather than individual `source_id`s) to let a user ask for a complete, pre-designed thematic map (e.g. "show me the volcanic land condition map") without Staff having to reconstruct it from raw layers. [ADR 0007](spec/adr/0007-style-references.md) proposes formalizing this; two thematic-only styles (extracted from [`hfu/kitavolca`](https://github.com/hfu/kitavolca)) are live at `https://stars.optgeo.org/style/vlcm` and `/style/vbm` as concrete evidence.

## Repository Guide — Quick Start

- What to read first: `spec/architecture-principles.md`, `spec/map-intent-vnext.md`, `spec/catalog-integration.md`, then `spec/background.md` and `spec/usecase.md` for context and examples.
- Normative vs informative: `spec/` contains normative specifications (MUST/SHOULD language), ADRs (`spec/adr/`) record decisions; other files provide background and examples.
- Cartographer consumption model: TileJSON 3.0 (collection) is the recommended canonical model; STAC is supported for discovery and should be materialized to TileJSON for rendering where PMTiles/tile assets exist.
- Contribution flow: propose changes via PR; record important architecture or policy changes as an ADR in `spec/adr/000N-*.md` and reference it from the PR.

If you plan to commit changes, use a clear message such as: "spec: add repository guide; standardize ingestion model to TileJSON 3.0".
