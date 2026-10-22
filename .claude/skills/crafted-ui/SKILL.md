---
name: crafted-ui
description: >
  Design and build app UIs that read as deliberately crafted by a product
  designer, not auto-generated. Use when building or reviewing the look and
  feel of an app — especially a native Android app in Jetpack Compose — and the
  goal is a polished, professional, non-generic result (e.g. a video editor
  that should feel like a pro tool). Covers escaping default/templated looks,
  building a real design system (type scale, restrained palette, spacing grid,
  radii, elevation), purposeful motion, full UI states, platform idioms,
  accessibility, and the craft details that separate shipped products from
  demos.
---

# Crafted UI

How to make an interface look designed, not defaulted. Most "AI-generated"
UIs are spotted by the same tells; avoid them on purpose and the result reads
as a product.

## The tells to avoid

Generic/templated UIs leak these signals. Treat each as a red flag in your own
output and in review:

- **Default theme untouched** — stock Material purple, default type, default
  component shapes. Ship a real theme instead.
- **Everything centered, evenly spaced, same size** — no hierarchy. Real
  layouts have a dominant element and deliberate asymmetry.
- **Gradient / shadow / rounded-corner on everything** — uniform decoration is
  noise. Decoration should mark meaning (one accent, one elevation level that
  means "raised").
- **Emoji as icons, emoji in headings** — use a real icon set.
- **Placeholder vibe** — three identical cards, lorem-ipsum rhythm, no empty /
  loading / error states.
- **No motion, or fade-everything** — absence of intent either way.
- **Ignoring the platform** — web patterns on Android, no edge-to-edge, system
  bars fought instead of used.

## Build a real design system first

Decide these once, encode them as tokens (Compose `MaterialTheme` +
custom tokens), and use them everywhere. Consistency is most of "polish".

- **Color**: a restrained palette — one or two neutrals carrying the UI, **one
  confident accent** used sparingly for the primary action and key state. Define
  full light + dark; dark is first-class (for a creative/pro tool, dark is
  usually the default). Check contrast (text ≥ 4.5:1). Don't tint everything the
  accent. (For charts/data specifically, load the `dataviz` skill.)
- **Type**: a real scale (e.g. display / title / body / label with intentional
  size+weight+line-height steps), one or two families, tabular figures for
  timecodes/counters. Hierarchy comes from the scale, not from centering.
- **Spacing**: a 4/8dp grid. Space is a tool — group related things tightly,
  separate groups generously. Uniform 16dp everywhere is a tell.
- **Shape & elevation**: a small set of corner radii with meaning; elevation
  only where something truly floats. Prefer tonal/color separation over drop
  shadows on dense pro UIs.
- **Iconography**: one coherent set (e.g. Material Symbols), consistent weight
  and optical size; custom glyphs where the domain needs them.

## Motion with intent

- Transitions should explain spatial relationships (shared-element, expand from
  origin), not just fade. In Compose use spring physics with sensible
  stiffness/damping; avoid long linear tweens.
- Micro-interactions signal quality: press states, haptics on key actions,
  a scrubber that tracks the finger exactly, snap feedback on the timeline.
- Respect "reduce motion" accessibility settings.

## Ship the whole state machine

A screen isn't done until it has: **empty**, **loading** (skeletons, not just a
spinner), **error** (with a way out), **populated**, and **offline/permission**
where relevant. Designing these is the difference between a demo and a product.

## Platform idioms (Android / Material 3)

- Edge-to-edge; draw behind system bars and inset content correctly.
- Follow Material 3 structure (navigation, FAB only when it fits, bottom sheets
  for contextual actions) but theme it — M3 structure, your skin.
- Touch targets ≥ 48dp; gestures that match platform expectation (back,
  swipe-to-dismiss). Support dynamic color only if it doesn't fight your brand.
- Content descriptions on every interactive element; test with TalkBack and
  large font scales (layouts must not break at 200%).

## Pro-tool / creative-app aesthetic (e.g. a video editor)

- Dark, dense, precise. Information-rich without clutter; let the media be the
  bright thing on a quiet chrome.
- Custom-draw the timeline (Compose `Canvas`): ruler with real timecode ticks,
  waveforms, thumbnails, clip handles with generous hit areas, a crisp playhead.
- Direct manipulation everywhere (drag to trim/move, pinch to zoom the
  timeline), with snapping and clear affordances. Keep destructive actions
  reversible (that undo stack is a UX feature, surface it).
- Keep the canvas/preview the hero; dock controls, don't scatter them.

## How to work

1. Look at two or three real reference apps in the genre; name the specific
   patterns you'll adopt and the ones you'll reject. Don't design from the
   default template.
2. Establish the token set before building screens.
3. Build one screen to a finished bar (all states, motion, a11y) as the quality
   reference, then match the rest to it.
4. Review against "the tells" above before calling it done.
