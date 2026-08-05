# Adeptus Mechanicus Quote Database

Planungs- und Ausgangsrepository für die **Mechanicus Quote Database (MQD)**: eine lokale, quellenbewusste YAML/JSON-Wissensbasis für Adeptus-Mechanicus-Zitate, Litaneien und verwandte Textformen. Die Daten sollen eine Techpriest-Persona mit geprüftem Flair versorgen und über eine review-pflichtige Kandidatenpipeline kontrolliert wachsen können.

## Aktueller Stand

Dieses Repository enthält zunächst die kommentierte Planungsgrundlage, noch keine fertige Anwendung und keine Sammlung urheberrechtlich geschützter Volltexte:

- [`docs/mechanicus-quote-database-build-plan.md`](docs/mechanicus-quote-database-build-plan.md) – vollständiges Idea-to-Agent-Plan-Paket mit Idea Draft, PRD, Meta-Prompt, Entwicklungsplan, Agenten-Taskgraph und Traceability-Matrix
- [`docs/mechanicus_quote_db_design_lastenheft.md`](docs/mechanicus_quote_db_design_lastenheft.md) – vorhandenes Design-/Lastenheft, unverändert als Primärquelle übernommen
- [`schema/quote_entry.schema.json`](schema/quote_entry.schema.json) – vorhandener Schemaentwurf, unverändert übernommen
- [`examples/quotes.example.yaml`](examples/quotes.example.yaml) – vorhandene, volltextfreie Beispielstruktur

## Wichtige Grenze

Das öffentliche Repository darf standardmäßig nur Schema, Werkzeuge, Dokumentation und Platzhalter bzw. selbst erstellte Testdaten enthalten. Drittanbieter-Volltexte bleiben in einem privaten, lokalen Datenbestand und dürfen einen öffentlichen Export nur nach expliziter Rechteprüfung passieren.

## Empfohlener nächster Schritt

Mit Phase 0 und Phase 1 des Build-Plans beginnen: Datenvertrag korrigieren, Repository-Grundgerüst aufbauen und einen dünnen, vollständig getesteten Pfad von YAML über Validierung zu JSON/Markdown implementieren.
