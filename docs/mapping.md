# Mapping-Spezifikation: Anton → CIDOC CRM (+ RiC-O)

Status: **Entwurf v0.1**. Leitmodell **CIDOC CRM 7.1.x**, Zweitmodell **RiC-O 1.0**.
Beispiele in RDF/Turtle.

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
# Anton-Instanzen, tenant-spezifisch (konfigurierbar):
@prefix a:     <https://{tenant}.anton.example/id/> .
```

### URI-Strategie

| Entität | URI-Muster | Beispiel |
|---|---|---|
| AntonObject | `a:object/{id}` | `a:object/4711` |
| Actor | `a:actor/{id}` | `a:actor/88` |
| Place | `a:place/{id}` | `a:place/12` |
| Keyword | `a:keyword/{id}` | `a:keyword/55` |
| AntonEvent | `a:event/{id}` | `a:event/903` |
| Media | `a:media/{id}` | `a:media/2001` |
| Term (Vokabular) | `a:term/{taxonomie}/{name}` | `a:term/level_of_description/fonds` |
| Identifier-Knoten | `a:object/{id}/identifier` | — |
| Title-Knoten | `a:object/{id}/title` | — |

Externe Authority-IDs (`resources`: GND, VIAF, Geonames, Wikidata) werden als
`owl:sameAs` / `skos:exactMatch` auf die jeweilige Linked-Data-URI gesetzt
(nicht als interne Identifier).

---

## 1. AntonObject — Verzeichnungseinheit

**CRM-Leitklasse:** `crm:E22_Human-Made_Object`
(Archivgut als hergestelltes Objekt; born-digital optional zusätzlich `crmdig:D1_Digital_Object`).
**RiC-O:** `rico:RecordResource` — Aggregationen (collection/fonds/series/class/recordgroup) →
`rico:RecordSet`, Einzelstücke (item/file) → `rico:Record` / `rico:RecordPart`.

| Anton-Feld | CRM | RiC-O |
|---|---|---|
| (Entität) | `a:object/{id} a crm:E22_Human-Made_Object` | `a rico:RecordResource` |
| `identifier` (Signatur) | `crm:P1_is_identified_by` → `crm:E42_Identifier` (Typ „reference number") | `rico:hasOrHadIdentifier` / `rico:identifier` |
| `identifier_old` | weiterer `crm:E42_Identifier`, Typ „former reference" | `rico:identifier` (Typ alt) |
| `title` (mehrsprachig JSON) | `crm:P102_has_title` → `crm:E35_Title`, Text via `crm:P190_has_symbolic_content` + `@lang` | `rico:hasOrHadTitle` / `rico:title` (langString) |
| `level_of_description_id` | `crm:P2_has_type` → `crm:E55_Type` (`a:term/level_of_description/…`) | `rico:hasRecordResourceType` bzw. `rico:RecordSetType` |
| `parent_id` (Hierarchie) | `crm:P46i_forms_part_of` → Eltern-Objekt (Eltern: `crm:P46_is_composed_of`) | `rico:isOrWasIncludedIn` ↔ `rico:includesOrIncluded` |
| `object_count` | `crm:P57_has_number_of_parts` (xsd:integer) | `rico:hasInstantiation` / `rico:recordResourceExtent` |
| `extent_cm` | `crm:P43_has_dimension` → `crm:E54_Dimension` (Einheit cm, `crm:E58_Measurement_Unit`) | `rico:hasExtent` → `rico:Extent` |
| `release_year` / Schutzfrist (`period_of_protection_id`) | `crm:P104_is_subject_to` → `crm:E30_Right` (Embargo/Sperrfrist) | `rico:hasOrHadHolder` / `rico:regulatedBy` → `rico:Rule` |
| `private` (bool) | Zugriffsstatus via `crm:E30_Right` / Filter beim Export | `rico:AccessRule` |
| `status_of_description_id` | `crm:P2_has_type` → `crm:E55_Type` (Bearbeitungsstatus) | `rico:descriptionLevel`-Begleitinfo |
| `location_id` (phys. Standort) | `crm:P55_has_current_location` → `crm:E53_Place` | `rico:hasOrHadPhysicalLocation` |
| Volltext (`full_text`) | (nicht exportiert; intern) | — |

### Beispiel (Fonds mit Untereinheit)

```turtle
a:object/100
    a crm:E22_Human-Made_Object ;
    crm:P2_has_type a:term/level_of_description/fonds ;
    crm:P1_is_identified_by a:object/100/identifier ;
    crm:P102_has_title a:object/100/title ;
    crm:P46_is_composed_of a:object/101 .

a:object/100/identifier
    a crm:E42_Identifier ;
    crm:P190_has_symbolic_content "CH-000123-1 N Müller" ;
    crm:P2_has_type a:term/identifier_type/reference_number .

a:object/100/title
    a crm:E35_Title ;
    crm:P190_has_symbolic_content "Nachlass Anna Müller"@de .

a:object/101
    a crm:E22_Human-Made_Object ;
    crm:P2_has_type a:term/level_of_description/series ;
    crm:P46i_forms_part_of a:object/100 .
```

---

## 2. AntonEvent — Ereignis (Kern des Modells)

Ein `antonevents`-Datensatz verknüpft Object, Actor, Place, Datum und einen
Ereignistyp (`event_type_id`). Der Typ steuert die CRM-Klasse.

**Generisch:** `a:event/{id} a crm:E5_Event` (bzw. spezifischere Subklasse), mit
- `crm:P14_carried_out_by` → Actor (für Aktivitäten)
- `crm:P7_took_place_at` → Place
- `crm:P4_has_time-span` → `crm:E52_Time-Span` (s. §2.1)
- `crm:P3_has_note` → `event_details`
- `crm:P2_has_type` → `crm:E55_Type` (Anton-Ereignistyp-Term)

**Verknüpfung zum Objekt** je nach Ereignistyp (s. Tabelle).

### Ereignistyp → CRM-Klasse

| Anton `event_type` | CRM-Klasse | Objekt-Anbindung | RiC-O |
|---|---|---|---|
| `creation`, `production` | `crm:E12_Production` | `crm:P108_has_produced` → Objekt | `rico:Activity` (`creation`) |
| `writing`, `text_author` | `crm:E65_Creation` | `crm:P94_has_created` → Objekt (E73) | `rico:Activity` |
| `acquisition` | `crm:E8_Acquisition` | `crm:P24_transferred_title_of` → Objekt | `rico:Activity` (acquisition) |
| `reception` | `crm:E10_Transfer_of_Custody` | `crm:P30_transferred_custody_of` → Objekt | `rico:Activity` |
| `provenance` | `crm:E10_Transfer_of_Custody` | `crm:P30_transferred_custody_of` | `rico:hasProvenance` |
| `digitisation`, `reproduction`, `ingest` | `crm:E12_Production` / `crmdig:D2_Digitization_Process` | erzeugt `Media` (§7) | `rico:Activity` |
| `preservation` | `crm:E11_Modification` | `crm:P31_has_modified` → Objekt | `rico:Activity` |
| `edition`, `performance`, `coloring`, `engravation`, `other` | `crm:E7_Activity` (+ `P2_has_type`) | `crm:P16i_was_used_for` / `P12_occurred_in_the_presence_of` | `rico:Activity` (typisiert) |

> **Vollständige, verifizierte Liste aller 17 Anton-Ereignistypen** (id 1000–1015,
> 1099 `other`) mit CRM-Klasse und Objekt-Property: siehe
> [`event-types.md`](event-types.md). Ereignistypen sind `antonfields` mit
> `type = 'antonevent'`; `event_type_id` referenziert deren `id`. Antons Typ wird
> in jedem Fall zusätzlich als `crm:P2_has_type → crm:E55_Type` mitgegeben.

### 2.1 Zeitangaben (`date_start`, `date_end`, `*_ca`, `datierung_text`)

```turtle
a:event/903
    a crm:E12_Production ;
    crm:P108_has_produced a:object/101 ;
    crm:P14_carried_out_by a:actor/88 ;
    crm:P7_took_place_at a:place/12 ;
    crm:P4_has_time-span a:event/903/timespan ;
    crm:P3_has_note "Reinschrift, eigenhändig"@de .

a:event/903/timespan
    a crm:E52_Time-Span ;
    crm:P82a_begin_of_the_begin "1898-01-01"^^xsd:date ;
    crm:P82b_end_of_the_end    "1899-12-31"^^xsd:date ;
    crm:P81_ongoing_throughout "1898/1899" ;          # bei Unschärfe
    crm:P3_has_note "ca. 1898–1899"@de .              # datierung_text / *_ca
```

- `date_start_ca` / `date_end_ca` → Unschärfe über `crm:P82a/P82b` (äußere Grenzen)
  vs. `crm:P81a/P81b` (innere, sichere Grenzen); `datierung_text` als `crm:P3_has_note`.
- RiC-O: `rico:Activity` `rico:hasBeginningDate` / `rico:hasEndDate` →
  `rico:Date` mit `rico:expressedDate` (Text) + `rico:normalizedDateValue`.

---

## 3. Actor — Person / Körperschaft / Familie

| Anton `type` | CRM | RiC-O |
|---|---|---|
| Person | `crm:E21_Person` | `rico:Person` |
| Körperschaft | `crm:E74_Group` (ggf. `crm:E40_Legal_Body`) | `rico:CorporateBody` |
| Familie | `crm:E74_Group` (Typ Familie) | `rico:Family` |

| Anton-Feld | CRM | RiC-O |
|---|---|---|
| `label` / `name` | `crm:P1_is_identified_by` → `crm:E41_Appellation` (`crm:P190_has_symbolic_content`) | `rico:hasOrHadName` → `rico:Name` |
| `alternative_names`, `variants`, `abbreviations` | weitere `crm:E41_Appellation` mit `crm:P2_has_type` (alt/variant/abbr.) | `rico:hasOrHadName` (Typ variant) |
| `type_id` | `crm:P2_has_type` → `crm:E55_Type` (`a:term/actor_type/…`) | `rico:hasAgentType` |
| `date_start` / `date_end` (Person) | `crm:P98i_was_born` → `crm:E67_Birth`; `crm:P100i_died_in` → `crm:E69_Death` (je `crm:E52_Time-Span`) | `rico:hasBirthDate` / `rico:hasDeathDate` |
| `date_start` / `date_end` (Körperschaft) | `crm:E66_Formation` (`P95i`), `crm:E68_Dissolution` (`P99i`) | `rico:hasBeginningDate` / `rico:hasEndDate` |
| `resources` (GND/VIAF/Wikidata) | `owl:sameAs` / `skos:exactMatch` → externe URI | `rico:hasOrHadIdentifier` + `owl:sameAs` |
| `notes` (Biogramm) | `crm:P3_has_note` bzw. `crm:E33_Linguistic_Object` (§5) | `rico:history` / `rico:descriptiveNote` |

**Rolle im Ereignis** (`antonevents.actor_id`): über das Event (§2),
`crm:P14_carried_out_by`. Spezifische Rolle via `crm:P14.1_in_the_role_of` → `crm:E55_Type`.

```turtle
a:actor/88
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

## 4. Place — Ort

**CRM:** `crm:E53_Place` · **RiC-O:** `rico:Place`.

| Anton-Feld | CRM | RiC-O |
|---|---|---|
| `label` / `name` / `city` | `crm:P1_is_identified_by` → `crm:E48_Place_Name` | `rico:hasOrHadName` → `rico:Name` |
| `type_id` | `crm:P2_has_type` → `crm:E55_Type` (`a:term/place_type/…`) | `rico:hasPlaceType` |
| `location` (Geo-Punkt, SRID 4326) | `crm:P168_place_is_defined_by` → WKT-Literal (`geo:asWKT`) | `rico:hasOrHadLocation` + `geo:asWKT` |
| `country_code_id` / `country` | `crm:P89_falls_within` → übergeordneter `crm:E53_Place` | `rico:isOrWasPartOf` |
| `resources` (Geonames) | `owl:sameAs` → Geonames-URI | `owl:sameAs` |

```turtle
a:place/12
    a crm:E53_Place ;
    crm:P1_is_identified_by [ a crm:E48_Place_Name ;
        crm:P190_has_symbolic_content "Bern"@de ] ;
    crm:P168_place_is_defined_by
        "POINT(7.4474 46.9480)"^^geo:wktLiteral ;
    crm:P89_falls_within a:place/ch ;
    owl:sameAs <https://www.geonames.org/2661552> .
```

---

## 5. Keyword & Deskriptoren (Aboutness)

Sachbegriffe und die Deskriptor-Verknüpfungen (`objects_keywords`,
`objects_places`, `actors_objects`) drücken *worüber* eine Einheit handelt aus —
**nicht** Prozesse. CRM unterscheidet hier von den Events.

| Anton | CRM | RiC-O |
|---|---|---|
| Keyword (Entität) | `crm:E55_Type` bzw. `crm:E89_Propositional_Object` (alt: `skos:Concept`) | `rico:concept` / `skos:Concept` |
| `keywords` (objects_keywords) | Objekt `crm:P67_refers_to` Keyword, **oder** `crm:P129_is_about` | `rico:hasOrHadSubject` |
| `places_descriptors` (objects_places) | Objekt `crm:P129_is_about` → `crm:E53_Place` | `rico:hasOrHadSubject` → `rico:Place` |
| `actors_descriptors` (actors_objects) | Objekt `crm:P129_is_about` → Actor | `rico:hasOrHadSubject` → `rico:Agent` |

> Wichtig: `actors_objects` (Deskriptor, „Objekt handelt von Person X") ist
> getrennt von `antonevents.actor_id` (Prozess, „Person X hat erstellt").

---

## 6. Note — typisierter mehrsprachiger Text

Polymorph an Objekt/Actor/Place/Keyword (`model_type`/`model_id`), Typ via
`note_type_id`, mit `locale`. Mehrsprachig & typisiert → eigene Ressource:

**CRM:** `crm:E33_Linguistic_Object` · **RiC-O:** `rico:descriptiveNote` / spezifische Properties.

```turtle
a:note/777
    a crm:E33_Linguistic_Object ;
    crm:P2_has_type a:term/note_type/scopecontent ;
    crm:P190_has_symbolic_content "Enthält Korrespondenz 1898–1910 …"@de .

a:object/101 crm:P67i_is_referred_to_by a:note/777 .   # bzw. crm:P3_has_note (kurz)
```

| Anton `note_type` | CRM-Typ / Property | RiC-O |
|---|---|---|
| `scopecontent` | `crm:E33` Typ scopecontent (Anbindung `crm:P67i`) | `rico:scopeAndContent` |
| `extent_text` | `crm:E33` Typ extent (oder `crm:E54_Dimension`, §1) | `rico:recordResourceExtent` |
| `internal_note` | `crm:E33` Typ internal (Export-Filter `private`) | `rico:descriptiveNote` (intern) |
| Biogramm/History (Actor) | `crm:P3_has_note` am Actor | `rico:history` |

Kurze, untypisierte Texte können vereinfacht als `crm:P3_has_note` (E62 String)
mit `crm:P3.1_has_type` ausgegeben werden.

---

## 7. Media — digitales Objekt / Repräsentation

**CRM:** `crmdig:D1_Digital_Object` (Subklasse von `crm:E73`) bzw. `crm:E36_Visual_Item`.

| Anton-Feld | CRM | RiC-O |
|---|---|---|
| (Entität) | `a:media/{id} a crmdig:D1_Digital_Object` | `rico:Instantiation` |
| Anbindung an Objekt | Objekt `crm:P128_carries` / `crm:P138i_has_representation` → Media | `rico:hasInstantiation` |
| `mime_type` | `crm:P2_has_type` → `crm:E55_Type` (Format) | `rico:hasFormatType` |
| `file_name` / `name` | `crm:P1_is_identified_by` → `crm:E41_Appellation` | `rico:hasOrHadIdentifier` |
| `custom_properties.md5sum` | `crm:P1_is_identified_by` → `crm:E42_Identifier` (Typ md5) | `rico:hasInstantiation` / checksum |
| `size`, `pronom_id` | `crm:P43_has_dimension` / `crm:P2_has_type` | `rico:Instantiation` Attribute |
| Volltext (`MediaText`) | `crm:P165i_is_incorporated_in` → `crm:E33_Linguistic_Object` | — |

---

## 8. Relation — generische Cross-Referenz

`relations` (polymorph: source/target beliebiger Typ, `name`/`label`).

- **Bekannter Relationstyp** → spezifische CRM-Property (Lookup-Tabelle, z. B.
  „Vorgänger/Nachfolger" → `crm:P134_continued` / `crm:P134i_was_continued_by`;
  „siehe auch" → `rdfs:seeAlso`).
- **Unbekannt/generisch** → reifiziert als `crm:E13_Attribute_Assignment` oder
  `crm:P67_refers_to` mit `crm:P2_has_type` (Relationstyp-Term).
- **RiC-O:** `rico:relation` / `rico:RelationType`, bzw. spezifische
  `rico:isAssociatedWith*`-Properties.

Eine Relationstyp-Lookup-Tabelle (`relations.name` → CRM/RiC-Property) ist
Teil der Konfiguration des Exporters und wird gemeinsam gepflegt. *(TODO: befüllen
aus tatsächlich vorkommenden `relations.name`-Werten.)*

---

## 9. Term / Taxonomie — kontrolliertes Vokabular

Jeder `terms`-Eintrag → `crm:E55_Type` (zugleich `skos:Concept`), gruppiert je
`taxonomies.name` als `skos:ConceptScheme`.

```turtle
a:term/level_of_description/fonds
    a crm:E55_Type , skos:Concept ;
    skos:prefLabel "Bestand"@de , "Fonds"@fr ;
    skos:inScheme a:scheme/level_of_description .
```

`term_translations` → `skos:prefLabel` (mehrsprachig). Übergeordnete Schemata
können bei Bedarf auf externe Vokabulare (z. B. RiC-O-Wertelisten,
EAD-Levels) per `skos:exactMatch` gemappt werden.

---

## Erkenntnisse aus dem Proof of Concept

Validiert an echten Daten (`examples/fma-hs-k-30-3.ttl`, Mandant `fondazionearp`,
227 Tripel, sauber nach Turtle/JSON-LD/RDF-XML serialisierbar). Export-kritische
Punkte, die der Exporter zwingend behandeln muss:

1. **Turtle-IRIs:** `/` ist in Turtle-Präfixnamen (`a:object/1`) **nicht erlaubt**.
   Lösung: `@base` + relative IRIs (`<object/1>`), oder Präfix-Lokalnamen ohne `/`.
2. **Geo-Koordinaten:** Anton speichert `POINT(lat lon)`; GeoSPARQL-`wktLiteral`
   verlangt `POINT(lon lat)` → beim Export **tauschen**.
3. **Datums-Normalisierung:** Reale Werte enthalten `0000-00-00` (kein Datum →
   Time-Span weglassen) und Null-Monat/-Tag wie `1951-00-00` (→ auf `1951`
   bzw. `xsd:gYear` reduzieren). `date_*_ca` → Unschärfe (P81/P82), `datierung_text`
   → `crm:P3_has_note`.
4. **Brief = creation + reception:** Anton modelliert einen Brief als
   `creation`-Event (Absender → `E12_Production` / `P14`) plus ein oder mehrere
   `reception`-Events (Empfänger → `E10_Transfer_of_Custody` / `P29_custody_received_by`).
   Das ergibt sauber Sender/Empfänger ohne Spezial-Property.
5. **Authority-Links:** `resources` ist reichhaltig (GND, VIAF, Wikidata, Geonames,
   LCNAF, HLS, SIKART …). Jede Zeile → `owl:sameAs` auf die `url`-Spalte. Die
   `url`-Spalte ist maßgeblich (das eingebettete `full_json` kann abweichende IDs
   enthalten). Auswahl stabiler LOD-Provider empfehlenswert.

## Verifikationsstand der offenen Punkte

Alle ehemals offenen Punkte sind an echten Daten geklärt:

1. ✅ **Relationstypen** — `relations` kaum genutzt (nur Test-Daten); zwei Typen
   (`references` → `P67_refers_to`, `isRelatedTo` → `rdfs:seeAlso`), v. a.
   CMS-Page-Links → für archivischen Export außerhalb Scope. Details: `vocabularies.md` §7.
2. ✅ **Ereignistypen** — vollständig verifiziert (17 Typen): `event-types.md`.
3. ✅ **Rechte/Zugriff** — `private`-Filter, Notiz-Privatheit via `note_type`,
   Schutzfristen → `E30_Right`/`AccessRule`: `vocabularies.md` §5.
4. ✅ **Actor-/Objekt-/Place-/Status-Typen + Notiztypen (38)** — verifiziert:
   `vocabularies.md` §1–4, §6.
5. ✅ **Proof of Concept** — `examples/fma-hs-k-30-3.ttl` (validiert).

## Nächster Schritt

**Implementierung des Exporters in Anton selbst** (Artisan-Command in
`anton.test`, Serialisierung via EasyRdf → Turtle/JSON-LD/RDF-XML). Diese
Spezifikation (`mapping.md` + `event-types.md` + `vocabularies.md`) und das PoC
dienen als Vorgabe; die Lookups (`antonevent → CRM-Klasse`, `note_type → ISAD/CRM`)
werden als Exporter-Konfiguration übernommen.
