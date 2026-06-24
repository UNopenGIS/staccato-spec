# Use Cases (Concept Exploration)

Created: 2026-06-24

1. Purpose and scope
- This document extracts and organizes primary usage scenarios from the CONOPS document. It avoids implementation judgments and focuses on observable behavior and failure modes.
- Note: concrete examples (agent names, PMTiles identifiers) are drawn from CONOPS and reproduction depends on the deployed catalog configuration and versions.

2. Actor definitions
- User: personnel or other users who want situational awareness via maps.
- Staff: enterprise-side actor (agent or human) that generates Map Intent from natural-language queries.
- Cartographer: Internet-side service that renders maps from Map Intent.
- Library: the set of data catalogs and geospatial resources (martin, layers.txt, STAC, PMTiles, etc.).

3. Common assumptions
- Staff is expected to reference only the catalogs configured at startup.
- The primary share artifact is Map Intent; URLs are not the principal sharing mechanism.
- Cartographer renders maps from posted Map Intent.

4. Use case index

- Case 1: Map generation for situational awareness (standard)
- Case 2: Missing layer due to catalog incompleteness
- Case 3: Sharing via Map Intent
- Case 4: Cartographer minimizing sensitive context

5. Detailed use cases

Case 1: Map generation for situational awareness (standard)
- Primary actor: User (example: disaster officer at Hokkaido Surveying Division)
- Trigger: User asks, e.g., "Show evacuation zones around Mount Usu"
- Preconditions: Staff (e.g., Genai or Microsoft Copilot) is running and required catalogs are configured at startup
- Basic flow:
  1. User submits a question to Staff
  2. Staff parses the question and generates a Map Intent (YAML), e.g., `required_layers: - source_id: jma.funka_keikai`
  3. User copies the Map Intent and posts it to Cartographer's `POST /`
  4. Cartographer fetches referenced PMTiles from FOIL4G Hokkaido (e.g. `hpg.hinan_kuiki.pmtiles`, `gsi.kazan_kihonzu.pmtiles`) and renders a MapLibre GL JS map
  5. User reviews the map and accompanying explanation to inform decisions
- Success criterion: the map displays expected information and the explanation identifies data sources
- Alternative flows / notes: when layers exist in multiple catalogs, resolution follows `catalog_context.resolution_policy.precedence`

Case 2: Missing layer due to catalog incompleteness
- Primary actors: Staff / User
- Trigger: Staff generates a Map Intent that references layers not present in configured catalogs
- Preconditions: Staff's configured catalog set is limited
- Basic flow:
  1. Staff generates Map Intent
  2. User posts Map Intent to Cartographer
  3. Cartographer consults FOIL4G PMTiles but cannot find requested layers
  4. Cartographer returns an error response that lists missing layers (e.g., `missing_layers: ["landuse.current"]`) and includes a `provenance_snapshot` of attempted resolution
- Failure scenarios:
  - Cartographer reports missing layers
  - Operators update catalog configuration and retry

Case 3: Sharing via Map Intent
- Primary actor: User
- Trigger: User wants to share the generated map with others
- Preconditions: Map Intent has been produced
- Basic flow:
  1. User clicks `Copy Map Intent` in Cartographer UI
  2. Recipient posts the Map Intent to their Cartographer instance via `POST /` to reproduce the map
- Success criterion: recipient reproduces a map reflecting the same intent
- Failure / caveat: differences in recipient catalog configuration may lead to different renderings; `catalog_context` and `provenance` within the intent are key to reproducibility

Case 4: Cartographer minimizing sensitive context
- Primary actor: Cartographer
- Trigger: Cartographer receives Map Intent
- Preconditions: operational guidance discourages including unnecessary sensitive data in Map Intent
- Basic flow:
  1. Cartographer uses only fields necessary for rendering and avoids persisting full intent payloads
  2. UI encourages copying the Map Intent but keeps the URL clean
- Success criterion: rendering occurs without embedding sensitive context in the URL or persistent logs
- Notes: the CONOPS suggests browser-side policies such as `Referrer-Policy: no-referrer`

6. Failure and exception scenarios (summary)
- Missing layers due to absent catalog entries
- Network constraints preventing access to external Library resources
- Reproducibility diverging due to differing recipient catalog sets

7. Observations about sharing and accountability
- Treating Map Intent as the share artifact centers human accountability in the handoff step
- This design reduces accidental leakage via URLs but lowers ad-hoc collaboration convenience, so operational rules and user training are important

8. Generalizable use-case patterns
- Intent generation (User → Staff)
- Human-mediated handoff (Staff output copied by User and posted to Cartographer)
- Internet-side rendering (Cartographer renders Map Intent)
- Feedback loop for catalog resolution failures

9. Open questions
- How detailed should Cartographer error messages be to be actionable without leaking sensitive context?
- Map Intent versioning and receiver-side fallback strategies
- How to signal differences when a recipient's catalog yields a different rendering?