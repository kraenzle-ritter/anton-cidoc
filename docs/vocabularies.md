# Verifizierte Vokabulare & Rechte (Anton → CRM/RiC)

An echten Daten verifiziert (mehrere Mandanten). Ergänzt `mapping.md`.

## 1. Actor-Typen — Taxonomie `authority_type`

`actors.type_id` → `terms` der Taxonomie **`authority_type`** (nicht „actor_type").

| Anton-Term | CRM | RiC-O |
|---|---|---|
| `person` | `crm:E21_Person` | `rico:Person` |
| `family` | `crm:E74_Group` (Typ Familie) | `rico:Family` |
| `corporate_body` | `crm:E74_Group` *(ggf. `crm:E40_Legal_Body`)* | `rico:CorporateBody` |
| `organisational_unit` | `crm:E74_Group` | `rico:CorporateBody` (Untereinheit) |
| `group` | `crm:E74_Group` | `rico:Group` / `rico:Agent` |
| `software` | `crmdig:D14_Software` *(Edge-Case)* | `rico:Mechanism` |

Anton-Term jeweils zusätzlich als `crm:P2_has_type → crm:E55_Type`.

## 2. Objekt-Typen — Taxonomie `object_type` (**kundenabhängig**)

`objects.object_type_id` ist **mandantenspezifisch** (in `fondazionearp` z. B.
`file`, `manuscript`, `photo`, `object`, `print`). Sie verfeinern die
`crm:E22_Human-Made_Object` via `crm:P2_has_type → crm:E55_Type`.

**CIDOC CRM bringt kein eigenes Objekttyp-Vokabular mit** — Typen werden bewusst an
externe Vokabulare delegiert. Empfohlenes Muster für die kundenabhängige Liste:
jeder Tenant-Term bleibt ein lokales `E55_Type` (`skos:Concept` im Tenant-Schema)
und erhält **einmalig** ein `skos:exactMatch`/`skos:closeMatch` auf ein globales
Vokabular → kundenspezifisch *und* interoperabel.

Empfohlenes Zielvokabular: **Getty AAT** (`vocab.getty.edu`, CRM-/Linked-Art-
Standard, mehrsprachig). Alternativen: Wikidata, LoC LCGFT (Genre/Form), RDA
content/carrier types, RiC `DocumentaryFormType`. Für digitale Formate ist
**PRONOM** (bereits via `media.pronom_id`) das richtige Vokabular.

Verifizierte AAT-Zuordnung der beobachteten Terme:

| Anton `object_type` | Getty AAT | URI |
|---|---|---|
| `photo` | photographs | `http://vocab.getty.edu/aat/300046300` |
| `print` | prints | `http://vocab.getty.edu/aat/300041273` |
| `manuscript` | manuscripts | `http://vocab.getty.edu/aat/300028569` |
| `file` | documents *(generisch; „file" ist eher Verzeichnungsstufe)* | `http://vocab.getty.edu/aat/300026030` |
| `object` | *(generisch — `E22` ohne AAT)* | — |

> Die Alignment-Tabelle (Tenant-Term → AAT) wird **pro Mandant** gepflegt (klein,
> einmalig) und ist Teil der Exporter-Konfiguration. Dasselbe Muster eignet sich
> für `keyword_type` und Notiztypen. **Nicht** für `place_type` — das ist bereits
> ein fester Standard (GeoNames feature classes, s. §3).

Optional zusätzlich spezifischere CRM-Klassen: `photo` → `crm:E38_Image`,
`manuscript`/`print` → zusätzlich `crm:E31_Document`.

## 3. Place-Typen — Taxonomie `place_type` = **GeoNames feature classes**

Die Einbuchstaben-Terme sind **nicht** kundenspezifisch, sondern die
**GeoNames feature classes** (mandantenunabhängig, fester Standard):

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

Zielvokabular daher direkt **GeoNames-Ontologie** (`gn:`, als Linked Data) — kein
per-Tenant-Alignment nötig. Mapping: `crm:P2_has_type → crm:E55_Type`, wobei der
E55-Term `skos:exactMatch` auf die GeoNames-Feature-Class-URI trägt; optional
zusätzlich für GeoNames-native Consumer `gn:featureClass` direkt am `E53_Place`.

```turtle
@prefix gn: <http://www.geonames.org/ontology#> .

<place/5> a crm:E53_Place ;
    crm:P2_has_type <term/place_type/P> ;
    gn:featureClass gn:P .                       # populated place

<term/place_type/P> a crm:E55_Type , skos:Concept ;
    skos:prefLabel "Ort (populated place)"@de ;
    skos:exactMatch gn:P .
```

`keyword_type` ist in den geprüften Mandanten leer.

## 4. Beschreibungs-Status (administrativ)

- `status_of_description`: `draft`, `final`
- `detail_of_description`: `minimal`, `partial`, `full`

→ RiC-O `rico:descriptionStatus` / Vollständigkeitsgrad, oder `crm:P2_has_type` an
der Beschreibung. Für öffentlichen Export i. d. R. weglassen (nur `final`).

## 5. Rechte & Zugriff (Export-kritisch)

| Quelle | Bedeutung | Mapping / Regel |
|---|---|---|
| `objects.private` (1 = privat) | Datensatz nicht öffentlich | **Export-Filter**: `private=1` ausschließen; `NULL`/`0` = öffentlich |
| `media.private_media` | Medium nicht öffentlich | analog filtern |
| `actors/places/keywords.private` | Authority privat | analog filtern |
| Notiz mit `note_type=internal_note` | private Notiz (notes hat **keine** `private`-Spalte!) | private Notiztypen ausschließen |
| `period_of_protection` (`public`, `standard`, `prolonged`, `standard_10`, `standard_20`, `forever`) | Schutzfrist-Regime | `crm:E30_Right` / `rico:AccessRule` |
| `release_year` / `expiry_year` (Notiz) | Ablauf der Schutzfrist | `crm:E30_Right` + Zeitangabe; RiC `rico:Rule` |

Modellierung Schutzfrist (CRM): Objekt `crm:P104_is_subject_to → crm:E30_Right`
(Typ = period_of_protection-Term), Fristende als zugeordnete Zeitangabe.
RiC-O: `rico:regulatedBy → rico:Rule` (`rico:AccessRule`) mit Geltungsdatum.

## 6. Notiztypen — Taxonomie der `antonevent`-losen `antonfields`

**38 Typen** mandantenübergreifend; überwiegend direkt aus **ISAD(G)/EAD**.
Generische Regel (gilt für alle): Notiz → `crm:E33_Linguistic_Object`
(`crm:P190_has_symbolic_content` + `@locale`, `crm:P2_has_type` = Notiztyp),
angebunden via `crm:P67i_is_referred_to_by` am beschriebenen Objekt/Actor.

Wesentliche Typen mit ISAD-/Semantik-Bezug:

| Anton `note_type` | ISAD(G) / Semantik | Sonderbehandlung |
|---|---|---|
| `scopecontent` | 3.3.1 Enthält/Darin | — |
| `bioghist` | 3.2.2 Verwaltungs-/Biografiegeschichte | gehört zum **Actor** (`rico:history` / `crm:P3` am Actor) |
| `custod_hist` | 3.2.3 Bestandsgeschichte | ggf. `crm:E10_Transfer_of_Custody` |
| `acqinfo` | 3.2.4 Abgebende Stelle | ggf. `crm:E8_Acquisition` |
| `provenance` | Provenienz | ggf. `crm:E10` |
| `appraisal` | 3.3.2 Bewertung/Kassation | — |
| `accruals` | 3.3.3 Neuzugänge | — |
| `arrangement` | 3.3.4 Ordnung/Klassifikation | — |
| `condition_of_access` | 3.4.1 Zugangsbestimmungen | **Rechte** (§5) → `E30_Right` |
| `condition_of_reproduction` | 3.4.2 Reproduktionsbestimmungen | **Rechte** (§5) |
| `language_script` | 3.4.3 Sprache/Schrift | ergänzt `objects_languages` |
| `physical_description` | 3.4.4 Physische Beschaffenheit | — |
| `finding_aids` | 3.4.5 Findmittel | — |
| `related_units` | 3.5.3 Verwandte Verzeichnungseinheiten | ggf. `crm:P67_refers_to` |
| `publications` | 3.5.4 Veröffentlichungen | — |
| `extent_text` | 3.1.5 Umfang | alt. `crm:E54_Dimension` |
| `general_note` | 3.6.1 Allgemeine Anmerkung | — |
| `internal_note`, `archivists_notes`, `comment` | interne Notizen | **privat** → Export-Filter (§5) |
| `original_inscription_*`, `image_caption`, `original_template` | objektspezifische Inschriften/Captions | — |
| `expiry_year`, `vacat_detail`, `date_comment` | Fristen/Datierungs-Detail | `expiry_year` → Rechte (§5) |

Vollständige Liste (38): accruals, acqinfo, appraisal, archivists_notes,
arrangement, bioghist, comment, condition_of_access, condition_of_reproduction,
custod_hist, date_comment, description, exhibitions, expiry_year, extent_text,
finding_aids, general_note, image_caption, internal_note, language_script,
location_details, original_inscription_manuscript, original_inscription_print,
original_template, parcel_number, physical_description, principal, provenance,
pub_place, publications, publisher, related_units, reproductions, rules_note,
scopecontent, sources, vacat_detail.

## 7. Relationen — Taxonomie/Tabelle `relations`

Kaum genutzt (in den geprüften Mandanten nur Test-Daten). Zwei Typen beobachtet:

| `relations.name` | CRM | Bemerkung |
|---|---|---|
| `references` | `crm:P67_refers_to` | — |
| `isRelatedTo` | `rdfs:seeAlso` *(generisch)* | — |

`source_type` ist meist `page` (CMS-Seite der Website/Edition), nicht ein
archivisches Objekt → für den **archivischen** RDF-Export weitgehend außerhalb des
Scopes. Bei Bedarf später erweitern.
