# Fidelity audit: where information is (and isn't) preserved

Status **draft v0.1**. Grounded in the real PoC (`examples/fma-hs-k-30-3.ttl`,
tenant `fondazionearp`, 239 triples, A+ emission profile). Companion to
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

Literal attributes (titles, identifiers, dates), events and their actors/places, and most
descriptive properties stay **CRM-side only**. This is a deliberate A+ choice, not an
oversight. **Exception:** note text is *also* emitted as the matching RiC-O note property
(loss #2 fix below).

## Field-coverage matrix

### AntonObject (unit of description)
| Anton | CRM | RiC-O (A+) | Verdict |
|---|---|---|---|
| `level_of_description` | `E55` + `skos:exactMatch`→RiC RecordSetTypes | class `rico:RecordSet`/`Record` | ✓ (recordgroup/class: ◑, no RiC term) |
| `parent_id` (hierarchy) | `P46`/`P46i` | `rico:isOrWasIncludedIn` | ✓ both views |
| `identifier`, `identifier_old` | `E42` typed | ✗ | CRM-only |
| `title` (multilingual) | `E35`/`P190`@lang | ✗ | CRM-only |
| `object_count`, `extent_cm` | `P57`, `E54` | ✗ | CRM-only |
| `location_id` | `P55` → `E53` | `rico:hasOrHadPhysicalLocation` → `rico:PhysicalLocation` | ✓ both views |
| `period_of_protection` | `P104` → `E30_Right` + term + `E52` release year | `rico:isOrWasRegulatedBy` → `rico:Rule`, `rico:conditionsOfAccess` | ✓ both views |
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
| typed multilingual note | `E33` + `P2 type` + `P190`@lang | `rico:scopeAndContent`/`history`/`recordResourceExtent`/… else `rico:note` | ✓ (loss #2 fixed) |
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
- **RiC-O-only view → relational skeleton + notes.** You get: `RecordSet`/`Record`/
  `Instantiation`/`Person`/`Place` classes; the inclusion hierarchy; `hasCreator` (the
  sender); `hasOrHadSubject`; `hasInstantiation`; and the note text via RiC-O note
  properties (`scopeAndContent`, `recordResourceExtent`, `note`). You **do not** get: the
  title/identifier/dates, the **recipients**, the place geometry, or the actors'
  birth/death. → a graph with the descriptive notes but still not a full description.

**Round-trip:** the Anton record is largely reconstructable from the CRM view (minus
`full_text` and some `object_type` nuance); it is **not** reconstructable from the RiC-O
view alone.

## The real losses, ranked

1. **The RiC-O view is a skeleton, by design (A+).** A pure-RiC-O consumer gets structure
   but not content/events/recipients. This is the single biggest "loss" — and it is the
   *opposite* of the RiC-AG caveat: here CRM is the rich view, RiC-O the lean one. Closing
   it = upgrade path B.
2. **Note-type semantics — FIXED.** Previously the RiC-O view carried nothing for notes
   (only a local CRM type term). Now the note text is emitted as the matching RiC-O
   datatype property on the entity (`rico:scopeAndContent`, `rico:recordResourceExtent`,
   `rico:history`, `rico:recordResourceStructure`, `rico:physicalCharacteristicsNote`,
   `rico:accrualsStatus`, `rico:conditionsOfAccess`/`conditionsOfUse`; generic `rico:note`
   otherwise — all verified against the RiC-O 1.1 RDF). Mapping table: `mapping.md` §6.
   *Note:* a `skos:exactMatch` to ISAD(G) was considered and rejected — ISAD has no concept
   URIs, so the semantic travels via the RiC-O **property**, not a concept alignment.
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

- [x] Loss #2: emit specific `rico:` note properties (done; verified against RiC-O 1.1).
- [ ] Decide if/when to take upgrade path **B** (RiC-O standalone) — currently not needed.
- [x] Re-run this audit once the exporter is implemented (Phase 1, anton#231, 2026-06-01).

## Phase 1 audit (2026-06-01)

Exporter implemented in Anton as the Artisan command `anton:export-rdf`
(`app/Services/Exporter/Rdf/`, sweetrdf/easyrdf, openspec change
`add-rdf-export-cidoc-crm`). Output verified against this fidelity matrix by
parsing the produced Turtle back into an `EasyRdf\Graph` and walking it. The A+
profile reproduces as designed; the table above (§ "Field-coverage matrix")
holds without correction. Specific points re-confirmed against the running
exporter on tenant `kr`:

- **AntonObject** — full CRM detail (E73 lead class with object_type subclass
  refinement, P46/P46i hierarchy, E42 identifier(s), E35 multilingual title,
  P57 part count), plus the RiC-O skeleton (`rico:RecordSet`/`Record` per
  level, `rico:isOrWasIncludedIn` reverse hierarchy). ✓
- **AntonEvent** — 17 event types looked up via config (event-types.md is the
  source of truth), CRM event class + object property + actor + place + dates
  via `DateNormaliser`, plus the Anton term redundantly via `P2_has_type`.
  Per A+ the RiC-O side emits `rico:hasCreator` (creation/writing/text_author/
  production/engravation/edition) and `rico:hasProvenance` (provenance/
  acquisition) on the linked object. ✓
- **Note doubling (loss #2 fix)** — typed `crm:E33_Linguistic_Object` AND the
  matching RiC-O datatype property (`rico:scopeAndContent`, `rico:history`,
  `rico:recordResourceExtent`, etc., generic `rico:note` fallback) emitted on
  the host entity. Verified end-to-end with a fixture note. ✓
- **Data quirks** — all three documented in `mapping.md` "Findings" handled:
  WKT `POINT(lat lon)` → `POINT(lon lat)`, `0000-00-00` drops the time-span,
  `YYYY-00-00` reduces to `xsd:gYear`, instance IRIs emitted as relative IRIs
  against a tenant-scoped `@base`. ✓
- **Access filter** — central `Support/AccessFilter.php`: `objects.private`,
  `media.private_media`, `actors.private`, plus private note types
  (`internal_note`, `archivists_notes`, `comment`). Private rows emit no
  triples, neither as subject nor as object — verified with private-object,
  private-note, and private-media fixtures. ✓

Open points carried into a future audit:
- Place `private` filter and Keyword `private` filter — the Anton `kr` schema
  has no `private` column on `places`/`keywords`, so the filter is a no-op
  there. When/if these columns are added (or another tenant has them) the
  filter logic in `AccessFilter::allows()` needs to be extended.
- WKT regression test currently skips (no place with `location` in the kr
  seed). Re-confirm against `fondazionearp` once the PoC tenant is reachable.
- JSON-LD and RDF/XML serialisations are not yet exercised; same in-memory
  graph just needs the alternative serializer wiring in Phase 2.
