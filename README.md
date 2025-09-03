# Gefriertruhe-Workflow (Home Assistant): Barcode/QR → To-do-Eintrag

Dieses Projekt verbindet einen Handy-Barcode/QR-Scanner mit Home Assistant (HA), um Lebensmittel in einer **To-do-Liste** zu erfassen.  
Ein Scan löst einen **Webhook** aus, der automatisch einen Eintrag mit Datum, fortlaufender ID und Fälligkeitsdatum erstellt und ihn in einer To-do-Liste ablegt. Ein kleines Dashboard macht die Bedienung bequem.

---

## Inhalt

- [Funktionen](#funktionen)
- [Voraussetzungen](#voraussetzungen)
- [Benötigte Helper & Entities](#benötigte-helper--entities)
- [Installation](#installation)
- [Webhook einrichten (Scanner → HA)](#webhook-einrichten-scanner--ha)
- [Wie es funktioniert](#wie-es-funktioniert)
- [Dashboard (Lovelace) hinzufügen](#dashboard-lovelace-hinzufügen)
- [Anpassen](#anpassen)
- [Troubleshooting](#troubleshooting)
- [Dateiübersicht](#dateiübersicht)
- [Lizenz](#lizenz)

---

## Funktionen

- **Scan → Eintrag:** POST/PUT an einen HA-Webhook erzeugt einen To-do-Eintrag.
- **Automatische Metadaten:**  
  - fortlaufende **ID**  
  - **Einfrierdatum** (Text in der Beschreibung)  
  - **Fälligkeitsdatum** = heute + **120 Tage** (zum Nachsehen)
- **Manuelle Erfassung:** Über das Dashboard kannst du auch ohne Scan Einträge anlegen.
- **Letzter Scan:** Der rohe Scantext wird separat angezeigt.

---

## Voraussetzungen

- Home Assistant (Core/Supervised/OS) mit aktivem **HTTP-Zugriff** von deinem Smartphone.
- Ein Barcode/QR-Scanner auf dem Handy, der **HTTP-Request (POST/PUT)** an eine URL senden kann  
  (z. B. iOS Kurzbefehle, Android Tasker/HTTP Request Shortcuts, diverse Scanner-Apps).
- Optional: HA Companion App (nicht zwingend).

---

## Benötigte Helper & Entities

> Lege diese unter **Einstellungen → Geräte & Dienste → Helfer** an (empfohlen).  
> Alternativ findest du unten Beispiel-YAML.

| Typ            | Name (Vorschlag)          | Entity-ID                         | Zweck |
|----------------|---------------------------|-----------------------------------|------|
| **To-do-Liste** (Helper) | Gefriertruhe 2               | `todo.gefriertruhe_2`             | Ziel-Liste für neue Einträge |
| **Zahl**       | Eingefroren ID             | `input_number.eingefroren_id`     | fortlaufende ID (wird automatisch erhöht) |
| **Auswahl**    | Speisen                    | `input_select.speisen`            | vordefinierte Speisen für manuelles Anlegen |
| **Text**       | Letzter Scan (Barcode, QR, …) | `input_text.last_qr_scn_gefriertruhe` | Anzeige des letzten Scan-Payloads |

**Empfohlene Startwerte:**

- `input_number.eingefroren_id`: min `1`, max `9999`, Schritt `1`, Initialwert z. B. `1`.
- `input_select.speisen`: Befülle die Optionsliste mit deinen Standardgerichten.
- `input_text.last_qr_scn_gefriertruhe`: max. Länge z. B. `255`.

**Beispiel-YAML (optional, falls du lieber in YAML definierst):**

```yaml
# configuration.yaml (oder in helpers.yaml eingebunden)

input_number:
  eingefroren_id:
    name: Eingefroren ID
    initial: 1
    min: 1
    max: 9999
    step: 1
    mode: box

input_text:
  last_qr_scn_gefriertruhe:
    name: Letzter Scan (Barcode, QR, …)
    max: 255

input_select:
  speisen:
    name: Speisen
    options:
      - Rindsgulasch (2 Portionen)
      - Chili con Carne (3 Portionen)
      - Lasagne (1 Blech)
      - Bolognese (4 Portionen)
```

> **Hinweis zur To-do-Liste:**  
> Erstelle die Liste **als Helper** in der UI (Name „Gefriertruhe 2“), damit die Entity-ID `todo.gefriertruhe_2` entsteht.

---

## Installation

1. **Dateien übernehmen**  
   - `automations` / `scripts` / `dashboard` aus diesem Repo in dein HA-Setup kopieren.  
2. **Script importieren**  
   - Inhalt von [`add_entry_script.yaml`](./add_entry_script.yaml) in **Einstellungen → Automationen & Szenen → Skripte** importieren oder in `scripts.yaml` einfügen.  
3. **Automation importieren**  
   - Inhalt von [`webhook_automation.yaml`](./webhook_automation.yaml) in **Automationen** importieren oder in `automations.yaml` einfügen.  
4. **Helper anlegen** (siehe oben).  
5. **Neustart / Reload**  
   - Unter **Einstellungen → Entwicklerwerkzeuge → YAML**: *Automationen neu laden*, *Skripte neu laden*.  
6. **Dashboard hinzufügen** (siehe Abschnitt unten).

---

## Webhook einrichten (Scanner → HA)

**Webhook-ID:** `barcode_scan_p`  
**URL:**  
```
https(s)://<deine-ha-url>/api/webhook/barcode_scan_p
```
**Methode:** `POST` (oder `PUT`)  
**Content-Type:** `application/json`

**Beispiel-Payloads:**
```json
// 1) Nur Code (Produktname unbekannt)
{ "code": "9001234567890" }

// 2) Code + Produktname (falls die App ihn erkennt/mitliefert)
{ "code": "9001234567890", "product": "Rindsgulasch (2 Portionen)" }
```

**cURL-Test:**
```bash
curl -X POST "https://<deine-ha-url>/api/webhook/barcode_scan_p"   -H "Content-Type: application/json"   -d '{"code":"9001234567890","product":"Chili con Carne (3 Portionen)"}'
```

> **Sicherheit:** HA-Webhooks benötigen **keinen Token**, die ID sollte daher geheim bleiben.

---

## Wie es funktioniert

1. **Automation (Webhook-Trigger)**  
   - Lauscht auf `POST/PUT` am Webhook `barcode_scan_p`.  
   - Schreibt eine Logbook-Zeile (`name: Scan`, `message: <code>`).  
   - Setzt `input_text.last_qr_scn_gefriertruhe` auf `"<code> :: <product>"`.  
   - Startet das Script `script.gefriertruhe_eintrag_anlegen_id_erhohen` und übergibt `speise` (Produktname oder – wenn leer – den Code).  

2. **Script („Eintrag anlegen & ID erhöhen“)**
   - Liest aktuelle **ID** aus `input_number.eingefroren_id`.  
   - Bestimmt **Speise**: übergebene Variable oder Auswahl aus `input_select.speisen`.  
   - Erzeugt **heutiges Datum** (`TT.MM.JJJJ`) und **Fälligkeitsdatum** (**+120 Tage**).  
   - Fügt **To-do-Eintrag** in `todo.gefriertruhe_2` hinzu:  
     - **Titel:** `speise`  
     - **Beschreibung:** `<ID> (eingefroren am <Datum>)`  
     - **Fällig:** `<heute + 120 Tage>`  
   - Erhöht `input_number.eingefroren_id` um 1.  

---

## Dashboard (Lovelace) hinzufügen

Du kannst die mitgelieferten Karten als eigenes Dashboard (YAML-Modus) verwenden oder die Karten per UI nachbauen.

**YAML-Beispiel (eigene Ansicht):**
```yaml
type: grid
cards:
  - type: todo-list
    entity: todo.gefriertruhe_2
    display_order: duedate_asc
    grid_options: { columns: 12, rows: 7 }
    hide_create: true

  - type: tile
    entity: todo.gefriertruhe_2
    features_position: bottom

  - type: tile
    entity: input_number.eingefroren_id
    features_position: bottom

  - type: entities
    title: ""
    show_header_toggle: false
    state_color: false
    grid_options: { columns: 12, rows: 2 }
    entities:
      - entity: input_select.speisen
        name: Speise wählen
      - type: call-service
        name: Einfrieren
        icon: mdi:snowflake
        action_name: Start
        service: script.gefriertruhe_eintrag_anlegen_id_erhohen

  - type: tile
    entity: input_text.last_qr_scn_gefriertruhe
    features_position: bottom
    vertical: false
    grid_options: { columns: 12, rows: 1 }
    name: "Letzter Scan (Barcode, QR, ...)"
column_span: 2
```

> Diese Karten entsprechen der mitgelieferten `dashboard.yaml`.

---

## Anpassen

- **Fälligkeitszeitraum ändern:**  
  In `add_entry_script.yaml` `timedelta(days=120)` anpassen.
- **Webhook-ID ändern:**  
  In `webhook_automation.yaml` `webhook_id` anpassen **und** die Scanner-App-URL aktualisieren.
- **Andere To-do-Liste verwenden:**  
  `todo.gefriertruhe_2` in Script & Dashboard ersetzen.
- **Manueller Button-Text/Icon:**  
  Im Dashboard die `call-service`-Karte bearbeiten.

---

## Troubleshooting

- **Kein Eintrag nach Scan:**  
  - Stimmt die Webhook-URL (inkl. `https://…/api/webhook/barcode_scan_p`)?  
  - Kommt ein **HTTP 200** zurück?  
  - In HA unter **Einstellungen → Protokolle** bzw. **Entwicklerwerkzeuge → Ereignisse/Logbook** prüfen.
- **Falsche Entity-IDs:**  
  - Prüfe, ob die Helper-IDs exakt mit den oben genannten übereinstimmen.  
- **Script läuft, aber keine ID-Erhöhung:**  
  - `input_number.eingefroren_id` existiert und ist nicht schreibgeschützt?

---

## Dateiübersicht

- `webhook_automation.yaml` – Automation: Webhook-Trigger, Log, Text setzen, Script starten.  
- `add_entry_script.yaml` – Script: To-do-Eintrag anlegen, ID hochzählen, Fälligkeit +120 Tage.  
- `dashboard.yaml` – Lovelace-Karten für To-do-Liste, ID-Counter, Auswahl & Scan-Anzeige.  

---

## Lizenz

MIT – siehe `LICENSE`.
