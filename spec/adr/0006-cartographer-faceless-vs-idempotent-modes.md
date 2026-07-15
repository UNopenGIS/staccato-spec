# ADR 0006: Cartographer Implementation Patterns: Faceless-by-Default vs. Idempotent Modes

Status: Proposed

Date: 2026-07-10

Deciders: Staccato spec maintainers, Cartographer implementers

## Context

ADR 0005 establishes three URL policy strategies available to Cartographer implementations:

1. No hash usage (classic faceless, ADR 0001 strict interpretation)
2. One-shot fragment hand-off (`#intent=...` read once on load, cleared, ADR 0004)
3. Session-scoped live reflection (user-toggled URL-fragment updates, ADR 0005 option 3)

In practice, implementing option 3 reveals a design pattern that generalizes across these strategies: Cartographer can be understood as operating in two distinct **modes**, each representing a different user interaction model and data-sharing posture.

The `hfu/faceless-cartographer` v0.2+ implementation of ADR 0005 demonstrates this pattern concretely, yielding UI/UX insights and implementation trade-offs worth formalizing as guidance for future Cartographer implementations.

## Decision

We introduce a conceptual framework for Cartographer implementation: **faceless mode** (default, safe, URL-clean) and **idempotent mode** (user opt-in, convenience, URL-reflective).

### Faceless Mode (Default)

- **URL policy**: Always remains at root path (`/` or equivalent), no state encoding in query string, path, or hash.
- **Sharing mechanism**: Primary artifact is the Map Intent text (YAML). Users paste it into the Cartographer's form field or receive a `#intent=...` fragment URL (one-shot, cleared on render).
- **Use case**: Sensitive contexts (restricted-access datasets, ad-hoc analysis, compliance-constrained workflows) where URL-clean session defaults are essential.
- **User experience**:
  - "Copy Map Intent" button provides the rendered Map Intent (with current view baked into `render_hints`).
  - Adrress bar never accumulates state; ページリロード returns to clean form.
  - Users who wish to share must explicitly copy the Map Intent and transmit it (via chat, email, ticket system, etc.).
- **Security/Privacy posture**: URL-clean baseline; no accidental at-rest retention in browser history, synced bookmarks, or shared log files.

### Idempotent Mode (User Opt-In, Session-Scoped)

- **URL policy**: Fragment (`#intent=...`) is live-updated as map state changes (pan, zoom, layer toggling).
- **Activation**: UI toggle (checkbox, button, etc.) allows user to switch from faceless to idempotent within a session.
- **Initial state**: Toggle defaults to the intent's `sharing_policy.url_share` value (false if unset), respecting intent author's data posture.
- **Sharing mechanism**: User copies the current URL directly from the browser address bar, which includes the live Map Intent fragment.
- **Use case**: Public or openly-shared data (published hazard zones, open-source basemaps, educational demonstrations) where address-bar copy-paste convenience is valued.
- **User experience**:
  - When toggle is ON, URL fragment updates in real time as user interacts with the map.
  - User sees live URL in address bar, can copy it to share current view with others.
  - Toggling OFF returns to faceless mode (URL reverts to clean state).
  - **Session-scoped only**: Page reload returns toggle to its default state (intent's `sharing_policy.url_share` value). Fragment is never persisted to localStorage, cookies, or server.
- **Security/Privacy posture**: Fragment is server-invisible and transient. User consciously enables URL state; it does not accumulate without explicit action. Upon reload, defaults back to intent author's sharing policy.

### Relationship to ADR 0001 & 0005

Both modes preserve ADR 0001's core principles:

- **URL policy (§2)**: Faceless mode is the guaranteed baseline; idempotent mode is a deliberate, session-scoped human choice, never a silent side effect.
- **Share policy (§3)**: The Map Intent text remains the primary, durable share artifact. URL fragments are a convenience layer.
- **Data minimization (§4)**: Fragments never reach the server; browser-local exposure only (history, clipboard, sync).

Idempotent mode is ADR 0005's option 3 in concrete form.

## Implementation Guidance

### UI/UX Patterns

1. **Toggle Label**: Name the control clearly to indicate what is being enabled. Examples:
   - "Reflect Map Intent in URL" (English)
   - "URLにMap Intentを反映" (Japanese, more direct than "URLに地図の状態を反映")
   - Avoid vague terms like "enable sharing" or "sync state"—be specific about *what* is in the URL.

2. **No Redundant Buttons**: If idempotent mode is enabled and URL is live-updated, a separate "Copy Shareable Link" button is redundant (users can copy from address bar). Prefer a single "Copy Map Intent" button for faceless mode.

3. **Layer Search/Filtering**: Implement across both modes. Real-time substring matching on layer labels helps users navigate intents with many layers, independent of sharing mode.

4. **Mobile Considerations** (375px viewport):
   - Toggle placement: below notices, above action buttons, or inline with layer list.
   - Sticky footer for action buttons ensures reach on narrow screens; idempotent mode's live URL updates do not conflict.

### State Management

1. **Session-Scoped Only**: Do not persist toggle state in localStorage, IndexedDB, or server-side session storage. Reload always returns to intent's default.
   - Rationale: Prevents accidental URL-state accumulation; honors intent author's original `sharing_policy.url_share` decision.

2. **Event Listeners**: Attach to map `moveend`, layer `change`, zoom/bearing/pitch updates. Encode current Map Intent as base64url fragment on every change.

3. **Performance**: For large Map Intents (many layers, complex `render_hints`), consider debouncing or batching fragment updates to avoid excessive re-encodes on rapid pan/zoom.

### Compatibility with ADR 0004

- One-shot `#intent=...` fragments (ADR 0004) can coexist with idempotent mode.
- On load: Cartographer reads and clears one-shot fragments (faceless baseline).
- After render: Toggle default is set to intent's `sharing_policy.url_share`; if user enables idempotent, live updates begin.
- Result: Users can receive a `#intent=...` link, see faceless mode render, then choose to enable idempotent mode to share the current view with others.

## Consequences

### Positive

- **Clear mental model**: Implementers and users understand two distinct sharing postures ("faceless by default, idempotent on demand").
- **Intent author autonomy**: `sharing_policy.url_share` is respected; implementations do not unilaterally impose URL state on all intents.
- **Diverse use cases**: Same Cartographer code serves both sensitive (faceless) and public (idempotent) workflows.
- **UI clarity**: Naming modes explicitly (`faceless`, `idempotent`) and labeling controls precisely ("Reflect Map Intent in URL") reduces confusion and improves onboarding.

### Negative / Trade-Offs

- **Learning curve**: Users must understand that toggling the control switches behaviors. Not intuitive without explanation or clear labeling.
- **Session-scoped friction**: Users who prefer persistent idempotent state must re-enable the toggle after reload. This is intentional (faceless baseline) but may feel inconvenient.
- **Fragmentation risk**: If implementations use wildly different UI for the mode toggle, users moving between Cartographers may not recognize the pattern.
- **Performance**: Live fragment encoding on every map interaction can introduce latency on slow devices or large Map Intents.

## Alternatives Considered

1. **Always-idempotent (fragment always live-updated).**
   - Rejected: Violates faceless baseline; intent authors who set `sharing_policy.url_share: false` would see silent URL accumulation.

2. **Persistent idempotent toggle (localStorage or server session).**
   - Rejected: Defeats faceless baseline. Users could enable idempotent, reload, find it still on, and accidentally share URL-clean-violating state.

3. **Deployment-wide mode selection (each Cartographer instance is either faceless-only or idempotent-only).**
   - Rejected: Inflexible. A deployment might need to serve both use cases; users or intents cannot opt in per interaction.

## Future Considerations

- **Persistent state (future ADR)**: If later use cases demand bookmarkable, persisted map state ("save this map as a bookmark"), that would require a distinct new decision and likely server-side storage, explicitly deviating from ADR 0001.
- **UI framework guidance**: As implementations grow (multiple Cartographers in different frameworks), a separate recommendation on accessible/mobile-friendly toggle patterns could be useful.
- **Fragment size limits**: Very large Map Intents might exceed URL length limits (~2000–2083 chars in browsers). Future guidance on truncation, compression, or server-side state management could be needed.

## References

- ADR 0001: Faceless Cartographer
- ADR 0004: One-shot fragment hand-off for Map Intent delivery
- ADR 0005: Session-scoped URL-fragment reflection controlled by intent policy
- Implementation reference: `hfu/faceless-cartographer` DECISIONS.md D34 (live reflection), D35 (button removal), D36 (faceless/idempotent mode naming)
