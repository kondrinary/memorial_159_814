# STATE.md

## Current project state

This repository contains a browser-based final version of **MEMORIAL**.

The app is a static web project written in plain HTML, CSS, and JavaScript. It uses Tone.js for sound and Firebase Realtime Database for persistent shared date storage.

The project is in a working prototype/final-web-version state, with the main architecture already implemented.

## Repository inspected

The working code was found in:

```text
kondrinary/memorial
```

The repository originally provided:

```text
kondrinary/memorial_159_814
```

appeared empty during inspection.

## Implemented features

### Core concept

Implemented:

- user can enter birth and death dates;
- dates are validated;
- dates are saved to Firebase;
- each date pair becomes part of the shared visual stream;
- each digit becomes a note/frequency;
- the full database becomes a cyclic musical timeline;
- playback is continuous and synchronized to a grid;
- new records are appended rather than replacing old ones.

### UI

Implemented:

- black full-screen interface;
- bottom fixed UI panel;
- project title and description card;
- start/connect button;
- birth date input;
- death date input;
- add/remember button;
- error bar;
- success bar;
- debug/status line;
- optional seed/test button;
- left “ear” showing date count after start;
- right “ear” switching language;
- scrollable visual stream of dates.

### Localization

Implemented:

- Russian UI text;
- English UI text;
- language switching between RU and EN;
- localized placeholders, buttons, project text, status messages, and debug strings.

### Date input

Implemented:

- automatic input formatting into `DD.MM.YYYY`;
- validation of real calendar dates;
- rejection of invalid format;
- rejection of death date earlier than birth date;
- conversion to digit-only strings before database write.

### Database

Implemented through Firebase Realtime Database:

- Firebase compat SDK loaded by CDN;
- config stored in `config.js`;
- database path controlled by `AppConfig.DB_PATH`;
- default date path: `dates`;
- timeline change log path: `control/changes`;
- records are pushed with server timestamps.

Expected date record:

```js
{
  birth: "DDMMYYYY",
  death: "DDMMYYYY",
  digits: [Number, Number, ...],
  ts: firebase.database.ServerValue.TIMESTAMP
}
```

### Playback

Implemented:

- Tone.js starts only after user click;
- synth initializes after `Tone.start()`;
- playback starts after the first non-empty database list;
- timeline is built from visual digit spans;
- one beat corresponds to one timeline digit;
- active digit is highlighted;
- sound is scheduled with lookahead;
- missed beat tolerance exists;
- new timelines are switched on deterministic boundaries.

### Synchronization

Implemented:

- fixed sync epoch;
- configurable grid duration;
- Firebase server-time offset;
- optional HTTP UTC time sync;
- smooth offset correction;
- activation windows for new records;
- timeline change log for deterministic offset reconstruction.

### Visual stream

Implemented:

- records are rendered old-to-new;
- visible dates include dots;
- playback timeline excludes dots;
- each digit span can be activated;
- active digit gets `.active`;
- optional deterministic character spacing config exists;
- new records are appended without full rebuild.

### Backgrounds

Implemented:

- `fire.html` initial Unicode flame canvas;
- `noise.html` Unicode noise canvas after start;
- iframes used as background layers;
- start click hides flame and shows noise.

### Overlay highlight

Implemented:

- `overlay.fx.js` creates one absolute overlay bar;
- the bar follows the currently active line;
- visual effect uses a transparent bar with white border;
- no additional canvas is used for the overlay.

### Sound design

Implemented:

- Tone.js voice pool;
- two-body oscillator morph through CrossFade;
- optional FM attack layer;
- high-pass filter;
- compressor;
- low-pass filter;
- dry branch;
- reverb branch;
- delay branch;
- echo reverb after delay;
- master EQ;
- digit-dependent release;
- digit-dependent delay time;
- digit-dependent feedback.

## Important current config

```js
FREQ_MIN: 90
FREQ_MAX: 500
PITCH_MODE: 'linear'
DB_PATH: 'dates'
SYNC_ENABLED: true
SYNC_EPOCH_MS: Date.UTC(2025,0,1,0,0,0)
SYNC_SEED: 123456789
RANDOM_MODE: 'seeded'
SYNC.GRID_MS: 1000
WINDOW.MS: 1000
WINDOW.DELAY_MS: 200
DUR.noteLen: 0.50
DUR.pairGap: 1000
SPEED: 1
```

Firebase project:

```text
memorial-b26b7
```

Database URL:

```text
https://memorial-b26b7-default-rtdb.europe-west1.firebasedatabase.app
```

## Current lifecycle

1. Page loads.
2. Flame background is visible.
3. Start/connect button waits for user click.
4. User clicks Start.
5. Flame hides; noise appears.
6. Tone.js starts.
7. Synth initializes.
8. Firebase initializes.
9. Server time offset tracking starts.
10. Form appears.
11. Database subscription starts.
12. If date records exist: visual stream builds, overlay initializes, player starts.
13. If user adds a date: date is validated, record is pushed to Firebase, success message appears, stream updates, player schedules timeline change.

## Known issues / cleanup candidates

### 1. `memorial_159_814` repository appears empty

The requested repository appeared to have size `0` and no accessible files. The working version was found in `kondrinary/memorial`.

### 2. Duplicate `digitToFreq` in `visual.js`

`visual.js` defines `digitToFreq(d)` twice. Both versions are currently equivalent, but one should be removed.

### 3. Conflicting seed button config

There is a global `const ENABLE_SEED = false` and also `AppConfig.ENABLE_SEED = true`. `main.js` checks the global `ENABLE_SEED`, not `AppConfig.ENABLE_SEED`.

Recommended fix: use only `AppConfig.ENABLE_SEED`.

### 4. `AppConfig` is initialized and later overwritten

At the beginning of `config.js`, properties are assigned to `window.AppConfig = window.AppConfig || {}`. Later the file does `window.AppConfig = { ... }`. This can overwrite earlier config properties.

Recommended fix: define `window.AppConfig` only once, or merge instead of replacing.

### 5. FM parameter name mismatch

`synth.js` reads `fmHarm`, while `synth.params.js` defines `fmHarmonicity`. Because of this mismatch, configured harmonicity may not be used.

### 6. Inline background switch duplicates `main.js`

`index.html` contains an inline click listener that hides `fireFrame` and shows `noiseFrame`. `main.js` also does the same.

### 7. Typo in Russian validation message

Russian `errDeathBeforeBirth` contains a typo: `дпопробуйте...`.

Recommended text:

```text
дата смерти не может быть раньше даты рождения
```

### 8. Public Firebase config

Firebase web config is public in `config.js`. This is normal for frontend Firebase apps, but database security depends on Firebase Rules.

### 9. CDN dependency

The app depends on external CDNs for Tone.js and Firebase. If CDN loading fails, audio/database cannot start.

### 10. WorldTimeAPI dependency

`data.js` optionally uses `https://worldtimeapi.org/api/timezone/Etc/UTC`. If unavailable, the code catches errors silently.

## Do not change without care

Sensitive areas:

1. Script order in `index.html`.
2. `Data.serverNow()` and sync logic.
3. `Player.onTimelineChanged()` rotation logic.
4. Firebase record format.
5. `Visual.timeline` item structure.
6. Start button / `Tone.start()` lifecycle.
7. `Synth.trigger()` timing arguments.
8. Active span references used by both `Player` and `OverlayFX`.

## Suggested next tasks

High priority:

1. Decide whether final repo is `kondrinary/memorial` or `kondrinary/memorial_159_814`.
2. Add `AGENTS.md`.
3. Add `docs/ARCHITECTURE.md`.
4. Add `docs/STATE.md`.
5. Fix seed config conflict.
6. Fix `fmHarm` / `fmHarmonicity` mismatch.
7. Remove duplicate `digitToFreq`.
8. Fix Russian typo in `errDeathBeforeBirth`.

Medium priority:

1. Clean duplicate background switching.
2. Add a short `README.md`.
3. Add manual setup instructions for Firebase Rules.
4. Add an exhibition-mode checklist.
5. Add a simple fallback message for database/CDN failure.

## Manual test checklist

### Loading

- [ ] Page opens without console errors.
- [ ] Flame background appears.
- [ ] Start/connect button appears.
- [ ] Form is hidden before start.
- [ ] RU/EN switch is visible.

### Start

- [ ] Clicking Start starts audio context.
- [ ] Flame disappears.
- [ ] Noise background appears.
- [ ] Form appears.
- [ ] Project text changes to play description.
- [ ] No browser autoplay error appears.

### Database

- [ ] Firebase initializes.
- [ ] Existing records load.
- [ ] Empty database shows a useful message.
- [ ] New valid record writes to Firebase.
- [ ] Invalid date format is rejected.
- [ ] Impossible date is rejected.
- [ ] Death before birth is rejected.

### Visuals

- [ ] Date stream renders.
- [ ] Dots appear visually but are not played.
- [ ] Digits highlight one by one.
- [ ] Overlay bar follows active line.
- [ ] Adding a record appends it visually.

### Sound

- [ ] Sound plays after start.
- [ ] Sound continues cyclically.
- [ ] Frequencies change by digit.
- [ ] No obvious clicks from scheduling.
- [ ] New record does not restart the whole audio engine.

### Localization

- [ ] RU texts display correctly.
- [ ] EN texts display correctly.
- [ ] Switching language does not stop playback.
- [ ] Status/debug strings update language where expected.
