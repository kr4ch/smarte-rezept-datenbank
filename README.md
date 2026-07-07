# Smarte Rezept-Automation (Mealie, n8n, Paperless & AI)

Dieses Repository enthält alle Materialien, Konfigurationen und Präsentationsfolien zu meinem MakerTalk im FabLab Winterthur: "Wenn die KI Zwiebelsuppe kocht – Smarte Rezept-Automation mit Mealie, Docker & n8n" vom 10.7.2026.

Ziel des Projekts ist es, Rezepte von beliebigen Webseiten (auch ohne strukturiertes JSON-LD) sowie physische Kochbuchseiten (via Smartphone-Foto und Paperless-ngx OCR) vollautomatisch über n8n und ein LLM (z. B. Google Gemini oder ein lokales Ollama-Modell) strukturiert in die eigene Rezeptdatenbank (Mealie) zu importieren.

## Projekt-Struktur

```.
├── doc/
│   └── anleitung.md                        # Ausführliche Installationsanleitung
├── slides/
│   ├── *.jpeg/png                          # Bbilder für die Präsentation
│   └── makertalk.md                        # Präsentationsfolien im Marp-Format
├── LICENSE                                 # MIT Lizenzvereinbarung
└── README.md                               # Diese Übersichtsdatei
```

## Schnelleinstieg

1. Infrastruktur selbst aufsetzen

    * Die vollständige Anleitung zum Aufsetzen der Docker-Umgebung (Mealie, n8n, Paperless-ngx) findest du unter `doc/anleitung.md`

2. Slides anschauen oder bearbeiten

    * Die Präsentationsfolien wurden mit Marp (Markdown Presentation Ecosystem) erstellt. Du kannst die Präsentation lokal rendern oder direkt in VS Code bearbeiten:

    * Installiere die VS Code Extension Marp for VS Code.

    * Öffne `slides/makertalk.md`.

    * Klicke oben rechts auf das Marp-Icon, um die Vorschau anzuzeigen oder die Slides als HTML/PDF zu exportieren.

Alternativ kannst du Marp über das CLI nutzen:
```
npx @marp-team/marp-cli@latest slides/makertalk.md --pdf
```

## Die n8n-Workflows im Überblick

### Use Case 1: Web-Scraper für unstrukturierte Rezeptseiten

Nimmt das ungefilterte HTML einer beliebigen Koch-Webseite entgegen, lässt die KI die Zutaten und Zubereitungsschritte heraussuchen und schickt das fertige JSON direkt an die Mealie-API.

### Use Case 2: Vom Kochbuch-Foto direkt in den Topf

Sobald du ein Foto mit der Paperless Mobile App machst und es mit dem Tag mealie versiehst, holt sich n8n den OCR-Text, korrigiert typische Scan-Fehler semantisch über das LLM und importiert es direkt.

## Lizenz

Dieses Repository ist unter zwei Lizenzen aufgeteilt, um sowohl Code als auch Medien fair zu teilen:

* Software, Konfigurationen & Workflows (docker-compose.yml, Skripte): Lizenziert unter der MIT-Lizenz (siehe LICENSE). Du kannst damit machen, was du willst – häng einfach meinen Namen an

* Präsentationsfolien, Dokumentation & Bilder (slides/, doc/): Lizenziert unter Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0).

Fragen oder Feedback?
Melde dich gerne per Issue oder schreib mir eine E-Mail!

Präsentiert im Juli 2026 beim FabLab Winti.