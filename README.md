# Gefriertruhe-Workflow (Home Assistant): Barcode/QR → To-do-Eintrag

Dieses Projekt verbindet einen Handy-Barcode/QR-Scanner mit Home Assistant (HA), um Lebensmittel in einer **To-do-Liste** zu erfassen.  
Ein Scan löst einen **Webhook** aus, der automatisch einen Eintrag mit Datum, fortlaufender ID und Fälligkeitsdatum erstellt und ihn in einer To-do-Liste ablegt. Ein kleines Dashboard macht die Bedienung bequem.

<!-- Quellenhinweis: Automationsdetails stammen aus webhook_automation.yaml -->
<!-- :contentReference[oaicite:0]{index=0} -->
<!-- Quellenhinweis: Dashboard-Karten stammen aus dashboard.yaml -->
<!-- :contentReference[oaicite:1]{index=1} -->
<!-- Quellenhinweis: Script-Logik stammt aus add_entry_script.yaml -->
<!-- :contentReference[oaicite:2]{index=2} -->

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
