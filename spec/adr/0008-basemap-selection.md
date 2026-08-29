# ADR 0008: Explicit `basemap` Selection on Map Intent

Status: Proposed

Date: 2026-08-28

Deciders: Staccato spec maintainers, Cartographer implementers

## Context

`dwg7/spiccato` (the reference Cartographer implementation this ADR is motivated by) currently has exactly one hardcoded background style: a vendored `bvmap` style optimized for Japan. Outside Japan, this background has no usable tiles — any request for a map of a non-Japanese area renders against a background that simply doesn't cover it.

A fix was first discussed in geometric terms: have Cartographer inspect `area.bbox` at render time and switch basemaps depending on whether the bbox falls inside Japan. That approach was deliberately rejected. The Staff role that builds the Map Intent already knows, from the user's actual natural-language query, whether the request is domestic or international — that's a strictly more reliable signal than reverse-engineering the same fact from a bbox after the fact (a bbox can be ambiguous near borders, or simply wrong if Staff mis-resolved the area). More importantly, a bbox-sniffing Cartographer would violate this architecture's existing division of responsibility: Staff resolves ambiguity and semantics into explicit Map Intent fields, and Cartographer renders mechanically from what it's given, without inventing its own judgment calls. ADR 0002 already establishes this precedent for catalog scope (Staff MUST NOT infer or auto-discover catalogs; scope is fixed and explicit at startup) — the same reasoning applies here. `area.bbox` itself is already an example of Staff's resolved judgment, not something Cartographer derives; basemap selection should follow the same pattern rather than becoming the one field Cartographer is left to infer on its own.

Separately, `dwg7/spiccato` has its own `#q=` URL shorthand (not yet documented anywhere in this spec) that mirrors Map Intent fields as compact URL parameters, including `rstyle=`/`ostyle=` for `required_styles`/`optional_styles`. A `basemap` field on Map Intent would naturally want an equivalent `#q=` parameter. That URL shorthand is worth its own separate ADR and is out of scope here; it's noted only as context that motivates keeping `basemap`'s wire format consistent with `required_styles`/`optional_styles`, so a future `#q=` ADR can extend the same pattern without rework.

## Decision

We extend the Map Intent schema (extends `map-intent-vnext.md` §4) with a single optional field:

```yaml
basemap:
  style_id: "bvmap-intl"
  label: "International Basemap"
```

- `basemap` is optional (`basemap?: StyleRef`).
- It reuses the exact `StyleRef` shape ADR 0007 defines for `required_styles`/`optional_styles` — `style_id` plus optional `label`. No new type is introduced, and no new resolution mechanism is introduced: `basemap.style_id` is resolved against `catalog_context.active_catalogs` the same way `required_styles`/`optional_styles` entries are (ADR 0007 §3 — resolution MUST only be attempted against `martin`-type catalogs, via `GET {base}/style/{style_id}`).
- `basemap` is semantically distinct from `required_styles`/`optional_styles` in two ways:
  1. **Cardinality**: exactly one active basemap at a time, not a required/optional plural set.
  2. **Compositing role**: a resolved `basemap` style *replaces* Cartographer's background, rather than being composed alongside thematic content the way a resolved `required_styles`/`optional_styles` entry is.
- **Default behavior**: if `basemap` is absent, Cartographer falls back to whatever its own default background is (implementation-defined — e.g. `dwg7/spiccato`'s vendored `bvmap`). This is unchanged, existing behavior.
- **When present**, Cartographer renders the resolved `basemap` style as the background instead of its own default.

This is purely additive: every existing Map Intent, and every existing `#q=` link, has no `basemap` field and is silently unaffected — Cartographer's current default-background behavior for those is preserved exactly.

## Consequences

### Positive

- Fixes the international-usability gap: a Staff that recognizes a request as non-Japanese can supply an appropriate `basemap`, instead of the user being stuck with a Japan-only background regardless of the area requested.
- Fully backward-compatible and additive — no existing Map Intent, consumer, or `#q=` link needs to change.
- Reuses `StyleRef` and its existing `martin`-catalog resolution path verbatim (ADR 0007) rather than introducing a new type or a new resolution mechanism, keeping the schema's surface area small.
- Keeps the architecture's division of responsibility intact: Cartographer still does no geometric inference or judgment-call logic of its own (consistent with ADR 0002's precedent).

### Negative / Trade-offs

- One more field Staff, Cartographer, and Library implementers must all be aware of and support, alongside `required_styles`/`optional_styles`.
- Correctness now depends on Staff reliably classifying a request as domestic vs. international (or otherwise choosing the right basemap) from the natural-language query. A misclassification is a Staff-side error rather than a Cartographer-side one — this is a deliberate shift of responsibility, not an incidental gap, but it does mean Staff prompts/logic need to actually make this determination.
- A Library must publish a basemap-appropriate style (background sources included) for `basemap` to be usable at all; this is the *opposite* recommendation from ADR 0007 §4, which asks `required_styles`/`optional_styles` publishers to keep styles thematic-only. Style authors need to understand which of the two roles a given published style is for.

### Operational implications

- Testing must cover an unresolved `basemap.style_id` (reported as missing, not a hard error — same posture ADR 0007 established for unresolved `style_id` in `required_styles`/`optional_styles`) and the absent-`basemap` default-background path.
- Because `basemap` styles are expected to include background sources/layers (unlike the thematic-only convention for `required_styles`/`optional_styles`), Cartographer implementations must ensure a resolved `basemap` fully replaces rather than layers on top of their own default background, to avoid double-rendering.

## Alternatives Considered

1. **Cartographer infers the basemap from `area.bbox` (geometric/heuristic check against Japan's extent).**
   Rejected — this is the approach that motivated this ADR by being rejected first. It requires Cartographer to perform interpretive/geometric judgment it structurally shouldn't own (per ADR 0002's precedent), is less reliable than Staff's direct knowledge of the user's query, and produces ambiguous results near Japan's borders or when `area.bbox` itself is imprecise.

2. **Leave Cartographer's basemap as a single hardcoded default; do nothing.**
   Rejected — does not address the international-usability gap; any request for a non-Japanese area continues to render against a background with no coverage there.

## Future Considerations

- `dwg7/spiccato`'s `#q=` URL shorthand is not yet documented in this spec at all. A future ADR should formalize it, including a `basemap=<style_id>[|label]` parameter alongside the existing `rstyle=`/`ostyle=` parameters, using the same wire format `basemap` establishes here. Not decided in this ADR.
- Whether Staff should have a standard vocabulary or catalog convention for "the domestic basemap" vs. "the international basemap" (e.g. well-known `style_id` naming) is left open for implementers; this ADR only adds the field, not a naming convention.

## References

- ADR 0002: Enforce Staff Startup Catalog Contract and Ban Hidden Fallback (precedent: Staff resolves ambiguity/scope explicitly; Cartographer/consumers MUST NOT infer or auto-discover)
- ADR 0007: Style References (`required_styles`/`optional_styles`) as an Alternative to Layer-Level Composition (`StyleRef` shape and `martin`-catalog resolution reused verbatim here)
- `map-intent-vnext.md` §4 (Schema)
- `catalog-integration.md` §3.1 (`martin` catalog type)
- Origin: `dwg7/spiccato` — single hardcoded Japan-only `bvmap` background motivating this proposal
