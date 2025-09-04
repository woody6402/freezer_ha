# Gefriertruhe -- Home Assistant 

This repository contains configuration files and dashboards for managing
a freezer ("Gefriertruhe") in **Home Assistant**.\
It includes: - Complete configuration (`gefriertruhe.yaml`) - A Lovelace
dashboard example (`dashboard.yaml`) - An **experimental web barcode
scanner** (`scanner_20.html`) as an add-on

------------------------------------------------------------------------

## 📦 Installation (Gefriertruhe)

### 1. Add helpers and scripts

Copy the content of [`gefriertruhe.yaml`](./gefriertruhe.yaml) into your
**Home Assistant `configuration.yaml`**, or split it into your existing
structure.\
This will install: - Helpers (`input_number`, `input_text`,
`input_select`) - Script to add freezer entries and increase IDs -

(Optional) Webhook automation for barcode scans

Restart Home Assistant after saving changes.

------------------------------------------------------------------------

### 2. Dashboard setup

The supplied [`dashboard.yaml`](./dashboard.yaml) contains a Lovelace
**grid layout** with: - To-do list for freezer items - Helper controls -
Last scanned barcode

⚠️ Note: Lovelace dashboards cannot be defined directly in
`configuration.yaml`.\
Import this file via the **Dashboard Editor** or use YAML mode with a
separate file.


------------------------------------------------------------------------

### 3. Dashboard screenshot

![Dashboard
Screenshot](Screenshot-20250903174802-1500x669.png)

------------------------------------------------------------------------

## 🧪 Experimental: Web Barcode Scanner

The file [`scanner_20.html`](./scanner_20.html) provides a **QR/Barcode
scanner** that runs in the browser.\
It is **experimental** and meant as an add-on:

-   Uses your device (i.e. handy) camera to scan codes
-   Can send results directly to Home Assistant via Webhook
-   Includes optional product lookup using [Open Food
    Facts](https://openfoodfacts.org)
-   Beware: only webhook is tested ....

### Usage

1.  Copy `scanner_20.html` into your Home Assistant `www` folder:

        config/www/scanner.html

2.  Access it from your browser via:

        https://<your-ha-url>:8123/local/scanner.html
        
    or using the buttons already placed in the dashboard (check the link if it's apropriate for you environment).

3.  Configure the webhook endpoint inside the UI.\
    (By default, it expects the `barcode_scan_p` webhook from the
    Gefriertruhe config.)

------------------------------------------------------------------------

## ⚠️ Notes

-   The scanner is experimental, some devices/browsers may not support
    all features (e.g. torch).
-   You can extend the food list in `input_select.speisen` with your own
    entries.
-   The `todo.gefriertruhe_2` entity in the config must match your own
    **To-do list** integration.
    https://www.home-assistant.io/integrations/local_todo/


------------------------------------------------------------------------
