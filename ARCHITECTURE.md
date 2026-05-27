# ARCHITECTURE.md

## Overview

**MEMORIAL** is a static web artwork that transforms dates of birth and death into a shared endless musical score.

The application has three main layers:

1. **Data layer** — Firebase Realtime Database stores user-submitted date pairs.
2. **Timeline / playback layer** — dates are converted into a sequence of digits; each digit becomes a timeline event with a frequency and a visual DOM reference.
3. **Interface / artwork layer** — the user sees a stream of dates, an active visual highlight, bilingual project text, animated Unicode backgrounds, and hears a synchronized Tone.js composition.

The project is implemented as plain HTML/CSS/JavaScript without a build system.

## Runtime environment

The app runs directly in the browser. External libraries are loaded from CDN in `index.html`:

- Tone.js `14.7.77`
- Firebase App compat SDK `9.23.0`
- Firebase Database compat SDK `9.23.0`
- Google Fonts / Inter

No npm, Node.js, backend server, or bundler is required.

## Entry point

The browser starts from `index.html`.

Important DOM elements:

- `#fireFrame` — initial flame iframe.
- `#noiseFrame` — noise iframe shown after start.
- `#left` — bottom UI panel.
- `#introBox` — project title and description.
- `#startBtn` — required audio-start button.
- `#formSection` — date input form, hidden before start.
- `#birthInput`, `#deathInput` — date inputs.
- `#addBtn` — add/remember button.
- `#errorBar`, `#okBar` — error/success messages.
- `#debugInfo` — current playback/status text.
- `#dbLeft` — date counter/debug ear.
- `#dbRight` — language switch/debug ear.
- `#right` — scrollable date stream container.
- `#stream` — rendered stream of date characters.

## Script order

The order is critical:

```html
<script src="config.js"></script>
<script src="synth.params.js"></script>
<script src="synth.js"></script>
<script src="visual.js"></script>
<script src="overlay.fx.js"></script>
<script src="player.js"></script>
<script src="data.js"></script>
<script src="main.js"></script>
```

`config.js` defines configuration and texts; `synth.params.js` defines sound parameters; module files expose `window.Synth`, `window.Visual`, `window.OverlayFX`, `window.Player`, `window.Data`; `main.js` connects everything to DOM and user events.

## File responsibilities

### `config.js`

Central configuration file. Contains RU/EN texts, artist/studio links, `CURRENT_LANG`, `TEXTS`, `window.AppConfig`, Firebase config, timing, frequency range, stream spacing, sync, activation window, and UI settings.

Important fields include:

```js
AppConfig.STREAM_SPACING
AppConfig.CLOCK
AppConfig.SYNC
AppConfig.WINDOW
AppConfig.DUR
AppConfig.FREQ_MIN
AppConfig.FREQ_MAX
AppConfig.PITCH_MODE
AppConfig.DB_PATH
AppConfig.firebaseConfig
```

### `synth.params.js`

Editable sound-design parameters. This is the safest place to tune the sound without rewriting the engine.

The conceptual audio graph:

```text
Body A + Body B → CrossFade → bodyGain
FM attack → attackGain
both layers → busGain
busGain → high-pass → compressor → low-pass
then parallel branches:
  dry
  reverb → EQ3
  ping-pong delay → echo reverb
then master → optional master EQ → Tone.Destination
```

### `synth.js`

Tone.js sound engine.

Public API:

```js
Synth.init()
Synth.trigger(freq, lenSec, vel, whenAbs, digit)
Synth.fx
```

The synth uses a voice pool, optional FM attack, filters, compressor, reverb, ping-pong delay, echo reverb, master gain, and optional master EQ. The current digit affects release duration, delay time, and feedback.

### `visual.js`

Renders Firebase date records into the right-side visual stream and builds the playable timeline.

Public API:

```js
Visual.build(list)
Visual.append(list)
Visual.setActiveIndex(idx)
Visual.getTimelineSnapshot()
Visual.timeline
Visual.knownIds
```

Visible dates include dots. Playback timeline excludes dots. Each timeline item contains `digit`, `freq`, `span`, and `pairEnd`.

### `overlay.fx.js`

Creates a lightweight highlight layer over the active line.

Public API:

```js
OverlayFX.init({ rootEl, heightFactor })
OverlayFX.pulseAtSpan(span)
OverlayFX.setHeightFactor(k)
```

It creates one absolute-positioned `div` with a white outline and moves it to the active line.

### `player.js`

Controls synchronized playback.

Public API:

```js
Player.start()
Player.stop()
Player.onTimelineChanged()
Player.rebuildAndResync()
```

Responsibilities:

- reads timeline from `Visual`;
- calculates current beat from `Data.serverNow()`;
- maps beat to timeline index;
- highlights active span;
- triggers `Synth.trigger()`;
- schedules notes with lookahead;
- handles missed beats;
- applies pending timeline updates on deterministic grid boundaries;
- logs timeline changes for deterministic reconstruction.

### `data.js`

Manages Firebase RTDB and synchronized time.

Public API:

```js
Data.init()
Data.pushDate(bDigits, dDigits)
Data.subscribe(handler, onError)
Data.getActiveList()
Data.currentWindowInfo()
Data.announceChange(k, beat, n)
Data.getChangeLogOnce()
Data.watchServerOffset()
Data.serverNow()
```

Main Firebase paths:

- `dates` — memorial date records.
- `control/changes` — timeline change log.

### `main.js`

UI controller. It handles DOM references, localization, input formatting, validation, start lifecycle, Tone.js start, synth init, Firebase init, subscriptions, visual build/append, overlay init, player start, user date submission, seed records, language switching, debug text, and counter updates.

## Data flow

### User submission flow

```text
User enters birth/death dates
↓
main.js formats input as DD.MM.YYYY
↓
main.js validates dates
↓
main.js removes dots
↓
Data.pushDate(bDigits, dDigits)
↓
Firebase RTDB writes record to dates
↓
Data.subscribe receives updated list
↓
Visual.append renders new record
↓
Player.onTimelineChanged schedules deterministic timeline switch
```

### Playback flow

```text
Firebase records
↓
Data.subscribe active list
↓
Visual.build / Visual.append
↓
Visual.timeline
↓
Player.start / Player.tick
↓
current beat from Data.serverNow()
↓
timeline index
↓
active span highlight + OverlayFX pulse
↓
Synth.trigger(freq, lenSec, vel, whenAbs, digit)
↓
Tone.js output
```

## Date format and storage

Visible format: `DD.MM.YYYY`.

Stored format: `DDMMYYYY`.

A complete memorial record consists of 16 digits: 8 birth digits and 8 death digits.

## Synchronization model

All clients use the same `SYNC_EPOCH_MS`. Beats are calculated from corrected server time. Every beat maps to one timeline index. New records are activated after a short window delay. Timeline changes are logged so clients can reconstruct offset/rotation.

## Visual architecture

The visual layer contains:

1. Background iframes: `fire.html` before start, `noise.html` after start.
2. Date stream: text rendered into `#stream`; every character is a span; digits receive `.digit`; active digit receives `.active`.
3. Overlay highlight: `overlay.fx.js` creates a viewport-width outlined bar.

## Audio architecture

High-level graph:

```text
Body voice pool
  ├─ Synth A
  ├─ Synth B
  └─ CrossFade
        ↓
body gain
        ↓
bus gain
        ↓
high-pass filter
        ↓
compressor
        ↓
low-pass filter
        ↓
        ├─ dry branch
        ├─ reverb → EQ3 branch
        └─ ping-pong delay → echo reverb branch
        ↓
master gain
        ↓
optional master EQ
        ↓
Tone.Destination
```

## Localization

Texts are stored in `TEXTS` inside `config.js`. Supported languages: `ru`, `en`. The language switch is `#dbRight`. `main.js` exposes `window.tr()` and `window.applyTexts()`.

## Known architectural risks

1. `config.js` initializes `window.AppConfig` and later reassigns it, which can overwrite earlier properties.
2. `visual.js` contains duplicated `digitToFreq()`.
3. `config.js` has both global `ENABLE_SEED` and `AppConfig.ENABLE_SEED`; `main.js` checks the global one.
4. `synth.js` reads `fmHarm`, while `synth.params.js` defines `fmHarmonicity`.
5. `index.html` and `main.js` both switch flame/noise backgrounds on start.
6. Firebase client config is public; RTDB security depends on Firebase Rules.
7. The app depends on CDN availability.

## Recommended change strategy

Keep changes small. Preserve static hosting. Modify `synth.params.js` for sound tuning, `config.js` for texts/timing/frequency range, `styles.css` for layout, `main.js` for UI lifecycle, `player.js` for sync/playback, and `data.js` for Firebase/time logic.
