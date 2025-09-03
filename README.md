# Gefriertruhe-Workflow (Home Assistant): Barcode/QR → To-do-Eintrag

Dieses Projekt nutzt ein **Home-Assistant-Dashboard**, mit dem du deine Gefriertruhe bequem mit **eigenen Speisen** befüllst. Über die Oberfläche wählst du Gerichte aus deiner Liste; **Einfrierdatum**, **fortlaufende ID** und **Fälligkeitsdatum** werden automatisch gesetzt und die Einträge in einer **To-do-Liste** abgelegt. Der **Webhook** zum Erfassen **gekaufter Produkte** ist ein optionales **Add-on** und kann bei Bedarf zusätzlich aktiviert werden.

---

## Inhalt

- [Funktionen](#funktionen)
- [Voraussetzungen](#voraussetzungen)
- [Benötigte Helper & Entities](#benötigte-helper--entities)
- [Installation](#installation)
- [Ablauf & Architektur](#ablauf--architektur)
- [Dashboard (Lovelace) hinzufügen](#dashboard-lovelace-hinzufügen)
- [Optionales Add-on: Webhook (Scanner → HA)](#optionales-add-on-webhook-scanner--ha)
- [Anpassen](#anpassen)
- [Troubleshooting](#troubleshooting)
- [Dateiübersicht](#dateiübersicht)
- [Lizenz](#lizenz)

---

## Funktionen

- **Dashboard-first:** Wähle Speisen aus einer **pflegbaren Liste** (Helper) und lege mit einem Button neue Einträge an.
- **Automatische Metadaten:**  
  - fortlaufende **ID**  
  - **Einfrierdatum** (in der Beschreibung)  
  - **Fälligkeitsdatum** = heute + **120 Tage**
- **Übersicht & Bedienung:** Kacheln für To-do-Liste, aktuelle ID und letzte Aktion; eine Entities-Karte zum Auswählen der Speise und Auslösen.
- **Optionales Add-on:** Ein **Webhook** kann zusätzlich genutzt werden, um **gekaufte Produkte** per Handy-Scan zu erfassen.

---

## Voraussetzungen

- Home Assistant (Core/Supervised/OS).
- Für das optionale Add-on: Eine Scanner-/Automation-App, die **HTTP-POST** an eine URL senden kann (z. B. iOS Kurzbefehle, Android Tasker/HTTP Request Shortcuts).

---

## Benötigte Helper & Entities

> Lege diese unter **Einstellungen → Geräte & Dienste → Helfer** an (empfohlen).  
> Alternativ findest du unten Beispiel-YAML.

| Typ            | Name (Vorschlag)          | Entity-ID                         | Zweck |
|----------------|---------------------------|-----------------------------------|------|
| **To-do-Liste** (local to do Erweiterung) | Gefriertruhe 2               | `todo.gefriertruhe_2`             | Ziel-Liste für neue Einträge |
| **Zahl**       | Eingefroren ID             | `input_number.eingefroren_id`     | fortlaufende ID (wird automatisch erhöht) |
| **Auswahl**    | Speisen                    | `input_select.speisen`            | vordefinierte Speisen für manuelles Anlegen |
| **Text**       | Letzter Scan (Barcode, QR, …) | `input_text.last_qr_scn_gefriertruhe` | Anzeige des letzten Scan-Payloads (nur fürs Add-on) |

**Empfohlene Startwerte:**

- `input_number.eingefroren_id`: min `1`, max `9999`, Schritt `1`, Initialwert z. B. `1`.
- `input_select.speisen`: Befülle die Optionsliste mit deinen Standardgerichten.
- `input_text.last_qr_scn_gefriertruhe`: max. Länge z. B. `255` (nur nötig, wenn du das Add-on nutzt).

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
      - Rindsgulasch 
      - Chili con Carne 
      - Lasagne 
      - Bolognese 
```

> **Hinweis zur To-do-Liste:**  
> Erstelle die Liste **als Helper** in der UI (Name „Gefriertruhe 2“), damit die Entity-ID `todo.gefriertruhe_2` entsteht.

---

## Installation

1. **Dateien übernehmen**  
   - Inhalte aus diesem Repo (Scripts, Automationen, Dashboard-Beispiel) in dein HA-Setup übernehmen:
2. **Script importieren**  
   - Inhalt von [`add_entry_script.yaml`](./add_entry_script.yaml) in **Einstellungen → Automationen & Szenen → Skripte** importieren oder in `scripts.yaml` einfügen.
3. **Helper anlegen** (siehe oben).
4. **Dashboard hinzufügen** (siehe Abschnitt unten).
5. **(Optional) Webhook-Add-on aktivieren**  
   - Inhalt von [`webhook_automation.yaml`](./webhook_automation.yaml) importieren **nur wenn** du Barcode/QR-Scans nutzen willst.
6. **Reload**  
   - Unter **Einstellungen → Entwicklerwerkzeuge → YAML**: *Automationen neu laden*, *Skripte neu laden*.

---

## Ablauf & Architektur

### Dashboard-Fluss (Standard)

- Du wählst in `input_select.speisen` ein Gericht und klickst den **„Einfrieren“**-Button.  
- Das Script `script.gefriertruhe_eintrag_anlegen_id_erhohen`:
  - liest die aktuelle **ID** (`input_number.eingefroren_id`),
  - berechnet **heute** und **Fälligkeit = heute + 120 Tage**,
  - erstellt einen **To-do-Eintrag** in `todo.gefriertruhe_2` (Titel = Speise, Beschreibung = „`<ID> (eingefroren am <Datum>)`“),
  - erhöht die **ID** um `+1`.

### Webhook-Add-on (optional)

- Ein externer Scan sendet einen HTTP-Request an den Home-Assistant-**Webhook**.  
- Die Automation schreibt den rohen Scan in `input_text.last_qr_scn_gefriertruhe` und ruft dasselbe **Script** auf – mit `speise` aus dem Payload (oder dem Code als Fallback).

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

> Die letzte Kachel (`input_text.last_qr_scn_gefriertruhe`) ist nur für das **Add-on** relevant – du kannst sie weglassen, wenn du keinen Webhook nutzt.

---

## Optionales Add-on: Webhook (Scanner → HA)

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

## Anpassen

- **Fälligkeitszeitraum ändern:** In `add_entry_script.yaml` `timedelta(days=120)` anpassen.
- **Andere To-do-Liste verwenden:** `todo.gefriertruhe_2` in Script & Dashboard ersetzen.
- **Speisenliste pflegen:** Optionen von `input_select.speisen` in den Helper-Einstellungen anpassen.
- **Webhook-ID ändern (nur Add-on):** In `webhook_automation.yaml` `webhook_id` anpassen **und** die Scanner-App-URL aktualisieren.

---

## Troubleshooting

### Dashboard
- **Kein Eintrag nach Klick:** Script/Automationen neu geladen? Existieren `todo.gefriertruhe_2` und `input_number.eingefroren_id`?
- **ID erhöht sich nicht:** Schreibrechte für `input_number.eingefroren_id` prüfen.

### Webhook-Add-on
- **Kein Eintrag nach Scan:** Stimmt die Webhook-URL (`.../api/webhook/barcode_scan_p`)? Kommt **HTTP 200** zurück? Logbook/Protokolle prüfen.
- **Produktname fehlt:** App liefert ggf. nur den Code – dann wird der Code als Titel verwendet.

---

## Dateiübersicht

- `add_entry_script.yaml` – Script: To-do-Eintrag anlegen, ID hochzählen, Fälligkeit +120 Tage.  
- `dashboard.yaml` – Lovelace-Karten für To-do-Liste, ID-Counter, Auswahl & (optional) Scan-Anzeige.  
- `webhook_automation.yaml` – **Optionales Add-on**: Webhook-Trigger → Script.

---

## Lizenz

MIT – siehe `LICENSE`.
