# Event types: Anton (`antonevent`) → CIDOC CRM

Verified against real data (`antonfields.type = 'antonevent'`, multiple tenants).

## Principle

CIDOC CRM contains **no** controlled list of event types, but a **class
hierarchy** of events/activities. Anton's event type becomes a
`crm:P2_has_type → crm:E55_Type` refinement *within* the appropriate CRM class.
→ Anton's vocabulary does **not** need to change; a lookup `antonevent → CRM class`
in the exporter is sufficient.

## Anton's 17 defined event types (antonfields id 1000–1015, 1099)

The "used in" column is verified across the tenants archives, kba, gosteli, be,
bahnarchiv, fondazionearp, adt, mp **and casparwolf** (printmaking — supplies the
technique-/edition-related types).

| Anton `antonevent` | used in | CRM class | object link |
|---|---|---|---|
| `creation` (1000) | many (most frequent) | `crm:E12_Production` *(alt. `E65_Creation`)* | `P108_has_produced` |
| `acquisition` (1001) | archives, adt | `crm:E8_Acquisition` | `P24_transferred_title_of` |
| `reproduction` (1002) | archives, bahnarchiv | `crm:E12_Production` (derivative) | `P108_has_produced` + `P130_shows_features_of` original |
| `digitisation` (1003) | archives, kba, … | `crmdig:D2_Digitization_Process` *(fallback `E12`)* | `L1_digitized` / `L11_had_output` → `D1` |
| `reception` (1004) | kba, fondazionearp | `crm:E10_Transfer_of_Custody` | `P30_transferred_custody_of` + `P29_custody_received_by` |
| `performance` (1005) | kba | `crm:E7_Activity` *(alt. LRMoo `F31_Performance`)* | `P16_used_specific_object` |
| `provenance` (1006) | gosteli, be | `crm:E10_Transfer_of_Custody` | `P30_transferred_custody_of` |
| `loaned` (1007) | – | `crm:E10_Transfer_of_Custody` (temporary) | `P30_transferred_custody_of` |
| `preservation` (1008) | archives, gosteli | `crm:E11_Modification` | `P31_has_modified` |
| `engravation` (1009) | **casparwolf** | `crm:E12_Production` | `P108_has_produced` (+ `P32_used_general_technique`) |
| `writing` (1010) | – | `crm:E65_Creation` | `P94_has_created` |
| `coloring` (1011) | **casparwolf** | `crm:E11_Modification` | `P31_has_modified` |
| `edition` (1012) | **casparwolf** | `crm:E12_Production` *(alt. LRMoo `F30`)* | `P108_has_produced` |
| `production` (1013) | **casparwolf** | `crm:E12_Production` | `P108_has_produced` |
| `text_author` (1014) | **casparwolf** | `crm:E65_Creation` | `P94_has_created` (+ `P14.1_in_the_role_of` author) |
| `ingest` (1015) | archives, kba | `crm:E7_Activity` / `crm:E10` (administrative) | `P16` or `P30` |
| `other` (1099) | **casparwolf** | `crm:E7_Activity` (+ `P2_has_type`) | `P16_used_specific_object` / `P12_occurred_in_the_presence_of` |

Common properties of all events:
`P14_carried_out_by` → actor · `P7_took_place_at` → place ·
`P4_has_time-span` → `E52_Time-Span` (date) · `P3_has_note` → `event_details`.

## Relevant CRM event classes (reference)

```
E2 Temporal Entity
└─ E4 Period
   └─ E5 Event
      ├─ E63 Beginning of Existence ──── P92 brought into existence
      │   ├─ E12 Production (also ⊂E11) ─ P108 has produced
      │   ├─ E65 Creation ─────────────── P94 has created
      │   ├─ E66 Formation ────────────── P95 has formed (E74 Group)
      │   ├─ E67 Birth ────────────────── P98 brought into life
      │   └─ E81 Transformation
      ├─ E64 End of Existence ─────────── P93 took out of existence
      │   ├─ E6 Destruction ───────────── P13 destroyed
      │   ├─ E68 Dissolution ──────────── P99 dissolved
      │   └─ E69 Death ────────────────── P100 was death of
      └─ E7 Activity ──────────────────── P14 carried out by, P16 used, P2 type
         ├─ E8 Acquisition ────────────── P24 transferred title of
         ├─ E9 Move ──────────────────── P25 moved (P26 to / P27 from)
         ├─ E10 Transfer of Custody ───── P30 (P28 surrendered / P29 received)
         ├─ E11 Modification ──────────── P31 has modified
         │   ├─ E79 Part Addition / E80 Part Removal
         │   └─ E12 Production
         └─ E13 Attribute Assignment

CRMdig (extension, for digital processes):
   D2 Digitization Process (⊂E11) ─ L1 digitized, L11 had output → D1 Digital Object
```

## Recommendation for adapting Anton

1. **No data migration needed** — the lookup above in the exporter is sufficient.
2. **Capture the actor role** (author/writer/sender/recipient) → becomes
   `crm:P14.1_in_the_role_of`. For letters, "creation = sender" is otherwise only implicit.
3. **Optionally add**, if needed in practice: `destruction` (→ `E6`), `move`
   (→ `E9`) — currently not in Anton's vocabulary.
