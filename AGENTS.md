# AGENTS.md

## Project

This repository contains the final web version of **MEMORIAL** — an interactive digital memorial.

The project is a static browser-based artwork. Users enter a birth date and a death date. The dates are saved into Firebase Realtime Database. Each pair of dates becomes part of a shared endless musical score: every digit is transformed into a frequency and played in a synchronized loop.

## Repository type

Plain static frontend project: HTML, CSS, JavaScript. No build system, no bundler, no backend server, no framework. Do not add React, Vite, Node, npm, TypeScript, webpack, or other infrastructure unless explicitly requested.

## Main files

- `index.html` — DOM structure, script loading order, background iframes, UI containers.
- `styles.css` — visual layout, typography, panels, stream styling.
- `config.js` — global config, texts, Firebase config, timing, frequency range, sync parameters.
- `synth.params.js` — editable sound-design parameters.
- `synth.js` — Tone.js sound engine and FX graph.
- `visual.js` — renders dates into visual stream and builds playable timeline.
- `overlay.fx.js` — active-line highlight overlay.
- `player.js` — synchronized playback scheduler, beat grid, timeline switching.
- `data.js` — Firebase RTDB connection, writes, subscriptions, server-time sync.
- `main.js` — UI lifecycle, validation, start/add flow, language switching.
- `fire.html` — initial Unicode flame background.
- `noise.html` — Unicode noise background after start.

## Critical script order

Do not casually change this order:

1. `config.js`
2. `synth.params.js`
3. `synth.js`
4. `visual.js`
5. `overlay.fx.js`
6. `player.js`
7. `data.js`
8. `main.js`

## Core rules

- The project is a digital memorial.
- User enters birth and death dates in `DD.MM.YYYY`.
- Internally, dates are stored as digits without dots.
- One record contains 16 digits: 8 birth digits + 8 death digits.
- Digits `0–9` map to frequencies.
- The database becomes a shared endless cyclic score.
- New entries append to the shared memory and must not replace old records.
- Audio must start only after explicit user interaction. Do not remove the Start/Connect button.

## Firebase rules

Default RTDB path is controlled by `AppConfig.DB_PATH`, currently `dates`.

Each pushed record should keep this structure:

```js
{
  birth: "DDMMYYYY",
  death: "DDMMYYYY",
  digits: [Number, ...],
  ts: firebase.database.ServerValue.TIMESTAMP
}
```

Do not rename fields unless all dependent modules are updated.

## Synchronization rules

Playback uses a deterministic time grid. Be careful with:

- `SYNC_EPOCH_MS`
- `SYNC.GRID_MS`
- `WINDOW.MS`
- `WINDOW.DELAY_MS`
- `CLOCK.USE_FIREBASE_OFFSET`
- `CLOCK.USE_HTTP_TIME`

The project uses Firebase server-time offset, optional HTTP UTC time, activation windows, and a change log at `control/changes`.

## Timeline rules

`Visual.timeline` contains only playable digits, never dots or separators.

Each item:

```js
{
  digit,
  freq,
  span,
  pairEnd
}
```

`pairEnd` marks the last digit of a date pair and is used for counting records.

## Sound rules

`Synth.init()` creates the Tone.js graph. `Synth.trigger(freq, lenSec, vel, whenAbs, digit)` plays one note. The `digit` affects release, delay time, and feedback. Do not create a new synth per note; keep the voice-pool architecture.

## Validation rules

Date inputs must auto-format to `DD.MM.YYYY`, reject invalid calendar dates, reject death earlier than birth, and save only digits to Firebase.

## Change strategy

Make small, targeted changes. Preserve static hosting compatibility. Do not refactor unrelated files. After changes, manually check: load, start, audio, Firebase read/write, visual stream, active highlight, RU/EN switch, and adding a new record during playback.
