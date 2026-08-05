# Design- und Lastenheft: Mechanicus Quote Database

**Projektname:** Mechanicus Quote Database  
**Kurzname:** MQD  
**Version:** 1.0 Konzept und Lastenheft  
**Stand:** 2026-05-28  
**Zielplattform:** JSON/YAML-first Datenbasis mit optionaler SQLite- und API-Erweiterung  
**Primärer Zweck:** Maschinenlesbare, gut kuratierte Quote-, Litany-, Hymn-, Creed- und Excerpt-Datenbank aus den Seiten „Cult Mechanicus Religious Excerpts“ und „Adeptus Mechanicus Quotes“ von Lexicanum.

> Hinweis: Dieses Dokument beschreibt Struktur, Datenmodell, Qualitätsregeln, Importlogik und spätere Nutzung. Es ist bewusst nicht als vollständiger Abdruck aller Quotes formuliert, sondern als umsetzbares Design für eine Datenbank, die diese Inhalte strukturiert verwaltet.

---

## 1. Ausgangslage

Zwei Lexicanum-Seiten liefern eine größere Sammlung von Adeptus-Mechanicus- und Cult-Mechanicus-Zitaten, religiösen Exzerpten, Litaneien und Sprüchen:

1. `Cult Mechanicus Religious Excerpts`
2. `Adeptus Mechanicus Quotes`

Die Inhalte sind für eine direkte maschinelle Nutzung nur bedingt geeignet, weil sie in Wiki-Strukturen vorliegen, teils tabellarisch, teils abschnittsweise, teils mit Quellenfußnoten, teils mit unklarer oder quarantänierter Quellenlage.

Ziel dieses Projekts ist es, daraus eine robuste Datenbank zu entwerfen, die für Menschen gut lesbar bleibt und gleichzeitig von Agenten, Bots, UIs, Random-Quote-Systemen, RAG-Pipelines und thematischen Persona-Systemen zuverlässig verarbeitet werden kann.

---

## 2. Zielbild

Die Datenbank soll als kanonische, textbasierte Wissensbasis funktionieren:

```text
mechanicus-quote-db/
├── README.md
├── LICENSE_NOTES.md
├── DATA_POLICY.md
├── sources.yaml
├── tags.yaml
├── schema/
│   ├── quote_entry.schema.json
│   ├── source_entry.schema.json
│   └── tag_registry.schema.json
├── data/
│   ├── quotes.yaml
│   ├── quotes.json
│   ├── excerpts.yaml
│   ├── excerpts.json
│   └── generated/
│       ├── quotes.normalized.json
│       ├── quotes.readable.md
│       └── quotes.sqlite
├── imports/
│   ├── lexicanum_cult_mechanicus_religious_excerpts.raw.html
│   ├── lexicanum_adeptus_mechanicus_quotes.raw.html
│   └── import_report.json
├── tools/
│   ├── validate_quotes.py
│   ├── build_json_from_yaml.py
│   ├── build_sqlite.py
│   ├── export_markdown.py
│   └── lint_taxonomy.py
└── docs/
    ├── design_lastenheft.md
    ├── data_dictionary.md
    ├── import_pipeline.md
    └── usage_patterns.md
```

YAML ist das primäre Pflegeformat. JSON ist das primäre Austauschformat. SQLite ist optional, aber für Suche, Filter und spätere Bot-/API-Nutzung vorgesehen.

---

## 3. Leitprinzipien

### 3.1 YAML-first, JSON-guaranteed

Die Daten werden manuell oder halbautomatisch in YAML gepflegt, weil YAML für Menschen angenehmer lesbar ist. Jede YAML-Datei muss automatisch in valides JSON konvertierbar sein.

### 3.2 Jede Aussage braucht Herkunft

Jeder Quote- oder Excerpt-Datensatz muss eine nachvollziehbare Quelle haben. Die Quelle darf mehrstufig sein:

- Quellseite: Lexicanum-URL
- Lexicanum-Seitenversion oder Abrufdatum
- In-Lore-Quelle, z. B. Aphorisms, Codex, Hymn, Litany, Character
- Publikationsquelle, z. B. Codex, Rulebook, Novel, Magazine
- Seitenzahl, Kapitel oder Fußnotenreferenz, falls vorhanden

### 3.3 Unklare Quellen werden nicht versteckt

Eintragstypen mit unsicherer Quellenlage werden nicht gelöscht, aber markiert. Das System kennt bewusst Qualitätsstatus wie `verified`, `lexicanum_cited`, `lexicanum_uncited`, `quarantined`, `needs_review`, `duplicate_candidate`.

### 3.4 Keine flache Zitatliste

Die Datenbank soll nicht nur Text speichern. Sie soll Bedeutung speichern:

- Sprecher oder Ursprung
- Kontext
- Liturgischer Typ
- Thema
- Tonalität
- Nutzungsfälle
- Trigger-Kontexte
- Eignung für Agentenantworten
- Eignung für UI-Ausgabe
- Quellenqualität

### 3.5 Erweiterbar ohne Datenbruch

Das Modell muss später problemlos zusätzliche Quellen, deutsche Übersetzungen, eigene Paraphrasen, Audio-Metadaten, persona-spezifische Nutzung und Embeddings aufnehmen können.

---

## 4. Begriffe und Datentypen

### 4.1 Quote

Ein prägnanter Ausspruch, meist von einem Sprecher, Text oder unbekanntem Ursprung. Beispielhafte Herkunftstypen:

- Aphorism
- Tech-Priest
- Magos
- Skitarii
- Fabricator General
- Unknown Author
- Common Litany

### 4.2 Excerpt

Längerer Auszug aus einem religiösen, technischen oder liturgischen Text. Excerpts können mehrzeilig sein und enthalten häufig Anweisungen, Refrains oder rituelle Struktur.

### 4.3 Litany

Ritualisierte, oft wiederholbare Formel. Für spätere Agenten- und UI-Nutzung besonders wertvoll, weil sie gut mit Systemereignissen gemappt werden kann.

### 4.4 Creed

Dogmatische Glaubensformel. Eignet sich für Persona-Kern, System-Prompts, Kapiteltrenner und starke thematische Marker.

### 4.5 Invocation

Ruf, Startformel oder Aktivierungsritual. Besonders geeignet für Events wie Boot, Deploy, Task Start, Container Start, Service Recovery.

### 4.6 Source

Strukturierte Quellenangabe. Nicht identisch mit `origin`. Ein Quote kann als Ursprung `Aphorisms 56` haben und als Publikationsquelle einen Codex oder ein Lexicanum-Zitat.

---

## 5. Primäre Datenobjekte

### 5.1 QuoteEntry

Ein `QuoteEntry` ist der zentrale Datensatz.

```yaml
id: MQD-AMQ-000001
record_type: quote
canonical_text: "<Originaltext>"
text_variants: []
normalized_text: "<Normalisierte Fassung für Suche>"
language: en
source_page: adeptus_mechanicus_quotes
source_page_url: "https://wh40k.lexicanum.com/wiki/Adeptus_Mechanicus_Quotes"
lexicanum_section: "Attributed > Texts > Aphorisms"
lexicanum_anchor: null
lexicanum_retrieved_at: "2026-05-28"
lexicanum_oldid: "758807"
origin:
  origin_type: text
  name: "Aphorisms"
  reference: "56"
  speaker: null
  speaker_role: null
  faction: "Adeptus Mechanicus"
publication_source:
  title: "<Publikationsquelle, falls vorhanden>"
  edition: null
  page: null
  chapter: null
  footnote_ref: "[1]"
classification:
  category: aphorism
  subcategory: doctrine
  is_liturgical: false
  is_prayer: false
  is_battle_related: false
  is_machine_spirit_related: false
  attribution_status: attributed
  source_quality: lexicanum_cited
semantic_tags:
  - xenos
  - doctrine
  - corruption
  - vigilance
style_tags:
  - doctrinal
  - terse
  - warning
usage:
  suitable_for:
    - agent_warning
    - loading_screen
    - lore_database
  trigger_contexts:
    - threat_detected
    - unknown_external_input
    - security_warning
  intensity: 3
  humour_level: 0
  persona_fit:
    techpriest: 5
    skitarii: 4
    magos: 4
rights:
  content_origin: third_party_copyrighted
  use_policy: reference_only
  reproduction_scope: local_private_database
review:
  status: needs_human_review
  reviewer: null
  reviewed_at: null
  notes: null
```

### 5.2 SourceEntry

Quellen werden getrennt gespeichert, damit Datensätze sauber referenzieren können.

```yaml
id: SRC-LEX-AMQ
kind: web_page
title: "Adeptus Mechanicus Quotes"
url: "https://wh40k.lexicanum.com/wiki/Adeptus_Mechanicus_Quotes"
site: "Warhammer 40k - Lexicanum"
retrieved_at: "2026-05-28"
oldid: "758807"
page_status:
  has_quarantine_notice: true
  has_uncited_content_warning: true
  has_hidden_citation_category: false
notes: "Große Quote-Sammlung mit Attributed/Unattributed-Bereichen und umfangreichem Quellenapparat."
```

### 5.3 TagEntry

Tags werden nicht frei wildwuchernd vergeben, sondern in einer Registry gepflegt.

```yaml
id: TAG-MACHINE-SPIRIT
name: machine_spirit
label: "Machine Spirit"
description: "Bezüge auf Machine Spirits, Maschinenverehrung, technische Beseeltheit oder rituelle Gerätepflege."
category: theme
aliases:
  - machine-god-device
  - spirit_of_machine
allowed_for:
  - quote
  - excerpt
  - litany
```

---

## 6. Pflichtfelder

Jeder Datensatz in `quotes.yaml` oder `excerpts.yaml` muss mindestens folgende Felder besitzen:

```yaml
id: string
record_type: quote | excerpt | litany | hymn | creed | invocation | prayer | chant
canonical_text: string
language: string
source_page: string
source_page_url: string
lexicanum_retrieved_at: date
origin:
  origin_type: string
  name: string | null
classification:
  category: string
  attribution_status: attributed | unattributed | unknown | mixed
  source_quality: verified | lexicanum_cited | lexicanum_uncited | quarantined | needs_review
semantic_tags: list[string]
style_tags: list[string]
usage:
  suitable_for: list[string]
review:
  status: draft | needs_human_review | reviewed | rejected
```

---

## 7. Optionale Felder

```yaml
text_variants: list[string]
normalized_text: string
translation:
  de: string | null
  de_style: literal | adapted | liturgical_rewrite | null
source_lineage:
  lexicanum_oldid: string | null
  source_footnotes: list[string]
  publication_candidates: list[string]
origin:
  speaker: string | null
  speaker_role: string | null
  faction: string | null
  forge_world: string | null
publication_source:
  title: string | null
  edition: string | null
  page: string | null
  chapter: string | null
  footnote_ref: string | null
audio:
  cadence: calm | solemn | aggressive | binharic | sermon | chant | null
  voice_hint: string | null
ui:
  short_display: string | null
  preferred_breaks: list[string]
  max_safe_line_length: integer | null
rag:
  embedding_ready: boolean
  summary: string | null
  keywords: list[string]
rights:
  content_origin: third_party_copyrighted | user_generated | transformed | public_domain_unknown
  use_policy: reference_only | private_use | generated_paraphrase_only
review:
  reviewer: string | null
  reviewed_at: date | null
  notes: string | null
```

---

## 8. ID-Konzept

IDs müssen stabil, sprechend und sortierbar sein.

### 8.1 Präfixe

```text
MQD-AMQ-000001     Eintrag aus Adeptus Mechanicus Quotes
MQD-CMRE-000001    Eintrag aus Cult Mechanicus Religious Excerpts
SRC-LEX-AMQ        Source: Lexicanum Adeptus Mechanicus Quotes
SRC-LEX-CMRE       Source: Lexicanum Cult Mechanicus Religious Excerpts
TAG-...            Tag Registry
```

### 8.2 Regeln

- IDs werden niemals wiederverwendet.
- Entfernte Einträge bekommen `review.status: rejected` oder `deprecated`, werden aber nicht still gelöscht.
- Doppelte Inhalte werden über `duplicate_of` oder `related_records` verknüpft.
- Wenn derselbe Quote einmal attributed und einmal unattributed auftaucht, bleibt die besser belegte Variante kanonisch.

---

## 9. Klassifikation

### 9.1 Kategorien

Empfohlene `classification.category` Werte:

```yaml
- aphorism
- battle_cry
- catechism
- chant
- creed
- excerpt
- hymn
- invocation
- litany
- mantra
- mourning_rite
- prayer
- prophecy
- technical_rite
- warning
- doctrinal_statement
- character_quote
- unknown_quote
```

### 9.2 Source Quality

```yaml
verified:
  meaning: "Gegen Primär- oder Sekundärquelle geprüft."
lexicanum_cited:
  meaning: "Lexicanum nennt Quelle/Fußnote, aber Primärquelle wurde noch nicht selbst geprüft."
lexicanum_uncited:
  meaning: "Lexicanum-Eintrag ohne belastbare Quellenangabe."
quarantined:
  meaning: "Lexicanum markiert oder diskutiert Inhalt als problematisch/ausgelagert."
needs_review:
  meaning: "Importiert, aber noch nicht redaktionell geprüft."
duplicate_candidate:
  meaning: "Möglicher Dublettenfund."
```

### 9.3 Attribution Status

```yaml
attributed:
  meaning: "Sprecher, Werk oder Ursprung erkennbar."
unattributed:
  meaning: "Unbekannter Autor oder Unknown."
mixed:
  meaning: "Mehrteiliger Eintrag mit teils Sprecher, teils Kommentar."
unknown:
  meaning: "Attribution nicht sauber extrahierbar."
```

---

## 10. Taxonomie der Tags

### 10.1 Thematische Tags

```yaml
- omnissiah
- machine_god
- machine_spirit
- flesh_weakness
- steel
- knowledge
- secrecy
- logic
- faith
- purity
- corruption
- xenos
- heresy
- fire
- war
- duty
- sacrifice
- augmentation
- cybernetics
- forge
- titan
- skitarii
- servitor
- noosphere
- binharic
- data
- weapon
- engine
- repair
- awakening
- maintenance
- mourning
- efficiency
- triumph
- annihilation
```

### 10.2 Stil-Tags

```yaml
- doctrinal
- liturgical
- martial
- mystical
- technical
- ritualistic
- terse
- sermon
- poetic
- warning
- humorous_dark
- solemn
- fanatical
- analytical
- triumphal
```

### 10.3 Nutzungstags

```yaml
- agent_startup
- agent_shutdown
- system_boot
- deployment_start
- deployment_success
- deployment_failure
- docker_start
- service_restart
- network_recovery
- security_warning
- error_message
- loading_screen
- dashboard_banner
- telegram_bot_reply
- mission_control_log
- fpv_workshop_flavor
- led_matrix_display
- rpg_handout
- persona_prompt_seed
```

---

## 11. Import- und Normalisierungsprozess

### 11.1 Stufe 1: Rohdaten sichern

Die zwei Quellseiten werden als HTML und optional als Markdown/Text gesichert.

Ergebnis:

```text
imports/lexicanum_adeptus_mechanicus_quotes.raw.html
imports/lexicanum_cult_mechanicus_religious_excerpts.raw.html
imports/import_metadata.yaml
```

`import_metadata.yaml`:

```yaml
retrieved_at: "2026-05-28"
sources:
  - id: SRC-LEX-AMQ
    url: "https://wh40k.lexicanum.com/wiki/Adeptus_Mechanicus_Quotes"
    oldid: "758807"
  - id: SRC-LEX-CMRE
    url: "https://wh40k.lexicanum.com/wiki/Cult_Mechanicus_Religious_Excerpts"
    oldid: "745013"
```

### 11.2 Stufe 2: Parserlauf

Der Parser erkennt:

- Tabellenähnliche Struktur
- Abschnittsüberschriften
- Sprecherzeilen
- Fußnotenreferenzen
- In-Lore-Textnamen
- Excerpt-Marker
- mehrzeilige Textblöcke
- bekannte Source-Verweise

### 11.3 Stufe 3: Rohdatensatz erzeugen

Der Parser legt zunächst `draft` Datensätze an:

```yaml
review:
  status: draft
classification:
  source_quality: needs_review
```

### 11.4 Stufe 4: Normalisierung

Normalisierung umfasst:

- Whitespace vereinheitlichen
- Zeilenumbrüche bei Litaneien erhalten
- Wiki-Fußnoten getrennt speichern
- typografische Apostrophe konsistent behandeln
- eindeutige Sprecher vom Zitattext trennen
- Quellenkommentare vom eigentlichen Quote trennen
- Mehrfachvorkommen erkennen

### 11.5 Stufe 5: Redaktionelle Prüfung

Jeder Datensatz bekommt Review-Status:

```yaml
review:
  status: reviewed
  reviewer: "<Name>"
  reviewed_at: "YYYY-MM-DD"
```

### 11.6 Stufe 6: Export

Aus YAML werden erzeugt:

```text
data/quotes.json
data/excerpts.json
data/generated/quotes.normalized.json
data/generated/quotes.readable.md
data/generated/quotes.sqlite
```

---

## 12. Qualitätsregeln

### 12.1 Validierung

Ein Datensatz ist nur gültig, wenn:

- `id` eindeutig ist.
- `canonical_text` nicht leer ist.
- `source_page_url` vorhanden ist.
- `classification.category` aus erlaubtem Vokabular stammt.
- `classification.source_quality` gesetzt ist.
- alle Tags in `tags.yaml` registriert sind.
- keine ungeprüften Dubletten unmarkiert bleiben.

### 12.2 Dublettenprüfung

Dubletten werden mit drei Methoden erkannt:

1. Exakte Textübereinstimmung
2. Normalisierte Textübereinstimmung
3. Fuzzy Matching ab Schwellwert, z. B. 92 Prozent Ähnlichkeit

Dublettenfeld:

```yaml
duplicate:
  duplicate_of: "MQD-AMQ-000075"
  duplicate_reason: "same_text_different_attribution"
  keep_as_variant: true
```

### 12.3 Lesbarkeitsprüfung

Für UI-Ausgaben werden Zeilenlängen und Blockstruktur erfasst:

```yaml
ui:
  short_display: "<gekürzte Fassung oder erster Satz>"
  preferred_breaks:
    - "after_each_verse"
    - "before_response_line"
  max_safe_line_length: 80
```

### 12.4 Review Flags

```yaml
review_flags:
  - needs_source_confirmation
  - possible_misattribution
  - possible_duplicate
  - long_excerpt
  - contains_editorial_note
  - needs_line_break_review
  - source_page_has_quarantine_notice
```

---

## 13. Maschinenlesbare Ausgabeformate

### 13.1 YAML als kanonische Pflegequelle

Vorteile:

- leicht lesbar
- gut für Git-Diffs
- Kommentare möglich
- angenehm für manuelle Kuration
- ideal für strukturierte Lore-Daten

### 13.2 JSON als technische Wahrheit für Programme

Vorteile:

- API-freundlich
- validierbar gegen JSON Schema
- kompatibel mit Web-UIs, Bots und Agenten
- direkt in TypeScript/Python nutzbar

### 13.3 SQLite als optionaler Suchindex

Vorteile:

- schnelle Volltextsuche
- einfache Filter
- lokal und robust
- gut für Telegram-Bots oder kleine Web-UIs

Empfohlene SQLite-Tabellen:

```sql
CREATE TABLE quote_records (
    id TEXT PRIMARY KEY,
    record_type TEXT NOT NULL,
    canonical_text TEXT NOT NULL,
    normalized_text TEXT,
    language TEXT NOT NULL,
    source_page TEXT NOT NULL,
    source_page_url TEXT NOT NULL,
    source_quality TEXT NOT NULL,
    attribution_status TEXT NOT NULL,
    category TEXT NOT NULL,
    origin_name TEXT,
    speaker TEXT,
    faction TEXT,
    review_status TEXT NOT NULL
);

CREATE TABLE quote_tags (
    quote_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    tag_type TEXT NOT NULL,
    FOREIGN KEY (quote_id) REFERENCES quote_records(id)
);

CREATE TABLE quote_usage_contexts (
    quote_id TEXT NOT NULL,
    usage_context TEXT NOT NULL,
    FOREIGN KEY (quote_id) REFERENCES quote_records(id)
);

CREATE TABLE sources (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    url TEXT NOT NULL,
    retrieved_at TEXT,
    oldid TEXT,
    notes TEXT
);
```

---

## 14. JSON Schema Kern

Das eigentliche Schema wird separat als `quote_entry.schema.json` gepflegt. Minimalkonzept:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.local/mechanicus-quote-db/schema/quote_entry.schema.json",
  "title": "Mechanicus Quote Entry",
  "type": "object",
  "required": [
    "id",
    "record_type",
    "canonical_text",
    "language",
    "source_page",
    "source_page_url",
    "lexicanum_retrieved_at",
    "origin",
    "classification",
    "semantic_tags",
    "style_tags",
    "usage",
    "review"
  ],
  "properties": {
    "id": { "type": "string", "pattern": "^MQD-[A-Z]+-[0-9]{6}$" },
    "record_type": {
      "type": "string",
      "enum": ["quote", "excerpt", "litany", "hymn", "creed", "invocation", "prayer", "chant"]
    },
    "canonical_text": { "type": "string", "minLength": 1 },
    "language": { "type": "string", "enum": ["en", "de", "la", "binharic_unknown"] },
    "source_page": { "type": "string" },
    "source_page_url": { "type": "string", "format": "uri" },
    "lexicanum_retrieved_at": { "type": "string", "format": "date" },
    "origin": { "type": "object" },
    "classification": { "type": "object" },
    "semantic_tags": { "type": "array", "items": { "type": "string" } },
    "style_tags": { "type": "array", "items": { "type": "string" } },
    "usage": { "type": "object" },
    "review": { "type": "object" }
  }
}
```

---

## 15. Beispielhafte Datensätze

### 15.1 Beispiel Quote ohne langen Volltext

```yaml
id: MQD-AMQ-000001
record_type: quote
canonical_text: "<Original quote text from source>"
language: en
source_page: adeptus_mechanicus_quotes
source_page_url: "https://wh40k.lexicanum.com/wiki/Adeptus_Mechanicus_Quotes"
lexicanum_section: "Attributed > Texts"
lexicanum_retrieved_at: "2026-05-28"
lexicanum_oldid: "758807"
origin:
  origin_type: text
  name: "Aphorisms"
  reference: "56"
  speaker: null
  speaker_role: null
  faction: "Adeptus Mechanicus"
classification:
  category: aphorism
  attribution_status: attributed
  source_quality: lexicanum_cited
semantic_tags:
  - xenos
  - doctrine
  - vigilance
style_tags:
  - doctrinal
  - terse
  - warning
usage:
  suitable_for:
    - security_warning
    - loading_screen
    - persona_prompt_seed
  trigger_contexts:
    - unknown_external_input
    - threat_detected
  intensity: 3
  persona_fit:
    techpriest: 5
    skitarii: 4
review:
  status: needs_human_review
  notes: "Text imported from Lexicanum, primary publication not yet independently checked."
```

### 15.2 Beispiel Litany/Excerpt

```yaml
id: MQD-CMRE-000001
record_type: litany
canonical_text: |
  <Multi-line litany text from source>
source_page: cult_mechanicus_religious_excerpts
source_page_url: "https://wh40k.lexicanum.com/wiki/Cult_Mechanicus_Religious_Excerpts"
lexicanum_retrieved_at: "2026-05-28"
lexicanum_oldid: "745013"
origin:
  origin_type: religious_text
  name: "Litany of Praise"
  reference: null
classification:
  category: litany
  attribution_status: attributed
  source_quality: lexicanum_cited
semantic_tags:
  - machine_god
  - march
  - praise
  - triumph
style_tags:
  - liturgical
  - chant
  - martial
usage:
  suitable_for:
    - system_boot
    - deployment_start
    - dashboard_banner
    - led_matrix_display
  trigger_contexts:
    - agent_startup
    - mission_start
  intensity: 4
review:
  status: needs_human_review
```

---

## 16. Nutzungskonzepte

### 16.1 OpenClaw / Agent Persona

Die Datenbank kann passende Quotes anhand von Trigger-Kontexten liefern:

```yaml
trigger_contexts:
  - system_boot
  - task_completed
  - task_failed
  - file_saved
  - git_commit
  - service_restart
```

Agentenlogik:

```pseudo
on_event(event_type):
    candidates = db.find_by_trigger(event_type)
    candidates = filter_by_source_quality(candidates, min="lexicanum_cited")
    candidates = filter_by_intensity(candidates, max=event.intensity)
    return weighted_random(candidates)
```

### 16.2 Telegram Bot

Bot-Befehle:

```text
/quote random
/quote tag machine_spirit
/quote context deployment_success
/quote category litany
/quote source_status needs_review
/quote search "Omnissiah"
```

Antwortformat:

```text
<Quote>

Origin: <origin.name>
Source quality: <source_quality>
Tags: <tags>
```

### 16.3 Mission-Control Dashboard

Dashboard-Funktionen:

- Quote of the Day
- Event-basierte Ritualzeile
- Filter nach Kategorie
- Filter nach Persona-Eignung
- Review-Board für ungeprüfte Einträge
- Dubletten-Ansicht
- Quellenstatus-Matrix

### 16.4 LED Matrix / Ambient Display

Kurze Quotes können über `ui.short_display` oder `ui.max_safe_line_length` ausgewählt werden.

Filter:

```yaml
usage.suitable_for contains led_matrix_display
ui.max_safe_line_length <= 64
classification.source_quality in [verified, lexicanum_cited]
```

### 16.5 RAG / Semantic Search

Für RAG-Nutzung werden längere Excerpts mit Summaries und Keywords versehen:

```yaml
rag:
  embedding_ready: true
  summary: "Liturgical machine invocation for startup or engine activation."
  keywords:
    - engine
    - invocation
    - machine_god
    - activation
```

---

## 17. Rechte- und Nutzungshinweise

Die Inhalte stammen aus einer Warhammer-40k-Fanwiki-Quelle, die wiederum Inhalte aus Games-Workshop- und Black-Library-Kontexten referenziert. Die Datenbank sollte daher als private, lokale Referenz- und Kurationsebene betrachtet werden.

Empfehlungen:

- Keine öffentliche Volltext-Republication ohne Rechteprüfung.
- Für öffentliche Software-Repositories nur Schema, Importskripte und leere Beispieldaten bereitstellen.
- Für private lokale Nutzung können Volltexte gepflegt werden.
- Für öffentliche Demos lieber paraphrasierte, selbst geschriebene „inspired by“-Texte oder kurze, rechtearme Platzhalter verwenden.
- Jedes Datenelement erhält ein `rights` Objekt.

Beispiel:

```yaml
rights:
  content_origin: third_party_copyrighted
  use_policy: private_reference_only
  public_export_allowed: false
```

---

## 18. Lastenheft

## 18.1 Projektziel

Es soll eine strukturierte Datenbank entstehen, die alle relevanten Quotes, Litaneien, Hymnen, Exzerpte und verwandten Textformen aus den zwei genannten Lexicanum-Seiten in YAML und JSON erfasst, validiert, klassifiziert und für spätere Anwendungen nutzbar macht.

## 18.2 Muss-Anforderungen

### L-001 Kanonisches YAML

Das System muss eine kanonische YAML-Datenbasis bereitstellen.

Akzeptanzkriterien:

- `data/quotes.yaml` existiert.
- `data/excerpts.yaml` existiert oder längere Excerpts sind in `quotes.yaml` über `record_type` differenziert.
- Alle Einträge besitzen stabile IDs.
- YAML ist syntaktisch valide.

### L-002 JSON-Export

Das System muss JSON aus YAML erzeugen können.

Akzeptanzkriterien:

- JSON-Dateien werden automatisiert generiert.
- JSON besteht Schema-Validierung.
- JSON ist UTF-8-kodiert.
- Mehrzeilige Texte bleiben korrekt erhalten.

### L-003 Quellenmodell

Jeder Datensatz muss Quelleninformationen enthalten.

Akzeptanzkriterien:

- `source_page_url` ist immer gesetzt.
- `lexicanum_retrieved_at` ist immer gesetzt.
- Falls vorhanden, wird `lexicanum_oldid` gespeichert.
- Publikationsquelle/Fußnote wird gespeichert, wenn im Quellmaterial vorhanden.

### L-004 Klassifikation

Jeder Datensatz muss klassifiziert werden.

Akzeptanzkriterien:

- Kategorie ist gesetzt.
- Attribution Status ist gesetzt.
- Source Quality ist gesetzt.
- Mindestens ein semantischer Tag ist gesetzt.
- Mindestens ein Stil-Tag ist gesetzt.

### L-005 Review Workflow

Jeder Datensatz muss einen Review-Status haben.

Akzeptanzkriterien:

- `review.status` ist Pflichtfeld.
- Statuswerte sind kontrolliert.
- Unsichere Datensätze werden markiert, nicht still übernommen.

### L-006 Tag Registry

Alle Tags müssen in einer zentralen Registry definiert sein.

Akzeptanzkriterien:

- `tags.yaml` existiert.
- Validierung schlägt fehl, wenn ein nicht registrierter Tag verwendet wird.
- Tags haben Beschreibung und Kategorie.

### L-007 Lesbare Markdown-Ausgabe

Das System soll eine menschenlesbare Markdown-Ausgabe erzeugen.

Akzeptanzkriterien:

- `data/generated/quotes.readable.md` kann erzeugt werden.
- Einträge sind nach Kategorie oder Ursprung gruppiert.
- Quellenstatus wird sichtbar angezeigt.
- Lange Excerpts bleiben strukturiert.

### L-008 Dublettenbehandlung

Das System muss Dubletten erkennen und markieren.

Akzeptanzkriterien:

- Exakte Dubletten werden gefunden.
- Normalisierte Dubletten werden gefunden.
- Mögliche Dubletten bekommen `duplicate_candidate` oder `duplicate_of`.

### L-009 Rechte-Metadaten

Jeder Datensatz muss Nutzungs- und Rechtehinweise aufnehmen.

Akzeptanzkriterien:

- `rights.content_origin` ist gesetzt.
- `rights.use_policy` ist gesetzt.
- Öffentliche Exportfähigkeit kann maschinell geprüft werden.

### L-010 Erweiterbarkeit

Das Schema muss zusätzliche Quellen, Sprachen und Nutzungskontexte aufnehmen können.

Akzeptanzkriterien:

- Neue `source_page` Werte sind möglich.
- Übersetzungen sind optional modelliert.
- Trigger-Kontexte sind als Liste modelliert.
- Optionales SQLite kann ohne Schemaumbruch ergänzt werden.

---

## 18.3 Soll-Anforderungen

### S-001 SQLite-Erzeugung

Das System soll aus JSON eine SQLite-Datenbank generieren.

### S-002 CLI-Tooling

Ein CLI soll Validierung, Export und Suche ermöglichen.

Beispiel:

```bash
python tools/validate_quotes.py
python tools/build_json_from_yaml.py
python tools/build_sqlite.py
python tools/export_markdown.py --group-by category
```

### S-003 Fuzzy Search

Das System soll unscharfe Suche über Quotes unterstützen.

### S-004 Trigger-Auswahl

Das System soll Quotes passend zu Nutzungskontexten ausgeben können.

### S-005 Persona Scoring

Einträge sollen für verschiedene Persona-Typen bewertet werden:

```yaml
persona_fit:
  techpriest: 5
  magos: 4
  skitarii: 3
  servitor: 2
  neutral_lore: 3
```

### S-006 Übersetzungsfeld

Deutsche Übersetzungen sollen optional möglich sein, aber klar als Übersetzung oder Adaption markiert werden.

---

## 18.4 Kann-Anforderungen

### K-001 Web UI

Eine kleine lokale Web UI kann später Filter, Suche und Review ermöglichen.

### K-002 Telegram Bot

Ein Telegram Bot kann die Datenbank abfragen.

### K-003 OpenClaw Adapter

Ein Adapter kann Ereignisse aus OpenClaw oder einem lokalen Agentensystem auf Quote-Trigger mappen.

### K-004 LED Matrix Export

Ein Export kann kurze, displaytaugliche Zeilen für LED-Matrix-Projekte erzeugen.

### K-005 Audio-/TTS-Metadaten

Quotes können mit Sprechstil, Pausen und Betonung versehen werden.

---

## 19. Nicht-Ziele

Diese Dinge gehören nicht in Phase 1:

- Vollautomatische, ungeprüfte Veröffentlichung aller extrahierten Volltexte.
- Öffentliche API mit urheberrechtlich geschützten Volltexten.
- Vollständige Primärquellen-Verifikation jedes Zitats.
- Automatische Übersetzung aller Einträge.
- Semantische Embeddings als Pflichtbestandteil.
- Web-Crawler, der fremde Seiten regelmäßig ohne manuelle Kontrolle scraped.

---

## 20. Phasenplan

### Phase 1: Schema und Repository-Grundlage

Ergebnis:

- Ordnerstruktur
- `quote_entry.schema.json`
- `sources.yaml`
- `tags.yaml`
- Beispiel-YAML
- Validierungsskript

### Phase 2: Manuell kuratierter Seed-Datensatz

Ergebnis:

- 20 bis 50 geprüfte Datensätze
- Kategorien und Tags validiert
- Markdown-Export funktioniert
- JSON-Export funktioniert

### Phase 3: Import-Assistent

Ergebnis:

- Parser für beide Lexicanum-Seiten
- Draft-Datensätze
- Import-Report
- Dublettenliste

### Phase 4: Review und Normalisierung

Ergebnis:

- Alle importierten Datensätze überprüft oder markiert
- Dubletten bereinigt
- Source Quality gesetzt
- UI-Felder ergänzt

### Phase 5: SQLite und Bot/API-Vorbereitung

Ergebnis:

- SQLite-Export
- Suchfunktionen
- Beispielabfragen
- Optionaler Telegram-Bot-Prototyp

---

## 21. Validierungslogik

### 21.1 YAML Validierung

```pseudo
load tags.yaml
load quote_entry.schema.json
for file in data/*.yaml:
    parse yaml
    for record in records:
        validate against schema
        validate id uniqueness
        validate tags exist in registry
        validate source quality enum
        validate review status enum
```

### 21.2 Textnormalisierung

```pseudo
normalize(text):
    strip leading/trailing whitespace
    convert repeated spaces to single spaces
    preserve intentional line breaks for record_type in [litany, hymn, chant, prayer]
    normalize typographic quotes for search index only
    keep canonical_text unchanged except import cleanup
```

### 21.3 Duplicate Detection

```pseudo
for each record:
    exact_key = lower(normalized_text)
    if exact_key exists:
        mark duplicate_candidate

for each pair above fuzzy threshold:
    create duplicate_report entry
```

---

## 22. Beispielabfragen

### 22.1 Alle Litaneien mit Machine-Spirit-Bezug

```jq
.[] | select(.record_type == "litany" and (.semantic_tags | index("machine_spirit")))
```

### 22.2 Alle Quotes für Systemstart

```jq
.[] | select(.usage.trigger_contexts | index("system_boot"))
```

### 22.3 Nur belastbare Einträge

```jq
.[] | select(.classification.source_quality == "verified" or .classification.source_quality == "lexicanum_cited")
```

### 22.4 Alle noch zu prüfenden Einträge

```jq
.[] | select(.review.status == "needs_human_review")
```

---

## 23. Akzeptanztest für MVP

Der MVP gilt als erfüllt, wenn:

1. mindestens 20 Datensätze aus beiden Quellseiten im YAML-Format vorhanden sind,
2. alle Datensätze gegen JSON Schema validieren,
3. alle Tags in `tags.yaml` registriert sind,
4. JSON-Export erfolgreich läuft,
5. Markdown-Export erfolgreich läuft,
6. mindestens drei Nutzungskontexte sinnvoll befüllt sind,
7. Source Quality sichtbar und filterbar ist,
8. Dublettenprüfung einen Report erzeugt,
9. unsichere Lexicanum-Inhalte nicht als `verified` markiert werden,
10. die Datenbank ohne Internetzugriff lokal lesbar und nutzbar bleibt.

---

## 24. Risiken und Gegenmaßnahmen

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| Unklare Quellenlage | Falsche Attribution | Source Quality und Review Flags erzwingen |
| Copyright-Probleme bei öffentlicher Nutzung | Rechtliches Risiko | Volltexte nur lokal/private, öffentliche Exporte filtern |
| Uneinheitliche Wiki-Struktur | Parserfehler | Draft-Import plus menschlicher Review |
| Dubletten mit anderer Attribution | Verwirrende Daten | Duplicate-Modell und kanonische Version |
| Tag-Wildwuchs | Schlechte Filterbarkeit | Tag Registry mit Linter |
| Zu lange Texte für UI | Unlesbare Anzeige | UI-Metadaten und short_display |
| Zu viel Handarbeit | Projekt versandet | Import-Assistent und Review-Board |

---

## 25. Empfehlung für die erste Umsetzung

Der beste erste Schritt ist nicht Scraping, sondern ein stabiler Datenkern:

1. Repository-Struktur anlegen.
2. `quote_entry.schema.json` finalisieren.
3. `tags.yaml` mit 30 bis 50 kontrollierten Tags anlegen.
4. 10 Quotes und 10 Excerpts manuell als Seed erfassen.
5. Validator schreiben.
6. YAML nach JSON exportieren.
7. Markdown-Lesefassung generieren.
8. Danach erst Parser/Importer bauen.

Damit entsteht kein Zitat-Friedhof, sondern ein brauchbares mechanisches Archiv: kuratiert, filterbar, erweiterbar, agententauglich.

---

## 26. Minimale Projektdefinition für Codex/OpenClaw/Manus

```text
Build a local-first YAML/JSON quote database for Adeptus Mechanicus and Cult Mechanicus texts based on two Lexicanum source pages. Implement a strict JSON Schema, source metadata model, tag registry, YAML canonical files, JSON export, Markdown export, duplicate detection, source-quality classification, review workflow, and optional SQLite export. Do not publish copyrighted full-text data by default. Provide only schema, tooling, and placeholder/example seed records unless the user supplies local private data. All entries must preserve source lineage, attribution status, category, semantic tags, style tags, usage contexts, and review status.
```

---

## 27. Definition of Done

Das Projekt ist aus Designsicht bereit zur Umsetzung, wenn folgende Artefakte existieren:

```text
schema/quote_entry.schema.json
data/quotes.yaml
data/excerpts.yaml
tags.yaml
sources.yaml
tools/validate_quotes.py
tools/build_json_from_yaml.py
tools/export_markdown.py
docs/data_dictionary.md
docs/import_pipeline.md
```

Und folgende Befehle erfolgreich laufen:

```bash
python tools/validate_quotes.py
python tools/build_json_from_yaml.py
python tools/export_markdown.py
```

---

## 28. Zusammenfassung

Die Mechanicus Quote Database soll kein simples Zitate-Sammelglas werden. Sie soll ein sauber gebauter Datenaltar sein: YAML für Menschen, JSON für Maschinen, SQLite für spätere Suche, Markdown für Lesbarkeit und ein strenger Review-/Quellenstatus gegen Datenkorruption.

Die entscheidenden Designentscheidungen sind:

- YAML als kanonische Pflegequelle.
- JSON als maschinenlesbarer Export.
- Jede Quote bekommt Quelle, Status, Tags, Nutzungskontext und Review-Status.
- Unsichere Inhalte werden markiert, nicht beschönigt.
- Öffentliche Nutzung wird rechtebewusst vom privaten Volltextbestand getrennt.
- Das System wird von Anfang an für Agenten, Bots, Dashboards und Ambient Displays gedacht.

Damit ist die Datenbank später nicht nur lesbar, sondern benutzbar: als Stimme eines Techpriest-Agenten, als Ritual-Engine für OpenClaw, als Lore-Suche, als Telegram-Bot-Gehirn oder als flackernde Litanei auf einer LED-Matrix.
