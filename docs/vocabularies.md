# Verified vocabularies & rights (Anton → CRM/RiC)

Verified against real data (multiple tenants). Complements `mapping.md`.

## 1. Actor types — taxonomy `authority_type`

`actors.type_id` → `terms` of the taxonomy **`authority_type`** (not "actor_type").

| Anton term | CRM | RiC-O |
|---|---|---|
| `person` | `crm:E21_Person` | `rico:Person` |
| `family` | `crm:E74_Group` (type family) | `rico:Family` |
| `corporate_body` | `crm:E74_Group` *(or `crm:E40_Legal_Body`)* | `rico:CorporateBody` |
| `organisational_unit` | `crm:E74_Group` | `rico:CorporateBody` (sub-unit) |
| `group` | `crm:E74_Group` | `rico:Group` / `rico:Agent` |
| `software` | `crmdig:D14_Software` *(edge case)* | `rico:Mechanism` |

Anton term additionally emitted as `crm:P2_has_type → crm:E55_Type`.

## 2. Object types — taxonomy `object_type` (**tenant-dependent**)

`objects.object_type_id` is **tenant-specific** (in `fondazionearp`, e.g.
`file`, `manuscript`, `photo`, `object`, `print`). They refine the
`crm:E22_Human-Made_Object` via `crm:P2_has_type → crm:E55_Type`.

**CIDOC CRM provides no object-type vocabulary of its own** — types are
deliberately delegated to external vocabularies. Recommended pattern for the
tenant-dependent list: each tenant term stays a local `E55_Type` (`skos:Concept`
in the tenant scheme) and gets — **once** — a `skos:exactMatch`/`skos:closeMatch`
to a global vocabulary → tenant-specific *and* interoperable.

Recommended target vocabulary: **Getty AAT** (`vocab.getty.edu`, the CRM/Linked-Art
standard, multilingual). Alternatives: Wikidata, LoC LCGFT (genre/form), RDA
content/carrier types, RiC `DocumentaryFormType`. For digital formats, **PRONOM**
(already via `media.pronom_id`) is the right vocabulary.

Verified AAT alignment of the observed terms:

| Anton `object_type` | Getty AAT | URI |
|---|---|---|
| `photo` | photographs | `http://vocab.getty.edu/aat/300046300` |
| `print` | prints | `http://vocab.getty.edu/aat/300041273` |
| `manuscript` | manuscripts | `http://vocab.getty.edu/aat/300028569` |
| `file` | documents *(generic; "file" is rather a level of description)* | `http://vocab.getty.edu/aat/300026030` |
| `object` | *(generic — `E22` without AAT)* | — |

> The alignment table (tenant term → AAT) is maintained **per tenant** (small,
> one-off) and is part of the exporter configuration. The same pattern suits
> `keyword_type` and note types. **Not** `place_type` — that is already a fixed
> standard (GeoNames feature classes, see §3).

Optionally also more specific CRM classes: `photo` → `crm:E38_Image`,
`manuscript`/`print` → additionally `crm:E31_Document`.

## 3. Place types — taxonomy `place_type` = **GeoNames feature classes**

The single-letter terms are **not** tenant-specific but the **GeoNames feature
classes** (tenant-independent, a fixed standard):

| Code | GeoNames feature class |
|---|---|
| `A` | country, state, region (administrative) |
| `H` | stream, lake (hydrographic) |
| `L` | parks, area |
| `P` | city, village (populated place) |
| `R` | road, railroad |
| `S` | spot, building, farm |
| `T` | mountain, hill, rock (hypsographic) |
| `U` | undersea |
| `V` | forest, heath (vegetation) |

Target vocabulary is therefore directly the **GeoNames ontology** (`gn:`, as linked
data) — no per-tenant alignment needed. Mapping: `crm:P2_has_type → crm:E55_Type`,
where the E55 term carries a `skos:exactMatch` to the GeoNames feature-class URI;
optionally, for GeoNames-native consumers, `gn:featureClass` directly on the
`E53_Place`.

```turtle
@prefix gn: <http://www.geonames.org/ontology#> .

<place/5> a crm:E53_Place ;
    crm:P2_has_type <term/place_type/P> ;
    gn:featureClass gn:P .                       # populated place

<term/place_type/P> a crm:E55_Type , skos:Concept ;
    skos:prefLabel "populated place"@en ;
    skos:exactMatch gn:P .
```

`keyword_type` is empty in the inspected tenants.

## 4. Description status (administrative)

- `status_of_description`: `draft`, `final`
- `detail_of_description`: `minimal`, `partial`, `full`

→ RiC-O `rico:descriptionStatus` / completeness level, or `crm:P2_has_type` on the
description. For a public export usually omitted (export `final` only).

## 5. Rights & access (export-critical)

| Source | Meaning | Mapping / rule |
|---|---|---|
| `objects.private` (1 = private) | record not public | **export filter**: exclude `private=1`; `NULL`/`0` = public |
| `media.private_media` | media not public | filter analogously |
| `actors/places/keywords.private` | authority private | filter analogously |
| note with `note_type=internal_note` | private note (`notes` has **no** `private` column!) | exclude private note types |
| `period_of_protection` (`public`, `standard`, `prolonged`, `standard_10`, `standard_20`, `forever`) | protection-period regime | `crm:E30_Right` / `rico:AccessRule` |
| `release_year` / `expiry_year` (note) | end of protection period | `crm:E30_Right` + date; RiC `rico:Rule` |

Protection-period modelling (CRM): object `crm:P104_is_subject_to → crm:E30_Right`
(type = period_of_protection term), end of period as an associated date.
RiC-O: `rico:regulatedBy → rico:Rule` (`rico:AccessRule`) with effective date.

## 6. Note types — taxonomy of the non-`antonevent` `antonfields`

**38 types** across tenants; mostly straight from **ISAD(G)/EAD**. Generic rule
(applies to all): note → `crm:E33_Linguistic_Object` (`crm:P190_has_symbolic_content`
+ `@locale`, `crm:P2_has_type` = note type), linked via
`crm:P67i_is_referred_to_by` to the described object/actor.

Key types with ISAD/semantic correspondence:

| Anton `note_type` | ISAD(G) / semantics | special handling |
|---|---|---|
| `scopecontent` | 3.3.1 Scope and content | — |
| `bioghist` | 3.2.2 Administrative/biographical history | belongs to the **actor** (`rico:history` / `crm:P3` on the actor) |
| `custod_hist` | 3.2.3 Archival history | possibly `crm:E10_Transfer_of_Custody` |
| `acqinfo` | 3.2.4 Immediate source of acquisition | possibly `crm:E8_Acquisition` |
| `provenance` | provenance | possibly `crm:E10` |
| `appraisal` | 3.3.2 Appraisal/destruction | — |
| `accruals` | 3.3.3 Accruals | — |
| `arrangement` | 3.3.4 System of arrangement | — |
| `condition_of_access` | 3.4.1 Conditions governing access | **rights** (§5) → `E30_Right` |
| `condition_of_reproduction` | 3.4.2 Conditions governing reproduction | **rights** (§5) |
| `language_script` | 3.4.3 Language/scripts | complements `objects_languages` |
| `physical_description` | 3.4.4 Physical characteristics | — |
| `finding_aids` | 3.4.5 Finding aids | — |
| `related_units` | 3.5.3 Related units of description | possibly `crm:P67_refers_to` |
| `publications` | 3.5.4 Publication note | — |
| `extent_text` | 3.1.5 Extent | alt. `crm:E54_Dimension` |
| `general_note` | 3.6.1 Note | — |
| `internal_note`, `archivists_notes`, `comment` | internal notes | **private** → export filter (§5) |
| `original_inscription_*`, `image_caption`, `original_template` | object-specific inscriptions/captions | — |
| `expiry_year`, `vacat_detail`, `date_comment` | period/dating detail | `expiry_year` → rights (§5) |

Full list (38): accruals, acqinfo, appraisal, archivists_notes, arrangement,
bioghist, comment, condition_of_access, condition_of_reproduction, custod_hist,
date_comment, description, exhibitions, expiry_year, extent_text, finding_aids,
general_note, image_caption, internal_note, language_script, location_details,
original_inscription_manuscript, original_inscription_print, original_template,
parcel_number, physical_description, principal, provenance, pub_place,
publications, publisher, related_units, reproductions, rules_note, scopecontent,
sources, vacat_detail.

## 7. Relations — table `relations`

Barely used (test data only in the inspected tenants). Two types observed:

| `relations.name` | CRM | note |
|---|---|---|
| `references` | `crm:P67_refers_to` | — |
| `isRelatedTo` | `rdfs:seeAlso` *(generic)* | — |

`source_type` is mostly `page` (a CMS page of the website/edition), not an
archival object → largely out of scope for the **archival** RDF export. Extend
later if needed.
