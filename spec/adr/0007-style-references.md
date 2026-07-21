# ADR 0007: Style References (`required_styles`/`optional_styles`) as an Alternative to Layer-Level Composition

Status: Proposed

Date: 2026-07-21

Deciders: Staccato spec maintainers, Cartographer implementers

## Context

`map-intent-vnext.md` §4 only lets Staff express a user's request as a set of individual `source_id`s, via `required_layers`/`optional_layers`. In practice, some requests are naturally a request for a *complete, pre-designed thematic map* rather than a pile of layers — for example, "I want to see Hokkaido's volcanic land condition map" (火山土地条件図を見たい). When Staff tries to decompose such a request into `source_id`s, a mismatch appears: the user's intent is the finished map product itself, but the only vocabulary Staff has is individual data sources. This gap was identified and worked through concretely in [`hfu/faceless-cartographer`#6](https://github.com/hfu/faceless-cartographer/issues/6).

Separately, a `martin`-type catalog (`catalog-integration.md` §3.1) is not only a tile-service publisher — [Martin](https://martin.rs) (the reference tile server used by this ecosystem's Library implementations) can also publish complete [MapLibre Style Spec](https://maplibre.org/maplibre-style-spec/) documents, exposed via `GET {base}/style/{style_id}`, alongside its existing `GET {base}/{source_id}` TileJSON endpoint. `catalog-integration.md` currently has no concept of this at all — `map-intent-vnext.md` and `catalog-integration.md` are both silent on "style" as a Map Intent concept.

`hfu/faceless-cartographer` implemented and deployed `required_styles`/`optional_styles` as an intentional deviation from current spec text (its own `DECISIONS.md` D39, following the same append-then-propose pattern as ADR 0003's D18 and ADR 0004–0006's D32/D34–D36). Concretely:

- Two thematic-only styles (extracted from [`hfu/kitavolca`](https://github.com/hfu/kitavolca)'s combined basemap+thematic style, keeping only the `vlcm-*`/`vbm-*` layers and their `vlcm`/`vbm` sources) are published live at `https://stars.optgeo.org/style/vlcm` and `/style/vbm`.
- A Map Intent can reference either with `required_styles: [{style_id: "vlcm"}]`, and `faceless-cartographer` resolves and renders it end-to-end, verified in a real browser against the live server (no mocking).
- Notably, the published styles are deliberately *thematic-only* — they omit the background sources (`bvmap`, `mapterhorn`, `seamlessphoto`) that `hfu/kitavolca`'s own combined style bundles for its own standalone use. Cartographer already draws its own background unconditionally (a project-local convention, `hfu/faceless-cartographer` D24); including a second, redundant background inside a resolved style would double-render it.

## Decision

We extend the Map Intent schema and the `martin` catalog type to support style-level references, as an alternative (not a replacement) to layer-level composition.

### 1. Map Intent schema (extends `map-intent-vnext.md` §4)

Add `required_styles`/`optional_styles`, structurally parallel to `required_layers`/`optional_layers`:

```yaml
required_styles:
  - style_id: "vlcm"
    label: "Volcanic Land Condition Map"

optional_styles:
  - style_id: "vbm"
    label: "Volcanic Basic Map"
```

- `style_id` is a new canonical term (extends `architecture-principles.md` §3): the canonical style identifier emitted into `required_styles`/`optional_styles`, resolved against a `martin`-type catalog's `GET {base}/style/{style_id}` endpoint.
- `label` is optional, same semantics as `required_layers[*].label`.

### 2. Validation (revises `map-intent-vnext.md` §6, rules 1–2)

- At least one of `required_layers` or `required_styles` MUST be a non-empty array. Neither is unconditionally mandatory on its own anymore.
- `required_styles`/`optional_styles` entries MUST each have a string `style_id`, mirroring the existing `source_id` shape rule.

### 3. Catalog integration (extends `catalog-integration.md` §3.1)

- A `martin`-type catalog MAY expose a `styles` object alongside `tiles` (Martin's own catalog root shape: `{"tiles": {...}, "styles": {...}}`). Style resolution MUST only be attempted against `martin`-type catalogs.
- `layers_txt` catalogs have no equivalent endpoint concept and MUST NOT be attempted for style resolution — this is a structural limitation of that catalog type (a static mirror with no server-side style-publishing capability), not a policy choice.
- A resolved style's `sources`/`layers` are composed into the Cartographer's rendered map in the same position as layer-level thematic content, not as a replacement for whatever fixed background (if any) a given Cartographer implementation always draws.

### 4. Recommendation for Library implementers publishing styles for Map Intent consumption

A style published for `required_styles`/`optional_styles` consumption SHOULD be **thematic-only**: it SHOULD NOT bundle its own basemap/background sources and layers. A Library may separately publish a *different*, complete standalone style (background included) for its own direct use (e.g. a project's own demo page) — the two are different artifacts with different consumers, and conflating them causes duplicate rendering when the complete one is composed into a Cartographer that already draws its own background.

This is a SHOULD, not a MUST, because a Cartographer implementation that draws no background of its own would have no such conflict — but any Cartographer following the common pattern of an always-on background (as `hfu/faceless-cartographer` D24 does) will double-render if given a style that bundles one.

## Consequences

### Positive

- Requests for a complete, pre-designed thematic product ("give me the volcanic land condition map") can be expressed directly, without Staff reconstructing it from individual `source_id`s.
- Library implementers can publish curated cartographic products — including layer ordering, symbology, and color design already reviewed and approved — as a first-class, independently referenceable resource, not just raw data sources.
- Matches (and is enabled by) an existing, real Martin server capability (`GET /style/{style_id}`), rather than inventing a project-specific protocol (`architecture-principles.md` §2.3, convention over custom protocol).

### Negative / Trade-offs

- A second resolution path (alongside `source_id`) that Staff, Cartographer, and Library implementers must all support; increases the surface area of the Map Intent schema.
- The thematic-only publishing convention (Decision §4) is an extra authoring/maintenance burden for Library implementers who already maintain a complete standalone style for their own use — two artifacts to keep in sync instead of one.
- Martin's own operational note applies: style *additions* require a server restart to take effect (removals/changes are immediate), an operational quirk Library operators need to plan around.

### Operational implications

- Testing must cover an unresolved `style_id` (reported as missing, not a hard error — same posture as an unresolved `source_id`) and a resolved-but-empty-`layers` style (treated as unrenderable, not a crash).
- `catalog-integration.md` §7's future `catalog_bindings` extension, if adopted, should be able to track style resolution provenance the same way it would track `source_id` provenance.

## Alternatives Considered

1. **Always decompose into `required_layers`, even for "give me a complete map" requests.**
   Rejected — this is the status quo that motivated this ADR; it structurally cannot express the user's actual intent for a pre-designed product, and pushes composition/design work onto Staff that a Library has already done and curated.

2. **A single unified array (e.g. `required_resources`) with a `kind: "layer" | "style"` discriminator, instead of two parallel array pairs.**
   Considered, and would be more extensible if more resource kinds are added later. Not adopted here: the reference implementation kept `required_styles`/`optional_styles` structurally parallel to the already-established `required_layers`/`optional_layers` precedent, prioritizing reviewability and minimal schema churn over generality. A future ADR could revisit unification if a third resource kind emerges.

3. **Embed style content directly in Map Intent (inline style JSON) instead of a `style_id` reference.**
   Rejected — breaks the reproducibility-via-catalog-provenance model that `source_id` already relies on, and bloats Map Intent (styles can carry many layers) against `architecture-principles.md` §2.5 (least disclosure / minimal payload).

## Future Considerations

- Formal guidance for Library implementers on authoring "thematic-only" styles (Decision §4) could grow into its own guidance document if more Library implementations start publishing styles.
- Whether `stac`-type catalogs could someday expose an equivalent concept is left open; no such mapping is proposed here.

## References

- ADR 0001: Faceless Cartographer
- ADR 0002: Enforce Staff Startup Catalog Contract and Ban Hidden Fallback
- `map-intent-vnext.md` §4 (Schema), §6 (Validation Rules)
- `catalog-integration.md` §3.1 (`martin` catalog type)
- `architecture-principles.md` §3 (Canonical Terminology), §2.3 (Convention over custom protocol), §2.5 (Least disclosure)
- Origin: [`hfu/faceless-cartographer`#6](https://github.com/hfu/faceless-cartographer/issues/6)
- Implementation reference: `hfu/faceless-cartographer` `DECISIONS.md` D39; live deployment at `https://stars.optgeo.org/style/vlcm` and `/style/vbm`
- Style source material: [`hfu/kitavolca`](https://github.com/hfu/kitavolca)
