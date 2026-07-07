# Schritt-für-Schritt-Anleitung: Smarte Rezept-Automation mit Mealie, Docker & n8n

Diese Anleitung beschreibt, wie du eine eigene, selbstgehostete Rezeptdatenbank (Mealie) aufbaust und sie über ein Automatisierungstool (n8n) mit künstlicher Intelligenz (LLMs) verbindest. Dadurch kannst du:

1. Rezepte von Webseiten importieren, die kein strukturiertes Datenformat (JSON-LD) anbieten.

2. Rezepte per Foto/OCR (z. B. aus Kochbüchern via Paperless-ngx) vollautomatisch digitalisieren und importieren.

# Systemarchitektur & Ablauf

```
[ Kochbuch-Foto / Mobile App ] ──> [ Paperless-ngx (OCR) ] ──┐
                                                             │
                                                             v
[ Website ohne JSON-LD ] ──────────────────────────────> [ n8n Workflow ] ──> [ OpenAI / LLM (Parser) ] ──> [ Mealie API ]
```

# Voraussetzungen

* Ein Server, Mini-PC (z. B. Raspberry Pi, Intel NUC) oder ein dauerhaft laufender Computer mit Linux (Ubuntu/Debian empfohlen).

* Installiertes Docker und Docker Compose (wird in Schritt 1 erklärt).

* Ein API-Schlüssel für einen KI-Anbieter (z. B. OpenAI, Anthropic) oder ein lokal laufendes Modell (z. B. Ollama).

# Schritt 1: Docker & Docker Compose vorbereiten

Solltest du Docker noch nicht installiert haben, kannst du dies mit folgendem Befehl auf deinem Linux-System tun:

```
# Paketquellen aktualisieren und Docker installieren
sudo apt update
sudo apt install docker.io docker-compose-plugin -y

# Docker-Dienst starten und aktivieren
sudo systemctl enable --now docker
```

Erstelle nun einen dedizierten Ordner für deine Rezept-Infrastruktur auf deinem Server:

mkdir -p ~/rezept-automation
cd ~/rezept-automation


# Schritt 2: Das Docker Compose Setup

Wir nutzen eine gemeinsame docker-compose.yml-Datei, um alle benötigten Dienste (Mealie, n8n und optional Paperless-ngx) in einem isolierten Netzwerk zu starten.

Erstelle die Datei im erstellten Ordner:
```
nano docker-compose.yml
```

Kopiere den folgenden Inhalt in die Datei:

```yaml
version: "3.8"

networks:
  rezept-net:
    driver: bridge

services:
  # --- MEALIE: Die Rezeptdatenbank ---
  mealie:
    image: ghcr.io/mealie-recipes/mealie:v1.12.0 # Nutze die aktuelle stabile Version
    container_name: mealie
    restart: always
    ports:
      - "9000:80"
    environment:
      - ALLOW_SIGNUP=true
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Zurich # Passe deine Zeitzone an
      - BASE_URL=http://localhost:9000
    volumes:
      - ./mealie-data:/app/data
    networks:
      - rezept-net

  # --- n8n: Das Automatisierungs-Tool ---
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - TZ=Europe/Zurich
      - N8N_SECURE_COOKIE=false # Auf true setzen, wenn HTTPS verwendet wird
    volumes:
      - ./n8n-data:/home/node/.n8n
    networks:
      - rezept-net

  # --- PAPERLESS-NGX: Für OCR-Scans von physischen Rezepten (Optional) ---
  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: paperless
    restart: always
    ports:
      - "8010:8000"
    environment:
      - PAPERLESS_OCR_LANGUAGE=deu+eng
      - PAPERLESS_TIME_ZONE=Europe/Zurich
      - PAPERLESS_ADMIN_USER=admin
      - PAPERLESS_ADMIN_PASSWORD=admin # Nach dem ersten Login dringend ändern!
    volumes:
      - ./paperless/data:/usr/src/paperless/data
      - ./paperless/media:/usr/src/paperless/media
      - ./paperless/consume:/usr/src/paperless/consume
    networks:
      - rezept-net
```

Speichere die Datei (Strg+O, Enter) und schließe den Editor (Strg+X).

Erstelle die Ordner auf der rechten Hälfte unter "volumes", oder passe den Pfad an:

```
    volumes:
      - ./paperless/data:/DIESEN/PFAD/ERSTELLEN
```

## Container starten:

```
docker compose up -d
```

Tipp für Einsteiger: Der Parameter -d sorgt dafür, dass die Container im Hintergrund ("detached") laufen und dein Terminal frei bleibt.

Überprüfe mit `docker ps`, ob alle drei Dienste (Ports: 9000 für Mealie, 5678 für n8n, 8010 für Paperless) laufen.

# Schritt 3: Mealie & API-Token konfigurieren

1. Öffne deinen Browser und rufe Mealie auf: `http://<deine-server-ip>:9000`

2. Erstelle einen Admin-Account.

3. Gehe im Menü auf User Settings (Benutzereinstellungen) -> API Tokens.

4. Generiere einen neuen API-Token (z. B. mit dem Namen "n8n-Automation").

5. Kopiere den Schlüssel sofort! Du wirst ihn später im n8n-Workflow benötigen.

# Schritt 4: Der n8n-Workflow (Die Magie im Hintergrund)

Wir erstellen einen Workflow in n8n, der unstrukturierte Daten (HTML einer Webseite oder OCR-Text eines Scans) entgegennimmt, diese strukturiert an eine KI übergibt und das resultierende JSON-Rezept an Mealie sendet.

1. n8n öffnen

Rufe n8n auf unter `http://<deine-server-ip>:5678` und richte deinen Benutzer ein.

2. Workflow-Struktur aufbauen

Erstelle einen neuen, leeren Workflow. Dein Workflow benötigt folgende Kernknoten:

## A. Die Trigger (Auslöser)

* Web-Import-Trigger: Ein `Webhook`-Knoten (POST), an den du eine URL sendest.

* OCR-Import-Trigger: Ein `Webhook`-Knoten (oder ein `Paperless-ngx`-Trigger), der reagiert, wenn ein neues Dokument mit dem Tag "mealie" in Paperless hochgeladen wird.

## B. Der Daten-Fetcher (für Webseiten)

Falls eine URL übergeben wurde, nutzt du einen `HTTP Request`-Knoten, um das rohe HTML der Webseite herunterzuladen.

## C. Der KI-Parser (Das "Gehirn")

Verwende einen Advanced AI-Knoten in n8n (z. B. `OpenAI Chat Model` oder `Ollama` für lokale Setups).

* System-Prompt für das LLM:

```
Du bist ein präziser Rezept-Parser. Deine Aufgabe ist es, aus dem folgenden unstrukturierten Text (HTML oder OCR-Scan) ein strukturiertes JSON-Format für die Rezept-App "Mealie" zu generieren.

WICHTIG: Antworte AUSSCHLIESSLICH mit dem puren JSON-Objekt. Verwende kein Markdown-Formatting (keine ```json Blöcke).

Das Ziel-Format muss genau dieser Struktur entsprechen:
{
  "name": "Name des Rezepts",
  "description": "Kurze Beschreibung",
  "recipeYield": "Portionsanzahl (z.B. '4 Portionen')",
  "recipeIngredient": [
    {"note": "Menge und Zutat 1"},
    {"note": "Menge und Zutat 2"}
  ],
  "recipeInstructions": [
    {"text": "Schritt 1 der Zubereitung"},
    {"text": "Schritt 2 der Zubereitung"}
  ]
}
```

## D. Der Mealie API-Post

Nutze einen `HTTP Request`-Knoten, um das generierte Rezept an Mealie zu senden.

* Methode: `POST`

* URL: `http://mealie:80/api/recipes` (Hinweis: Da n8n und Mealie im selben Docker-Netzwerk sind, kannst du direkt den Container-Namen mealie auf Port 80 ansprechen!)

* Headers:

  * `Authorization` : `Token <DEIN_KOPIERTER_MEALIE_API_TOKEN>`

  * `Content-Type` : `application/json`

* Body: Nutze den Output des KI-Knotens.

# Schritt 5: OCR-Import mit Paperless-ngx einrichten

Um handschriftliche Rezepte oder Ausschnitte aus Kochbüchern zu importieren:

1. Richte die App Paperless Mobile auf deinem Smartphone ein und verbinde sie mit deinem Paperless-Server (`http://<deine-server-ip>:8010`).

2. Scanne ein Rezept mit deiner Handykamera.

3. Vergib in der App das Tag "mealie".

4. In n8n lauscht dein Workflow auf Dokumente von Paperless. Sobald ein neues Dokument mit dem Tag "mealie" auftaucht:

    * Holt sich n8n den extrahierten OCR-Text aus Paperless.

    * Sendet diesen Text an das LLM.

    * Das LLM generiert das Rezept-JSON und lädt es direkt in Mealie hoch.

# Typische Stolpersteine & Fehlerbehebung (Troubleshooting)

1. "Die KI halluziniert und kocht immer Zwiebelsuppe!"

    * Problem: Wenn die Webseite blockiert wird (z. B. durch Cloudflare) oder das OCR-Dokument komplett unleserlich ist, bekommt die KI keinen Text. Ohne Kontext neigen manche LLMs dazu, Standardantworten zu erfinden (wie die berühmte französische Zwiebelsuppe).

    * Lösung: Baue in n8n einen `If`-Knoten ein. Prüfe, ob der HTTP-Request der Webseite erfolgreich war und Text enthält. Ist der Text leer oder kürzer als 100 Zeichen, stoppe den Workflow und sende eine Fehlermeldung (z. B. via Matrix/Telegram/Discord).

2. "n8n kann Mealie nicht erreichen"

    * Problem: Fehlermeldung `Connection refused` im HTTP-Knoten.

    * Lösung: Stelle sicher, dass beide Container im selben Docker-Netzwerk (rezept-net) liegen. Nutze innerhalb des Docker-Netzwerks die URL `http://mealie:80` (nicht Port 9000, da dieser nur nach außen zum Host-System hin geöffnet ist).

3. "Mealie meldet Unauthorized"

    * Problem: Statuscode `401` beim API-Request.

    * Lösung: Überprüfe die Groß- und Kleinschreibung im HTTP-Header von n8n. Es muss exakt lauten: `Authorization` als Name und `Token <dein-token-wert>` als Wert. Das Wort "Token" muss mit einem Leerzeichen vom eigentlichen Schlüssel getrennt sein.