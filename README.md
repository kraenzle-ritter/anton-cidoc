# anton-cidoc — Mapping der Anton-Daten auf CIDOC CRM & Records in Context

Ziel: ein präzises, serialisierungsneutrales **Mapping** der Anton-Daten
(`../anton.test`) auf **CIDOC CRM** (Leitmodell) mit **RiC-O**-Alignment
(Records in Context Ontology), als Grundlage für einen RDF-Export.

## Entscheidungen

| Punkt | Entscheidung | Begründung |
|---|---|---|
| Leitmodell | **CIDOC CRM 7.1.x** | breiter etabliert; objekt-/ereigniszentriert |
| Zweitmodell | **RiC-O 1.0** | archivspezifisch, CRM-kompatibles Alignment, nah an ISAD/ISAAR/ISDIAH |
| Kanon-Serialisierung | **RDF/Turtle** | am lesbarsten; Spec-Beispiele in Turtle |
| Weitere Ausgaben | **JSON-LD, RDF/XML** | aus demselben Graphen verlustfrei erzeugbar |
| Export-Architektur | Graph **einmal** bauen → in alle Formate serialisieren | z. B. EasyRdf in Laravel |

## Kompatibilitäts-Einschätzung

**Gesamturteil: hoch.** Antons Datenmodell passt überdurchschnittlich gut auf
CIDOC CRM und noch besser auf RiC-O — **ohne Umbau von Anton**. Der Aufwand liegt
in *Konfiguration* (Lookups, Vokabular-Alignment) und *Normalisierung*
(Daten-Quirks), nicht in einer Datenmigration.

**Was die Kompatibilität trägt**
- **Ereignis-Zentrierung:** `antonevents` (*Object × Actor × Place × Datum × Typ*)
  ist strukturell schon das, was CRM erzwingt (E5/E7/E12/E65 …) und entspricht
  RiC-Os Activity/Agent-Modell. Die meisten Systeme müssen Events erst künstlich
  rekonstruieren — Anton nicht.
- **Hierarchie:** Closure-Table → `P46`/`P46i` verlustfrei.
- **Authority + Normdaten:** `actors`/`places`/`keywords` mit `resources`
  (GND/VIAF/Geonames) → `owl:sameAs`, sofort Linked-Open-Data-fähig.
- **Orte besonders sauber:** Identität via GeoNames-URI **und** Typ via
  `place_type` = GeoNames feature classes (`gn:`) — beides standardgebunden,
  ganz ohne Pflegeaufwand.
- **Mehrsprachigkeit** (JSON-Labels) und **typisierte Notizen** (38 Typen ≈
  ISAD(G)/EAD) übersetzen sauber.

**Reibungspunkte (lösbar, aber bewusst zu entscheiden)**
- **Klassenwahl `E22` vs. `E73`:** Anton trennt physischen Träger und geistigen
  Inhalt nicht — Export muss sich festlegen (Empfehlung: `E22`, born-digital
  zusätzlich `crmdig:D1`).
- **`object_type` kundenabhängig:** braucht ein **per-Mandant**-Alignment auf
  Getty AAT — Pflegeaufwand, keine Modellfrage. (Nur dieser Typ; `place_type`
  ist standardgebunden, s. o.)
- **Daten-Hygiene:** Datums-Quirks (`0000-00-00`, Null-Monat) und Geo-Achsen
  (`POINT(lat lon)` → `(lon lat)`) müssen normalisiert werden, sonst syntaktisch
  valides, aber semantisch falsches RDF.
- **Schwach modelliert:** Rechte/Schutzfristen und `relations` sind in CRM/RiC
  reicher darstellbar als aktuell in Anton geführt → Export eher grob.

**CRM vs. RiC-O**
- **RiC-O ist der engere Fit** (eigens für Archive, direkte ISAD/ISAAR/ISDIAH-
  Abstammung) — Verzeichnungsstufen, Agenten und Beziehungen nahezu 1:1.
- **CRM ist allgemeiner** und damit verbreiteter, bei rein archivischen
  Konstrukten etwas umständlicher (Stufen nur via `P2_has_type`). Da RiC-O ein
  offizielles CRM-Alignment mitbringt, sind beide **kohärent gemeinsam**
  exportierbar — kein Widerspruch.

## Relevante Anton-Entitäten (Quellseite)

| Anton-Modell | Tabelle | Rolle |
|---|---|---|
| AntonObject | `objects` | Verzeichnungseinheit (hierarchisch, closure table) |
| AntonEvent | `antonevents` | Ereignis (Object↔Actor↔Place↔Datum↔Typ) |
| Actor | `actors` | Person / Körperschaft / Familie |
| Place | `places` | Ort (inkl. Geo-Punkt, Country) |
| Keyword | `keywords` | Sachbegriff / Deskriptor |
| Note | `notes` | typisierter, mehrsprachiger Text (polymorph) |
| Relation | `relations` | generische Cross-Referenz (polymorph) |
| Media | `media` | digitales Objekt / Repräsentation |
| Location | `locations` | physischer Aufbewahrungsort |
| Term / Taxonomie | `terms` / `taxonomies` | kontrolliertes Vokabular |

> Die Tabellen `isads` / `isaars` / `isdiahs` sind nur Definitionen der ICA-Standards
> selbst (Referenz), **keine** Records — für das Mapping irrelevant.

Pivot-/Deskriptor-Verknüpfungen: `actors_objects`, `objects_places`,
`objects_keywords`, `objects_languages` (Aboutness/Deskriptoren, im Gegensatz zu
den prozessualen `antonevents`).

## Dateien

- [`docs/mapping.md`](docs/mapping.md) — die eigentliche Mapping-Spezifikation, Entität für Entität
- [`docs/event-types.md`](docs/event-types.md) — alle 17 Anton-Ereignistypen → CRM-Klassen (verifiziert)
- [`docs/vocabularies.md`](docs/vocabularies.md) — verifizierte Vokabulare (Actor-/Objekt-/Notiztypen, Rechte/Zugriff, Relationen) + AAT-Alignment
- [`examples/fma-hs-k-30-3.ttl`](examples/fma-hs-k-30-3.ttl) — Proof of Concept aus echten Daten
  (Mandant `fondazionearp`, Brief Janco→Arp): Fonds-Hierarchie, Produktions-/Empfangsereignis,
  Actors mit Authority-Links, Ort mit Geo, typisierte Notizen, digitales Medium. Validiert
  (227 Tripel, parst nach Turtle/JSON-LD/RDF-XML).
- `docs/namespaces.md` — verwendete Namespaces & URI-Strategie *(folgt)*
