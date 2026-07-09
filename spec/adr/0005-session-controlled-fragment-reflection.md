# ADR 0005: Session-Scoped URL-Fragment Reflection Controlled by Intent Policy

Status: Proposed
Date: 2026-07-10
Deciders: Staccato spec maintainers

## Context

ADR 0004 (fragment hand-off) enables one-click Map Intent delivery via `#intent=<base64url>`, read once and immediately cleared, preserving the faceless baseline of persistent URL cleanliness.

However, use-case analysis reveals a tension: **public-data intents** (e.g., published hazard zones, open-source basemaps) require users to self-service share a discovered map view via the address bar — they expect to copy and paste the URL as-is, without intermediate paste-into-textarea steps. **Sensitive-context intents** (e.g., restricted-access datasets, ad-hoc analysis), by contrast, require the URL to stay clean by default, to avoid at-rest retention in browser history, synced bookmarks, or clipboard logs.

ADR 0001 §3 already defines `sharing_policy.url_share` (a SHOULD-false policy field) as the mechanism by which an intent declares "this data is OK for deeplinking." The current implementation (faceless-cartographer v0.2 and prior) ignores this field entirely when deployed in a faceless configuration — treating all intents uniformly as "no persistent URL state allowed."

This creates a silent UX failure for S3 (public data): address-bar copy yields `/` (empty form) to the recipient. And a silent security failure for S4 (sensitive context) if URL reflection were always-on: the rendered session's URL would accumulate in browser history/sync despite user intent otherwise.

The solution is to make URL reflection **intent-driven by default, human-overridable per session**.

## Decision

We extend the faceless URL policy to support three strategies:

1. **No hash usage** (classic faceless: ADR 0001 as written, ADR 0004 option 1).
2. **One-shot fragment hand-off** (ADR 0004 option 2; `#intent=...` read once on load, cleared).
3. **Session-scoped live reflection** (new): a Cartographer MAY offer a UI toggle that lets a human enable/disable URL-fragment reflection on a per-session basis. When enabled:
   - Every map state change (`moveend`, layer visibility toggle, etc.) triggers a `history.replaceState` to update `location.hash` with the current Map Intent encoded as base64url.
   - The initial toggle state MUST default to the intent's `sharing_policy.url_share` value (false if unset).
   - The toggle state is NOT persisted (sessionStorage, cookies, etc.) — reloading returns to the intent's default.
   - A rendered session's clean URL behavior is preserved: the fragment is always advisory and transient, never becoming a bookmark or server-logged artifact (identical to ADR 0004's guarantee for one-shot hand-offs, except the user decides when to enable it).

Implementations using option 3 remain fully subject to ADR 0001's core principles:

- **§2 (URL policy)**: Rendered sessions keep the URL clean by default (via policy-driven default); persistence is a deliberate, session-scoped human choice, not a silent side effect.
- **§3 (Share policy)**: The intent text remains the primary share artifact; a fragment URL is a convenience layer the human consciously enables, not a competing mechanism.
- **§4 (Data minimization)**: Fragments are server-invisible; the only exposure is browser-local (history, clipboard history, sync).

This ADR does not change §1 (Endpoint model) or §5 (Spec linkage).

## Consequences

Positive:

- Addresses both S3 (public data wants deeplink-friendly URLs) and S4 (sensitive context wants URL-clean default) without silent failures in either case.
- Respects intent author's intent via `sharing_policy.url_share`, yet preserves human autonomy: a person using a public-data intent can still disable reflection if they wish, and vice versa.
- Reduces surprise: the toggle is explicit; no hidden URL persistence occurs without user action.
- Scales with maturity: an intent author can signal their data's sharing posture; implementations can choose to expose (or not) a reflection UI; humans make the final call.

Negative / trade-offs:

- Implementation complexity increases: Cartographer must wire up a UI toggle, manage session state, attach move/layer-change listeners, and encode/decode fragments on the fly.
- The toggle is session-scoped (not persisted), so a user who enables reflection must re-enable it after reload. This is intentional (faceless baseline) but may be perceived as friction by some users.
- Two separate fragment behaviors exist in the wild: ADR 0004's one-shot hand-off (always cleared on load) vs. option 3's live reflection (stays, updates on demand). A person encountering both must understand the difference: a URL with `#intent=` is not automatically safe to bookmark.

## Alternatives Considered

1. **Always-ON live reflection (no toggle).**
   Rejected: silent security failure for sensitive contexts; S4 would accumulate URL state in history without explicit consent, violating intent author's intent and ADR 0001 §3's notion of intentional sharing.

2. **Always-OFF live reflection (ignore `sharing_policy.url_share`).**
   Rejected: silent UX failure for public data; S3 users get no address-bar shortcut, losing the convenience that `sharing_policy.url_share` was designed to enable.

3. **Global Cartographer setting (live reflection ON/OFF for all intents).**
   Rejected: violates intent author autonomy; a Staff agent that publishes hazard zones should not have their sharing intent overridden by deployment configuration.

4. **Crypto/server-side single-use tokens for reflection.**
   Rejected: adds complexity without benefit over "human toggles it on" + "clear on load" guarantees; there is no server to enforce single-use.

## Status

Proposed. `hfu/faceless-cartographer` D34 is offered as evidence that session-scoped reflection is viable and addresses real use-case tension without reintroducing persistent URL state.

## Future Considerations

- **Reflection UI design**: Different Cartographer implementations may place the toggle in different UI locations or use different language/icons. The spec does not mandate UI details, only the behavior (default to policy, session-scoped, clear on reload).
- **Performance optimization**: Encoding/decoding base64url fragments on every move event may cause latency on large Map Intents. Implementations might defer updates, debounce, or batch them.
- **Persistence reconsideration**: If future use cases demand persistent reflection (e.g., a "bookmark this map state" feature), that would require a new decision to store state server-side or in client-persistent storage — a material change requiring explicit re-evaluation of ADR 0001 consequences.
