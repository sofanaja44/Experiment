# ReaCut → Android (Kotlin/Compose/Media3) — Spesifikasi Port

> Target stack: **Kotlin + Jetpack Compose + Media3 (ExoPlayer preview +
> Transformer export)**. Sumber: ReaCut (MIT). Dokumen ini merinci seluruh
> sistem edit ReaCut dan cara mem-port-nya. Pelengkap `reacut-core-engine.md`.

## 0. Lisensi

ReaCut = MIT → port adalah karya turunan. **Wajib** sertakan file `NOTICE`
dengan copyright ReaCut + teks MIT. Jangan masukkan kode/aset proprietary
CapCut/Jianying. Aman selama hanya source MIT ReaCut yang diadaptasi.

## 1. Sistem edit ReaCut (detail yang harus di-port akurat)

### 1.1 Model (`src/core/types.ts`)
`Project{version,id,name,width,height,fps,background,assets[],tracks[],clips[]}`
→ Kotlin data class immutable. Default proyek: 1920×1080 @30, background
`#101015`, 3 track awal (text, video, audio). `Clip` = unit atomik (lihat tabel
field di `reacut-core-engine.md`).

### 1.2 Primitif (`src/core/model.ts`) — port 1:1 sebagai fungsi murni
- `snapToFrame(t,fps) = round(t*fps)/fps` — **semua** waktu di-snap ke frame.
- `splitClip(project,clipId,time)`: belah di `at`; klip kanan `sourceIn +=
  leftDuration*speed`; keyframe & fade di-slice (lihat 1.3/1.4); minimal 1 frame
  per sisi.
- `trimClip(project,clipId,edge,time)`: ubah tepi, **clamp** terhadap batas
  sumber (`start` tak boleh lewat `start - sourceIn/speed`; `end` tak lewat
  `start + (asset.duration - sourceIn)/speed`). Image = tak terbatas.
- `changeClipSpeed(project,clipId,speed)`: `ratio = oldSpeed/newSpeed`;
  `duration *= ratio`; semua `keyframe.time *= ratio`; semua fade `*= ratio`.
  Rentang 0.1–8.
- `evaluateKeyframes(clip,time)`: interpolasi linear/easeInOut per properti.

### 1.3 Keyframe + easing (titik paling halus)
- Properti animatable: `x, y, scale, rotation, opacity, volume`.
- `easeInOut` memakai polinomial kubik dengan **easing window** `[start,end]`
  (potongan kurva smoothstep). Koefisien:
  `a=6·s(1−s)`, `b=3·Δ(1−2s)`, `c=−2Δ²` (Δ=end−start); `ratio' =
  ratio·(a+b·ratio+c·ratio²)/(a+b+c)`.
- `sliceKeyframes(clip,from,to)` (dipakai split/trim): **remap** interval kurva
  tanpa mengubah nilai/bentuk — termasuk menghitung ulang `easingWindow` untuk
  potongan. Port persis; ini sumber bug halus kalau disederhanakan.
- Batas: ≤400 keyframe/klip.

### 1.4 Fade (`FadeWindow`)
Fade disimpan sebagai envelope `{offset,duration,fadeIn,fadeOut}` relatif ke
timeline lokal klip, dipertahankan saat cut (`preserveFades` menggeser
`offset`). `evaluateFade = clamp(local/fadeIn) · clamp((dur−local)/fadeOut)`.

### 1.5 Operasi timeline (`src/core/timelineEdits.ts`)
- **3 mode** `placeAsset`: `insert` (ripple semua klip setelahnya),
  `overwrite` (potong hanya klip tujuan + pasangan link), `overlay`.
- **Linked clips** (`linkId`): video+audio yang dipisah tetap trim/split/geser
  bareng. `trimLinked` meng-clamp delta tepi terhadap handle sumber tiap member;
  `splitLinked` menghasilkan grup kiri/kanan independen; `detachLinkedAudio`
  memisah audio ke track audio baru dan menandai video `muted`.
- Semua menghormati track `locked` dan snap frame.

### 1.6 Validasi (`validateProject`) — jadikan invariant model Kotlin
fps ∈ {24,25,30,50,60}; dims 16–7680 (integer); speed 0.1–8; volume 0–4;
scale 1e-6–10; opacity 0–1; brightness −1..1; contrast/saturation 0..3;
temperature −1..1; blur 0..100; ≤10000 aset/klip, ≤1000 track, ≤400 keyframe;
durasi total ≤24 jam. Validator menolak data tak tepercaya tanpa coercion —
pertahankan sikap defensif ini saat load file proyek.

### 1.7 State & undo (`src/editor/store.ts`)
Zustand store → **ViewModel + StateFlow**. State: `project, path?, selectedIds,
playhead, playing, zoom, snapping, magnetic, dirty, past[], future[],
clipboard[]`. `commit(project)` push clone ke `past` (cap **150**), kosongkan
`future`. **Transaction** (begin/end) supaya satu drag = satu entri undo. Aksi:
place/split/trim/move/duplicate/copy/paste/addText/addSubtitles/track ops.

### 1.8 Render (ReaCut: FFmpeg) — konsep yang dipetakan ke Media3
Teks dirender via canvas → PNG overlay (`renderText`: multi-line, stroke,
background, font CJK fallback). Mask dirender ke PNG alpha (`renderMask`). Speed
video `setpts`, audio rantai `atempo`. Komposit `overlay ... enable='gte(t,
start)*lt(t,end)'`. Detail lengkap di `reacut-core-engine.md`.

## 2. Pemetaan ke Media3

Lihat tabel lengkap di skill **`android-video-editor`** (ReaCut concept →
Media3 target). Ringkas: track → `EditedMediaItemSequence`; overlap/PiP →
sequence terpisah dalam `Composition` atau `OverlayEffect`; trim →
`ClippingConfiguration`; transform → `ScaleAndRotateTransformation`+`Presentation`;
crop → `Crop`; adjustments → `Contrast`/`HslAdjustment`/color-matrix (+blur GL);
chromaKey & mask & temperature → **custom `GlShaderProgram`**; text → `Bitmap`
+`BitmapOverlay`; export → `Transformer` (H.264, mp4/mov).

**Risiko utama** (jujur): compositing berlapis dengan transform+keyframe per
klip lebih kaya di FFmpeg daripada Media3 bawaan → siapkan **GL effect custom**
untuk PiP, chroma-key, mask feather, blur. Kalau mentok, fallback render frame
via GL lalu umpan ke Transformer.

## 3. Milestone

1. **Core model Kotlin** + port operasi edit (model.ts + timelineEdits.ts) +
   **unit test JVM** (pakai kasus yang sama dgn ReaCut). Tanpa UI/perangkat.
2. **Load/save proyek** (JSON, bentuk sama) + `validateProject`.
3. **Timeline Compose** (track, klip, playhead, zoom, seleksi) + undo + transaksi.
4. **Preview**: ExoPlayer track dasar → GL compositing lapisan (evaluasi
   keyframe per frame).
5. **Export Media3**: 1 track → multi-track → efek → teks.
6. **Shader custom**: chroma-key, mask, blur, temperature.

Milestone 1–2 bisa dikerjakan & diuji **headless di server ini** (JVM, Gradle).
Milestone 3–6 butuh emulator/perangkat.

**Progres** (repo `sofanaja44/klipa-android`, branch `ccr-40d3051a-fe67to`):
- ✅ Milestone 1–2 selesai — `:core:model` (model, evaluasi keyframe/fade,
  `validateProject`, `ProjectJson`) + `:core:edit` (split/trim/speed + semua
  operasi linked/`placeAsset`). 49 test JUnit lulus
  (`./gradlew :core:model:test :core:edit:test`).
- ✅ Milestone 3 selesai — `EditorState` (port `store.ts`: undo 150 langkah,
  transaksi, clipboard, hapus magnetik), `EditorViewModel`, timeline Canvas
  (scrub+fling, pinch zoom, trim/move dengan snapping), monitor program
  (frame via `MediaMetadataRetriever`), import media, autosave. APK debug
  ter-build; 92 test lulus; lint bersih; screenshot via Roborazzi.
- ✅ Milestone 4 selesai (v0.4.0) — preview real-time dengan suara lewat
  Media3 `CompositionPlayer`: setiap track jadi sequence (gap eksplisit),
  sequence 0 = lapisan teratas (dicek di bytecode `DefaultCompositorGlProgram`),
  transform/keyframe/fade per frame via `VideoCompositorSettings`, fallback ke
  frame diam bila player gagal. Thumbnail & waveform di timeline. APK release
  (R8, ~1,9 MB) ditandatangani dev key yang di-commit. 107 test lulus, lint
  bersih. **Belum diuji di perangkat fisik** (server tanpa emulator/KVM).
- Temuan saat port: `CompositionPlayer` wajib `EditedMediaItem.setDurationUs`
  untuk semua item (bukan hanya gambar), dan gap di awal sequence butuh
  `trackTypes` dideklarasikan — keduanya tertangkap test Robolectric.
- 🐞 0.4.1 — laporan pengguna: "media tidak bergerak saat diputar" setelah
  upload. Akar masalah (dari dokumentasi resmi + demo Media3 + issue tracker):
  multi-input video graph dipakai untuk 1 track (jalur belum didukung resmi),
  player dipakai ulang antar-komposisi, reload player tiap waveform selesai,
  `setRemoveAudio` (#2854). Perbaikan: single-input untuk 1 track video, player
  baru tiap komposisi, `setPerStreamMediaProgressionEnabled` (#3453), gain 0
  untuk mute, reload hanya saat data playback berubah, fallback bertingkat,
  sheet "Diagnostik pemutaran" (menu ⋮) untuk laporan bug. Plus edit teks &
  volume, decoder thumbnail/monitor terpisah. 111 test lulus.
  Catatan: timestamp efek per-item = waktu komposisi (dikonfirmasi di source
  `PlaybackVideoGraphWrapper`).
- ✅ 0.4.1 dikonfirmasi pengguna: media sudah bisa diputar.
- ✅ Milestone 5 selesai (v0.5.0) — export MP4 (H.264/AAC) lewat `Transformer`
  dari `Composition` yang sama dengan preview; semua lapisan video dikomposit.
  Teks = `OverlayEffect` di level komposisi (setelah compositing), satu
  `BitmapOverlay` per *lane* teks yang tidak tumpang-tindih, bitmap dirender
  saat klip muncul (port `renderText` ReaCut, dipakai juga oleh monitor
  preview → teks identik di preview & export). Resolusi 480p/720p/1080p
  (`Presentation` di akhir), estimasi ukuran dari rumus bitrate default Media3
  (0,14 bit/piksel/frame, kini diset eksplisit), `FrameDropEffect` bila media
  lebih cepat dari fps proyek, HDR → SDR (OpenGL, API 29+). Hasil disalin ke
  `Movies/Klipa` via MediaStore (Android 8–9: izin storage; tanpa izin tetap
  bisa dibagikan via FileProvider). Progres + batal; preview melepas decoder
  selama export. 138 test lulus, lint bersih.
- Temuan saat port (dicek di bytecode Media3 1.11.1):
  - Efek komposisi (`Composition.effects`) diteruskan ke video graph tunggal
    maupun multi-input, jadi overlay teks berlaku di kedua jalur.
  - `BitmapOverlay` meng-upload tekstur hanya bila objek bitmap atau
    `generationId` berubah → bitmap per klip di-cache selama klip tampil.
  - Perubahan speed dipasang sebagai *pre-processing* sebelum audio processor
    item (CompositionPlayer & Transformer) → posisi di `GainAudioProcessor`
    sudah waktu timeline. Bug fade/volume pada klip ber-speed ≠ 1× diperbaiki.
  - Transformer menentukan foto vs video dari MIME `MediaItem`, atau
    `ContentResolver.getType` untuk URI `content://`.
- ✅ Milestone 6 selesai (v0.6.0) — satu shader GLSL ES 1.00 kustom
  (`core/media/src/main/res/raw/klipa_look_fragment_es2.glsl`) mem-port urutan &
  rumus 8-bit ReaCut: chroma key → mask → kecerahan/kontras (clamp antar tahap)
  → saturasi Rec.709 → suhu warna (±35 level R/B). Diuji di WebGL (Chromium
  headless, SwiftShader) terhadap fungsi asli ReaCut (`applyChromaKey`,
  `applyShapeMask`, `applyColorAdjustments`): 18 kasus, selisih ≤ 2/255
  (`tools/shader-check`). Blur = `SeparableConvolution` Media3 dengan sigma
  dibaca per frame dan resolusi diperkecil untuk radius besar. Alpha: lurus
  untuk compositor, premultiplied sebelum blur (+ shader unpremultiply), atau
  "di atas hitam" untuk 1 layer. UI: tool Atur (8 preset filter ReaCut +
  slider), Chroma key, Masker; slider look & volume tampil live tanpa reload
  player (`LiveProject` dibaca di thread GL/audio; signature playback hanya
  memuat ada/tidak look, potong alpha, bucket blur, volume ≠ 1). Frame diam
  memakai color matrix. 158 test lulus, lint bersih.
- Temuan saat port:
  - Pemetaan lama Klipa ke `Contrast`/`HslAdjustment` Media3 salah: Media3
    `Contrast(c)` memakai faktor (1+c)/(1−c), jadi kontras 2× ReaCut menjadi
    ekstrem; HSL saturation ≠ matriks Rec.709 ReaCut. Kini diganti shader.
  - `HslShaderProgram` Media3 menolak input HDR; shader Klipa tidak.
  - `SeparableConvolutionShaderProgram` membuat ulang tekstur kernel bila
    `getConvolution(t)` berubah (`equals`), dan sigma dihitung dalam piksel
    input walau output diperkecil.
  - `GlProgram` mendukung uniform int/float/vec/mat (bukan bool).
- ✅ v0.7.0 — edit langsung di monitor: ketuk untuk memilih (ketuk judul
  terpilih untuk edit teks), geser klip terpilih, cubit 2 jari untuk skala &
  rotasi; snap ke garis tengah, 100% dan sudut siku; keyframe properti yang
  sama ikut bergeser (`transformClip`). Hit-test & outline memakai `LayerBox`
  (ukuran media = fit ke frame proyek, teks = `textLayerSize` yang sama dengan
  render). Placement dibaca live oleh komposisi preview (matriks gerak selalu
  ada di mode 1 layer saat live; compositor membaca `LiveProject`), jadi
  menggeser klip tidak me-reload player. Tool **Overlay**: media jadi PiP di
  track video di atas track utama (`placeOverlays`, mode Overlay ReaCut), 50%.
  175 test lulus (termasuk smoke test gesture di aplikasi asli), lint bersih.
- ⏭ Berikutnya: konfirmasi export, efek & gesture di HP pengguna; lalu export
  di latar belakang (foreground service), tool crop, editor keyframe, daftar
  proyek.
- Catatan lingkungan build (lokal, tidak di-commit): Android SDK di
  `/root/android-sdk` (`local.properties`), dan init script
  `~/.gradle/init.d/maven-central-mirror.gradle.kts` yang memakai mirror
  Google untuk Maven Central karena `repo.maven.apache.org` membalas 429.
- `backups/klipa-android.bundle` hanya berisi scaffold awal; sumber utama
  sekarang repo GitHub di atas.

## 4. UI/UX

Ikuti skill **`crafted-ui`**: estetika pro-tool (gelap, padat, presisi),
timeline di-`Canvas` sendiri (ruler timecode, waveform, thumbnail, handle),
manipulasi langsung (drag trim/move, pinch zoom), semua state (empty/loading/
error), edge-to-edge, target sentuh ≥48dp, a11y. Target: tidak terlihat
template/AI-generated.

---
*Referensi source: `/home/user/xiangyu-cas/rea-capcut` (read-only, MIT).*
