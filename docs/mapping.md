# Mapping specification: Anton → CIDOC CRM (+ RiC-O)

Status: **draft v0.1**. Lead model **CIDOC CRM 7.1.x**, secondary model **RiC-O 1.0**.
Examples in RDF/Turtle.

## Namespaces

```turtle
@prefix crm:   <http://www.cidoc-crm.org/cidoc-crm/> .
@prefix rico:  <https://www.ica.org/standards/RiC/ontology#> .
@prefix rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs:  <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl:   <http://www.w3.org/2002/07/owl#> .
@prefix skos:  <http://www.w3.org/2004/02/skos/core#> .
@prefix geo:   <http://www.opengis.net/ont/geosparql#> .
@prefix crmdig:<http://www.ics.forth.gr/isl/CRMdig/> .
@prefix xsd:   <http://www.w3.org/2001/XMLSchema#> .
# Anton instances, tenant-specific (configurable) — used as @base:
@base          <https://{tenant}.anton.example/id/> .
```

> Turtle note: instance IRIs are written as **relative IRIs** against `@base`
> (e.g. `<object/1>`). A `/` is **not** allowed in Turtle prefixed names
> (`a:object/1` would be invalid), so `@base` + relative IRIs is the idiomatic
> form. See finding #1 below.

### URI strategy

| Entity | IRI pattern (relative to `@base`) | Example |
|---|---|---|
| AntonObject | `<object/{id}>` | `<object/4711>` |
| Actor | `<actor/{id}>` | `<actor/88>` |
| Place | `<place/{id}>` | `<place/12>` |
| Keyword | `<keyword/{id}>` | `<keyword/55>` |
| AntonEvent | `<event/{id}>` | `<event/903>` |
| Media | `<media/{id}>` | `<media/2001>` |
| Term (vocabulary) | `<term/{taxonomy}/{name}>` | `<term/level_of_description/fonds>` |
| Identifier node | `<object/{id}/identifier>` | — |
| Title node | `<object/{id}/title>` | — |

External authority IDs (`resources`: GND, VIAF, GeoNames, Wikidata) are emitted as
`owl:sameAs` / `skos:exactMatch` to the respective linked-data URI (not as
internal identifiers).

---

## 1. AntonObject — unit of description

**CRM lead class:** `crm:E22_Human-Made_Object`
(archival material as a produced object; born-digital optionally additionally `crmdig:D1_Digital_Object`).
**RiC-O:** `rico:RecordResource` — aggregations (collection/fonds/series/class/recordgroup) →
`rico:RecordSet`, single items (item/file) → `rico:Record` / `rico:RecordPart`.

| Anton field | CRM | RiC-O |
|---|---|---|
| (entity) | `<object/{id}> a crm:E22_Human-Made_Object` | `a rico:RecordResource` |
| `identifier` (reference code) | `crm:P1_is_identified_by` → `crm:E42_Identifier` (type "reference number") | `rico:hasOrHadIdentifier` / `rico:identifier` |
| `identifier_old` | another `crm:E42_Identifier`, type "former reference" | `rico:identifier` (type former) |
| `title` (multilingual JSON) | `crm:P102_has_title` → `crm:E35_Title`, text via `crm:P190_has_symbolic_content` + `@lang` | `rico:hasOrHadTitle` / `rico:title` (langString) |
| `level_of_description_id` | `crm:P2_has_type` → `crm:E55_Type` (`<term/level_of_description/…>`) | `rico:hasRecordResourceType` / `rico:RecordSetType` |
| `parent_id` (hierarchy) | `crm:P46i_forms_part_of` → parent object (parent: `crm:P46_is_composed_of`) | `rico:isOrWasIncludedIn` ↔ `rico:includesOrIncluded` |
| `object_count` | `crm:P57_has_number_of_parts` (xsd:integer) | `rico:hasInstantiation` / `rico:recordResourceExtent` |
| `extent_cm` | `crm:P43_has_dimension` → `crm:E54_Dimension` (unit cm, `crm:E58_Measurement_Unit`) | `rico:hasExtent` → `rico:Extent` |
| `release_year` / protection period (`period_of_protection_id`) | `crm:P104_is_subject_to` → `crm:E30_Right` (embargo) | `rico:hasOrHadHolder` / `rico:regulatedBy` → `rico:Rule` |
| `private` (bool) | access status via `crm:E30_Right` / export filter | `rico:AccessRule` |
| `status_of_description_id` | `crm:P2_has_type` → `crm:E55_Type` (editing status) | `rico:descriptionLevel` accompanying info |
| `location_id` (physical location) | `crm:P55_has_current_location` → `crm:E53_Place` | `rico:hasOrHadPhysicalLocation` |
| full text (`full_text`) | (not exported; internal) | — |

### Example (fonds with sub-unit)

```turtle
<object/100>
    a crm:E22_Human-Made_Object ;
    crm:P2_has_type <term/level_of_description/fonds> ;
    crm:P1_is_identified_by <object/100/identifier> ;
    crm:P102_has_title <object/100/title> ;
    crm:P46_is_composed_of <object/101> .

<object/100/identifier>
    a crm:E42_Identifier ;
    crm:P190_has_symbolic_content "CH-000123-1 N Müller" ;
    crm:P2_has_type <term/identifier_type/reference_number> .

<object/100/title>
    a crm:E35_Title ;
    crm:P190_has_symbolic_content "Nachlass Anna Müller"@de .

<object/101>
    a crm:E22_Human-Made_Object ;
    crm:P2_has_type <term/level_of_description/series> ;
    crm:P46i_forms_part_of <object/100> .
```

---

## 2. AntonEvent — event (core of the model)

An `antonevents` record links object, actor, place, date and an event type
(`event_type_id`). The type drives the CRM class.

**Generic:** `<event/{id}> a crm:E5_Event` (or a more specific subclass), with
- `crm:P14_carried_out_by` → actor (for activities)
- `crm:P7_took_place_at` → place
- `crm:P4_has_time-span` → `crm:E52_Time-Span` (see §2.1)
- `crm:P3_has_note` → `event_details`
- `crm:P2_has_type` → `crm:E55_Type` (Anton event-type term)

**Link to the object** depends on the event type (see table).

### Event type → CRM class

| Anton `event_type` | CRM class | object link | RiC-O |
|---|---|---|---|
| `creation`, `production` | `crm:E12_Production` | `crm:P108_has_produced` → object | `rico:Activity` (`creation`) |
| `writing`, `text_author` | `crm:E65_Creation` | `crm:P94_has_created` → object (E73) | `rico:Activity` |
| `acquisition` | `crm:E8_Acquisition` | `crm:P24_transferred_title_of` → object | `rico:Activity` (acquisition) |
| `reception` | `crm:E10_Transfer_of_Custody` | `crm:P30_transferred_custody_of` → object | `rico:Activity` |
| `provenance` | `crm:E10_Transfer_of_Custody` | `crm:P30_transferred_custody_of` | `rico:hasProvenance` |
| `digitisation`, `reproduction`, `ingest` | `crm:E12_Production` / `crmdig:D2_Digitization_Process` | produces `Media` (§7) | `rico:Activity` |
| `preservation` | `crm:E11_Modification` | `crm:P31_has_modified` → object | `rico:Activity` |
| `edition`, `performance`, `coloring`, `engravation`, `other` | `crm:E7_Activity` (+ `P2_has_type`) | `crm:P16i_was_used_for` / `P12_occurred_in_the_presence_of` | `rico:Activity` (typed) |

> **Full, verified list of all 17 Anton event types** (id 1000–1015, 1099 `other`)
> with CRM class and object property: see [`event-types.md`](event-types.md).
> Event types are `antonfields` with `type = 'antonevent'`; `event_type_id`
> references their `id`. Anton's type is in every case additionally emitted as
> `crm:P2_has_type → crm:E55_Type`.

### 2.1 Dates (`date_start`, `date_end`, `*_ca`, `datierung_text`)

```turtle
<event/903>
    a crm:E12_Production ;
    crm:P108_has_produced <object/101> ;
    crm:P14_carried_out_by <actor/88> ;
    crm:P7_took_place_at <place/12> ;
    crm:P4_has_time-span <event/903/timespan> ;
    crm:P3_has_note "Reinschrift, eigenhändig"@de .

<event/903/timespan>
    a crm:E52_Time-Span ;
    crm:P82a_begin_of_the_begin "1898-01-01"^^xsd:date ;
    crm:P82b_end_of_the_end    "1899-12-31"^^xsd:date ;
    crm:P81_ongoing_throughout "1898/1899" ;          # when uncertain
    crm:P3_has_note "ca. 1898–1899"@de .              # datierung_text / *_ca
```

- `date_start_ca` / `date_end_ca` → uncertainty via `crm:P82a/P82b` (outer bounds)
  vs. `crm:P81a/P81b` (inner, certain bounds); `datierung_text` as `crm:P3_has_note`.
- RiC-O: `rico:Activity` `rico:hasBeginningDate` / `rico:hasEndDate` →
  `rico:Date` with `rico:expressedDate` (text) + `rico:normalizedDateValue`.

---

## 3. Actor — person / corporate body / family

| Anton `type` | CRM | RiC-O |
|---|---|---|
| person | `crm:E21_Person` | `rico:Person` |
| corporate body | `crm:E74_Group` (or `crm:E40_Legal_Body`) | `rico:CorporateBody` |
| family | `crm:E74_Group` (type family) | `rico:Family` |

> Verified actor types (taxonomy `authority_type`): person, family,
> corporate_body, organisational_unit, group, software. Full mapping incl. edge
> cases: [`vocabularies.md`](vocabularies.md) §1.

| Anton field | CRM | RiC-O |
|---|---|---|
| `label` / `name` | `crm:P1_is_identified_by` → `crm:E41_Appellation` (`crm:P190_has_symbolic_content`) | `rico:hasOrHadName` → `rico:Name` |
| `alternative_names`, `variants`, `abbreviations` | further `crm:E41_Appellation` with `crm:P2_has_type` (alt/variant/abbr.) | `rico:hasOrHadName` (type variant) |
| `type_id` | `crm:P2_has_type` → `crm:E55_Type` (`<term/authority_type/…>`) | `rico:hasAgentType` |
| `date_start` / `date_end` (person) | `crm:P98i_was_born` → `crm:E67_Birth`; `crm:P100i_died_in` → `crm:E69_Death` (each with `crm:E52_Time-Span`) | `rico:hasBirthDate` / `rico:hasDeathDate` |
| `date_start` / `date_end` (corporate body) | `crm:E66_Formation` (`P95i`), `crm:E68_Dissolution` (`P99i`) | `rico:hasBeginningDate` / `rico:hasEndDate` |
| `resources` (GND/VIAF/Wikidata) | `owl:sameAs` / `skos:exactMatch` → external URI | `rico:hasOrHadIdentifier` + `owl:sameAs` |
| `notes` (biographical) | `crm:P3_has_note` or `crm:E33_Linguistic_Object` (§6) | `rico:history` / `rico:descriptiveNote` |

**Role in an event** (`antonevents.actor_id`): via the event (§2),
`crm:P14_carried_out_by`. Specific role via `crm:P14.1_in_the_role_of` → `crm:E55_Type`.

```turtle
<actor/88>
    a crm:E21_Person ;
    crm:P1_is_identified_by [ a crm:E41_Appellation ;
        crm:P190_has_symbolic_content "Müller, Anna" ] ;
    crm:P98i_was_born  [ a crm:E67_Birth ;
        crm:P4_has_time-span [ crm:P82a_begin_of_the_begin "1870-03-04"^^xsd:date ] ] ;
    crm:P100i_died_in  [ a crm:E69_Death ;
        crm:P4_has_time-span [ crm:P82b_end_of_the_end "1945-11-20"^^xsd:date ] ] ;
    owl:sameAs <https://d-nb.info/gnd/118585045> .
```

---

## 4. Place — place

**CRM:** `crm:E53_Place` · **RiC-O:** `rico:Place`.

| Anton field | CRM | RiC-O |
|---|---|---|
| `label` / `name` / `city` | `crm:P1_is_identified_by` → `crm:E48_Place_Name` | `rico:hasOrHadName` → `rico:Name` |
| `type_id` | `crm:P2_has_type` → `crm:E55_Type` (= GeoNames feature class, see [`vocabularies.md`](vocabularies.md) §3) | `rico:hasPlaceType` |
| `location` (geo point, SRID 4326) | `crm:P168_place_is_defined_by` → WKT literal (`geo:wktLiteral`) | `rico:hasOrHadLocation` + `geo:asWKT` |
| `country_code_id` / `country` | `crm:P89_falls_within` → enclosing `crm:E53_Place` | `rico:isOrWasPartOf` |
| `resources` (GeoNames) | `owl:sameAs` → GeoNames URI | `owl:sameAs` |

```turtle
<place/12>
    a crm:E53_Place ;
    crm:P1_is_identified_by [ a crm:E48_Place_Name ;
        crm:P190_has_symbolic_content "Bern"@de ] ;
    # NB: GeoSPARQL WKT is POINT(lon lat) — Anton stores POINT(lat lon), so swap.
    crm:P168_place_is_defined_by "POINT(7.4474 46.9480)"^^geo:wktLiteral ;
    crm:P89_falls_within <place/country/CH> ;
    owl:sameAs <https://www.geonames.org/2661552> .
```

---

## 5. Keyword & descriptors (aboutness)

Subject terms and the descriptor links (`objects_keywords`, `objects_places`,
`actors_objects`) express what a unit is *about* — **not** processes. CRM keeps
this separate from events.

| Anton | CRM | RiC-O |
|---|---|---|
| Keyword (entity) | `crm:E55_Type` or `crm:E89_Propositional_Object` (alt.: `skos:Concept`) | `rico:concept` / `skos:Concept` |
| `keywords` (objects_keywords) | object `crm:P67_refers_to` keyword, **or** `crm:P129_is_about` | `rico:hasOrHadSubject` |
| `places_descriptors` (objects_places) | object `crm:P129_is_about` → `crm:E53_Place` | `rico:hasOrHadSubject` → `rico:Place` |
| `actors_descriptors` (actors_objects) | object `crm:P129_is_about` → actor | `rico:hasOrHadSubject` → `rico:Agent` |

> Important: `actors_objects` (descriptor, "the object is about person X") is
> distinct from `antonevents.actor_id` (process, "person X created it").

---

## 6. Note — typed multilingual text

Polymorphic on object/actor/place/keyword (`model_type`/`model_id`), type via
`note_type_id`, with `locale`. Multilingual & typed → its own resource:

**CRM:** `crm:E33_Linguistic_Object` · **RiC-O:** `rico:descriptiveNote` / specific properties.

```turtle
<note/777>
    a crm:E33_Linguistic_Object ;
    crm:P2_has_type <term/note_type/scopecontent> ;
    crm:P190_has_symbolic_content "Enthält Korrespondenz 1898–1910 …"@de .

<object/101> crm:P67i_is_referred_to_by <note/777> .   # or crm:P3_has_note (short)
```

| Anton `note_type` | CRM type / property | RiC-O |
|---|---|---|
| `scopecontent` | `crm:E33` type scopecontent (link `crm:P67i`) | `rico:scopeAndContent` |
| `extent_text` | `crm:E33` type extent (or `crm:E54_Dimension`, §1) | `rico:recordResourceExtent` |
| `internal_note` | `crm:E33` type internal (export filter `private`) | `rico:descriptiveNote` (internal) |
| biographical/history (actor) | `crm:P3_has_note` on the actor | `rico:history` |

There are 38 note types in total, mostly mapping to ISAD(G)/EAD elements — full
list and ISAD correspondence: [`vocabularies.md`](vocabularies.md) §6. Short,
untyped texts may be emitted simply as `crm:P3_has_note` (E62 String) with
`crm:P3.1_has_type`.

---

## 7. Media — digital object / representation

**CRM:** `crmdig:D1_Digital_Object` (subclass of `crm:E73`) or `crm:E36_Visual_Item`.

| Anton field | CRM | RiC-O |
|---|---|---|
| (entity) | `<media/{id}> a crmdig:D1_Digital_Object` | `rico:Instantiation` |
| link to object | object `crm:P128_carries` / `crm:P138i_has_representation` → media | `rico:hasInstantiation` |
| `mime_type` | `crm:P2_has_type` → `crm:E55_Type` (format) | `rico:hasFormatType` |
| `file_name` / `name` | `crm:P1_is_identified_by` → `crm:E41_Appellation` | `rico:hasOrHadIdentifier` |
| `md5sum` | `crm:P1_is_identified_by` → `crm:E42_Identifier` (type md5) | `rico:hasInstantiation` / checksum |
| `size`, `pronom_id` | `crm:P43_has_dimension` / `crm:P2_has_type` | `rico:Instantiation` attributes |
| full text (`MediaText`) | `crm:P165i_is_incorporated_in` → `crm:E33_Linguistic_Object` | — |

---

## 8. Relation — generic cross-reference

`relations` (polymorphic: source/target of any type, `name`/`label`).

- **Known relation type** → specific CRM property (lookup table, e.g.
  "predecessor/successor" → `crm:P134_continued` / `crm:P134i_was_continued_by`;
  "see also" → `rdfs:seeAlso`).
- **Unknown/generic** → reified as `crm:E13_Attribute_Assignment` or
  `crm:P67_refers_to` with `crm:P2_has_type` (relation-type term).
- **RiC-O:** `rico:relation` / `rico:RelationType`, or specific
  `rico:isAssociatedWith*` properties.

Verified: `relations` is barely used (test data only); two types observed
(`references` → `crm:P67_refers_to`, `isRelatedTo` → `rdfs:seeAlso`), mostly
linking CMS pages → largely out of scope for the archival export. Details:
[`vocabularies.md`](vocabularies.md) §7.

---

## 9. Term / Taxonomy — controlled vocabulary

Each `terms` entry → `crm:E55_Type` (also `skos:Concept`), grouped per
`taxonomies.name` as a `skos:ConceptScheme`.

```turtle
<term/level_of_description/fonds>
    a crm:E55_Type , skos:Concept ;
    skos:prefLabel "Bestand"@de , "Fonds"@fr ;
    skos:inScheme <scheme/level_of_description> .
```

`term_translations` → `skos:prefLabel` (multilingual). Where useful, schemes can be
aligned to external vocabularies (e.g. RiC-O value lists, EAD levels, Getty AAT,
GeoNames feature classes) via `skos:exactMatch` — see
[`vocabularies.md`](vocabularies.md).

---

## Findings from the proof of concept

Validated against real data (`examples/fma-hs-k-30-3.ttl`, tenant `fondazionearp`,
227 triples, serializes cleanly to Turtle/JSON-LD/RDF-XML). Export-critical points
the exporter must handle:

1. **Turtle IRIs:** `/` is **not** allowed in Turtle prefixed names (`a:object/1`).
   Solution: `@base` + relative IRIs (`<object/1>`), or prefixed local names without `/`.
2. **Geo coordinates:** Anton stores `POINT(lat lon)`; GeoSPARQL `wktLiteral`
   requires `POINT(lon lat)` → **swap** on export.
3. **Date normalization:** real values contain `0000-00-00` (no date → omit the
   time-span) and null month/day such as `1951-00-00` (→ reduce to `1951` /
   `xsd:gYear`). `date_*_ca` → uncertainty (P81/P82), `datierung_text` →
   `crm:P3_has_note`.
4. **Letter = creation + reception:** Anton models a letter as a `creation` event
   (sender → `E12_Production` / `P14`) plus one or more `reception` events
   (recipient → `E10_Transfer_of_Custody` / `P29_custody_received_by`). This yields
   sender/recipient cleanly without a special property.
5. **Authority links:** `resources` is rich (GND, VIAF, Wikidata, GeoNames, LCNAF,
   HLS, SIKART …). Each row → `owl:sameAs` to the `url` column. The `url` column is
   authoritative (the embedded `full_json` may carry diverging IDs). Selecting
   stable LOD providers is recommended.

## Verification status of the open points

All formerly open points have been clarified against real data:

1. ✅ **Relation types** — `relations` barely used (test data only); two types
   (`references` → `P67_refers_to`, `isRelatedTo` → `rdfs:seeAlso`), mostly
   CMS-page links → out of scope for the archival export. Details: `vocabularies.md` §7.
2. ✅ **Event types** — fully verified (17 types): `event-types.md`.
3. ✅ **Rights/access** — `private` filter, note privacy via `note_type`,
   protection periods → `E30_Right`/`AccessRule`: `vocabularies.md` §5.
4. ✅ **Actor/object/place/status types + note types (38)** — verified:
   `vocabularies.md` §1–4, §6.
5. ✅ **Proof of concept** — `examples/fma-hs-k-30-3.ttl` (validated).

## Next step

**Implement the exporter in Anton itself** (Artisan command, serialization via
EasyRdf → Turtle/JSON-LD/RDF-XML). This specification (`mapping.md` +
`event-types.md` + `vocabularies.md`) and the PoC serve as the brief; the lookups
(`antonevent → CRM class`, `note_type → ISAD/CRM`) are carried over as exporter
configuration.
