# ADR 0001: Adopt Faceless Cartographer as Baseline

Status: Accepted
Date: 2026-06-24
Deciders: Staccato spec maintainers

## Context

The architecture separates four parties: User, Staff, Cartographer, and Library.
Cartographer is internet-facing, while Staff operates in an enterprise context.

Without explicit constraints, URL-based state sharing and server-side persistence can increase perceived and actual information leakage risk:

- URL links can become accidental share artifacts.
- URL parameters/hash can encode mission context.
- Persistent logs/storage can create retention and misuse concerns.
- Stakeholders may misinterpret Cartographer as an enterprise data collector.

The project requires a practical model where the primary share artifact is Map Intent text, not URL state.

## Decision

We adopt the Faceless Cartographer pattern as the baseline deployment model.

1. Endpoint model
- Public interactive endpoint is `/`.
- `GET /` returns an HTML page to submit Map Intent.
- `POST /` accepts Map Intent and renders map output.

2. URL policy
- URL paths, query parameters, and hash MUST NOT carry map state.
- Rendered sessions keep the URL clean as `/`.

3. Share policy
- Primary share artifact is Map Intent text.
- UI SHOULD provide `Copy Map Intent`.
- URL sharing is not a recommended workflow.

4. Data minimization
- Implementations SHOULD avoid persistent storage of posted Map Intent.
- Logs MUST NOT include raw Map Intent payloads.
- Client-side automatic persistence SHOULD be disabled by default.

5. Spec linkage
- Map Intent documents catalog provenance and sharing intent.
- Catalog selection remains constrained by startup-specified catalog configuration.

## Consequences

Positive:

- Clear accountability boundary: human-mediated transfer remains explicit.
- Reduced chance of accidental leakage through shared URLs.
- Stronger alignment between security narrative and product behavior.
- Easier explanation to governance and legal stakeholders.

Negative / trade-offs:

- No deep-link convenience for map state by URL.
- Collaboration requires explicit copy/paste of Map Intent.
- Some UX expectations from common web maps are intentionally not supported.

Operational implications:

- Frontend must include robust Map Intent copy/export affordances.
- Observability design must avoid payload capture while retaining operational metrics.
- Support playbooks should guide users to share Map Intent, not URLs.

## Alternatives Considered

1. URL-shareable map state (query/hash)
- Rejected for baseline due to leakage and governance risk.

2. Opaque permalink IDs with server-side storage
- Rejected for baseline because stored payload/index becomes a sensitive control point.

3. Encrypted token in URL
- Not selected for baseline due to key management and operational complexity.
- May be reconsidered for a future profile if strict controls are introduced.

## Status

Accepted as baseline architecture for current specification set.
Future profiles may define additional sharing modes, but they MUST NOT weaken this baseline without a superseding ADR.
