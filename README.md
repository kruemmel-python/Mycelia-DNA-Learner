# Mycelia DNA Learner
<img width="2752" height="1536" alt="info_en" src="https://github.com/user-attachments/assets/96c8ab87-4962-4d10-938c-3fcdc33143f2" />

A learnable Python program for image and audio processing, inspired by three DNA sources:

- Bio vision hierarchy (YCbCr, ZCA, dictionary learning, k-WTA, temporal pooling)
- Enum-tag validation logic (strict/permissive type checks)
- Audio editor effect logic (noise reduction, trim, volume, speed, pitch, fade, echo, reverb, distort)

File: `mycelia_dna_learner.py`

---

## Contents

1. Goal and Features
2. Architecture and Learning Logic
3. Requirements
4. Installation
5. CLI Overview
6. Quick Start (with `data/images` and `data/audio`)
7. Detailed Commands and Parameters
8. Output Files
9. Troubleshooting
10. TODO (with Examples)
11. FAQ

---

## 1. Goal and Features

The program combines two learning pipelines:

- **Vision-DNA Pipeline**
  - Trains a multi-channel model on `Y`, `Cb`, `Cr`
  - Uses `ZCA`, sparse activation (`k-WTA`), dictionary atoms
  - Optional biologically inspired modulation (contact matrix + methyl state)
  - Reconstructs single images or sequences with temporal invariance (`max`/`avg` pooling)
  - Uses a stable postprocess by default for more realistic colors and tonal values

- **Audio-DNA Pipeline**
  - Learns an effect sequence ("genome") via evolutionary search
  - Optimizes against robust audio features (including RMS, ZCR, spectral flatness, band-energy ratios)
  - Can learn with or without target audio
  - Applies the learned genome to new WAV files

Additional:
- **Enum demo** for strict/permissive type validation in the style of JSDoc enum logic.

---

## 2. Architecture and Learning Logic

### Vision Branch

1. RGB is converted to YCbCr.
2. Patches are extracted per channel.
3. Patches are normalized with `ZCA`.
4. A dictionary is learned from whitened patches.
5. Activation: ReLU projection + `k-WTA` sparsity.
6. Optional bio modulation:
   - Contact matrix (`c`) for co-activation support
   - Methyl vector (`m`) for adaptive scaling
   - Online adapt modes:
     - `m`: adapt methyl only (stable)
     - `cm`: adapt methyl + contact matrix (more dynamic)
7. Decoding + overlap-add reconstruction (Hann or Rect window).
8. Stable postprocess (enabled by default):
   - Tone/chroma matching to input
   - Edge-aware chroma smoothing
   - Chroma compression against neon/oversaturation
   - Fidelity anchors to original (luma/chroma)
   - Delta clamps against implausible color/tonal drifts
   - light dithering against posterization
9. Temporal pooling over:
   - synthetic jitter frames (`reconstruct`)
   - real neighboring frames (`reconstruct_seq`)

### Audio Branch

1. Load mono WAV (`16-bit`, internal float32).
2. Genome = list of genes; each gene:
   - `op` (e.g. `echo`, `reverb`)
   - `params` (e.g. delay, decay, mix)
3. Evolution:
   - random initial population via mutations
   - scoring over robust features:
     - RMS, ZCR, centroid, spread, peak
     - spectral flatness
     - band energy ratios (low/mid/high)
   - feature distance (with target) or noise/energy heuristics (without target)
4. Best genome is saved as JSON and can be applied reproducibly.
5. Safety layer per gene:
   - DC offset removal
   - soft clip as emergency brake
   - minimum duration guard

### Enum Branch

- Validates dictionaries against expected types.
- `strict=False`: warnings
- `strict=True`: errors on type mismatch

---

## 3. Requirements

- Python 3.10+ (3.11 recommended)
- Packages:
  - `numpy`
  - `Pillow`
- Optional for MP3 conversion:
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

Optional checks:

```bash
python mycelia_dna_learner.py --help
python -m py_compile mycelia_dna_learner.py
```

---

## 5. CLI Overview

```bash
python mycelia_dna_learner.py <command> [options]
```

Available commands:

- `train_vision`
- `reconstruct`
- `reconstruct_seq`
- `learn_audio`
- `learn_audio_folder`
- `apply_audio`
- `enum_demo`

---

## 6. Quick Start (with `data/images` and `data/audio`)

### 6.1 Train vision model

```bash
python mycelia_dna_learner.py train_vision data/images \
  --model_out data/out/vision_dna_model.npz \
  --patch 8 --n_patches 8000 --epochs 5 \
  --y_dict 128 --cb_dict 40 --cr_dict 40 \
  --k_y 8 --k_cb 4 --k_cr 4 \
  --max_size 512 --seed 42
```

### 6.2 Reconstruct single image

```bash
python mycelia_dna_learner.py reconstruct data/images/ralf.png \
  --model data/out/vision_dna_model.npz \
  --out data/out/ralf_reconstruct.png \
  --stride 4 --window hann \
  --tp_len 5 --tp_mode max \
  --jitter_deg 4 --jitter_px 2
```

Note: This command already uses the stable default postprocess.

Baseline without postprocess (for A/B comparison only):

```bash
python mycelia_dna_learner.py reconstruct data/images/ralf.png \
  --model data/out/vision_dna_model.npz \
  --out data/out/ralf_reconstruct_baseline.png \
  --no_postprocess
```

### 6.2.1 Use existing model only (no retraining)

Batch reconstruction for all images in `data/images` with an already trained model:

```powershell
$model='data/out/vision_dna_model_hq.npz'; $outDir='data/out/reconstruct_images_hq'; New-Item -ItemType Directory -Force -Path $outDir | Out-Null; Get-ChildItem data/images -File | Where-Object { $_.Extension.ToLower() -in '.png','.jpg','.jpeg','.bmp','.webp' } | Sort-Object Name | ForEach-Object { $name=[System.IO.Path]::GetFileNameWithoutExtension($_.Name); python mycelia_dna_learner.py reconstruct $_.FullName --model $model --out (Join-Path $outDir ($name + '_recon.png')) --stride 2 --window hann --tp_len 1 --jitter_deg 0 --jitter_px 0 --max_size 1024 }
```

Single-image reconstruction with existing model:

```powershell
python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg --model data/out/vision_dna_model_hq.npz --out data/out/bild1002_recon.png --stride 2 --window hann --tp_len 1 --jitter_deg 0 --jitter_px 0 --max_size 1024
```

### 6.2.2 Audio controls vision reconstruction (new)

Single image with audio folder as control source (features are aggregated across all audio files):

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

As a one-liner (copy/paste):

```powershell
python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg --model data/out/vision_dna_model_hq.npz --out data/out/bild1002_recon_audio_ctrl.png --stride 2 --window hann --tp_len 1 --tp_mode max --audio_control data/audio --audio_control_strength 0.65 --audio_control_sr 16000 --audio_control_max_seconds 45
```

Frame sequence with audio file as control source:

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

Note: The console prints additional `[audio_control]` lines with features and derived parameters.

### 6.2.3 Reconstruct video frames and export MP4

Reconstruct frames from `data/vid` into an output folder:

```powershell
python mycelia_dna_learner.py reconstruct_seq data/vid --model data/out/vision_dna_model_hq.npz --out_dir data/out/reconstructed_video --tp_len 5 --tp_mode avg
```

Then merge generated frames (`00000.png`, `00001.png`, ...) into an MP4:

```powershell
ffmpeg -y -framerate 30 -i data/out/reconstructed_video/%05d.png -c:v libx264 -pix_fmt yuv420p -crf 18 data/out/reconstructed_video.mp4
```

### 6.3 Prepare audio (MP3 -> WAV)

```bash
ffmpeg -y -i data/audio/dreh_zurueck.mp3 -ac 1 -ar 16000 data/audio/dreh_zurueck.wav
```

### 6.4 Learn audio genome (single WAV)

Short learning run:

```bash
python mycelia_dna_learner.py learn_audio data/audio/dreh_zurueck.wav \
  --genome_out data/out/audio_genome.json \
  --preview_out data/out/audio_preview.wav \
  --iterations 120 --gene_len 4 --seed 7
```

Longer learning run:

```bash
python mycelia_dna_learner.py learn_audio data/audio/dreh_zurueck.wav \
  --genome_out data/out/audio_genome_long.json \
  --preview_out data/out/audio_preview_long.wav \
  --iterations 90 --gene_len 5 --seed 17
```

### 6.5 Learn audio genome (entire audio folder)

```bash
python mycelia_dna_learner.py learn_audio_folder data/audio \
  --sample_rate 16000 \
  --genome_out data/out/audio_genome_folder.json \
  --preview_out data/out/audio_preview_folder.wav \
  --iterations 120 --gene_len 5 --seed 17
```

Optional limits for very large folders:

```bash
python mycelia_dna_learner.py learn_audio_folder data/audio \
  --max_seconds_per_file 60 \
  --max_total_seconds 900 \
  --genome_out data/out/audio_genome_folder.json
```

### 6.6 Apply genome

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

## 7. Detailed Commands and Parameters

## `train_vision`

Trains the multi-channel vision model.

```bash
python mycelia_dna_learner.py train_vision <images_folder> [options]
```

Important options:

- `--model_out`: output model path (`.npz`)
- `--patch`: patch size (default `8`)
- `--n_patches`: number of training patches
- `--epochs`: dictionary training epochs
- `--y_dict`, `--cb_dict`, `--cr_dict`: atoms per channel
- `--k_y`, `--k_cb`, `--k_cr`: k-WTA sparsity per channel
- `--zca_eps`: ZCA regularization
- `--no_bio`: disables DNA/methyl modulation
- `--max_size`: image edge limit
- `--seed`: reproducibility

## `reconstruct`

Reconstructs a single image with optional jitter-based temporal pooling.

```bash
python mycelia_dna_learner.py reconstruct <image_file> [options]
```

Important options:

- `--model`: input model (`.npz`)
- `--out`: output image path
- `--stride`: patch stride
- `--window`: `hann` or `rect`
- `--tp_len`: number of temporal samples
- `--tp_mode`: `max` or `avg`
- `--recon_chunk`: patch batch size for memory-safe reconstruction (default `4096`)
- `--jitter_deg`, `--jitter_px`: synthetic frame variation
- `--online_adapt`: adaptive modulator updates while encoding
- `--adapt_mode`: `m` or `cm` (only with `--online_adapt`)
- `--audio_control`: audio file or folder for parameter-based vision control
- `--audio_control_strength`: blend strength `0..1` (how strongly audio overrides parameters)
- `--audio_control_sr`: sampling rate for feature analysis (default `16000`)
- `--audio_control_max_seconds`: max audio duration for analysis (default `30`)
- `--no_postprocess`: disable stable postprocess (comparison only)
- `--tone_match`: default `0.88`
- `--chroma_match`: default `0.92`
- `--chroma_smooth`: default `1.0`
- `--chroma_edge_strength`: default `8.0`
- `--detail_strength`: default `0.18`
- `--detail_radius`: default `1.2`
- `--dither_strength`: default `0.05`
- `--chroma_compress`: default `0.65`
- `--fidelity_luma_anchor`: default `0.70`
- `--fidelity_chroma_anchor`: default `0.86`
- `--max_luma_delta`: default `0.18`
- `--max_chroma_delta`: default `0.12`

## `reconstruct_seq`

Reconstructs all frames in a folder with pooling across neighboring frames.

```bash
python mycelia_dna_learner.py reconstruct_seq <frames_dir> [options]
```

Important options:

- `--out_dir`: output directory
- `--tp_len`: neighbor-frame window length
- `--tp_mode`: `max`/`avg`
- `--stride`, `--window`, `--online_adapt`
- `--adapt_mode`: `m` or `cm` (only with `--online_adapt`)
- `--recon_chunk`: patch batch size (reduce on OOM, e.g. `2048` or `1024`)
- `--audio_control`, `--audio_control_strength`, `--audio_control_sr`, `--audio_control_max_seconds`
- `--seed`: default `123`
- `--no_postprocess` and the same postprocess parameters as in `reconstruct`

Example (frames -> MP4):

```bash
python mycelia_dna_learner.py reconstruct_seq data/vid --model data/out/vision_dna_model_hq.npz --out_dir data/out/reconstructed_video --tp_len 5 --tp_mode avg
ffmpeg -y -framerate 30 -i data/out/reconstructed_video/%05d.png -c:v libx264 -pix_fmt yuv420p -crf 18 data/out/reconstructed_video.mp4
```

## `learn_audio`

Learns an audio genome.

```bash
python mycelia_dna_learner.py learn_audio <input_wav> [options]
```

Important options:

- `--target`: optional target WAV (match-oriented learning)
- `--genome_out`: JSON genome output
- `--preview_out`: best-candidate WAV
- `--metrics_out`: per-iteration log (`.json` or `.csv`)
- `--iterations`: number of search iterations
- `--gene_len`: initial genome length
- `--seed`: reproducibility

## `learn_audio_folder`

Learns an audio genome from all audio files in a folder. Supports `wav`, `mp3`, `flac`, `ogg`, `m4a` and more (decoded via ffmpeg).

```bash
python mycelia_dna_learner.py learn_audio_folder <audio_dir> [options]
```

Important options:

- `--target`: optional target audio file
- `--sample_rate`: shared training sample rate (default `16000`)
- `--max_seconds_per_file`: limit per file (`0` = full file)
- `--max_total_seconds`: global limit across files (`0` = unlimited)
- `--genome_out`: JSON with learned genome
- `--preview_out`: best-candidate WAV
- `--metrics_out`: per-iteration log (`.json` or `.csv`)
- `--iterations`: number of search iterations
- `--gene_len`: initial genome length
- `--seed`: reproducibility

## `apply_audio`

Applies an existing genome to a WAV file.

```bash
python mycelia_dna_learner.py apply_audio <input_wav> --genome <genome.json> --out <output.wav>
```

## `enum_demo`

Demonstrates strict/permissive type validation.

```bash
python mycelia_dna_learner.py enum_demo
python mycelia_dna_learner.py enum_demo --strict
```

---

## 8. Output Files

Typical artifacts:

- Vision:
  - `vision_dna_model.npz`
  - reconstructed images (`.png`)
  - sequence reconstructions (`out_seq/*.png`)
- Audio:
  - `audio_genome*.json`
  - `audio_preview*.wav`
  - `*_audio_out*.wav`
  - optional MP3 exports

Example from this project:

- `data/out/vision_dna_model.npz`
- `data/out/ralf_reconstruct.png`
- `data/out/reconstruct_images_stable/*.png`
- `data/out/audio_genome_long.json`
- `data/out/dreh_zurueck_audio_out_long.wav`
- `data/out/dreh_zurueck_audio_out_long.mp3`

---

## 9. Troubleshooting

- **`No images found`**
  - Check path and extensions (`.png`, `.jpg`, `.jpeg`, `.bmp`, `.webp`).

- **Audio learning takes too long**
  - Reduce iterations (`--iterations 50`)
  - learn on a short clip first, then apply to full file
  - keep sample rate at `16 kHz`

- **`Unsupported sample width`**
  - Convert audio to PCM WAV with ffmpeg:
    `ffmpeg -y -i input.mp3 -ac 1 -ar 16000 output.wav`

- **High memory/CPU load in vision**
  - reduce `--n_patches`, `--y_dict`
  - use a smaller `--max_size`
  - for reconstruction: reduce `--recon_chunk` (e.g. `2048` or `1024`)

- **`_ArrayMemoryError` in `reconstruct`**
  - Cause: too many patches allocated at once.
  - Fix: reduce `--recon_chunk` and/or limit `--max_size`.
  - Example:
    ```bash
    python mycelia_dna_learner.py reconstruct data/images/bild1002.jpg \
      --model data/out/vision_dna_model_hq.npz \
      --out data/out/bild1002_recon_audio_ctrl.png \
      --max_size 1536 \
      --recon_chunk 1024
    ```

- **Reconstruction looks too stylized/over-colored**
  - Check if `--no_postprocess` was accidentally enabled.
  - For more original fidelity: slightly increase `--fidelity_luma_anchor` and `--fidelity_chroma_anchor`.

- **Noise/banding in smooth regions**
  - keep `--dither_strength` low (`0.03..0.06`)
  - keep `--chroma_smooth` moderate (`0.8..1.2`)

---

## 10. TODO (with Examples)

1. **Adaptive vision presets (per image)**
   - Goal: derive postprocess parameters automatically from image statistics (portrait, landscape, low-light, high-contrast)
   - Example CLI:
     ```bash
     python mycelia_dna_learner.py reconstruct data/images/bild9.jpg \
       --model data/out/vision_dna_model_multi.npz \
       --preset auto
     ```

2. **Auto-parameter search for reconstruction**
   - Goal: small local search per image over `tone_match/chroma_match/anchors`, optimized for MSE/PSNR/SSIM
   - Example CLI:
     ```bash
     python mycelia_dna_learner.py tune_reconstruct data/images/bild9.jpg \
       --model data/out/vision_dna_model_multi.npz \
       --search_steps 24 \
       --out_params data/out/reconstruct_params_bild9.json
     ```

3. **Quality gates in batch mode**
   - Goal: mark batch as "passed" only if minimum quality is reached
   - Example CLI:
     ```bash
     python mycelia_dna_learner.py reconstruct_batch data/images \
       --model data/out/vision_dna_model_multi.npz \
       --min_psnr 18.0 \
       --max_mse 0.015
     ```

4. **Add perceptual metrics (vision)**
   - Goal: include SSIM and LPIPS-like distance in addition to MSE/PSNR (if available)
   - Example output:
     - `data/out/reconstruction_metrics_extended.csv`

5. **Region protection masks for faces**
   - Goal: stabilize eyes/lips/skin transitions separately to avoid segmentation-like artifacts
   - Example:
     ```bash
     python mycelia_dna_learner.py reconstruct data/images/portrait.jpg \
       --face_protect on
     ```

6. **Unit tests**
   - Goal: stable regression checks for core functions
   - Example:
     - ZCA roundtrip test
      - `k_wta` sparsity test
      - reconstruction postprocess bounds/anchor test
      - audio operator range/NaN test

---

## 11. FAQ

**Q: Can I use MP3 directly?**  
A: In `learn_audio`, no (WAV only). In `learn_audio_folder`, yes, if `ffmpeg` is installed.

**Q: Why are there two audio outputs (`normal` and `long`)?**  
A: Those are two separate learning runs with different parameters/iterations.

**Q: Why is output duration sometimes shorter than input?**  
A: Some operators (e.g. `trim`, `speed`) change duration. That is part of the learned genome.

**Q: How do I make runs reproducible?**  
A: Use fixed seeds (`--seed`) for training and learning.

**Q: Which audio formats can the script itself write?**  
A: WAV. For MP3, use ffmpeg as a separate export step.

**Q: How do I disable bio-inspired vision modulation?**  
A: Set `--no_bio` during training.

**Q: What is recommended for online adapt in vision?**  
A: `--online_adapt --adapt_mode m` is usually more stable than `cm`.

**Q: Is stable image mode enabled by default?**  
A: Yes. `reconstruct` and `reconstruct_seq` use the stable postprocess by default.

**Q: How do I get the old comparison mode without stabilization?**  
A: Set `--no_postprocess`.

**Q: How do I get observability in audio learning?**  
A: Set `--metrics_out` (JSON or CSV) to get an iteration log.

**Q: What is a good starting point for quick tests?**  
A: Vision with a small dictionary (`y_dict=64`) and audio with `iterations=30..60`.

**Q: Where do I see which files were generated?**  
A: In the output folder, e.g. `data/out`.

**Q: Is it correct that the system combines image and audio and creates a new "heard-and-seen" image?**  
A: Yes, in this sense: image structure comes from vision reconstruction (patch codes), and audio is not mixed in as pixels. Instead, audio drives reconstruction parameters (e.g. `tp_len`, `tp_mode`, `detail_strength`, `chroma_smooth`) via `--audio_control`. The result is an audio-modulated, visually reconstructed image.
