# ADR 0002: Enforce Staff Startup Catalog Contract and Ban Hidden Fallback

Status: Accepted
Date: 2026-06-24
Deciders: Staccato spec maintainers

## Context

Staff interprets user intent and emits Map Intent.
If Staff can silently consult unspecified catalogs, output provenance becomes ambiguous and replayability is weakened.

The specification already introduces multi-catalog support (`martin`, `layers_txt`, `stac`) and requires provenance in Map Intent.
To make this auditable and reproducible, catalog scope must be fixed at startup and hidden fallback behavior must be prohibited.

## Decision

We establish a strict startup catalog contract for Staff.

1. Startup contract
- Staff MUST receive an explicit `catalog_context.active_catalogs` configuration at startup.
- Each configured catalog entry MUST include `id`, `type`, and `uri`.

2. Scope restriction
- Staff MUST only use configured catalogs for interpretation and layer resolution.
- Staff MUST NOT query, infer, or auto-discover external catalogs outside configured scope during normal operation.

3. Hidden fallback prohibition
- Hidden fallback catalogs are prohibited.
- If required layers cannot be resolved from configured catalogs, Staff MUST return a bounded failure outcome instead of silently substituting alternative catalogs.

4. Deterministic conflict handling
- Staff MUST apply `catalog_context.resolution_policy.precedence`.
- Staff MUST apply declared conflict behavior (default: `first_match`).

5. Provenance output
- Staff MUST include `catalog_context` and `provenance` in emitted Map Intent.
- Catalog types listed in `resolution_policy.precedence` MUST be a subset of configured catalog types.

## Consequences

Positive:

- Stronger reproducibility and auditability of Map Intent generation.
- Clear governance boundary for enterprise-side interpretation behavior.
- Reduced risk of unapproved data source usage.

Negative / trade-offs:

- Lower convenience when configured catalogs are incomplete.
- More operational burden to maintain startup catalog configuration quality.

Operational implications:

- Runtime startup validation is required for Staff configuration.
- Failure responses should guide operators to update catalog configuration rather than bypassing policy.
- Testing must cover unresolved-layer and conflict scenarios.

## Alternatives Considered

1. Best-effort fallback to any reachable catalog
- Rejected due to provenance ambiguity and policy non-compliance risk.

2. User-approved interactive fallback at runtime
- Not selected for baseline because it complicates deterministic replay and introduces inconsistent operator behavior.

3. Environment-specific fallback allowlist
- Deferred for a future profile; baseline remains strict no-hidden-fallback.

## Status

Accepted as baseline behavior for Staff in the current specification set.
Any relaxation of fallback constraints requires a superseding ADR.
