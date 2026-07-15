# ADR Index

Architecture Decision Records (ADR) for normative design decisions.

Recommended file naming:

- `0001-title.md`
- `0002-title.md`

Suggested ADR template sections:

1. Context
2. Decision
3. Consequences
4. Alternatives Considered
5. Status

## Records

- [0001-faceless-cartographer.md](0001-faceless-cartographer.md): Adopt Faceless Cartographer as baseline
- [0002-staff-startup-catalog-contract.md](0002-staff-startup-catalog-contract.md): Enforce Staff startup catalog contract and ban hidden fallback
- [0003-spa-endpoint-model.md](0003-spa-endpoint-model.md): Client-side SPA transition satisfies the Faceless Cartographer endpoint model (Proposed)
- [0004-fragment-handoff.md](0004-fragment-handoff.md): One-shot, cleared-on-load URL fragments satisfy the Faceless Cartographer URL policy (Proposed)
- [0005-session-controlled-fragment-reflection.md](0005-session-controlled-fragment-reflection.md): Session-scoped URL-fragment reflection controlled by intent policy (Proposed)
- [0006-cartographer-faceless-vs-idempotent-modes.md](0006-cartographer-faceless-vs-idempotent-modes.md): Cartographer implementation patterns: faceless-by-default vs. idempotent modes (Proposed)
