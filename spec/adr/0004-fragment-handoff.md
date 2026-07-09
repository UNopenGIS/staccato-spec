# ADR 0004: One-Shot, Cleared-on-Load URL Fragments Satisfy the Faceless Cartographer URL Policy

Status: Proposed
Date: 2026-07-09
Deciders: Staccato spec maintainers

## Context

ADR 0001 §2 (URL policy) states:

> URL paths, query parameters, and hash MUST NOT carry map state.
> Rendered sessions keep the URL clean as `/`.

[`hfu/faceless-cartographer`](https://github.com/hfu/faceless-cartographer) (D32, [issue #3](https://github.com/hfu/faceless-cartographer/issues/3)) implements a convenience feature that touches this clause: a Map Intent can be supplied as a URL fragment (`#intent=<base64url>`), letting a Staff agent or a human hand off a fully-specified Map Intent as a single clickable link, without pasting YAML into the textarea.

Unlike a query string or path segment, a URL fragment is never transmitted in any HTTP request — it is parsed client-side only. This means a fragment-based hand-off cannot violate ADR 0001 §4 (Data minimization): there is nothing for a server to log, because the server never receives it. The literal wording of §2, however, forbids hash from carrying map state at all, with no exception for this case.

The reference implementation's mitigation: the fragment is read at most once, synchronously, on initial page load, and immediately cleared via `history.replaceState` before any rendering occurs. A user who copies the address bar URL after the map has rendered gets a clean `/` — the same as pasting Map Intent text through the form. The fragment never becomes a bookmarkable or reloadable representation of map state; it functions purely as a one-shot transfer channel, analogous to a Staff agent handing a human a block of YAML text to paste, except the paste step is automated.

## Decision

We clarify ADR 0001 §2 (URL policy) as follows. The baseline requirement remains that **no persistent or reloadable URL may carry map state**. This MAY be satisfied by either:

1. **No hash usage at all**, as the original wording implies.
2. **A one-shot fragment hand-off**: an implementation MAY read `location.hash` for an initial Map Intent, provided it does so at most once per page load, and clears it (e.g. via `history.replaceState`) before rendering any map state derived from it. A rendered session's URL MUST be indistinguishable from one reached without a fragment (i.e., still clean as `/`).

Implementations choosing option 2 remain fully subject to ADR 0001 §3 (Share policy: Map Intent text is still the primary share artifact; a fragment URL is a convenience wrapper around that same text, not a competing mechanism) and §4 (Data minimization: a fragment is server-invisible by construction, so it introduces no new logging/storage concern).

This ADR does not change §1 (Endpoint model) or §5 (Spec linkage).

## Consequences

Positive:

- Enables one-click Map Intent hand-off (e.g., a Staff agent producing a clickable link instead of a block of YAML to copy-paste), without reintroducing any server-visible or persistent URL state.
- Keeps the core guarantee ADR 0001 was written to protect — no accidental leakage through a shared/logged/bookmarked URL — fully intact, since a cleared-on-load fragment cannot be replayed by revisiting a saved URL.
- Makes explicit that §2's constraint is about *persistence and server-visibility*, not about the hash character class categorically.

Negative / trade-offs:

- Two valid hash behaviors (none, vs. read-once-and-clear) means implementers must check which one a given Cartographer instance uses before assuming a hash-bearing URL is safe to bookmark or replay (it generally is not, by design).
- A cleared fragment is still visible in browser history / clipboard history / any tooling that captured the URL before the JS ran — this is a real but bounded exposure (identical in kind to a person pasting Map Intent text anywhere), not a server-side leak. Implementations should not describe this as more private than "the same as Map Intent text shared in this way."

## Alternatives Considered

1. **Leave ADR 0001's wording as an unconditional hash prohibition and treat this feature as a non-conforming deviation.**
   Rejected: the feature satisfies every actual concern §2/§4 were written to address (no server visibility, no persistent/bookmarkable state), and the "MUST NOT carry map state" language was written with query strings and paths — which persist across reloads/bookmarks/server logs — in mind, not a value that's read once and discarded before render.

2. **Edit ADR 0001's text in place.**
   Rejected in favor of a new, clarifying ADR, consistent with this repository's existing pattern (ADR 0003 coexists with ADR 0001 rather than rewriting it).

3. **Require server-side or cryptographic proof that a fragment was single-use (e.g., signed, time-limited tokens).**
   Rejected for baseline: adds real implementation complexity for a guarantee already provided more simply by "clear it before render, never read it again" — there is no server to enforce single-use against, and the threat model here is accidental persistence, not adversarial replay.

## Status

Proposed. `hfu/faceless-cartographer` D32 is offered as evidence that the one-shot fragment interpretation is a viable, already-deployed instance of the Faceless Cartographer baseline, not a hypothetical relaxation.
