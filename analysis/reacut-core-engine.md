# ReaCut — Arsitektur Mesin Edit Video Inti

> Editor video open-source (MIT) — repo `Xiangyu-CAS/rea-capcut`.
> Stack: Electron + React + TypeScript + FFmpeg. Editing jalan **lokal, tanpa
> akun/cloud/backend**. Legal untuk dipelajari & dimodifikasi (lisensi MIT).
> Analisis dari source langsung (bukan decompile).

## Gambaran besar

ReaCut adalah NLE (non-linear editor) klasik dengan satu ide inti yang elegan:
**seluruh timeline dikompilasi menjadi SATU pemanggilan FFmpeg `-filter_complex`.**
Tidak ada render per-klip lalu digabung — model proyek diterjemahkan langsung
jadi satu graf filter besar, lalu di-encode sekali dengan libx264.

Alurnya: **Model data (UI, React)** → **operasi edit (immutable)** → **compiler
graf FFmpeg** → **proses ffmpeg (Electron main)**.

## 1. Model data — `src/core/types.ts`

Hierarki: `Project` → `Track[]` + `Clip[]` + `MediaAsset[]`.

- **`Project`**: kanvas (`width/height/fps/background`) + daftar aset, track, klip.
- **`MediaAsset`**: media terimpor (video/audio/image) — `duration`, dimensi,
  `fps`, `hasAudio`, `waveform[]`, `thumbnail`.
- **`Track`**: lapisan (video/audio/image/text) dengan `muted/hidden/locked`.
- **`Clip`** — unit atomik editor, memuat seluruh "resep" render satu potongan:
  | Field | Fungsi |
  |---|---|
  | `start`, `duration` | posisi & panjang di timeline |
  | `sourceIn` | titik trim di dalam sumber |
  | `speed` | kecepatan putar (memengaruhi video & audio) |
  | `volume`, `muted` | audio |
  | `transform` | x/y/scale/rotation/opacity/flip |
  | `adjustments` | brightness/contrast/saturation/temperature/blur |
  | `text` | gaya teks (untuk track text) |
  | `fadeIn/Out`, `*FadeWindow` | fade video & audio |
  | `keyframes[]` | animasi properti seiring waktu |
  | `crop`, `chromaKey`, `mask` | potong, green-screen, masking bentuk |
  | `linkId` | mengikat klip video & audio terpisah agar tetap sinkron |

Model ini **murni data** (serializable ke JSON = format proyek, `version: 1`).

## 2. Operasi edit — `src/core/timelineEdits.ts`

Semua edit adalah **transformasi immutable**: terima `Project`, kembalikan
`Project` baru (pola cocok untuk undo/redo & state React). Semua menghormati
track `locked` dan snap ke grid frame (`snapToFrame`).

**Tiga mode penempatan** (`placeAsset`):
- `insert` — **ripple**: geser semua klip sesudahnya (sisipkan).
- `overwrite` — timpa hanya klip tujuan (+ pasangan ter-link).
- `overlay` — taruh di track lain tanpa mengganggu.

**Operasi inti:**
- `placeAsset` — masukkan aset ke track (validasi rentang sumber, buat track bila perlu).
- `splitLinked` — belah klip di titik waktu; menghasilkan grup kiri/kanan independen.
- `trimLinked` — ubah tepi (start/end), di-clamp terhadap handle sumber tiap member.
- `moveLinked` / `slipLinked` / `speedLinked` — geser posisi / geser isi sumber / ubah kecepatan.
- `detachLinkedAudio` — pisahkan audio dari video jadi track audio tersendiri (via `linkId`).

**Konsep "linked clips" (`linkId`)**: video & audio yang dipisah tetap bergerak,
ter-trim, dan ter-split bareng — ini yang bikin terasa seperti editor profesional.

## 3. Mesin render — `electron/export.ts`  ⭐ inti teknisnya

`createExportArguments(project, options)` membangun graf `-filter_complex`.
Untuk tiap track, tiap klip (urut `start`):

1. **Kecepatan**: `setpts=(PTS-STARTPTS)/speed` untuk video; audio pakai rantai
   `atempo` (`tempoFilters`: pecah jadi faktor 0.5–2.0 karena atempo dibatasi segitu).
2. **Skala & SAR**: `scale=W:H`, `setsar=1`.
3. **Green-screen** (`chromaKey`): `colorkey=warna:similarity:blend` lalu
   `alphaextract` → channel alpha `[keyalpha]`.
4. **Mask bentuk**: render mask → `alphaextract` → `[shapealpha]`.
5. **Keyframe** (`keyframeExpression`): daftar keyframe diubah jadi **ekspresi
   waktu FFmpeg** bertingkat `if(lt(t,...), interpolasi, ...)`. Easing `easeInOut`
   memakai polinomial kubik (koefisien dihitung analitik agar window kecil akurat),
   dievaluasi per-frame (`eval=frame`).
6. **Opacity & fade**: digeser lewat `setpts`/`asetpts`.
7. **Komposit**: ditumpuk dengan `overlay=x=...:y=...:enable='gte(t,start)*lt(t,start+dur)'`,
   membentuk rantai `[base0]→[base1]→…` (tiap klip jadi satu lapisan).
8. **Audio**: tiap klip `asetpts` + fade, lalu semua di-`amix=inputs=N:duration=longest`
   + `apad`/`atrim` + `alimiter` (cegah clipping).
9. **Output**: `format=yuv420p[vout]` → `-map [vout]` + `-c:v libx264`.

Jadi efek, transisi, animasi, dan layering semuanya **dinyatakan secara deklaratif
di dalam satu filtergraph**, bukan diproses manual frame-demi-frame di JS. Ini
kunci kenapa editornya ringan — kerja berat diserahkan ke FFmpeg.

## 4. Lokasi FFmpeg — `electron/mediaTools.ts`

`resolveMediaTool` mencari `ffmpeg`/`ffprobe`: override env (`REACUT_FFMPEG`),
lalu direktori bundel, lalu `PATH`, lalu `/opt/homebrew/bin`, `/usr/local/bin`,
`/usr/bin`. (Pesan error-nya dalam bahasa Mandarin.)

## 5. Soal "fitur AI"

Fitur AI generatif (portrait cutout, speech-to-subtitle) di README **belum
diimplementasikan**. Yang ada: **asisten LLM** (plugin Codex/Claude Code lokal,
`src/core/assistantEdits.ts` + `assistantTypes.ts`) yang memanipulasi **model
proyek yang sama** di atas — jadi "AI" di sini = agen yang memanggil operasi edit
di bagian 2, bukan model video/gambar bawaan.

## Peta file inti

| File | Peran |
|---|---|
| `src/core/types.ts` | Definisi model data (Project/Track/Clip/Asset) |
| `src/core/model.ts` | Primitif: createClip, trimClip, splitClip, changeClipSpeed, validateProject |
| `src/core/timelineEdits.ts` | Operasi edit tingkat-tinggi (place/split/trim/move/link) |
| `src/editor/store.ts` | State editor (React) |
| `src/editor/previewLayers.ts` | Preview real-time di kanvas |
| `electron/export.ts` | **Compiler timeline → graf FFmpeg + proses render** |
| `electron/mediaTools.ts` | Menemukan binary ffmpeg/ffprobe |
| `electron/capcutProjectIO.ts` | Impor proyek Jianying/CapCut (eksperimental, read-only) |

---
*Repo dibaca read-only via git proxy di `/home/user/xiangyu-cas/rea-capcut`.*
