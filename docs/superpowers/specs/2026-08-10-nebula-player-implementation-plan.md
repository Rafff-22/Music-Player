# Nebula Player — Implementation Plan

**Spec:** `docs/superpowers/specs/2026-08-10-nebula-player-pwa-design.md`
**Approach:** Incremental, 5 layers
**Stack:** Vanilla HTML/CSS/JS, ES Modules, zero framework, zero build tool

---

## Layer 1: PWA Shell

**Goal:** Installable PWA skeleton — manifest, service worker, minimal HTML, icon generator.

### Step 1.1 — `manifest.json`

Create `manifest.json` exactly as spec section 5.1. Fields: name, short_name, description, start_url, display (standalone), orientation (portrait-primary), theme_color (#080810), background_color (#080810), icons (192 + 512 maskable).

**File:** `manifest.json`

### Step 1.2 — `generate-icons.mjs`

Node.js script using `canvas` package to generate `icons/icon-192.png` and `icons/icon-512.png`.

- Background fill: `#8b5cf6`
- White music note symbol centered
- Create `icons/` directory if missing
- Output both sizes

**File:** `generate-icons.mjs`
**Run:** `npm install canvas && node generate-icons.mjs`
**Verify:** `icons/icon-192.png` and `icons/icon-512.png` exist and are valid PNGs

### Step 1.3 — `sw.js`

Service worker with cache-first strategy.

- Cache name: `nebula-v1`
- **Install event:** Pre-cache static assets only: `./`, `index.html`, `style.css`, `script.js`, `player.js`, `db.js`, `manifest.json`, `icons/icon-192.png`, `icons/icon-512.png`. Do NOT pre-cache Google Fonts.
- **Fetch event:**
  - For requests matching `fonts.googleapis.com` or `fonts.gstatic.com`: runtime cache-first (try cache, fallback fetch, store response in cache for next time)
  - For all other requests: cache-first from pre-cache
- **Activate event:** Delete all caches where key !== `nebula-v1`

**File:** `sw.js`

### Step 1.4 — `index.html` (minimal skeleton)

Minimal HTML5 boilerplate with:

- `<meta charset="UTF-8">`, viewport (responsive), theme-color (#080810)
- Apple mobile web app meta tags (capable, status-bar-style black-translucent)
- `<link rel="manifest" href="manifest.json">`
- Google Fonts link: Space Grotesk (400,700), Inter (400,500), JetBrains Mono (400)
- Lucide Icons via unpkg CDN
- `<link rel="stylesheet" href="style.css">`
- `<script type="module" src="script.js"></script>`
- SW registration in a separate non-module `<script>` block
- `<body>` with a single placeholder `<p>Nebula Player loading...</p>`

**File:** `index.html`

### Step 1.5 — `style.css` (foundation only)

CSS reset + design tokens in `:root` (all custom properties from spec section 8.1). Body styling: flex center, Inter font, --bg-primary background, overflow-x hidden.

**File:** `style.css`

### Step 1.6 — `script.js` (stub)

```javascript
import * as SongDB from './db.js'
import { Player } from './player.js'

document.addEventListener('DOMContentLoaded', () => {
  console.log('Nebula Player booting...')
})
```

**File:** `script.js`

### Step 1.7 — `db.js` (stub) + `player.js` (stub)

Empty module stubs so ES Module imports don't fail.

- `db.js`: export empty async functions (addSong, updateCover, getAllSongs, deleteSong, clearAll)
- `player.js`: export Player object with no-op methods

**Files:** `db.js`, `player.js`

### Layer 1 Verification

- Serve with `npx serve .` (or any static server)
- Open in Chrome → DevTools → Application tab
  - Manifest loads correctly, shows name/icons
  - Service Worker registers and activates
  - Cache Storage shows `nebula-v1` with pre-cached assets
- No console errors
- Lighthouse PWA audit: check installability criteria

---

## Layer 2: Data Layer (`db.js`)

**Goal:** Full IndexedDB wrapper — add, read, update cover, delete songs. Test via console.

### Step 2.1 — `db.js` complete implementation

Replace stub with full implementation per spec section 3.

Internal `openDB()`:
- Lazy, returns cached promise (singleton)
- Opens `NebulaDB` v1
- `onupgradeneeded`: create `songs` store with `{ keyPath: 'id', autoIncrement: true }`
- Wrap in try/catch; throw meaningful error if IDB unavailable (private browsing)

Exported functions:

**`addSong(audioFile, meta)`**
- Open transaction (readwrite) on `songs` store
- Build record: `{ title: meta.title, artist: meta.artist, duration: meta.duration, audioBlob: audioFile, coverBlob: null, dateAdded: Date.now() }`
- Add record, get resulting id from `request.result`
- Create `audioURL = URL.createObjectURL(audioFile)`
- Return `{ id, audioURL }`

**`updateCover(id, imageFile)`**
- Open transaction (readwrite), get existing record by id
- Set `record.coverBlob = imageFile`, put back
- Return `URL.createObjectURL(imageFile)`

**`getAllSongs()`**
- Open transaction (readonly), getAll from store
- Map each record: create `audioURL` from `audioBlob`, `coverURL` from `coverBlob` (if not null)
- Return array of `{ id, title, artist, duration, audioURL, coverURL, dateAdded }`

**`deleteSong(id)`**
- Open transaction (readwrite), delete by id

**`clearAll()`**
- Open transaction (readwrite), clear store

**File:** `db.js`

### Layer 2 Verification

- Open app in browser, open DevTools console
- Test sequence:
  ```javascript
  import * as SongDB from './db.js'
  // Create a test blob
  const blob = new Blob(['test'], { type: 'audio/mp3' })
  const file = new File([blob], 'test.mp3', { type: 'audio/mp3' })
  const result = await SongDB.addSong(file, { title: 'Test Song', artist: 'Test', duration: '3:00' })
  console.log(result) // { id: 1, audioURL: 'blob:...' }
  const songs = await SongDB.getAllSongs()
  console.log(songs) // [{ id: 1, title: 'Test Song', ... }]
  await SongDB.deleteSong(1)
  const empty = await SongDB.getAllSongs()
  console.log(empty) // []
  ```
- Check IndexedDB in DevTools → Application → IndexedDB → NebulaDB → songs
- Close browser, reopen, verify data persists

---

## Layer 3: Audio Layer (`player.js`)

**Goal:** Full audio playback + Media Session. Test with a hardcoded audio file.

### Step 3.1 — `player.js` complete implementation

Replace stub with full implementation per spec section 4.

Internal state:
- `let audio = null` (HTMLAudioElement, created in init)
- `let callbacks = {}` (stored from init)

**`init(cbs)`**
- Store callbacks
- Create `audio = new Audio()`
- Wire events:
  - `timeupdate` → `cbs.onTimeUpdate?.(audio.currentTime, audio.duration)`
  - `ended` → `cbs.onEnded?.()`
  - `error` → `cbs.onError?.(audio.error)`
  - `waiting` → `cbs.onLoading?.()`
  - `canplay` → `cbs.onReady?.()`
  - `loadedmetadata` → `cbs.onMetadata?.({ duration: audio.duration })`

**`load(src, { title, artist, coverSrc })`**
- Set `audio.src = src`
- If `'mediaSession' in navigator`:
  - Set `navigator.mediaSession.metadata = new MediaMetadata({ title, artist, artwork })`
  - Set action handlers:
    - `play` → `audio.play()`
    - `pause` → `audio.pause()`
    - `previoustrack` → `cbs.onPrev?.()`
    - `nexttrack` → `cbs.onNext?.()`
    - `seekbackward` → `audio.currentTime = Math.max(0, audio.currentTime - 15)`
    - `seekforward` → `audio.currentTime = Math.min(audio.duration, audio.currentTime + 15)`

**`play()`** → `return audio.play()` (wrapped in try/catch)
**`pause()`** → `audio.pause()`
**`toggle()`** → if paused call play(), else pause()
**`seek(pct)`** → `audio.currentTime = (pct / 100) * audio.duration`
**`setVolume(v)`** → `audio.volume = v`

Getters:
- `get currentTime()` → `audio?.currentTime || 0`
- `get duration()` → `audio?.duration || 0`
- `get isPlaying()` → `audio ? !audio.paused : false`

**File:** `player.js`

### Layer 3 Verification

- Temporarily add a test in script.js:
  ```javascript
  Player.init({
    onTimeUpdate: (ct, dur) => console.log(`${ct}/${dur}`),
    onEnded: () => console.log('ended'),
    onMetadata: ({ duration }) => console.log('duration:', duration)
  })
  ```
- Upload a real MP3 file via console: create objectURL, call `Player.load(url, meta)`, then `Player.play()`
- Verify: audio plays, timeupdate fires, metadata callback fires
- On Android Chrome: verify lock screen shows metadata and controls
- Remove test code after verification

---

## Layer 4: UI (HTML Markup + CSS)

**Goal:** Complete visual UI — all markup, all styling, responsive, aurora, glassmorphism. Non-functional (no JS wiring yet).

### Step 4.1 — `index.html` full markup

Replace placeholder body with complete markup per spec section 6 and PRD section PROMPT 3.

Structure inside `<body>`:
1. Aurora background (3 orbs, aria-hidden)
2. `<main class="player-wrapper">`
   - `<section class="player-card">` containing:
     - Album art container (SVG progress ring + img + fallback + spinner)
     - Song info (title + artist + like/share buttons)
     - Progress bar (seekable, with time display)
     - Controls (shuffle/prev/play/next/repeat)
     - Volume (mute + slider + display)
     - Visualizer container (empty, filled by JS)
     - Mobile playlist toggle button
   - `<aside class="playlist-card">` containing:
     - Header (Library title + track count)
     - Search (input with icon + clear button)
     - Playlist body (ul for track list + empty state + no results state)
     - Upload zone (drag-drop area + "Tambah Lagu" button + hidden file input)
     - Install banner (hidden by default)
3. Mobile overlay (for bottom sheet backdrop)
4. Hidden cover-input file picker
5. Toast container

All elements must have correct ids, aria-labels, roles, data-lucide attributes as listed in the PRD.

**File:** `index.html`

### Step 4.2 — `style.css` complete

Replace foundation-only CSS with full styling per spec section 8 and PRD section PROMPT 3.

Sections to implement (in order):
1. CSS reset (*, box-sizing, margin/padding 0, smooth scrolling)
2. Design tokens (:root variables — already from Layer 1)
3. Body styling
4. Aurora background (3 orbs, keyframes aurora-float-1/2/3)
5. Player wrapper (flex, max-width 900px, centered)
6. Player card (glassmorphism, 360px, flex column)
7. Album art container + SVG ring (270px circle, glow, spin animation)
8. Song info (title marquee, artist, action buttons)
9. Progress bar (seekable, fill gradient, thumb, time display)
10. Controls (flex row, play button prominent with gradient)
11. Volume (mute button, custom range slider, percentage)
12. Visualizer (24 bars, gradient, height animation keyframes)
13. Mobile playlist toggle (hidden on desktop)
14. Playlist card (glassmorphism, flex column)
15. Playlist header (title + track count)
16. Search (input with icon, clear button)
17. Playlist list + items (thumbnail, info, duration, hover actions)
18. Upload zone (dashed border, drag-over state, upload button)
19. Install banner (purple accent, install/dismiss buttons)
20. Toast container + toast styles (info/success/error, animations)
21. Scrollbar customization (webkit + Firefox)
22. Responsive mobile (max-width 767px) — stacked layout, bottom sheet, overlay
23. Reduced motion (@media prefers-reduced-motion)
24. Focus visible outlines

**File:** `style.css`

### Layer 4 Verification

- Open in desktop browser (1280px+): player card left, playlist card right, side-by-side
- Aurora orbs animate smoothly
- Album art area shows circular placeholder with glow
- All glassmorphism surfaces visible (blur, border, shadow)
- Resize to <768px:
  - Layout stacks vertically
  - Album art shrinks to 200px
  - Playlist card hidden, toggle button visible
- Check: no scrollbar issues, no overflow, no broken layout
- Fonts load correctly (Space Grotesk, Inter, JetBrains Mono)
- Lucide icons render
- Test reduced-motion: enable in OS → no animations

---

## Layer 5: Glue (`script.js`)

**Goal:** Wire everything together — state, events, uploads, playback, search, install prompt.

### Step 5.1 — `script.js` complete implementation

Replace stub with full orchestrator per spec section 7 and PRD section PROMPT 4.

**Imports and state:**
```javascript
import * as SongDB from './db.js'
import { Player } from './player.js'

const state = {
  songs: [],
  currentIndex: -1,
  isShuffle: false,
  repeatMode: 'none',
  liked: new Set(),
  shuffleOrder: [],
  deferredInstallPrompt: null,
}
```

**DOM refs:** Grab all elements by id at top of file.

**Functions to implement (grouped):**

Initialization:
- `init()` — load songs from IDB, load prefs from localStorage, init Player with callbacks, build visualizer, render playlist, load first song if exists, setup install prompt, create Lucide icons

Visualizer:
- `buildVisualizer(n)` — create n bar divs with random CSS vars (--max-h, --dur, --delay)

Song loading:
- `loadSong(index)` — set currentIndex, call Player.load, update album art, update song info, reset progress, set active item, update document title
- `updateAlbumArt(song)` — show cover image or gradient+initial fallback
- `updateSongInfo(title, artist)` — update DOM text

Playback:
- `togglePlay()` — Player.toggle, update button icon, toggle .playing class
- `playNext()` — handle shuffle/repeat logic, load next song
- `playPrev()` — restart if >3s, else prev song
- `stopPlayer()` — pause + remove .playing + update icon
- `handleEnded()` — call playNext

Progress:
- `updateProgress(currentTime, duration)` — update fill width + current time text
- `updateProgressRing(ratio)` — update SVG ring stroke-dashoffset
- `resetProgress()` — zero out fill + times + ring
- `seekTo(e)` — calculate ratio from click position, call Player.seek

Volume:
- `updateVolume(e)` — set Player volume, update display + icon + slider style + localStorage
- `updateVolumeSliderStyle(v)` — linear-gradient fill on range input
- `toggleMute()` — mute/unmute, update icon
- `updateVolumeIcon(v)` — set correct Lucide icon (volume-x/volume-1/volume-2)

Shuffle + Repeat:
- `toggleShuffle()` — toggle state, Fisher-Yates if on, update button, toast
- `toggleRepeat()` — cycle modes, update icon + button, toast

Like:
- `toggleLike()` — toggle in Set, update button class, toast

Utilities:
- `formatTime(sec)` — "M:SS" format
- `generateGradient(str)` — deterministic gradient from string hash
- `showToast(msg, type, duration)` — create toast element, auto-remove
- `showSpinner(show)` — toggle loading spinner display

Playlist rendering:
- `renderPlaylist(filter)` — build playlist items, show/hide empty state, update count
- `setActiveItem(index)` — toggle .active class on playlist items
- `updateNowPlayingMini(title)` — update mini title in mobile toggle button

File upload:
- `handleFileUpload(files)` — filter audio files, extract duration via temp Audio(), call SongDB.addSong, push to state, render, auto-play if first
- `handleCoverUpload(songId, file)` — call SongDB.updateCover, update state + UI
- `handleDeleteSong(songId)` — call SongDB.deleteSong, update state + UI
- `triggerCoverUpload(songId)` — open hidden file picker, wire onchange

Search:
- Filter songs by title/artist on input event, call renderPlaylist(filtered), show/hide clear button and no-results state

PWA install:
- `setupInstallPrompt()` — listen for beforeinstallprompt, show banner; listen for appinstalled, hide banner + toast
- Install button click → `state.deferredInstallPrompt.prompt()`
- Dismiss button click → hide banner

Preferences:
- `loadPrefs()` — restore volume, repeatMode, isShuffle from localStorage, apply to UI

**Event listeners (wired in init):**

- Progress bar: click/mousedown/mousemove/mouseup + touch equivalents for seeking
- Volume slider: input event
- Buttons: play, prev, next, shuffle, repeat, like, mute, add-songs, install, dismiss, share
- File input: change → handleFileUpload
- Upload zone: dragover/dragleave/drop
- Playlist delegation: click on items/cover-upload-btn/delete-btn
- Search input: input + clear button
- Mobile: playlist toggle + overlay click
- Keyboard: Space (play), ArrowLeft (prev), ArrowRight (next), M (mute), L (like), Escape (close playlist) — skip if focus on input
- Copy track info: share-btn → clipboard API with fallback

**File:** `script.js`

### Layer 5 Verification

End-to-end test sequence:
1. Open app → empty state shown, no errors
2. Click "Tambah Lagu" → pick MP3 files → songs appear in playlist with duration
3. Click a song → album art updates (fallback gradient), title/artist shown
4. Play → audio plays, progress bar moves, ring fills, visualizer animates, .playing class active
5. Pause → audio stops, visualizer stops, icon switches
6. Seek → click progress bar → audio jumps to position
7. Volume → drag slider → volume changes, icon updates, percentage updates
8. Mute → click mute → volume 0, icon changes, unmute restores
9. Next/Prev → song changes, auto-plays
10. Shuffle → toast, order randomized
11. Repeat → cycle modes (none/all/one), toast per mode
12. Upload cover → pick image → thumbnail + album art update
13. Delete song → removed from list, next song loads
14. Search → type in search → list filters, clear button works
15. Close + reopen browser → songs still in playlist (IndexedDB persist)
16. Volume/shuffle/repeat preferences restored from localStorage
17. Keyboard shortcuts work (Space, arrows, M, L, Esc)
18. Mobile: playlist toggle opens/closes bottom sheet, overlay works
19. Desktop: side-by-side layout correct
20. No console errors

---

## Post-Implementation Checks

After all 5 layers are complete:

1. **Lighthouse audit:** Run PWA audit, check installability
2. **Android test:** Deploy to GitHub Pages or Netlify, test on Chrome Android
   - Install banner appears
   - Install from home screen → standalone mode
   - Lock screen controls work (play/pause/prev/next with cover art)
   - Offline: disable network → app still works, songs play
3. **Cross-browser:** Test on Firefox, Safari (desktop)
4. **Accessibility:** Tab through all controls, verify focus outlines, screen reader labels
5. **Reduced motion:** Enable in OS settings → verify no animations
6. **No console.log in production:** Remove all debug logging

---

## File Dependency Graph

```
generate-icons.mjs  →  icons/*.png  (dev-time only)
manifest.json       ←  index.html (link rel=manifest)
sw.js               ←  index.html (registration script)
db.js               ←  script.js (import)
player.js           ←  script.js (import)
style.css           ←  index.html (link rel=stylesheet)
script.js           ←  index.html (script type=module)
```

Build order respects this: shell first (manifest, sw, html skeleton), then independent modules (db, player), then UI (html+css), then glue (script).

---

## Implementation Order Summary

| Step | Files | What |
|------|-------|------|
| 1.1 | `manifest.json` | PWA manifest |
| 1.2 | `generate-icons.mjs` | Icon generator + run it |
| 1.3 | `sw.js` | Service worker (cache-first + runtime font cache) |
| 1.4 | `index.html` | Minimal HTML skeleton |
| 1.5 | `style.css` | CSS reset + design tokens |
| 1.6 | `script.js` | Stub import |
| 1.7 | `db.js`, `player.js` | Module stubs |
| **Verify Layer 1** | | SW registers, manifest valid, no errors |
| 2.1 | `db.js` | Full IndexedDB implementation |
| **Verify Layer 2** | | Console CRUD tests pass, data persists |
| 3.1 | `player.js` | Full audio + Media Session |
| **Verify Layer 3** | | Audio plays, callbacks fire, lock screen works |
| 4.1 | `index.html` | Full markup (all components) |
| 4.2 | `style.css` | Full styling (glassmorphism, aurora, responsive) |
| **Verify Layer 4** | | Layout correct desktop + mobile, aurora animates |
| 5.1 | `script.js` | Full orchestrator (state, events, flows) |
| **Verify Layer 5** | | End-to-end: upload → play → persist → offline |
