---
name: android-video-editor
description: >
  Build or port a native Android video editor in Kotlin with Jetpack Compose
  for UI and AndroidX Media3 (ExoPlayer for preview, Transformer for export).
  Use when implementing a timeline/NLE editor on Android: a project/track/clip
  data model, a scrubbable multi-track timeline UI, real-time preview
  compositing, clip operations (split/trim/ripple/speed/keyframes/fades), text
  and chroma-key and mask layers, and exporting the timeline to MP4/MOV. Also
  use when porting the MIT-licensed ReaCut editor (TS/Electron/FFmpeg) to
  Android. Covers how ReaCut's FFmpeg filter_complex concepts map onto Media3,
  and where that mapping needs GL effects or a custom renderer.
---

# Android Video Editor (Kotlin + Compose + Media3)

Guidance for a native Android NLE. The reference architecture is **ReaCut**
(MIT, `Xiangyu-CAS/rea-capcut`) — a clean timeline→render pipeline worth
mirroring. Keep ReaCut's model and edit logic; swap its FFmpeg render engine
for **Media3 Transformer**.

> Media3 APIs move fast and this guidance predates your session. Treat method
> names here as *conceptual*; before writing export code, verify the current
> `androidx.media3.transformer` / `androidx.media3.effect` API against the
> installed version's sources or official docs. Do not invent signatures.

## License hygiene (do this first)

ReaCut is MIT. A port is a derivative work: keep a `NOTICE`/`THIRD_PARTY`
entry with ReaCut's copyright and the MIT text. This is non-negotiable and
cheap. Do not copy any CapCut/Jianying proprietary code, assets, or
decompiled output into the app — only ReaCut's own MIT source.

## Architecture

```
Compose UI  ──state──▶  EditorViewModel (StateFlow<EditorState>)
    ▲                        │  pure edit ops (Project -> Project)
    │ events                 ▼
 Timeline / Inspector    core model (Kotlin data classes)
                             │
         preview ◀───────────┤ ExoPlayer + GL preview compositor
         export  ◀───────────┘ Media3 Transformer (Composition)
```

- **Core model** = Kotlin `data class`es, ported 1:1 from ReaCut `src/core/types.ts`:
  `Project(width,height,fps,background, assets, tracks, clips)`, `Track(kind,
  muted,hidden,locked)`, `Clip(start,duration,sourceIn,speed,volume,transform,
  adjustments,text?,fades,keyframes,crop?,chromaKey?,mask?,linkId?)`. Keep them
  immutable (`copy()`); edits return a new `Project`.
- **Edit operations** = pure functions ported from `src/core/model.ts` +
  `src/core/timelineEdits.ts`: `splitClip`, `trimClip` (clamp to source
  handles), `changeClipSpeed` (ratio-remap duration+keyframes+fades),
  `placeAsset` (insert=ripple / overwrite / overlay), `moveLinked`,
  `slipLinked`, `detachLinkedAudio`, keyframe slicing/evaluation
  (`evaluateKeyframes` with the easeInOut cubic). These are platform-agnostic —
  port them verbatim and unit-test them against the same cases ReaCut uses.
- **State/undo** = ViewModel holding `past: List<Project>` / `future:` stacks
  (cap ~150), a `commit()` that pushes a clone, and transactions so a drag
  collapses into one undo entry (mirrors ReaCut's zustand store).

## Preview (real-time)

Use **ExoPlayer** for the primary/base video track. For overlays, positioned
layers (PiP), text, and adjustments, composite with a GL layer: `ExoPlayer`
video frames into a `SurfaceTexture`, then a Compose `AndroidView`/`GLSurfaceView`
(or Media3's preview effects pipeline) draws each visible clip with its
transform/opacity/crop. Keyframed properties are evaluated per frame from the
ported `evaluateKeyframes`. Keep preview and export driven by the *same* model
so what you see matches what you render.

## Export — mapping ReaCut's filter_complex to Media3

ReaCut compiles the whole timeline into one FFmpeg `-filter_complex`. Media3
expresses editing as a **`Composition`** of **`EditedMediaItemSequence`s**:
items *within* a sequence play back-to-back; *multiple* sequences are
**composited** together. The mapping:

| ReaCut concept | Media3 target |
|---|---|
| A track (clips back-to-back, with gaps) | one `EditedMediaItemSequence`; fill gaps with blank/transparent items |
| Overlapping clips / PiP / overlays | separate sequences composited in the `Composition`, or `OverlayEffect` |
| `sourceIn` + `duration` trim | `MediaItem.ClippingConfiguration` (start/end) |
| `speed` | a speed effect (video) + matching audio speed; verify current API (`SpeedChangeEffect`/sonic) |
| `transform` scale/rotate/translate | `ScaleAndRotateTransformation` + `Presentation`; translation via overlay placement |
| `crop` | `Crop` effect |
| `adjustments` (brightness/contrast/saturation/temperature/blur) | `Contrast`, `HslAdjustment`, `RgbFilter`/color-matrix; blur via a custom `GlShaderProgram` |
| `chromaKey` | a custom `GlEffect` chroma-key shader (no stock colorkey) |
| `mask` (rect/ellipse + feather) | alpha `BitmapOverlay` or a custom `GlShaderProgram` |
| `text` layer | `TextOverlay`, or render text to a `Bitmap` (as ReaCut does with canvas) and use `BitmapOverlay` |
| `fadeIn/Out`, keyframed opacity | `OverlayEffect` alpha over time / `AlphaScale`; animate via an effect that reads time |
| audio `volume`, fades, `amix` | per-item volume/fade `AudioProcessor`s; mixing is implicit across sequences |
| output `libx264`, mp4/mov | `Transformer` with the H.264 MIME + muxer; progress via its listener |

**Where the mapping breaks** (flag these, don't pretend they're free):
- Arbitrary multi-layer positioned compositing is richer in FFmpeg than in
  stock Media3. For PiP with per-clip transform + keyframes, expect to write
  **custom `GlEffect`/`GlShaderProgram`** passes, or render layers yourself.
- Chroma-key, feathered masks, gaussian blur, temperature → **custom GL
  shaders**; budget for them.
- If a timeline uses effects Media3 can't express per frame, consider a
  frame-accurate fallback: render frames via GL and feed them to Transformer.

## Build / validation on a headless CI box

- Use the Android Gradle plugin + command-line SDK; assemble with
  `./gradlew assembleDebug` and run unit tests (`testDebugUnitTest`) for the
  ported pure edit-logic — those run on the JVM without a device/emulator.
- Media3 export and Compose UI need an emulator or device; on a headless
  server, keep export/preview behind interfaces and test the model layer only.
- Port ReaCut's `validateProject` bounds (fps ∈ {24,25,30,50,60}, dims
  16–7680, speed 0.1–8, ≤400 keyframes/clip, etc.) as model invariants.

## Suggested milestones

1. Kotlin core model + ported edit ops + JVM unit tests (no UI).
2. Project load/save (JSON, same shape as ReaCut) + `validateProject`.
3. Compose timeline (tracks, clips, playhead, zoom) + selection + undo.
4. ExoPlayer preview of the base track; then GL compositing of layers.
5. Media3 Transformer export: single track → multi-track → effects → text.
6. Chroma-key / mask / blur custom GL shaders.
