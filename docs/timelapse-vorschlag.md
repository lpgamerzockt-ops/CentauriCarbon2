# Phase 2 – Timelapse-Änderungsvorschlag (noch NICHT anwenden)

**Status:** Warte auf Freigabe durch Manuel.  
**Patch-Datei:** `/cursor/stores/self/docs/timelapse-vorschlag.patch`  
**Analyse:** `/cursor/stores/self/docs/timelapse-analyse.md`

Keine behavior-changing Commits in diesem Turn. Der Patch ist nur Vorschlag.

---

## 1. Parameter-Tabelle

| Parameter | Ist | Vorgeschlagen (sinnvolles Maximum / erster Schritt) | Begründung | Extra Speicher | Extra Compute |
|-----------|-----|-----------------------------------------------------|------------|----------------|---------------|
| Capture/Encode width×height | 640×360 | **Stufe A (sicher):** unverändert 640×360. **Stufe B (nur nach Hardware-Check):** 1280×720 | Ohne Geräte-Enum riskant; 720p ≈ 4× Pixel | Stufe B: JPEG ~3–6×, RAM ~4× pro decode | Encode ~3–5× |
| `VIDEO_FPS` (Output) | 30 | **24** oder **30** belassen | 30 ist schon flüssig; höher macht Video kürzer wirkend bei gleicher Layerzahl / mehr Bitrate-Druck. Empfehlung: **30 lassen** oder 24 für etwas mehr bits/frame | ~gleich | ~gleich |
| `CAMERA_FPS` | 20 | 20 belassen (Timelapse = Einzelbild) | Capture ist event-basiert; höher hilft Timelapse kaum | — | etwas mehr USB/CPU live |
| H.264 `bit_rate` | 2_000_000 | **4_000_000** (640p) bzw. **8_000_000** (falls 720p) | Mehr Bits gegen Blockiness bei `stillimage` | MP4 ~2× | leicht höher |
| x264 `preset` | `ultrafast` | **`medium`** (Kompromiss) oder `slow` nur offline/nach Print | Größter Qualitätshebel bei gleicher Bitrate | etwas kleinere Dateien möglich | **deutlich** mehr CPU / längere Composite-Zeit |
| CRF | unset | **`crf=20`** + bit_rate als Upper-Cap **oder** CRF-only | Qualitätsziel statt starrem CBR; 18–23 = high quality (auch im toten Code `video_format_convert.cpp:71` erwähnt) | variabel | mittel |
| `profile` / `level` | baseline / 3.0 | **main** / **3.1** (640p) oder **4.0** (720p) | Bessere Effizienz; Baseline nur nötig für uralte Player | — | gering |
| `PIC_MAX_SIZE` / `SIZE_OF_FRAME` | 100 KiB | **300 KiB** (640p high-q) oder **512 KiB** (720p) | Verhindert Truncation wenn Kamera größere JPEGs liefert | RAM + Disk | — |
| JPEG-Löschung nach Composite | sofort `rm -rf` | Config `keep_source_jpegs: False` default; optional True | Re-Encode / Debug; kostet Disk | hoch wenn True | — |

### Empfohlene erste Freigabe (Phase 3 „safe“)

Nur Software-Encoder, **keine** Auflösungsänderung:

1. `bit_rate` → 4 Mbps  
2. `preset` → `medium`  
3. `crf` → 20  
4. `PIC_MAX_SIZE` / `SIZE_OF_FRAME` → 300 KiB (Safety)  
5. Alles über Config-Defaults, Ist-Werte als Fallback  

Auflösung 1280×720 = **separate Freigabe** nach `v4l2`-Check auf dem Gerät.

---

## 2. Config statt Hardcode (Vorschlag)

Neue Sektion in `printer.cfg` (Parser muss in Phase 3 verdrahtet werden; bis dahin nur Dokumentation + Header-Defaults):

```ini
# [timelapse]  — proposed; not wired in firmware yet
# width: 640
# height: 360
# output_fps: 30
# bit_rate: 4000000
# preset: medium
# crf: 20
# profile: main
# level: 3.1
# pic_max_size: 307200
# keep_source_jpegs: False
```

Vorgeschlagene zentrale Defaults-Datei im Code: `elegoo/extras/ai_camera/module/camera/timelapse_quality.h` (siehe Patch).

Bis der Config-Parser angebunden ist: Werte nur über dieses Header (Build-time). **Kein** Flash in Phase 2.

---

## 3. Patch

Siehe unified diff: `/cursor/stores/self/docs/timelapse-vorschlag.patch`

Inhalt skizziert:

- Neues `timelapse_quality.h` mit makros / Defaults (safe raise)  
- `uvc_config_info.h` / `time_lapse_photo.h` / `video_list.h` → nutzen diese Makros  
- `video_encoder.cpp` → bitrate/preset/crf/profile aus Makros; CRF setzen  
- Kommentierter Block in `debugfile/printer.cfg`  

**Nicht** `git apply` ohne Manuels OK.

---

## 4. Risiken der Vorschläge

- **`medium`/`slow`:** Composite kann Minuten dauern und CPU sättigen → **Vermutung:** UI/Nachbearbeitung ruckelt; nicht während kritischer Motion ausführen (Composite ist ohnehin post-capture).  
- **Höhere Bitrate:** volle SD/eMMC schneller.  
- **720p ohne Puffer-Anpassung:** Buffer overflows / fehlgeschlagene Captures.  
- **Motor/Heater/Safety:** unberührt (Patch nur `ai_camera`).

---

## 5. Freigabe-Checkliste für Manuel

- [ ] Safe-Encoder-Paket (bitrate/preset/crf/buffer) freigeben?  
- [ ] Auflösung belassen bei 640×360 für Phase 3?  
- [ ] Später 720p nach Hardware-Check?  
- [ ] `keep_source_jpegs` gewünscht?  
- [ ] Config-Parsing in `printer.cfg` in Phase 3 mitbauen oder erst Header-only?

**STOP – warte auf Freigabe. Phase 3 nicht starten.**
