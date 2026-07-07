# ADR 0003: Client-Side SPA Transition Satisfies the Faceless Cartographer Endpoint Model

Status: Proposed
Date: 2026-07-06
Deciders: Staccato spec maintainers

## Context

ADR 0001 (Faceless Cartographer) specifies an "Endpoint model" in literal HTTP terms:

> Public interactive endpoint is `/`.
> `GET /` returns an HTML page to submit Map Intent.
> `POST /` accepts Map Intent and renders map output.

The first working reference implementation of Cartographer, [`hfu/faceless-cartographer`](https://github.com/hfu/faceless-cartographer), does not implement this literally. It is a static single-page app deployed to GitHub Pages, with no server component at all. Submitting the form is a client-side JavaScript state transition (parse Map Intent → resolve catalog → build MapLibre style → swap the DOM), not an HTTP `POST /` request-response cycle. The URL never changes at all, not even to reflect a "submitted" state.

This was a deliberate, recorded implementation choice (see that repository's `DECISIONS.md`, D18), made because:

- The core rendering pipeline has no server-side dependency (no LLM, no persistence) that would require a server process.
- Removing the server entirely also removes essentially all of the leakage surface ADR 0001 §4 (Data minimization) is concerned with — there is no log, no persistent storage, and no payload capture to worry about, because there is no server to do the capturing.
- A static site can be hosted for free on GitHub Pages and requires no operational maintenance, which matters for a small reference implementation.

The implementer noted this as an intentional deviation from ADR 0001's literal wording but did not have the standing to amend the spec repository directly. This ADR is the follow-up: reconcile the spec's text with the one implementation that has actually been built and deployed against it.

## Decision

We clarify ADR 0001 §1 (Endpoint model) as follows. The baseline requirement is a **single interactive surface at `/`** that never encodes map state in the URL (§2) and treats Map Intent text as the primary share artifact (§3). This MAY be satisfied by either:

1. **Literal HTTP**, as originally described: `GET /` returns a form, `POST /` accepts Map Intent and renders output server-side.
2. **A client-side SPA transition**: a single static page where submitting Map Intent is handled entirely in the browser (parse, resolve, render), with no navigation, no additional HTTP request beyond fetching catalog/tile data, and no URL change of any kind.

Implementations choosing option 2 automatically satisfy ADR 0001 §4 (Data minimization) for any concern that depends on a server process existing (log capture, persistent storage) simply because no server process is present to violate it. They remain subject to §4's concerns that apply client-side (e.g., not persisting Map Intent to browser storage without cause).

This ADR does not change §2 (URL policy) or §3 (Share policy); a compliant SPA implementation trivially satisfies both, since its URL never changes under any circumstance.

## Consequences

Positive:

- Cartographer implementations with no server-dependent features (no LLM, no persistence) can be deployed as static sites, lowering hosting cost and operational burden to near zero. This broadens who can stand up a spec-compliant Cartographer.
- Removes an ambiguity that would otherwise mark the only working reference implementation as non-compliant with the spec it was built against.
- Makes explicit that the ADR's constraint is behavioral (no URL state, Map Intent as share artifact, minimal payload retention), not a mandate for a particular transport mechanism.

Negative / trade-offs:

- Implementations that do need a server (e.g., for an LLM-based explanation feature, per `hfu/faceless-cartographer`'s own backlog) still require one; this ADR does not exempt them from the rest of ADR 0001 once a server exists.
- Two valid endpoint shapes (literal HTTP vs. SPA) means spec readers and future implementers must check which one a given Cartographer instance uses before assuming `POST /` is a meaningful integration point (e.g., for automated testing or monitoring).

## Alternatives Considered

1. **Leave ADR 0001's text as literal HTTP and treat the SPA implementation as a non-conforming deviation.**
   Rejected: the SPA implementation satisfies every actual concern ADR 0001 was written to address (leakage risk, URL state, share artifact), and arguably exceeds the baseline (URL never changes at all, vs. "kept clean as `/`"). Marking it non-compliant over transport mechanics the ADR's own rationale never cared about would be a spec/reality mismatch with no corresponding safety benefit.

2. **Edit ADR 0001's text in place.**
   Rejected in favor of a new, clarifying ADR, consistent with this repository's existing pattern of `spec/adr/` records being append-only historical entries (ADR 0002 coexists with ADR 0001 rather than rewriting it). A new ADR keeps the original decision's context legible and dated, and makes the clarification itself reviewable and revertible independent of the original.

## Status

Proposed. `hfu/faceless-cartographer` is offered as evidence that the SPA interpretation is a viable, already-deployed instance of the Faceless Cartographer baseline, not a hypothetical relaxation.
