# Mycelia DNA Learner
<img width="2752" height="1536" alt="info" src="https://github.com/user-attachments/assets/2b6ebedf-e32b-451e-b8bb-c592bfba9d7a" />

Ein lernfaehiges Python-Programm fuer Bild- und Audioverarbeitung, inspiriert von drei DNA-Quellen:

- Bio-Vision-Hierarchie (YCbCr, ZCA, Dictionary Learning, k-WTA, Temporal Pooling)
- Enum-Tag-Validierungslogik (strict/permissive Typpruefung)
- Audio-Editor-Effektlogik (Rauschreduktion, Trim, Volume, Speed, Pitch, Fade, Echo, Reverb, Distort)

Datei: `mycelia_dna_learner.py`

---

## Inhalt

1. Ziel und Funktionen
2. Architektur und Lernlogik
3. Voraussetzungen
4. Installation
5. CLI Uebersicht
6. Schnellstart (mit `data/images` und `data/audio`)
7. Detaillierte Befehle und Parameter
8. Output-Dateien
9. Troubleshooting
10. TODO (mit Beispielen)
11. FAQ

---

## 1. Ziel und Funktionen

Das Programm kombiniert zwei lernende Pipelines:

- **Vision-DNA Pipeline**
  - Trainiert ein mehrkanaliges Modell auf `Y`, `Cb`, `Cr`
  - Nutzt `ZCA`, sparse Aktivierung (`k-WTA`), Dictionary-Atome
  - Optional mit biologisch inspirierter Modulation (Kontaktmatrix + Methyl-Status)
  - Rekonstruiert Einzelbilder oder Sequenzen mit temporaler Invarianz (`max`/`avg` Pooling)
  - Nutzt standardmaessig einen stabilen Postprocess fuer realitaetsnaehere Farben/Tonwerte

- **Audio-DNA Pipeline**
  - Lernt eine Effektsequenz ("Genom") ueber evolutionaere Suche
  - Optimiert auf Basis robuster Audio-Features (u. a. RMS, ZCR, Spectral Flatness, Band-Energy-Ratios)
  - Kann mit oder ohne Target-Audio lernen
  - Wendet gelerntes Genom auf neue WAV-Dateien an

Zusatz:
- **Enum-Demo** fuer strict/permissive Typvalidierung im Stil der JSDoc-Enum-Logik.

---

## 2. Architektur und Lernlogik

### Vision-Zweig

1. RGB wird zu YCbCr konvertiert.
2. Pro Kanal werden Patches extrahiert.
3. Patches werden ueber `ZCA` normalisiert.
4. Ein Dictionary wird aus den whitened Patches gelernt.
5. Aktivierung: ReLU-Projektion + `k-WTA` Sparsity.
6. Optional: Bio-Modulation
   - Kontaktmatrix (`c`) fuer Co-Aktivierungsunterstuetzung
   - Methyl-Vektor (`m`) fuer adaptive Skalierung
   - Online-Adapt-Modi:
     - `m`: nur Methyl adaptieren (stabil)
     - `cm`: Methyl + Kontaktmatrix adaptieren (dynamischer)
7. Dekodierung + Overlap-Add Rekonstruktion (Hann oder Rect Window).
8. Stabiler Postprocess (default aktiv):
   - Tone/Chroma-Matching zur Eingabe
   - Edge-aware Chroma-Smoothing
   - Chroma-Compression gegen Neon/Uebersaettigung
   - Fidelity-Anker auf Original (Luma/Chroma)
   - Delta-Clamps gegen unplausible Farb-/Tonwertdrifts
   - leichtes Dithering gegen Posterization
9. Temporal Pooling ueber:
   - kuenstliche Jitter-Frames (`reconstruct`)
   - echte Nachbarframes (`reconstruct_seq`)

### Audio-Zweig

1. Lade Mono-WAV (`16-bit`, intern float32).
2. Genom = Liste von Genen; jedes Gen:
   - `op` (z. B. `echo`, `reverb`)
   - `params` (z. B. Delay, Decay, Mix)
3. Evolution:
   - random initial population ueber Mutationen
   - scoring ueber robuste Features:
     - RMS, ZCR, Centroid, Spread, Peak
     - Spectral Flatness
     - Band Energy Ratios (Low/Mid/High)
   - Feature-Abstand (mit Target) oder Noise/Energy-Heuristik (ohne Target)
4. Bestes Genom wird als JSON gespeichert und kann reproduzierbar angewendet werden.
5. Safety-Layer pro Gen:
   - DC-Offset-Entfernung
   - Soft-Clip als Notbremse
   - Mindestdauer-Sicherung

### Enum-Zweig

- Validiert Dictionaries gegen erwartete Typen.
- `strict=False`: Warnungen
- `strict=True`: Fehler bei Typabweichungen

---

## 3. Voraussetzungen

- Python 3.10+ (empfohlen 3.11)
- Pakete:
  - `numpy`
  - `Pillow`
- Optional fuer MP3-Konvertierung:
  - `ffmpeg`

---

## 4. Installation

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Linux/macOS:
# source .venv/bin/activate

pip install numpy pillow
```

Optional testen:

```bash
python mycelia_dna_learner.py --help
python -m py_compile mycelia_dna_learner.py
```

---

## 5. CLI Uebersicht

```bash
python mycelia_dna_learner.py <command> [options]
```

Verfuegbare Commands:

- `train_vision`
- `reconstruct`
- `reconstruct_seq`
- `learn_audio`
- `learn_audio_folder`
- `apply_audio`
- `enum_demo`

---

## 6. Schnellstart (mit `data/images` und `data/audio`)

### 6.1 Vision trainieren

```bash
python mycelia_dna_learner.py train_vision data/images \
  --model_out data/out/vision_dna_model.npz \
  --patch 8 --n_patches 8000 --epochs 5 \
  --y_dict 128 --cb_dict 40 --cr_dict 40 \
  --k_y 8 --k_cb 4 --k_cr 4 \
  --max_size 512 --seed 42
```

### 6.2 Einzelbild rekonstruierten

```bash
python mycelia_dna_learner.py reconstruct data/images/ralf.png \
  --model data/out/vision_dna_model.npz \
  --out data/out/ralf_reconstruct.png \
  --stride 4 --window hann \
  --tp_len 5 --tp_mode max \
  --jitter_deg 4 --jitter_px 2
```

Hinweis: Dieser Befehl verwendet bereits den stabilen Standard-Postprocess.

Baseline ohne Postprocess (nur fuer A/B-Vergleich):

```bash
python mycelia_dna_learner.py reconstruct data/images/ralf.png \
  --model data/out/vision_dna_model.npz \
  --out data/out/ralf_reconstruct_baseline.png \
  --no_postprocess
```

### 6.2.1 Nur vorhandenes Modell nutzen (ohne erneutes Training)

Batch-Rekonstruktion fuer alle Bilder in `data/images` mit einem bereits trainierten Modell:

```powershell
$model='data/out/vision_dna_model_hq.npz'; $outDir='data/out/reconstruct_images_hq'; New-Item -ItemType Directory -Force -Path $outDir | Out-Null; Get-ChildItem data/images -File | Where-Object { $_.Extension.ToLower() -in '.png','.jpg','.jpeg','.bmp','.webp' } | Sort-Object Name | ForEach-Object { $name=[System.IO.Path]::GetFileNameWithoutExtension($_.Name); python mycelia_dna_learner.py reconstruct $_.FullName --model $model --out (Join-Path $outDir ($name + '_recon.png')) --stride 2 --window hann --tp_len 1 --jitter_deg 0 --jitter_px 0 --max_size 1024 }
```

Einzelbild-Rekonstruktion mit vorhandenem Modell:

```powershell
python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg --model data/out/vision_dna_model_hq.npz --out data/out/bild1002_recon.png --stride 2 --window hann --tp_len 1 --jitter_deg 0 --jitter_px 0 --max_size 1024
```

### 6.2.2 Audio steuert Vision-Rekonstruktion (neu)

Einzelbild mit Audio-Ordner als Steuerung (Features werden ueber alle Audiodateien gemittelt):

```powershell
python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg \
  --model data/out/vision_dna_model_hq.npz \
  --out data/out/bild1002_recon_audio_ctrl.png \
  --stride 2 --window hann --tp_len 1 --tp_mode max \
  --audio_control data/audio \
  --audio_control_strength 0.65 \
  --audio_control_sr 16000 \
  --audio_control_max_seconds 45
```

Als Einzeiler (copy/paste):

```powershell
python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg --model data/out/vision_dna_model_hq.npz --out data/out/bild1002_recon_audio_ctrl.png --stride 2 --window hann --tp_len 1 --tp_mode max --audio_control data/audio --audio_control_strength 0.65 --audio_control_sr 16000 --audio_control_max_seconds 45
```

Frame-Sequenz mit Audio-Datei als Steuerung:

```powershell
python mycelia_dna_learner.py reconstruct_seq data/images \
  --model data/out/vision_dna_model_hq.npz \
  --out_dir data/out/reconstruct_seq_audio_ctrl \
  --stride 2 --tp_len 5 --tp_mode avg --window hann \
  --audio_control data/audio/dreh_zurueck.wav \
  --audio_control_strength 0.7 \
  --audio_control_sr 16000 \
  --audio_control_max_seconds 30
```

Hinweis: In der Konsole erscheinen zusaetzliche `[audio_control]`-Zeilen mit Features und den abgeleiteten Parametern.

### 6.2.3 Video-Frames rekonstruieren und MP4 erzeugen

Frames aus `data/vid` rekonstruiert in einen Ausgabeordner schreiben:

```powershell
python mycelia_dna_learner.py reconstruct_seq data/vid --model data/out/vision_dna_model_hq.npz --out_dir data/out/reconstructed_video --tp_len 5 --tp_mode avg
```

Anschliessend die erzeugten Frames (`00000.png`, `00001.png`, ...) zu einem MP4 zusammenfuehren:

```powershell
ffmpeg -y -framerate 30 -i data/out/reconstructed_video/%05d.png -c:v libx264 -pix_fmt yuv420p -crf 18 data/out/reconstructed_video.mp4
```

### 6.3 Audio vorbereiten (MP3 -> WAV)

```bash
ffmpeg -y -i data/audio/dreh_zurueck.mp3 -ac 1 -ar 16000 data/audio/dreh_zurueck.wav
```

### 6.4 Audio-Genom lernen (einzelne WAV)

Kurzer Lernlauf:

```bash
python mycelia_dna_learner.py learn_audio data/audio/dreh_zurueck.wav \
  --genome_out data/out/audio_genome.json \
  --preview_out data/out/audio_preview.wav \
  --iterations 120 --gene_len 4 --seed 7
```

Laengerer Lernlauf:

```bash
python mycelia_dna_learner.py learn_audio data/audio/dreh_zurueck.wav \
  --genome_out data/out/audio_genome_long.json \
  --preview_out data/out/audio_preview_long.wav \
  --iterations 90 --gene_len 5 --seed 17
```

### 6.5 Audio-Genom lernen (ganzer Audio-Ordner)

```bash
python mycelia_dna_learner.py learn_audio_folder data/audio \
  --sample_rate 16000 \
  --genome_out data/out/audio_genome_folder.json \
  --preview_out data/out/audio_preview_folder.wav \
  --iterations 120 --gene_len 5 --seed 17
```

Optionale Begrenzung bei sehr grossen Ordnern:

```bash
python mycelia_dna_learner.py learn_audio_folder data/audio \
  --max_seconds_per_file 60 \
  --max_total_seconds 900 \
  --genome_out data/out/audio_genome_folder.json
```

### 6.6 Genom anwenden

```bash
python mycelia_dna_learner.py apply_audio data/audio/dreh_zurueck.wav \
  --genome data/out/audio_genome_long.json \
  --out data/out/dreh_zurueck_audio_out_long.wav
```

### 6.7 Optional WAV -> MP3

```bash
ffmpeg -y -i data/out/dreh_zurueck_audio_out_long.wav -codec:a libmp3lame -b:a 192k data/out/dreh_zurueck_audio_out_long.mp3
```

---

## 7. Detaillierte Befehle und Parameter

## `train_vision`

Trainiert das mehrkanalige Vision-Modell.

```bash
python mycelia_dna_learner.py train_vision <images_folder> [options]
```

Wichtige Optionen:

- `--model_out`: Pfad zur Modellausgabe (`.npz`)
- `--patch`: Patchgroesse (Default `8`)
- `--n_patches`: Anzahl Trainingspatches
- `--epochs`: Dictionary-Trainingsepochen
- `--y_dict`, `--cb_dict`, `--cr_dict`: Atome pro Kanal
- `--k_y`, `--k_cb`, `--k_cr`: k-WTA Sparsity pro Kanal
- `--zca_eps`: ZCA Regularisierung
- `--no_bio`: deaktiviert DNA/Methyl Modulation
- `--max_size`: Bildkantenlimit
- `--seed`: Reproduzierbarkeit

## `reconstruct`

Rekonstruiert ein Einzelbild mit optionalem Jitter-basiertem Temporal Pooling.

```bash
python mycelia_dna_learner.py reconstruct <image_file> [options]
```

Wichtige Optionen:

- `--model`: Eingangsmodell (`.npz`)
- `--out`: Zielbild
- `--stride`: Patch-Schrittweite
- `--window`: `hann` oder `rect`
- `--tp_len`: Anzahl zeitlicher Samples
- `--tp_mode`: `max` oder `avg`
- `--recon_chunk`: Patch-Batchgroesse fuer speicherschonende Rekonstruktion (Default `4096`)
- `--jitter_deg`, `--jitter_px`: kuenstliche Frame-Variation
- `--online_adapt`: adaptive Modulator-Updates beim Encodieren
- `--adapt_mode`: `m` oder `cm` (nur mit `--online_adapt`)
- `--audio_control`: Audio-Datei oder Audio-Ordner zur parameterbasierten Vision-Steuerung
- `--audio_control_strength`: Mischstaerke `0..1` (wie stark Audio die Parameter ueberschreibt)
- `--audio_control_sr`: Samplingrate fuer Feature-Analyse (Default `16000`)
- `--audio_control_max_seconds`: maximale Audiodauer fuer Analyse (Default `30`)
- `--no_postprocess`: stabilen Postprocess deaktivieren (nur Vergleichszwecke)
- `--tone_match`: Default `0.88`
- `--chroma_match`: Default `0.92`
- `--chroma_smooth`: Default `1.0`
- `--chroma_edge_strength`: Default `8.0`
- `--detail_strength`: Default `0.18`
- `--detail_radius`: Default `1.2`
- `--dither_strength`: Default `0.05`
- `--chroma_compress`: Default `0.65`
- `--fidelity_luma_anchor`: Default `0.70`
- `--fidelity_chroma_anchor`: Default `0.86`
- `--max_luma_delta`: Default `0.18`
- `--max_chroma_delta`: Default `0.12`

## `reconstruct_seq`

Rekonstruiert alle Frames in einem Ordner mit Pooling ueber Nachbarframes.

```bash
python mycelia_dna_learner.py reconstruct_seq <frames_dir> [options]
```

Wichtige Optionen:

- `--out_dir`: Ausgabeverzeichnis
- `--tp_len`: Fensterlaenge ueber Nachbarframes
- `--tp_mode`: `max`/`avg`
- `--stride`, `--window`, `--online_adapt`
- `--adapt_mode`: `m` oder `cm` (nur mit `--online_adapt`)
- `--recon_chunk`: Patch-Batchgroesse (bei OOM kleiner setzen, z. B. `2048` oder `1024`)
- `--audio_control`, `--audio_control_strength`, `--audio_control_sr`, `--audio_control_max_seconds`
- `--seed`: Default `123`
- `--no_postprocess` und dieselben Postprocess-Parameter wie bei `reconstruct`

Beispiel (Frames -> MP4):

```bash
python mycelia_dna_learner.py reconstruct_seq data/vid --model data/out/vision_dna_model_hq.npz --out_dir data/out/reconstructed_video --tp_len 5 --tp_mode avg
ffmpeg -y -framerate 30 -i data/out/reconstructed_video/%05d.png -c:v libx264 -pix_fmt yuv420p -crf 18 data/out/reconstructed_video.mp4
```

## `learn_audio`

Lernt ein Audio-Genom.

```bash
python mycelia_dna_learner.py learn_audio <input_wav> [options]
```

Wichtige Optionen:

- `--target`: optionales Ziel-WAV (match-orientiertes Lernen)
- `--genome_out`: JSON mit Genom
- `--preview_out`: Best-Candidate WAV
- `--metrics_out`: Iterations-Log (`.json` oder `.csv`)
- `--iterations`: Anzahl Suchiterationen
- `--gene_len`: Startlaenge des Genoms
- `--seed`: Reproduzierbarkeit

## `learn_audio_folder`

Lernt ein Audio-Genom aus allen Audiodateien im Ordner. Unterstuetzt u. a. `wav`, `mp3`, `flac`, `ogg`, `m4a` (Dekodierung ueber ffmpeg).

```bash
python mycelia_dna_learner.py learn_audio_folder <audio_dir> [options]
```

Wichtige Optionen:

- `--target`: optionales Ziel-Audiofile
- `--sample_rate`: gemeinsame Trainings-Samplingrate (Default `16000`)
- `--max_seconds_per_file`: Begrenzung pro Datei (`0` = volle Datei)
- `--max_total_seconds`: Begrenzung ueber alle Dateien (`0` = unbegrenzt)
- `--genome_out`: JSON mit gelerntem Genom
- `--preview_out`: Best-Candidate WAV
- `--metrics_out`: Iterations-Log (`.json` oder `.csv`)
- `--iterations`: Anzahl Suchiterationen
- `--gene_len`: Startlaenge des Genoms
- `--seed`: Reproduzierbarkeit

## `apply_audio`

Wendet ein bestehendes Genom auf ein WAV an.

```bash
python mycelia_dna_learner.py apply_audio <input_wav> --genome <genome.json> --out <output.wav>
```

## `enum_demo`

Demonstriert Typvalidierung strict/permissive.

```bash
python mycelia_dna_learner.py enum_demo
python mycelia_dna_learner.py enum_demo --strict
```

---

## 8. Output-Dateien

Typische Artefakte:

- Vision:
  - `vision_dna_model.npz`
  - rekonstruierte Bilder (`.png`)
  - Sequenz-Rekonstruktionen (`out_seq/*.png`)
- Audio:
  - `audio_genome*.json`
  - `audio_preview*.wav`
  - `*_audio_out*.wav`
  - optional MP3 Exporte

Beispiel aus diesem Projekt:

- `data/out/vision_dna_model.npz`
- `data/out/ralf_reconstruct.png`
- `data/out/reconstruct_images_stable/*.png`
- `data/out/audio_genome_long.json`
- `data/out/dreh_zurueck_audio_out_long.wav`
- `data/out/dreh_zurueck_audio_out_long.mp3`

---

## 9. Troubleshooting

- **`No images found`**
  - Pruefe den Pfad und Dateiendungen (`.png`, `.jpg`, `.jpeg`, `.bmp`, `.webp`).

- **Audio-Lernen dauert zu lange**
  - Iterationen reduzieren (`--iterations 50`)
  - zuerst auf kurzem Clip lernen und dann auf volle Datei anwenden
  - Samplingrate auf `16 kHz` halten

- **`Unsupported sample width`**
  - Audio per ffmpeg in PCM WAV umwandeln:
    `ffmpeg -y -i input.mp3 -ac 1 -ar 16000 output.wav`

- **Speicher/CPU Last hoch bei Vision**
  - `--n_patches`, `--y_dict` senken
  - `--max_size` kleiner setzen
  - bei Rekonstruktion: `--recon_chunk` kleiner setzen (z. B. `2048` oder `1024`)

- **`_ArrayMemoryError` bei `reconstruct`**
  - Ursache: zu viele Patches gleichzeitig im RAM.
  - Loesung: `--recon_chunk` reduzieren und/oder `--max_size` begrenzen.
  - Beispiel:
    ```bash
    python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg \
      --model data/out/vision_dna_model_hq.npz \
      --out data/out/bild1002_recon_audio_ctrl.png \
      --max_size 1536 \
      --recon_chunk 1024
    ```

- **Rekonstruktion wirkt zu stilisiert/farbig**
  - Pruefe, ob `--no_postprocess` versehentlich aktiv ist.
  - Fuer noch mehr Originaltreue: `--fidelity_luma_anchor` und `--fidelity_chroma_anchor` leicht erhoehen.

- **Rauschen/Stufen in glatten Flaechen**
  - `--dither_strength` klein halten (`0.03..0.06`)
  - `--chroma_smooth` moderat lassen (`0.8..1.2`)

---

## 10. TODO (mit Beispielen)

1. **Adaptive Vision-Presets (pro Bild)**
   - Ziel: Postprocess-Parameter automatisch aus Bildstatistiken ableiten (Portrait, Landschaft, Low-Light, High-Contrast)
   - Beispiel-CLI:
     ```bash
     python mycelia_dna_learner.py reconstruct data/images/bild9.jpg \
       --model data/out/vision_dna_model_multi.npz \
       --preset auto
     ```

2. **Auto-Parameter-Suche fuer Rekonstruktion**
   - Ziel: pro Bild kleine lokale Suche ueber `tone_match/chroma_match/anchors`, optimiert auf MSE/PSNR/SSIM
   - Beispiel-CLI:
     ```bash
     python mycelia_dna_learner.py tune_reconstruct data/images/bild9.jpg \
       --model data/out/vision_dna_model_multi.npz \
       --search_steps 24 \
       --out_params data/out/reconstruct_params_bild9.json
     ```

3. **Qualitaets-Gates im Batch-Modus**
   - Ziel: Batch nur dann als \"bestanden\" markieren, wenn Mindestqualitaet erreicht ist
   - Beispiel-CLI:
     ```bash
     python mycelia_dna_learner.py reconstruct_batch data/images \
       --model data/out/vision_dna_model_multi.npz \
       --min_psnr 18.0 \
       --max_mse 0.015
     ```

4. **Perceptual-Metriken ergaenzen (Vision)**
   - Ziel: neben MSE/PSNR auch SSIM, LPIPS-aehnliche Distanz (falls verfuegbar)
   - Beispiel-Output:
     - `data/out/reconstruction_metrics_extended.csv`

5. **Region-Schutzmasken fuer Gesichter**
   - Ziel: Augen/Lippen/Hautuebergaenge gesondert stabilisieren, um Segmentierungslook zu vermeiden
   - Beispiel:
     ```bash
     python mycelia_dna_learner.py reconstruct data/images/portrait.jpg \
       --face_protect on
     ```

6. **Unit-Tests**
   - Ziel: stabile Regression-Checks fuer Kernfunktionen
   - Beispiel:
     - ZCA Roundtrip Test
      - `k_wta` Sparsity Test
      - Rekonstruktions-Postprocess Bounds/Anchor Test
      - Audio-Operator Range/NaN Test

---

## 11. FAQ

**F: Kann ich MP3 direkt benutzen?**  
A: In `learn_audio` nein (nur WAV). In `learn_audio_folder` ja, wenn `ffmpeg` installiert ist.

**F: Warum gibt es zwei Audio-Outputs (`normal` und `long`)?**  
A: Das sind zwei verschiedene Lernlaeufe mit unterschiedlichen Parametern/Iterationen.

**F: Warum ist die Ausgabezeit manchmal kuerzer als der Input?**  
A: Manche Operatoren (z. B. `trim`, `speed`) veraendern die Dauer. Das ist Teil des gelernten Genoms.

**F: Wie mache ich den Lauf reproduzierbar?**  
A: Nutze feste Seeds (`--seed`) fuer Training und Lernen.

**F: Welche Audio-Formate werden vom Script selbst geschrieben?**  
A: WAV. Fuer MP3 nutze ffmpeg als separaten Export-Schritt.

**F: Wie deaktiviere ich bio-inspirierte Vision-Modulation?**  
A: Beim Trainieren `--no_bio` setzen.

**F: Was ist fuer Online-Adapt in Vision empfehlenswert?**  
A: `--online_adapt --adapt_mode m` ist meist stabiler als `cm`.

**F: Ist der stabile Bildmodus standardmaessig aktiv?**  
A: Ja. `reconstruct` und `reconstruct_seq` nutzen default den stabilen Postprocess.

**F: Wie bekomme ich den alten Vergleichsmodus ohne Stabilisierung?**  
A: `--no_postprocess` setzen.

**F: Wie bekomme ich Messbarkeit im Audio-Lernen?**  
A: `--metrics_out` setzen (JSON oder CSV), dann bekommst du ein Iterations-Log.

**F: Was ist ein guter Start fuer schnelle Tests?**  
A: Vision mit kleinem Dictionary (`y_dict=64`) und Audio mit `iterations=30..60`.

**F: Wo sehe ich, welche Dateien erzeugt wurden?**  
A: Im Output-Ordner, z. B. `data/out`.

**F: Sehe ich das richtig, dass das System Bild und Audio kombiniert und daraus ein neues \"gehoertes-und-gesehenes\" Bild erstellt?**  
A: Ja, in diesem Sinn: Die Bildstruktur kommt aus der Vision-Rekonstruktion (Patch-Codes), und Audio wird nicht als Pixel \"hineingemischt\", sondern steuert Rekonstruktionsparameter (z. B. `tp_len`, `tp_mode`, `detail_strength`, `chroma_smooth`) via `--audio_control`. Das Ergebnis ist ein audio-moduliertes, visuell rekonstruiertes Bild.
