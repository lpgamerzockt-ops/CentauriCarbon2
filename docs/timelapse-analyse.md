# Timelapse-Analyse Centauri Carbon 2 (Phase 1)

**Stand:** 2026-11-02  
**Repo-Commit Basis:** `5a2ea7fc03e707552701b1a69f463699cbd39230` (`optimierung`)  
**Scope:** Analyse only — no behavior-changing code applied.

Jede Aussage mit Datei:Zeile. Unsicheres als **Vermutung** markiert.

---

## 1. Kurzfazit / Verdict

**Ja, Code-Änderungen können die Timelapse-Qualität messbar verbessern** — vor allem Encoder-Parameter (`preset ultrafast`, `bit_rate 2_000_000`, kein CRF im aktiven Pfad) und ggf. Ausgabe-FPS.  

**Die räumliche Auflösung der Quellframes ist hart auf 640×360 begrenzt** (`CAMERA_FARME_WIDTH` / `CAMERA_FARME_HIGHT`). Die Kamera liefert bereits **komprimierte MJPEG/JPEG**-Frames über V4L2; höhere Pixelqualität braucht Hardware-Unterstützung *und* größere Software-Puffer (`PIC_MAX_SIZE` / `SIZE_OF_FRAME` = 100 KiB). Ob die UVC-Kamera >640×360 kann, ist in diesem Repo **nicht** fest verdrahtet enumeriert (Enum-Hilfe existiert, ist aber auskommentiert) → **Vermutung:** oft 720p möglich, muss am Gerät verifiziert werden.

---

## 2. Pipeline (evidenziert)

```
UVC /dev/video0 (MJPEG) @ 640×360, CAMERA_FPS=20
        │  V4L2 SetFormat + Streaming
        ▼
VideoList ring buffer (SIZE_OF_FRAME = 100 KiB)
        │  GetFrameOfMjpeg()
        ▼
AICamera::TimeLapseSavePicture → TimeLapseVideo::Capture
        │  raw JPEG bytes → /opt/usr/picture/<name>.temp/NNNN.jpeg
        ▼  (pro Layer / Capture-Event vom Host)
TimeLapseStop → rename .temp → final pic dir
        ▼
method "composite_video" → ImageToVideoEncoder::encode_from_directory
        │  stbi_load JPEG→RGB, swscale→YUV420P, libx264 → MP4
        ▼
/opt/usr/video/<name>.mp4
        │
        ▼
rm -rf Quellbilder (nach erfolgreicher/versuchter Composite)
```

### 2.1 Kamera-Input

| Parameter | Wert | Fundstelle |
|-----------|------|------------|
| JPEG/MJPEG device | `/dev/video0` | `elegoo/extras/ai_camera/module/camera/uvc_config_info.h:26` |
| H264 device (Streaming, nicht Timelapse-Capture) | `/dev/video2` | `uvc_config_info.h:28` |
| Capture-Format Timelapse | `V4L2_PIX_FMT_MJPEG` (`UvcCameraInfoV2`) | `uvc_config_info.h:52–56` |
| Resolution | **640 × 360** | `uvc_config_info.h:20–21` |
| Camera FPS (V4L2) | **20** (`CAMERA_FPS`) | `uvc_config_info.h:17` |
| Format setzen | `VIDIOC_S_FMT` width/height/pixelformat | `uvc_v4l2.cpp:187–207` |
| Frame lesen | `GetFrameOfMjpeg` → `VideoList::ReadOneFrame` | `uvc_camera_without_uart.cpp:119–129` |

`REAL_TIME_SAVE_VIDEO` ist **0** → Pfad „erst JPEGs, dann Composite“ aktiv (`ai_camera.h:35`).

### 2.2 Frame pro Layer / Capture

- JSON-Methode `time_lapse` mit `status: capture` + `index` → `TimeLapseSavePicture` (`ai_camera.cpp:245–252`, `294–320`).
- Holt MJPEG-Frame in `pic_buf_` (Größe `PIC_MAX_SIZE`) und schreibt Datei `….jpeg` (`time_lapse_control.cpp:65–83`, `205–224`).
- Layer-Trigger kommt von außen (Host/GUI); GCode-Kommentare `timelapse_type = 0` in `gcodefile/*` sind Metadaten, keine Capture-Logik.

### 2.3 Storage

| Pfad | Makro | Fundstelle |
|------|-------|------------|
| Bilder | `/opt/usr/picture/` | `time_lapse_photo.h:17` |
| Videos | `/opt/usr/video/` | `time_lapse_photo.h:18` |

Nach Composite: symlink `/tmp/pic_link` + **`rm -rf` auf Bildverzeichnis** (`ai_camera.cpp:433–439`). Incomplete stop löscht Temp-Dir (`time_lapse_control.cpp:199–201`).

### 2.4 Encoding / Export

Aktiver Encoder: **`ImageToVideoEncoder`** (libav), nicht der CLI-`CombineImagesToVideo`-Pfad in `TimeLapseVideo` (der wird in `Composite` nicht mehr primär genutzt; `ai_camera.cpp` `#else`-Zweig ruft Encoder direkt).

| Encoder-Parameter | Aktueller Wert | Fundstelle |
|-------------------|----------------|------------|
| Codec | H.264 (`AV_CODEC_ID_H264`) | `video_encoder.cpp:84` |
| Output size | `CAMERA_FARME_WIDTH/HIGHT` = 640×360 | `ai_camera.cpp:413` |
| Output FPS | `VIDEO_FPS` = **30** | `uvc_config_info.h:18`; `ai_camera.cpp:413` |
| `bit_rate` | **2_000_000** (2 Mbps) | `video_encoder.cpp:114` |
| `preset` | **`ultrafast`** | `video_encoder.cpp:118` |
| `tune` | `stillimage` | `video_encoder.cpp:119` |
| `profile` | `baseline` | `video_encoder.cpp:120` |
| `level` | `3.0` | `video_encoder.cpp:121` |
| Pixel format | YUV420P | `video_encoder.cpp:113` |
| GOP / B-frames | gop 30, max_b 3 | `video_encoder.cpp:115–116` |
| CRF | **nicht gesetzt** im aktiven `#else`-Block | `video_encoder.cpp:109–125` |

Legacy/unused CLI-Pfad mit `ffmpeg … -preset ultrafast` und `scale=W:H`: `time_lapse_control.cpp:280–284`.  
Weitere Legacy-Hilfe in `common_tools.cpp:309–311` hart `scale=640:360` (nicht der aktive Composite-Pfad laut `ai_camera.cpp`).

Export-API-Konstante: `EXPORT_TIMELAPSE_VIDEO` in `elegoo/common/method.h:50` (GUI/Host-seitig; Timelapse-Qualität steckt in den Werten oben).

---

## 3. Quality-limiting sites (vollständig für Timelapse)

| # | Limit | Wert | Datei:Zeile | Wirkung |
|---|-------|------|-------------|---------|
| 1 | Capture/Encode-Auflösung | 640×360 | `uvc_config_info.h:20–21`; genutzt `ai_camera.cpp:278–281`, `413` | Max. Pixeldetail der Timelapse |
| 2 | V4L2 Capture-FPS | 20 | `uvc_config_info.h:17` | Live-Stream/Capture-Rate (Timelapse nimmt Einzelbilder) |
| 3 | Output-Video-FPS | 30 | `uvc_config_info.h:18` | Abspielgeschwindigkeit der fertigen MP4 |
| 4 | JPEG-Puffer | 100 KiB (`PIC_MAX_SIZE`) | `time_lapse_photo.h:16`; Alloc `ai_camera.cpp:40` | Truncation-Risiko bei größeren JPEG |
| 5 | Frame-Queue-Slot | 100 KiB (`SIZE_OF_FRAME`) | `video_list.h:24` | Gleicher Caps auf Framegröße |
| 6 | H.264 bitrate | 2 Mbps | `video_encoder.cpp:114` | Blockiness bei Details/Bewegung |
| 7 | x264 preset | `ultrafast` | `video_encoder.cpp:118` | Schlechtere Kompressionseffizienz → schlechtere Qualität bei gleicher Bitrate |
| 8 | Kein CRF (CBR-ish bit_rate) | — | `video_encoder.cpp:109–125` | Weniger „quality target“-Steuerung |
| 9 | profile/level | baseline / 3.0 | `video_encoder.cpp:120–121` | Feature-/Effizienz-Limit |
| 10 | Nach Encode: JPEG-Löschung | `rm -rf` pic dir | `ai_camera.cpp:437–439` | Kein Re-Encode später möglich |
| 11 | Incomplete Timelapse verworfen | Temp gelöscht | `time_lapse_control.cpp:199–201` | Speicher, aber Datenverlust |
| 12 | Hardcoded scale in Legacy ffmpeg | 640:360 | `common_tools.cpp:310` | Nur relevant falls dieser Helper genutzt wird |

**Nicht gefunden (in Timelapse-Pfad):** explizites `crf`/`qp` im aktiven Encoder; JPEG-Qualitätsfaktor der UVC-Kamera (Hardware/Treiber); Auto-Delete alter **Videos** nach Quota (nur Bild-Cleanup nach Composite + Temp-Cleanup). Speicherlimit „max N videos“: **nicht** im `ai_camera`-Timelapse-Code gefunden.

---

## 4. Liefert die Kamera schon komprimierte Frames?

**Ja.** Timelapse nutzt `GetFrameOfMjpeg` und schreibt den Buffer unverändert als `.jpeg` (`TimeLapseSavePicture` → `Capture` → `SavePicture` fwrite). Format am Device: MJPEG (`V4L2_PIX_FMT_MJPEG`).

**Folge:**

- Software kann **Encoder-Qualität** der MP4 verbessern (preset/bitrate/CRF/profile).
- Software kann **Auflösung nur erhöhen**, wenn (a) die Kamera die höhere Mode akzeptiert (`VIDIOC_S_FMT`) und (b) Puffer ≥ Framegröße sind.
- JPEG-Artefakte aus der Kamera lassen sich durch Re-Encode **nicht** rückgängig machen; höchstens weniger zusätzlich zerstören.

---

## 5. Risiken CPU / RAM / Disk

| Ressource | Risiko | Begründung |
|-----------|--------|------------|
| **CPU** | Hoch bei aggressiverem Encode | `ultrafast` → `medium`/`slow` erhöht Encode-Last stark; Composite läuft auf dem Drucker-SoC (Allwinner/sunxi-Toolchain laut README). Während/nach Print kann UI/Print-Stack leiden. |
| **RAM** | Mittel | `stbi_load` dekodiert jedes JPEG voll nach RGB (`video_encoder.cpp:235`); 640×360×3 ≈ 0.7 MiB/Frame + YUV-Frame. Bei 1280×720 ≈ 2.8 MiB/Frame. Zusätzlich `pic_buf_` und VideoList (4×100 KiB). |
| **Disk** | Hoch bei längeren Prints / höherer Res | Ein JPEG ~tens of KiB bei 640×360; bei 1000 Layern leicht zig–hundert MiB unter `/opt/usr/picture/` bis Composite. Höhere Res + weniger JPEG-Kompression → Vielfaches. Videos unter `/opt/usr/video/` bleiben bis manuell gelöscht (**kein** Auto-Quota im untersuchten Pfad). |

**Vermutung:** `/opt/usr` liegt auf begrenztem eMMC/userdata — volle Disk bricht Capture/Encode ab.

---

## 6. Konfiguration heute

- **Keine** Timelapse-Qualitätsparameter in `debugfile/printer.cfg` gefunden (Suche: camera/timelapse/video/fps → nur bed `resolution=0.3`, irrelevant).
- Alles hardcodiert in Headern / `video_encoder.cpp`.

---

## 7. Offene Fragen an Manuel

1. Welche **Firmware-Version** läuft auf dem Gerät? (für BACKUP.md)
2. Kann die eingebaute Kamera per `v4l2-ctl --list-formats-ext` höhere Modi (z. B. 1280×720)?  
3. Wie viel freier Speicher unter `/opt/usr`?  
4. Ist Stockeln während Composite / nach Print akzeptabel für höhere Encode-Qualität?  
5. Soll Auflösungs-Upgrade Teil von Phase 3 sein, oder zuerst nur Encoder (sicheres Qualitätsplus)?

---

## 8. Nächster Schritt

→ Phase-2-Vorschlag: `/cursor/stores/self/docs/timelapse-vorschlag.md` + Patch `/cursor/stores/self/docs/timelapse-vorschlag.patch`  
**Kein Apply ohne Freigabe.**
