# ADR 0010: Narrative Sequencing as an Ordered Wrapper Around Independent Map Intents

Status: Proposed

Date: 2026-09-07

Deciders: Staccato spec maintainers, Cartographer implementers

## Context

[ADR 0009](0009-vocabulary-flexibility.md) clarified that Map Intent is the required baseline Staff→Cartographer vocabulary but not the exclusive one, citing `dwg7/ferspas57`'s guided-narrative feature as the motivating example. ADR 0009 deliberately left the narrative format's *internal shape* undefined, deferring any standardization until "more than one implementation wants this" — its own stated trigger for reconsideration.

That trigger has now been met, by real convergence rather than speculation:

- **`dwg7/ferspas57`** (a Cartographer for FAO's Hand-in-Hand Initiative and GAEZ data) has 373 real narratives in production under its current `NARRATIVE-FORMAT.md` v1 shape: `{narrative_version, title, steps: [{center, zoom, layers, caption}]}` — a lightweight, Cartographer-specific schema, not composed from Map Intent.
- **`dwg7/kataribe`** (a Staff recognizing which meaning-layer of a pre-verified event dossier a given asker needs) independently concluded, from its own real dossier content, that a selected layer "is just an ordinary narrative with one step" — and chose `dwg7/spiccato`'s single-shot Map Intent link over forking `ferspas57`'s mechanism, since its actual interaction shape ("one question → one recognized layer → one render") never needed a multi-step sequence.

`staccato-spec` proposed a redesign direction, relayed through `dwg7/staccato-ecosystem`: redefine the narrative wrapper as **an ordered sequence of independent, complete Map Intent documents, each with its own optional caption**, rather than an independently-shaped competing schema — leaving Map Intent's own schema completely untouched. This is a deliberately different direction from ADR 0009's own rejected Alternative 1 (growing an optional `narrative` field onto Map Intent itself), which caused a real problem: a document trying to be both a single state and a sequence at once, forcing a redundant single-state fallback for non-narrative consumers. The new direction reverses the composition: narrative wraps Map Intent, Map Intent never wraps or grows to accommodate narrative.

Three open points were raised for `ferspas57` and `kataribe` to resolve directly against their own real content, rather than settling them from design opinion alone. Both converged independently on all three:

1. **`catalog_context`/`provenance` duplicating verbatim across steps is accepted**, not hoisted to a wrapper level. `ferspas57` flagged a concrete, non-blocking migration cost: its 373 existing narratives are in the current lightweight per-step shape, and the LZString-compressed URL-length impact of giving every step a full Map Intent needs to be measured before migrating, especially for its longer (7–20 step) tours.
2. **`caption` is not redundant with Map Intent's existing `goal` field.** Both sides checked this against real production content, independently, and reached the same reasoning: `goal` describes the *requester's* want ("a concise statement of what map the user wants" — one speech-act direction), while `caption` is *author-to-viewer* narration addressed to whoever is looking at the render (the opposite direction). `ferspas57`'s evidence: its DR Congo maize-siting narrative's captions are third-person mystery/contrast prose referencing the *previous* step ("Why would the 'constrained' land score higher?"). `kataribe`'s evidence: a `map_caveat` field — a safety warning that a displayed hazard polygon doesn't cover the disaster's actual reach, so viewers don't misread it. Two independently-motivated real fields landing on the same structural argument is real convergence, not coincidence.
3. **Sequence identity (an ordering/`narrative_id`-equivalent) belongs at the wrapper level**, not as fields inside individual Map Intent steps.

## Decision

We recommend — **SHOULD**, not MUST, matching [ADR 0007](0007-style-references.md) §4's precedent for implementer guidance rather than core-architecture requirements — that a Staff→Cartographer narrative/sequencing vocabulary, where an implementation chooses to build one, take the following shape:

### 1. Map Intent's schema is unchanged

No field is added to `map-intent-vnext.md`. This is the point this ADR does not compromise on: composition is one-directional. A narrative wraps Map Intent; Map Intent never has to know whether it is being used standalone or as one step of a sequence. This is what ADR 0009's Alternative 1 got wrong — growing Map Intent itself to carry a `narrative.steps[]` field made every Map Intent answer "am I a step or a whole," which is exactly the ambiguity a one-directional wrapper avoids.

### 2. Recommended wrapper shape

```yaml
narrative_id: "..."       # wrapper-level identifier, not inside any step
title: "..."               # optional
steps:
  - map_intent: { ... }     # one complete, independent Map Intent document
    caption: "..."          # optional, wrapper-level, never part of Map Intent
  - map_intent: { ... }
    caption: "..."
```

- Each step's `map_intent` is a **complete, independent, valid Map Intent** on its own — not a diff or partial state relative to the previous step. A Cartographer that receives just one step in isolation still renders correctly.
- `caption` lives on the wrapper's step object, never inside `map_intent`. §2 above establishes why: `goal` and `caption` are different speech acts (requester-to-system vs. author-to-viewer) and don't collapse into one field.
- Sequence identity and ordering (`narrative_id`, array order) belong to the wrapper, never to individual steps' Map Intent content.

### 3. The degenerate one-step case

If there is no caption to carry and no more than one step, a bare Map Intent is used directly — no wrapper is needed at all. A wrapper only earns its existence when there is a caption to carry, or genuinely more than one step, since Map Intent intentionally has nowhere to put caption/narration content (§2's whole point). This also makes `kataribe`'s own finding literal: "a selected dossier layer is just an ordinary narrative with one step" is, under this shape, sometimes not even a narrative at all — just a captioned (or uncaptioned) Map Intent.

### 4. Duplication is accepted, not solved

`catalog_context` and `provenance` repeating verbatim across every step of one authored narrative is a deliberate, named trade-off, not an oversight. Hoisting shared fields to the wrapper level would reintroduce exactly the step-vs-whole asymmetry this design avoids by keeping every step a genuinely complete, independently-valid Map Intent. This matches ADR 0007 Alternative 2's own precedent of prioritizing reviewability and structural simplicity over generality.

## Consequences

### Positive

- Map Intent's schema stays exactly as simple as ADR 0009 already committed to keeping it — this ADR adds a recommended composition pattern, not a new required field anywhere in the core architecture.
- Any future Map Intent field addition (e.g. [ADR 0008](0008-basemap-selection.md)'s `basemap`) becomes available inside every narrative step automatically, with no need for the narrative wrapper schema to separately track Map Intent's own evolution.
- Two independent, real implementations converging on the same shape from their own production content is stronger evidence than either implementation's opinion alone — the kind of cross-implementation validation ADR 0009 named as its own reconsideration trigger.
- Directly simplifies `kataribe`'s own design: single-layer selection no longer needs any narrative-specific machinery at all, just a Map Intent, optionally captioned.

### Negative / Trade-offs

- Per-step duplication of `catalog_context`/`provenance` makes narrative documents more verbose than the current, more compact per-Cartographer shapes (e.g. `ferspas57`'s current `{center, zoom, layers, caption}` steps). Accepted deliberately (§4).
- This is still a **recommendation** for implementations that choose to build a narrative-style vocabulary, not a normative requirement — narrative-style formats remain outside this spec's compliance floor per ADR 0009, and are still not guaranteed portable between different Cartographers.
- Existing production narratives built to a pre-recommendation shape (`ferspas57`'s 373 entries) face a real migration cost, not addressed by this ADR — see Operational Implications.

### Operational Implications

- `ferspas57` flagged, and this ADR does not resolve, a concrete pre-migration task: measuring the LZString-compressed URL-length impact of full per-step Map Intents against its longer (7–20 step) tour narratives, before migrating its 373 existing entries to this shape. This is an implementation-phase concern for whoever migrates, not a blocker to adopting this recommendation going forward.
- Implementers adopting this shape for new narrative content do not face this migration cost — it only applies to converting pre-existing, differently-shaped narrative libraries.

## Alternatives Considered

1. **Grow Map Intent's own schema with a `narrative` field** (ADR 0009's own Alternative 1, already rejected there).
   Rejected again here for the same reason: forces every narrative-bearing Map Intent to also carry a redundant single-state fallback, entangling two structurally different kinds of artifact.

2. **Hoist `catalog_context`/`provenance` to the wrapper level, overridable per step.**
   Considered during convergence discussion. Rejected: reintroduces a step-vs-whole asymmetry (some fields live on the wrapper, some on the step, and a step is no longer a complete document on its own) in exchange for reduced verbosity — judged not worth it, matching ADR 0007 Alternative 2's precedent of accepting duplication for structural simplicity.

3. **Fold `caption` into Map Intent's existing `goal` field, avoiding a new concept entirely.**
   Rejected — both implementations checked this against real content and found `goal` and `caption` are different speech acts (requester-to-system vs. author-to-viewer) that don't collapse into one field without losing real, load-bearing content (a mystery-story caption, a safety caveat).

4. **Standardize this now as a new normative sibling document type in `staccato-spec` itself (e.g. "Narrative Intent"), rather than a SHOULD-level recommendation.**
   Rejected for now: this ADR follows ADR 0007 §4's precedent of a recommendation for implementers, not a new required document type in the core architecture. Two implementations converging on a shape is good evidence for a recommendation; whether it should ever become a MUST-level, spec-defined document type is a larger step this ADR does not take.

## Status

Proposed. Evidentiary basis: `dwg7/ferspas57`'s 373 production narratives and `dwg7/kataribe`'s dossier-driven design, converging independently on the three open points above from their own real content — the same evidentiary pattern ADR 0003/0007/0008/0009 each used.

## References

- ADR 0009: Map Intent Is the Required Baseline Vocabulary, Not the Only One (the parent clarification this ADR refines)
- ADR 0007 §4: Recommendation for Library implementers (precedent for SHOULD-level implementer guidance, and for accepting duplication over generality per its Alternative 2)
- ADR 0008: Explicit `basemap` Selection on Map Intent (an example of a Map Intent field addition that this ADR's design lets every narrative step inherit automatically)
- `map-intent-vnext.md` §4 (Schema)
- `dwg7/ferspas57`: `NARRATIVE-FORMAT.md` (current v1 shape), `DECISIONS.md` D55 (convergence record)
- `dwg7/kataribe`: `DECISIONS.md` D2 (dossier vs. narrative.json split, "just an ordinary narrative with one step"), D3 (Cartographer choice: `spiccato` first), D4 (convergence record)
