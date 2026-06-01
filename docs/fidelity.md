# Fidelity audit: where information is (and isn't) preserved

Status **draft v0.1**. Grounded in the real PoC (`examples/fma-hs-k-30-3.ttl`,
tenant `fondazionearp`, 236 triples, A+ emission profile). Companion to
[`mapping.md`](mapping.md).

This answers a concrete question: **where does the Anton → RDF export actually lose
information?** It replaces the blanket RiC-AG caveat (*"mapping RiC to CIDOC CRM loses
specificity"*) with a field-by-field check. That caveat is about the direction RiC→CRM;
our situation is different — the **source is Anton**, and we emit a CRM-primary graph plus
a selective RiC-O layer (profile **A+**, see below).

## Method

1. **Field-coverage matrix** — every Anton field/relation × {captured in CRM? captured in
   the RiC-O layer? verdict}.
2. **Single-model read test** — read only the `crm:` triples, then only the `rico:`
   triples of the PoC; record what each view is missing.
3. **Round-trip** — could the Anton record be reconstructed from each view?

Legend: ✓ full · ◑ degraded/partial · ✗ not emitted · — n/a

## A+ emission profile (what the RiC-O layer carries)

CRM is the lead model and carries the **full** detail. RiC-O is emitted **selectively**:

- **Class typing:** `rico:RecordSet` / `rico:Record` / `rico:Instantiation` /
  `rico:Person` / `rico:CorporateBody` / `rico:Family` / `rico:Place`.
- **Relational skeleton:** `rico:isOrWasIncludedIn` (hierarchy), `rico:hasCreator`
  (and `rico:hasProvenance`), `rico:hasOrHadSubject` (aboutness), `rico:hasInstantiation`.
- **Shared (model-neutral):** `owl:sameAs` authority links, `skos:Concept` vocabularies.

Literal attributes (titles, identifiers, dates, note text), events and their actors/places,
and most descriptive properties stay **CRM-side only**. This is a deliberate A+ choice, not
an oversight.

## Field-coverage matrix

### AntonObject (unit of description)
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| `level_of_description` | `E55` + `skos:exactMatch`→RiC RecordSetTypes | class `rico:RecordSet`/`Record` | ✓ (recordgroup/class: ◑, no RiC term) |
| `parent_id` (hierarchy) | `P46`/`P46i` | `rico:isOrWasIncludedIn` | ✓ both views |
| `identifier`, `identifier_old` | `E42` typed | ✗ | CRM-only |
| `title` (multilingual) | `E35`/`P190`@lang | ✗ | CRM-only |
| `object_count`, `extent_cm` | `P57`, `E54` | ✗ | CRM-only |
| `location_id` | `P55` → `E53` | ✗ | CRM-only |
| `period_of_protection` | `E30_Right` (coarse) | ✗ | ◑ (source-side poverty) |
| `private` | export filter | export filter | ✓ (excluded) |
| `full_text` | not exported | ✗ | intentional drop |

### AntonEvent (the core)
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| event + `event_type` | `E5`/subclass + `P2 type` | — (no `rico:Activity` node) | CRM ✓ / RiC ◑ |
| creator (`P14` actor) | `P14_carried_out_by` | `rico:hasCreator` | ✓ both views |
| recipient / custody | `E10`/`P29` | ✗ | **RiC view loses recipients** |
| place (`P7`) | `P7_took_place_at` | ✗ | CRM-only |
| dates (`P82`/`P81`) | time-span + note | ✗ | CRM-only; circa → ◑ |
| role (`P14.1`) | `in_the_role_of` → `E55` | ✗ | CRM-only |

### Actor
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| type (person/family/corporate) | `E21`/`E74` + `P2` | `rico:Person`/`Family`/`CorporateBody` | ✓ both |
| `name` + variants | `E41` typed | ✗ | CRM-only |
| birth/death, formation/dissolution | `E67`/`E69`/`E66`/`E68` | ✗ | CRM-only |
| `resources` (GND/VIAF/Wikidata) | `owl:sameAs` | `owl:sameAs` (shared) | ✓ |
| biographical notes | `E33`/`P3` | ✗ | CRM-only |

### Place
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| class | `E53` | `rico:Place` | ✓ both |
| `name` | `E48` | ✗ | CRM-only |
| `place_type` (GeoNames class) | `E55` + `skos:exactMatch` `gn:` | ✗ | CRM-only |
| geo point | `P168` WKT (lat/lon swap) | ✗ (`geo:asWKT` not emitted) | CRM-only |
| country | `P89_falls_within` | ✗ | CRM-only |
| `resources` (GeoNames) | `owl:sameAs` | `owl:sameAs` (shared) | ✓ |

### Keyword / descriptors (aboutness)
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| keyword entity | `E55`/`skos:Concept` | `skos:Concept` (shared) | ✓ |
| subject (`objects_keywords`) | `P129_is_about` | `rico:hasOrHadSubject` | ✓ both |
| place/actor descriptor | `P129_is_about` | `rico:hasOrHadSubject` | ✓ (aboutness ≠ process preserved) |

### Note
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| typed multilingual note | `E33` + `P2 type` + `P190`@lang | ✗ (`rico:note` fallback only) | ◑ **see loss #2** |
| private note types | export filter | export filter | ✓ (excluded) |

### Media
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| media entity | `crmdig:D1` | `rico:Instantiation` | ✓ both |
| link to object | `P138i`/`P128` | `rico:hasInstantiation` | ✓ both |
| `mime_type`/`pronom_id` | `P2 type` | ✗ | CRM-only |
| `md5sum`, `size` | `E42`, `E54` | ✗ | CRM-only |

### Relation (generic), Term/Taxonomy
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| known relation type | specific CRM property | ✗ | CRM-only (sparse data) |
| unknown relation type | `E13` reified / `P67` | ✗ | ◑ semantic flattened (sparse) |
| `terms`/`taxonomies` | `E55` + `skos` | `skos` (shared) | ✓ |

## Single-model read test (PoC)

- **CRM-only view → full fidelity.** Hierarchy, identifiers, titles, both events
  (creation + the two receptions), actors with birth/death + authority links, place with
  geo, typed notes, media with checksum/dimension. Nothing essential is lost.
- **RiC-O-only view → relational skeleton.** You get: `RecordSet`/`Record`/`Instantiation`/
  `Person`/`Place` classes; the inclusion hierarchy; `hasCreator` (the sender);
  `hasOrHadSubject`; `hasInstantiation`. You **do not** get: any title/identifier/date/note
  text, the **recipients**, the place geometry, or the actors' birth/death. → a navigable
  graph, **not** a usable description.

**Round-trip:** the Anton record is largely reconstructable from the CRM view (minus
`full_text` and some `object_type` nuance); it is **not** reconstructable from the RiC-O
view alone.

## The real losses, ranked

1. **The RiC-O view is a skeleton, by design (A+).** A pure-RiC-O consumer gets structure
   but not content/events/recipients. This is the single biggest "loss" — and it is the
   *opposite* of the RiC-AG caveat: here CRM is the rich view, RiC-O the lean one. Closing
   it = upgrade path B.
2. **Note-type ISAD/RiC semantics are not machine-readable.** The 38 note types are mapped
   to ISAD(G) in [`vocabularies.md`](vocabularies.md) §6, but the export emits only the
   *local* type term (`<term/note_type/scopecontent>`), with no `skos:exactMatch` and no
   specific RiC property. **Cheap fix, recommended next:** add `skos:exactMatch` on the
   note-type `E55` terms and emit `rico:scopeAndContent`/`rico:history`/… where RiC-O has a
   matching property; fall back to `rico:note` otherwise.
3. **Work/carrier split is partial.** Switching the unit class to `E73` removed the
   `E22`+`rico:Record` contradiction, but there is still no separate physical-carrier node
   (RiC's strict work/instantiation split). Acceptable for now; full split = upgrade path C.
4. **Intermediate levels without a RiC term.** `recordgroup`, `class` → generic
   `rico:RecordSet` only (no RecordSetType). Documented, low impact.
5. **Date "circa" softened.** The boolean `*_ca` becomes `P81`/`P82` inner/outer bounds +
   a free-text note; the explicit "this is an estimate" flag is approximated, not typed.
6. **Generic `relations` flattened.** Unknown relation types lose their specific semantic.
   Near-zero impact (sparse, mostly CMS-page links, out of scope).

> Not a mapping loss: protection periods and `relations` are *thin in the source* — CRM/RiC
> could express more than Anton records. That is source-side poverty, not export loss.

## Upgrade paths (deferred)

- **B — fuller RiC-O parallelism:** add `rico:` datatype properties (title, identifier,
  dates) and event/agent relations so the RiC-O view becomes self-sufficient. Cost: ~2×
  triples, redundancy. Do this if a consumer needs to read RiC-O standalone.
- **C — named graphs + clean work/carrier split:** one CRM graph, one RiC-O graph, the
  intellectual `rico:Record` and the physical/digital carrier as **distinct** nodes linked
  by `rico:hasInstantiation`. Semantically cleanest, heaviest to build, needs TriG/quads.

## Concrete next actions

- [ ] Loss #2: emit `skos:exactMatch` + specific `rico:` note properties (cheap, high value).
- [ ] Decide if/when to take upgrade path **B** (RiC-O standalone) — currently not needed.
- [ ] Re-run this audit once the exporter is implemented (it must reproduce the A+ profile).
