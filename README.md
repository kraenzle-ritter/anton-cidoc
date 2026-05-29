# anton-cidoc — Mapping Anton data to CIDOC CRM & Records in Context

Goal: a precise, serialization-neutral **mapping** of the data held in the Anton
archival management application onto **CIDOC CRM** (lead model) with a
**RiC-O** alignment (Records in Context Ontology), as the basis for an RDF export.

## Decisions

| Topic | Decision | Rationale |
|---|---|---|
| Lead model | **CIDOC CRM 7.1.x** | more widely adopted; object-/event-centric |
| Secondary model | **RiC-O 1.0** | archive-specific, CRM-compatible alignment, close to ISAD/ISAAR/ISDIAH |
| Canonical serialization | **RDF/Turtle** | most readable; spec examples in Turtle |
| Further outputs | **JSON-LD, RDF/XML** | losslessly derivable from the same graph |
| Export architecture | build the graph **once** → serialize to every format | e.g. EasyRdf in Laravel |

## Compatibility assessment

**Overall verdict: high.** Anton's data model maps to CIDOC CRM well above
average, and to RiC-O even better — **without restructuring Anton**. The effort
lies in *configuration* (lookups, vocabulary alignment) and *normalization*
(data quirks), not in a data migration.

**What carries the compatibility**
- **Event-centricity:** `antonevents` (*object × actor × place × date × type*) is
  structurally already what CRM requires (E5/E7/E12/E65 …) and matches RiC-O's
  activity/agent model. Most systems have to reconstruct events artificially —
  Anton does not.
- **Hierarchy:** closure table → `P46`/`P46i`, lossless.
- **Authorities + linked data:** `actors`/`places`/`keywords` with `resources`
  (GND/VIAF/GeoNames) → `owl:sameAs`, immediately linked-open-data ready.
- **Places are especially clean:** identity via GeoNames URI **and** type via
  `place_type` = GeoNames feature classes (`gn:`) — both bound to standards, with
  no maintenance overhead.
- **Multilinguality** (JSON labels) and **typed notes** (38 types ≈ ISAD(G)/EAD)
  translate cleanly.

**Points of friction (solvable, but decide consciously)**
- **Class choice `E22` vs. `E73`:** Anton does not separate the physical carrier
  from the intellectual content — the export must commit (recommendation: `E22`,
  born-digital additionally `crmdig:D1`).
- **`object_type` is tenant-dependent:** needs a **per-tenant** alignment to
  Getty AAT — maintenance work, not a modelling issue. (Only this type;
  `place_type` is standard-bound, see above.)
- **Data hygiene:** date quirks (`0000-00-00`, null month) and geo axes
  (`POINT(lat lon)` → `(lon lat)`) must be normalized, otherwise the result is
  syntactically valid but semantically wrong RDF.
- **Weakly modelled:** rights/protection periods and `relations` can be expressed
  more richly in CRM/RiC than Anton currently records them → export stays coarse.

**CRM vs. RiC-O**
- **RiC-O is the closer fit** (purpose-built for archives, direct
  ISAD/ISAAR/ISDIAH lineage) — levels of description, agents and relations map
  nearly 1:1.
- **CRM is more general** and therefore more widely adopted, somewhat more verbose
  for purely archival constructs (levels only via `P2_has_type`). Since RiC-O
  ships an official CRM alignment, the two are **coherently exportable together** —
  no contradiction.

## Why the event model fits

Anton is already **event-centric**: `antonevents` links
*object × actor × place × date × event type*. This maps 1:1 onto CIDOC CRM's
event logic (E5/E7/E12/E65 …) and corresponds to RiC-O's activity/agent model.

## Relevant Anton entities (source side)

| Anton model | Table | Role |
|---|---|---|
| AntonObject | `objects` | unit of description (hierarchical, closure table) |
| AntonEvent | `antonevents` | event (object ↔ actor ↔ place ↔ date ↔ type) |
| Actor | `actors` | person / corporate body / family |
| Place | `places` | place (incl. geo point, country) |
| Keyword | `keywords` | subject term / descriptor |
| Note | `notes` | typed, multilingual text (polymorphic) |
| Relation | `relations` | generic cross-reference (polymorphic) |
| Media | `media` | digital object / representation |
| Location | `locations` | physical storage location |
| Term / Taxonomy | `terms` / `taxonomies` | controlled vocabulary |

> The `isads` / `isaars` / `isdiahs` tables only hold the ICA **standard
> definitions** themselves (reference) — they are **not** records and are
> irrelevant to the mapping.

Pivot / descriptor links: `actors_objects`, `objects_places`, `objects_keywords`,
`objects_languages` (aboutness / descriptors, as opposed to the processual
`antonevents`).

## Files

- [`docs/mapping.md`](docs/mapping.md) — the actual mapping specification, entity by entity
- [`docs/event-types.md`](docs/event-types.md) — all 17 Anton event types → CRM classes (verified)
- [`docs/vocabularies.md`](docs/vocabularies.md) — verified vocabularies (actor/object/note types, rights/access, relations) + AAT & GeoNames alignment
- [`examples/fma-hs-k-30-3.ttl`](examples/fma-hs-k-30-3.ttl) — proof of concept from real data
  (tenant `fondazionearp`, letter Janco→Arp): fonds hierarchy, production/reception event,
  actors with authority links, place with geo, typed notes, digital object. Validated
  (227 triples, parses to Turtle/JSON-LD/RDF-XML).
