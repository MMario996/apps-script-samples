# Gemini Gem: „Apps Script Architekt“

Ein umfassender Gemini-Gem, der beim Programmieren mit **Google Apps Script** und den **Google Workspace APIs** hilft: Code schreiben, Fehler finden, Reviews, Architektur, clasp/GitHub-Workflow, Add-ons, Chat-Apps und KI-Integration.

## Inhalt

| Datei | Verwendung |
|---|---|
| `01_GEM_ANWEISUNGEN.md` | Text für das Feld **„Anweisungen“** des Gems |
| `wissen/01_apps-script-kern.md` | Wissen: Laufzeit, Manifest, Dienste, Trigger, Quotas, Performance |
| `wissen/02_workspace-apis.md` | Wissen: Gmail, Calendar, CalDAV, Chat, Docs, Drive (+ Activity, Labels, Picker), Keep, Forms, Meet, Sheets, Slides, Tasks, Vault, Admin |
| `wissen/03_webapps-html-json.md` | Wissen: doGet/doPost, JSON, CORS, HtmlService, Picker, UrlFetchApp |
| `wissen/04_addons-chat-cards-ai.md` | Wissen: Workspace-/Editor-Add-ons, CardService, Chat-Apps, Custom Functions, Gemini/Vertex AI |
| `wissen/05_clasp-github-typescript.md` | Wissen: clasp, Git, GitHub Actions, TypeScript, Biome, Tests |
| `wissen/06_rezepte.md` | Wissen: 20 geprüfte Code-Muster |
| `wissen/07_ressourcen-und-samples.md` | Wissen: Linksammlung, Bibliotheken, Index der offiziellen Samples |

## Einrichtung (ca. 3 Minuten)

1. https://gemini.google.com öffnen → links **„Gems entdecken“** → **„Neues Gem“**.
2. **Name**: `Apps Script Architekt`
3. **Anweisungen**: Den kompletten Inhalt von `01_GEM_ANWEISUNGEN.md` einfügen (reiner Text, direkt kopierbar).
4. **Wissen**: Die 7 Dateien aus `wissen/` hochladen. Erlaubt sind maximal 10 Dateien, es bleiben also 3 Plätze frei, z. B. für eigene Projektdateien oder `GEMINI.md` aus dem Samples-Repo.
5. Optional: Statt Upload die Dateien als Google Docs in Drive ablegen und über **„Drive“** verknüpfen. Dann kann man sie später aktualisieren, ohne den Gem neu zu bauen.
6. **Speichern**. Für Code-Aufgaben am besten ein „Pro“- bzw. „Thinking“-Modell auswählen.

## Beispiel-Prompts

- `Projekt: Ein Sheet sammelt Urlaubsanträge per Google Form; Vorgesetzte sollen per Mail genehmigen und der Urlaub soll in einen Teamkalender.`
- `Schreib mir eine Web-App, die JSON per POST annimmt und in ein Sheet schreibt – inkl. curl-Test und fetch()-Beispiel aus dem Browser.`
- `Review:` + eingefügter Code
- `Mein Skript bricht ab mit "Exceeded maximum execution time" – hier der Code: …`
- `clasp: Richte mir ein Projekt mit TypeScript, esbuild, Biome und GitHub-Actions-Deploy ein.`
- `Extrahiere alle Bilder aus einem Google Doc in einen Drive-Ordner.`
- `Baue eine Chat-App, die /standup sammelt und täglich eine Zusammenfassung mit Gemini postet.`
- `Erklär:` + Code (für Einsteiger)
- `Migration: Dieses alte Rhino-Skript mit var und DriveApp-Schleifen auf V8 + Drive API v3 umbauen.`

## Gem aktuell halten

- Quotas, Modellnamen (Gemini) und Preview-APIs ändern sich. Die Dateien `01_…` und `04_…` regelmäßig mit der offiziellen Doku abgleichen.
- Eigene, oft genutzte Code-Muster in `06_rezepte.md` ergänzen.
