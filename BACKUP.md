# BACKUP.md – Sicherungen vor Timelapse-Optimierung / Safeguards

**Branch:** `optimierung`  
**Baseline-Commit (unmodified):** `5a2ea7fc03e707552701b1a69f463699cbd39230`  
**Repo-Fork:** https://github.com/lpgamerzockt-ops/CentauriCarbon2  
**Upstream (GPL-3.0, read-only):** https://github.com/elegooofficial/CentauriCarbon2

Dieses Dokument ist die Checkliste für Manuel **bevor** irgendetwas auf dem Drucker geflasht oder installiert wird.  
This is Manuel’s checklist **before** any flash/install on the printer.

---

## 1. Firmware-Version notieren / Note current printer firmware version

- [ ] Auf dem Drucker (Display / Settings / Info) die **aktuelle Firmware-Version** notieren.
- [ ] Optional: Kamera-Firmware-Version notieren (falls separat angezeigt; Code kennt `CAMERA_GET_VERSION` / `AI_CAMERA_GET_VERSION` in `elegoo/extras/ai_camera/module/camera/`).
- [ ] Notiz hier eintragen:

```
Printer FW version: ____________________
Camera FW version (if shown): __________
Date noted: ____________________________
```

**Hinweis:** In diesem Quell-Repo steht **keine** kanonische „installed on device“-Versionsnummer für Manuels Gerät. Build-Scripts erwarten Versionsstrings wie `00.00.00.02` (`README.md`, `mcu_build.sh`), das ist **nicht** automatisch die Version auf dem Drucker.

---

## 2. Original-Firmware von Elegoo lokal speichern / Download original Elegoo firmware

- [ ] Offizielle Centauri Carbon 2 Firmware von Elegoo herunterladen (Support-/Download-Seite des Herstellers).
- [ ] Dateien **lokal** archivieren (USB-Stick + PC), z. B.:
  - Gesamtes Image / `.swu` / `.ipk` – je nachdem, was Elegoo ausliefert
  - Checksummen (SHA256), falls vom Hersteller angegeben
- [ ] Speicherort notieren:

```
Local path to original Elegoo package(s): ____________________
SHA256 (if available): _______________________________________
```

**Vermutung:** Elegoo liefert Recovery über USB (`upan`) und/oder OTA. Im Repo gibt es Update-Skripte unter `ipkg-build/update_ipk.sh` (opkg / swupdate), aber **kein** vollständiges Endnutzer-Restore-Handbuch.

---

## 3. `printer.cfg` und alle Druckereinstellungen sichern / Back up config

Im Repo liegt eine Referenz-Config unter `debugfile/printer.cfg` – das ist **nicht** automatisch die Config auf dem Drucker.

- [ ] Vom Drucker / aus der laufenden Installation kopieren und lokal speichern:
  - [ ] `printer.cfg`
  - [ ] `printer_dsp.cfg` (falls vorhanden)
  - [ ] `temperature_sensors.cfg` (falls vorhanden)
  - [ ] weitere Custom-Configs / Macros
  - [ ] Netzwerkeinstellungen, Kalibrierdaten, Benutzerprofile (sofern exportierbar)
- [ ] Timelapse-Medien separat sichern, falls behalten werden sollen:
  - Bilder: `/opt/usr/picture/` (siehe `TIME_LAPSE_PIC_PATH` in `elegoo/extras/ai_camera/module/camera/time_lapse_photo.h`)
  - Videos: `/opt/usr/video/` (siehe `TIME_LAPSE_VIDEO_PATH` ebd.)

```
Local backup of printer.cfg: ____________________
Local backup of other settings: _________________
```

---

## 4. Unmodified Source-Archive (dieses Repo) / Repo snapshot

Phase 0 hat ein Archiv des **unveränderten** Trees erzeugt (ohne `.git`):

| Item | Value |
|------|--------|
| Date (UTC) | 2026-11-02 |
| Commit | `5a2ea7fc03e707552701b1a69f463699cbd39230` |
| Size | 108 455 767 bytes (~104 MiB) |
| SHA256 | `daf1995a004114d28bdcc3dbe1add82405a48d1632093944ac57e707806d7f78` |

**Durable store (split parts, wegen Store-Größenlimit ~100 MiB Einzeldatei):**  
`/cursor/stores/self/docs/backups/`

Reassemble:

```bash
cd /cursor/stores/self/docs/backups
cat centauri-carbon2-2026-11-02-5a2ea7fc03e707552701b1a69f463699cbd39230.tar.gz.*.part \
  > centauri-carbon2-2026-11-02-5a2ea7fc03e707552701b1a69f463699cbd39230.tar.gz
sha256sum -c SHA256.txt
```

Also during this agent run:  
`/workspace/.phase0-backup/centauri-carbon2-2026-11-02-5a2ea7fc03e707552701b1a69f463699cbd39230.tar.gz`  
(gitignored; not committed)

Details: `/cursor/stores/self/docs/backups/README.md`

---

## 5. Original-Firmware wiederherstellen / How to restore original firmware

### Was im Repo **evidenziert** ist

- `ipkg-build/update_ipk.sh` installiert Pakete per `opkg install … -d opt` und kennt Modi `upan` / `ota`.
- Bei fehlgeschlagenem Update versucht das Skript Recovery aus `$INST_DIR/firmware_version/` (Kopie des zuletzt erfolgreichen IPK) – siehe Kommentare/Logik um „update failed!recovery…“ in `ipkg-build/update_ipk.sh`.
- `ipkg-build/readme.md` dokumentiert nur generische `opkg install` / `opkg remove` Befehle, **kein** vollständiges CC2-Restore.
- `README.md` beschreibt **Build** von MCU-`factory.bin` und App-Layer, **nicht** das Zurückspielen der Hersteller-Firmware auf ein Gerät.

### Was **nicht** evidenziert ist → STOP

In diesem Repository und seinen Docs gibt es **kein** vollständiges, vom Hersteller freigegebenes Schritt-für-Schritt-Verfahren „Original-Elegoo-Firmware auf Centauri Carbon 2 zurückspielen“ (Display-Menü, USB-Dateiname, swupdate-Kanäle, Risiken, A/B-Partitionen für Endnutzer).

**Deshalb:** Hier werden **keine** erfundenen Flash-/Restore-Schritte beschrieben.

- [ ] Restore nur nach **offizieller Elegoo-Anleitung** (Support-Seite / Handbuch) durchführen.
- [ ] Bis dahin: **nichts** flashen, kein `swupdate`/`opkg` auf dem Drucker ausführen im Rahmen dieses Optimierungsprojekts.

Wenn Manuel eine offizielle Restore-Quelle hat (URL/PDF), kann Phase 3/Backup diese Checkliste später mit **zitierten** Schritten ergänzen.

---

## 6. Hard rules (Projekt)

- Kein Flash/Install auf dem Drucker in Phase 0–2.
- Keine Änderungen an Motor-, Heater-, Temperature- oder Safety-Code.
- Arbeiten nur auf Branch `optimierung`; `main` bleibt Baseline.
- Qualitätsänderungen erst nach Freigabe von Phase-2-Vorschlag (Phase 3).
