# System-1 Patch Map

A visual starting-point map for the Roland System-1 synthesizer's front panel.
Not a sound engine, not an emulator — a reference tool that mirrors the
hardware's knob layout so a musician can navigate away from "stuck on some
random preset" toward "I want a sound in this direction" without hunting
through internal patch banks.

This document is for whoever picks this project up next. It covers *why*
things are built the way they are, not just what the code does.

---

## 1. The problem this solves

The owner plays a small Roland Aira rig (TB-3, TR-8, VT-4, System-1). The
System-1 is a real analog-modeling synth with physical knobs, but its sound
banks are deep and easy to get lost in. In practice, the owner would land on
an unfamiliar factory patch, not know how to get from there to the sound in
their head, and default to always starting from the same one or two known
patches — a creative bottleneck.

The fix isn't a plugin or a preset browser (the System-1 already has one).
It's a **map**: a picture of the actual panel, pre-filled with musically
useful starting knob positions, organized by genre and era, so the player can
say "I want something like a Reese bass" or "I want 80s synth-pop pad,"
land on a sensible starting posture, and then perform the fine-tuning by ear
on the real hardware. The app never makes sound. It tells you where to put
your hands.

---

## 2. Two parallel builds — and why

| File | Environment | Purpose |
|---|---|---|
| `system1-patch-map.jsx` | Renders inside Claude's chat as a React artifact | Fast iteration, no deployment step, good for reviewing changes in conversation |
| `index.html` + `sw.js` + `manifest.webmanifest` (this folder) | Standalone PWA, deployed to GitHub Pages / Netlify | The version actually used in the studio — installs to a phone home screen, works fully offline |

They are **not** the same codebase mechanically (one is a React component, one
is vanilla DOM manipulation), but they share the same *data* — the patch
library, panel/knob definitions, info text, and section metadata are
authored once and kept in sync by hand across both files. See §6 for the
implication this has on how to make changes.

Why two builds instead of one:
- Claude's artifact sandbox cannot register a service worker, so it can
  never be made installable or offline-capable — a real PWA needs to live
  outside that sandbox.
- The standalone version deliberately has **zero dependencies** (no React,
  no build tooling) so it's a single `index.html` that works by being
  opened, with no `npm install` / bundler step between edit and deploy.
  For a small, mostly-static UI like this, pulling in a framework would add
  weight for no real benefit.

---

## 3. Architecture (standalone PWA)

```
index.html            structure + all CSS (inline <style>) + all JS (inline <script>)
manifest.webmanifest  installability metadata (name, icons, colors, start_url)
sw.js                 service worker: cache-first, enables offline use
icon-192.png / icon-512.png   home-screen icons (generated, a green knob glyph)
README.md             this file
README.txt            end-user setup instructions (hosting + install steps)
```

Everything is in one HTML file on purpose — for a project this size, file
navigation overhead would cost more than the "one big file" readability
tradeoff. If this grows substantially, splitting `LIBRARY`/`INFO`/
`SECTION_META` into a separate `data.js` would be the first refactor, not
before.

### Rendering approach
No canvas, no SVG library, no charting dependency. Each knob is a small
hand-built `<svg>` (arc track + arc fill + pointer line), positioned and
animated with plain trigonometry (`polar()`/`arc()` helpers). This keeps the
whole app dependency-free and keeps knob rendering fast and predictable.

### State
A single in-memory `state` object holds:
- `values` — current position (0–100) of all 44 panel parameters
- `current` — which patch is loaded (`kind`: `"factory"` or `"user"`, plus
  `id`/`name`/`note`/`base` for edited-detection)
- `genre` — which library tab is open
- `userPatches` — the array of user-saved patches
- `confirmDel` — which patch row is in delete-confirmation state

State is persisted to **`localStorage`** under two keys:
- `s1.patches` — the user's saved patch collection (durable, no expiry)
- `s1.session` — full session snapshot (knob values, loaded patch, genre
  tab), written debounced (400ms after the last change) so reopening the
  app resumes exactly where you left off, including mid-edit

This is deliberately *not* IndexedDB or any heavier storage — the data is
small (a few KB even with dozens of saved patches) and `localStorage`'s
synchronous API is simpler for a project this size.

---

## 4. The data model

Three data structures drive the whole UI. Understanding these is 90% of
understanding the codebase.

### `SECTIONS` + `LABELS`
`SECTIONS` is an ordered array of the 9 physical panel sections (LFO, OSC 1,
OSC 2, MIXER, PITCH, FILTER, FILTER ENV, AMP, EFFECTS), each listing which
parameter keys belong to it, **in the hardware's left-to-right order.**
`LABELS` maps each parameter key to the short text printed under its knob.

This ordering is load-bearing: the whole point of the app is that its
layout matches the physical synth, so a player's hand can move from the
screen to the real knob without translation. Do not reorder `SECTIONS`
without checking against the physical panel photo.

### `LIBRARY`
An array of genre groups, each `{ genre, patches }`. Each patch is:
```js
{ id, name, tag, note, v: { /* partial parameter values, 0–100 */ } }
```
`v` only needs to specify non-zero parameters — anything omitted falls back
to `ZERO` (see below) when loaded. `tag` is the short subtitle shown on the
patch chip (e.g. "the classic"). `note` is the "START HERE" guidance text
shown when the patch is loaded — written to name the *one or two* controls
that matter most for that sound, not to describe the whole patch.

Genres are ordered chronologically (`STARTERS` first, then 1981→2008) so
browsing the tabs left-to-right is a walk through electronic music history.
Deliberately mixes famous, easily-recognized sounds (Mentasm, supersaw, 303
acid) with obscure ones (Belgian New Beat, Sheffield Bleep, EBM) — the goal
is a genuinely useful reference, not just a "greatest hits" list.

### `STEPPED`, `BIG`, `BIPOLAR`, `ZERO`
- `STEPPED` — parameters that are discrete switches rather than continuous
  knobs (waveform selectors, range switches, on/off toggles). The knob still
  renders as a rotary control (it matches the real hardware's rotary
  selectors), but snaps to N evenly-spaced positions and shows a text
  readout (`SAW`/`SQR`/`TRI`, `64`/`32`/…, `OFF`/`ON`) instead of a number.
- `BIG` — the two knobs (`cutoff`, `reso`) rendered larger, because they're
  the ones actually played live and deserve more precision under a finger.
- `BIPOLAR` — parameters whose real-world meaning is "how far from center"
  (`o2Tune`, `pitchEnv`) — displayed as `+N`/`-N` around a 50 midpoint
  rather than 0–100.
- `ZERO` — the all-parameters-at-rest default. Not literally all zero: a few
  params default away from 0 to match how the real panel behaves at rest
  (`o1Range`/`o2Range` default to 8', `pitchEnv` defaults to center/off,
  `ampTone` defaults open, `delayTime` defaults centered). Any patch or
  imported file that omits a parameter inherits these, not raw 0 — this
  matters for backward compatibility (see §7).

### `INFO`
One paragraph per parameter (all 44), written for the touch-and-hold detail
popup. Style: plain language, states what the knob *does* and gives a
practical usage cue, not spec-sheet phrasing. Kept short enough to read in
a few seconds without leaving the flow of playing.

### `SECTION_META`
One entry per section: `tag` (the short subtitle under the section name),
`info` (what the whole section is for), and `pairs` — a list of non-obvious
cross-knob interactions specific to that section (e.g. "ENV needs headroom:
if cutoff is already open, the filter envelope has nothing to push against").
Surfaced when the player taps a section title.

---

## 5. Design decisions — the "why," not just the "what"

These are choices that weren't arbitrary and are worth preserving intent for
if this project changes hands.

- **Words over icons for section labels.** The panel is already a dense
  symbolic language (waveform glyphs, ADSR shapes, arrows). A second icon
  system to learn would work against the goal of the app being immediately
  legible. Two lowercase words in mono type ("movement & wobble") reads
  instantly without a legend.

- **Touch-and-hold, not double-tap, for the info popup.** A hold is
  unambiguous: movement means "turn the knob," stillness means "tell me
  about it," and the two can't be confused mid-gesture. Double-tap adds a
  recognition delay to *every* tap (the browser has to wait to see if a
  second tap is coming) and collides with the platform's native
  pinch/double-tap-to-zoom gesture. The implementation cancels the info
  timer the instant the finger moves more than 5px, and un-does any
  micro-jitter that happened during the hold so reading about a knob never
  silently changes its value.

- **Three zoom levels of guidance**, added incrementally as the real gap
  became clear through use: patch notes (which knob matters *for this
  specific sound*) → section taglines + tap-for-guide (which *area* of the
  panel to look in, plus the pairings that aren't obvious from any single
  knob) → hold-a-knob detail (what *this exact control* does). This mirrors
  how a person actually thinks: direction first, mechanism second.

- **No audio engine, ever, by design.** Building a Web Audio synth engine
  that approximates the System-1 would be a much bigger project, would
  never sound quite right, and would undermine the actual goal — getting
  the player's hands onto the real hardware faster, not replacing it.

- **Dependency-free standalone build.** No React, no bundler, no npm
  install step for the PWA. Anyone can open `index.html` directly and read
  top to bottom. This was a conscious tradeoff against code reuse with the
  JSX artifact (see §6) in favor of zero deployment friction.

- **Export/import as human-readable JSON**, not a binary or compressed
  format. It needs to survive being emailed, pasted into a notes app, or
  read years later by someone with no context — plain JSON with `name`,
  `note`, and `v` per patch is self-describing enough to reconstruct by
  hand if the app itself is ever lost.

- **Service worker cache version is bumped by hand on every deploy**
  (`s1-patch-map-v6` at time of writing). There's no automated cache-busting
  — this is a manual discipline the next engineer needs to keep, or updates
  won't reach phones that already have the app installed and cached.

---

## 6. Making changes safely

### Adding a new patch
Add an object to the appropriate genre's `patches` array in `LIBRARY`:
```js
{ id: "unique-id", name: "PATCH NAME", tag: "short subtitle",
  note: "Which knob matters most and why.",
  v: { /* only the parameters that differ from ZERO */ } }
```
Do this in **both** `system1-patch-map.jsx` and `index.html` — there is no
shared build step, so the data has to be copy-pasted into both files by
hand. This is the main maintenance cost of the two-build approach; see the
validation note below for how to catch mistakes.

### Adding a new genre tab
Add a new `{ genre, patches }` entry to `LIBRARY`, positioned chronologically
if it fits the timeline, in both files.

### Adding a new panel parameter (rare — only if the hardware mapping is
genuinely incomplete)
This touches more places than it looks like it should. You must update, in
both files:
1. `SECTIONS` — add the key to the right section's `params` array, in the
   correct physical position
2. `LABELS` — the short knob-face label
3. `STEPPED` (if it's a switch/selector, not a continuous knob)
4. `BIG` / `BIPOLAR` (if applicable)
5. `ZERO` (its rest value)
6. `INFO` — its detail-popup paragraph
7. Optionally `SECTION_META` pairings if it interacts non-obviously with
   another control in its section
8. Any existing patches that should actually use the new parameter

### Validating changes
There's no automated test suite, but every non-trivial data change during
development was checked with a quick throwaway Python script that:
- extracts the JS with regex,
- confirms every patch's `v` keys are all known parameter names,
- confirms every value is in 0–100,
- confirms stepped parameters (`o1Wave`, `lfoWave`, etc.) land exactly on a
  valid step,
- confirms `SECTIONS` params and `LABELS` keys are the same set.

Re-running an equivalent check (see git history / prior conversation for
the exact script) before shipping a data change is cheap insurance against
a typo silently breaking a knob.

### Keeping the two files in sync
There is currently no tooling enforcing that `system1-patch-map.jsx` and
`index.html` have identical `LIBRARY`/`INFO`/`SECTION_META` data. If this
project grows, the highest-value refactor would be extracting that data to
a single `data.js` that both builds import — the JSX artifact would need
the data inlined at export time, but the standalone build could load it
as a real module. Not done yet because the two-build split has been small
enough to manage by hand.

---

## 7. Compatibility notes

- **Import/export format** (`{ app, version, exported, patches: [...] }`)
  is versioned (`version: 1`) but the parser is lenient — it fills any
  missing parameter with its `ZERO` default rather than failing, so files
  exported before a given parameter existed still import cleanly.
- **Session/patch storage keys** (`s1.patches`, `s1.session` in the
  standalone build) have not changed since introduced. Don't rename them
  without a migration path — it would silently "lose" every user's saved
  collection (it isn't actually lost, just orphaned in localStorage under
  the old key).

## 8. Known limitations / explicit non-goals

- No sound. This is a map, not a synth.
- No MIDI I/O, no direct communication with the physical System-1 (the
  System-1 has no way to be remotely queried for its current panel state
  anyway).
- No real-time collaboration/sync between devices — patches live in one
  browser's local storage per device; export/import is the sharing
  mechanism, not a server.
- The panel is a *simplified, curated* subset of the real System-1 (44 of
  its controls) — deliberately excludes global/performance settings
  (arpeggiator, Scatter, portamento, tempo, mono/poly, MOD-routing
  selectors) that aren't part of a patch's sonic identity. See the
  conversation history / commit messages for the reasoning on what was
  cut and why.

## 9. Deployment

Standalone build is a static site with no build step — deploy by copying
this folder's contents as-is. Currently set up for GitHub Pages (serve from
repo root on `main`) and was previously hosted via Netlify Drop. See
`README.txt` for end-user-facing setup steps.
