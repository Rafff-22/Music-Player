# Nebula Player PWA — P0 Design Spec

**Date:** 2026-08-10  
**Scope:** MVP (P0) — Core playback, upload, persistence, PWA, offline  
**Target:** Android PWA, also desktop-responsive  
**Build approach:** Incremental (5 layers)

---

## 1. Vision

Nebula Player is a personal music player PWA for Android. Users upload MP3/audio files, which persist permanently on-device via IndexedDB. Playback controls are accessible from Android lock screen via Media Session API. The UI features glassmorphism design with animated aurora background and a rotating album art "planet" at the center. Fully offline after install.

---

## 2. File Structure & Architecture

```
music-player/
├── index.html          ← Entry point, PWA meta tags, markup skeleton
├── style.css           ← All styling, animations, responsive design
├── script.js           ← UI coordinator (state, events, DOM)
├── player.js           ← Deep module: HTMLAudioElement + Media Session
├── db.js               ← Deep module: IndexedDB wrapper
├── manifest.json       ← PWA manifest (installable metadata)
├── sw.js               ← Service Worker (cache-first strategy)
├── generate-icons.mjs  ← Node.js dev tool (icon generator)
└── icons/
    ├── icon-192.png    ← Generated PWA icon
    └── icon-512.png    ← Generated PWA icon (maskable)
```

### 2.1 Module Design (Deep Modules)

**`db.js` — SongDB (IndexedDB wrapper)**
- **Exported interface:** `addSong()`, `updateCover()`, `getAllSongs()`, `deleteSong()`, `clearAll()`
- **Hidden internals:** IDBDatabase, transactions, cursors, objectStore, onupgradeneeded
- **Lazy initialization:** DB opened on first operation, promise cached (singleton)
- **Error handling:** All operations wrapped in try/catch; meaningful error messages if private browsing or storage full

**`player.js` — Player (Audio + Media Session)**
- **Exported interface:** `init()`, `load()`, `play()`, `pause()`, `toggle()`, `seek()`, `setVolume()`, getters for `currentTime`, `duration`, `isPlaying`
- **Hidden internals:** HTMLAudioElement, MediaMetadata, setActionHandler, Media Session API calls
- **Media Session:** Guards all calls with `if ('mediaSession' in navigator)`; wires play/pause/previoustrack/nexttrack/seekbackward/seekforward
- **Callbacks:** init() accepts callbacks `onTimeUpdate`, `onEnded`, `onError`, `onLoading`, `onReady`, `onMetadata`; previoustrack/nexttrack actions call back via optional `onPrev`/`onNext` callbacks

**`script.js` — UI Coordinator**
- **Responsibility:** State management, DOM updates, event listeners, routing between Player and SongDB
- **Never touches:** HTMLAudioElement directly, IndexedDB directly
- **Data flow:** User action → script.js → Player/SongDB → callbacks → DOM update

---

## 3. IndexedDB Schema

**Database name:** `NebulaDB`  
**Version:** 1  
**Object Store:** `songs`

| Field | Type | Description |
|-------|------|-------------|
| `id` | number (autoIncrement, keyPath) | Primary key |
| `title` | string | Song title (derived from filename at upload) |
| `artist` | string | Artist name (default: "Unknown") |
| `duration` | string | Duration in format "M:SS" (extracted at upload) |
| `audioBlob` | Blob | Raw audio file (MP3/WAV/OGG/FLAC/AAC/M4A) |
| `coverBlob` | Blob \| null | Cover image (JPEG/PNG), nullable |
| `dateAdded` | number | Timestamp `Date.now()` |

### 3.1 db.js Interface

```javascript
export async function addSong(audioFile, meta)
  // Input: audioFile (File), meta { title, artist, duration }
  // Process: Store audioBlob + meta in IDB, extract duration if not provided
  // Output: Promise<{ id: number, audioURL: string }>

export async function updateCover(id, imageFile)
  // Input: id (number), imageFile (File)
  // Output: Promise<coverURL: string> (fresh objectURL)

export async function getAllSongs()
  // Output: Promise<Song[]> where Song = { id, title, artist, duration, audioURL, coverURL, dateAdded }
  // Note: Returns fresh objectURLs for audioBlob and coverBlob (if not null)

export async function deleteSong(id)
  // Removes song record from store
  // Output: Promise<void>

export async function clearAll()
  // Clears entire store (dev utility)
  // Output: Promise<void>
```

---

## 4. Player Module (player.js)

### 4.1 Exported Interface

```javascript
export const Player = {
  init({ onTimeUpdate, onEnded, onError, onLoading, onReady, onMetadata, onPrev, onNext }),
  load(src, { title, artist, coverSrc }),
  play(),           // Promise<void>
  pause(),          // void
  toggle(),         // void
  seek(pct),        // pct: 0-100, void
  setVolume(v),     // v: 0-1, void
  get currentTime(),    // number
  get duration(),       // number
  get isPlaying()       // boolean
}
```

### 4.2 Event Callbacks

- `onTimeUpdate(currentTime: number, duration: number)` — fires on every `timeupdate` event
- `onEnded()` — fires when track ends
- `onError(error: Error)` — fires on playback error
- `onLoading()` — fires on `waiting` event (buffering)
- `onReady()` — fires on `canplay` event
- `onMetadata({ duration: number })` — fires on `loadedmetadata` event
- `onPrev()` — wired to Media Session `previoustrack` action
- `onNext()` — wired to Media Session `nexttrack` action

### 4.3 Media Session

Set in `load()`:
```javascript
navigator.mediaSession.metadata = new MediaMetadata({
  title,
  artist,
  artwork: coverSrc ? [{ src: coverSrc, sizes: '512x512', type: 'image/jpeg' }] : []
})

setActionHandler('play', () => /* play */)
setActionHandler('pause', () => /* pause */)
setActionHandler('previoustrack', () => onPrev?.())
setActionHandler('nexttrack', () => onNext?.())
setActionHandler('seekbackward', () => /* seek -15s */)
setActionHandler('seekforward', () => /* seek +15s */)
```

All guarded by `if ('mediaSession' in navigator)`.

---

## 5. PWA & Service Worker

### 5.1 manifest.json

```json
{
  "name": "Nebula Player",
  "short_name": "Nebula",
  "description": "Personal music player — always with you",
  "start_url": "./",
  "display": "standalone",
  "orientation": "portrait-primary",
  "theme_color": "#080810",
  "background_color": "#080810",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

### 5.2 Service Worker (sw.js)

**Strategy:** Cache-first (try cache, fallback network, cache response)

**Install:**
- Pre-cache static assets only: `./`, `index.html`, `style.css`, `script.js`, `player.js`, `db.js`, `manifest.json`, `icons/icon-192.png`, `icons/icon-512.png`
- Cache name: `nebula-v1`
- **Do NOT pre-cache Google Fonts** — CDN URLs are content-hashed and dynamic, unknown at install time

**Fetch:**
- Static assets: cache-first (serve from pre-cache)
- `fonts.googleapis.com` and `fonts.gstatic.com` requests: **runtime cache-first** (cache on first fetch, serve from cache thereafter). This handles the dynamic CDN URLs correctly.
- No special handling for audio blobs (they're in IndexedDB, not network)

**Activate:**
- Delete caches with name !== `nebula-v1`

### 5.3 Icon Generator (generate-icons.mjs)

Node.js script that generates 192×192 and 512×512 PNG icons.
- Background: `#8b5cf6` (purple)
- Content: white music note (♫) centered
- Output: `icons/icon-192.png`, `icons/icon-512.png`

Run: `node generate-icons.mjs`

---

## 6. UI Layout & Components

### 6.1 Desktop Layout (>767px)

Side-by-side, max-width 900px, centered.

```
┌─ Player Card (360px) ─┬─ Playlist Card (flex) ─┐
│ ┌─ Album Art ─────┐   │ ┌─ Header ────────────┐│
│ │  ◯ (ring)       │   │ │ Library    3 songs  ││
│ │   rotating      │   │ └─────────────────────┘│
│ └─────────────────┘   │ ┌─ Search ────────────┐│
│ Title / Artist ♡ 📤   │ │ 🔍 Search...        ││
│ ━━━━●━━━━━━━━━ 3:24   │ └─────────────────────┘│
│ 🔀 ⏮ ▶️ ⏭ 🔁         │ ┌─ Track List ───────┐│
│ 🔊 ━━━━━ 80%          │ │ Song 1 ─────── 3:24││
│ ┌─ Visualizer ─┐      │ │ Song 2 ─────── 4:01││
│ │ ▁ ▂ ▃ ▂ ▁   │      │ │ Song 3 ─────── 2:55││
│ └───────────────┘      │ └─────────────────────┘│
│                        │ ┌─ Upload Zone ──────┐│
│                        │ │ [+ Tambah Lagu]     ││
│                        │ └─────────────────────┘│
│                        │ [Install Banner]      │
└────────────────────────┴───────────────────────┘
```

### 6.2 Mobile Layout (<768px)

Stacked, player full-width, playlist hidden in bottom sheet.

```
┌────────────────────────┐
│ ┌─ Album Art ────────┐ │
│ │  ◯ (ring)         │ │
│ │   rotating        │ │
│ └───────────────────┘ │
│ Title / Artist    ♡ 📤 │
│ ━━━●━━━━━━━━━ 3:24    │
│ 🔀 ⏮ ▶️ ⏭ 🔁         │
│ 🔊 ━━━━━ 80%          │
│ ┌─ Visualizer ──────┐ │
│ │ ▁ ▂ ▃ ▂ ▁        │ │
│ └────────────────────┘ │
│ [📋 Playlist ⬆️]       │ ← toggle button
└────────────────────────┘

Bottom Sheet (72vh, collapsed by default):
┌────────────────────────┐
│ Library    3 songs     │
│ 🔍 Search...           │
│ Song 1 ─────── 3:24 📷 │
│ Song 2 ─────── 4:01 📷 │
│ Song 3 ─────── 2:55 📷 │
│ [+ Tambah Lagu]        │
│ [Install Banner]       │
└────────────────────────┘
```

### 6.3 Components

1. **Aurora Background** — 3 blurred orbs (fixed, z-index 0, non-interactive)
2. **Album Art Container** — Circular image, 270px desktop/200px mobile, with purple glow, rotating SVG progress ring
3. **Song Info** — Title (marquee if overflow) + artist + like/share buttons
4. **Progress Bar** — Horizontal seekable bar, JetBrains Mono time display
5. **Controls** — Shuffle / Prev / Play (prominent) / Next / Repeat
6. **Volume** — Mute button + range slider + percentage
7. **Visualizer** — 24 animated bars (gradient purple→cyan), below volume
8. **Playlist Card** — Header (Library + count) + search + scrollable track list + upload zone + install banner
9. **Playlist Item** — 40px circular thumbnail + title/artist + duration + cover upload (hover) + delete (hover)
10. **Toast Container** — Fixed bottom-right, auto-dismiss after 3s

---

## 7. State Management (script.js)

```javascript
const state = {
  songs: [],              // Song[] from SongDB.getAllSongs()
  currentIndex: -1,       // -1 = no song selected
  isShuffle: false,
  repeatMode: 'none',     // 'none' | 'all' | 'one'
  liked: new Set(),       // Set<id> per-session (not persisted)
  shuffleOrder: [],       // Array<index> when shuffle enabled
  deferredInstallPrompt: null,
}

// Persisted to localStorage:
//   nebula-volume (float 0-1)
//   nebula-repeat ('none'|'all'|'one')
//   nebula-shuffle ('true'|'false')
```

### 7.1 Key Flows

**Upload Flow:**
1. User picks files (file picker or drag-drop)
2. Filter audio files (MIME type or extension)
3. For each file:
   - Create temp Audio() to extract duration via `loadedmetadata` event
   - Call `SongDB.addSong(file, { title, artist, duration })`
   - Returns `{ id, audioURL }`
4. Push to `state.songs`, call `renderPlaylist()`
5. If first song, `loadSong(0)` + `Player.play()`

**Load Song Flow:**
1. `loadSong(index)` — set `state.currentIndex = index`
2. Get song from `state.songs[index]`
3. `Player.load(song.audioURL, { title, artist, coverSrc: song.coverURL })`
4. Update album art (cover image or gradient fallback)
5. Update song info (title, artist)
6. Reset progress display
7. Mark item as active in playlist
8. Set document title

**Playback Control Flows:**

- **Play/Pause:** `Player.toggle()` + update UI button icon + add/remove `.playing` class
- **Next:** Respects shuffle order (if enabled) and repeat mode
  - If `repeatMode === 'one'`: seek to 0 and play
  - If shuffle: pick next from shuffleOrder
  - Else: `currentIndex + 1`
  - If end of list and `repeatMode === 'all'`: loop to start
  - If end of list and `repeatMode === 'none'`: stop playback
- **Prev:** If currentTime > 3s, seek to 0; else go to previous song
- **Seek:** Update `Player.currentTime` via percentage of total duration
- **Shuffle:** Toggle + regenerate shuffleOrder (Fisher-Yates) + show toast
- **Repeat:** Cycle through modes (none → all → one → none) + show toast

**Error Handling:**
- Upload error → show toast error, continue batch upload
- Playback error → show toast error, don't crash
- IDB unavailable → show toast, app degrades gracefully

---

## 8. Styling & Visual Design

### 8.1 Design Tokens

```css
:root {
  /* Colors */
  --bg-primary: #080810;
  --bg-secondary: #0f0f1a;
  --bg-glass: rgba(255,255,255,0.04);
  --bg-glass-hover: rgba(255,255,255,0.07);
  --border-glass: rgba(255,255,255,0.08);
  --border-glass-strong: rgba(255,255,255,0.15);
  --color-purple: #8b5cf6;
  --color-purple-glow: rgba(139,92,246,0.4);
  --color-cyan: #06b6d4;
  --color-pink: #f472b6;
  --text-primary: #f1f5f9;
  --text-secondary: #94a3b8;
  --text-muted: #475569;
  --text-accent: #c4b5fd;

  /* Aurora */
  --aurora-1: rgba(76,29,149,0.6);
  --aurora-2: rgba(14,116,144,0.5);
  --aurora-3: rgba(131,24,67,0.4);

  /* Spacing & Radius */
  --radius-sm: 8px;
  --radius-md: 16px;
  --radius-lg: 24px;
  --radius-xl: 32px;
  --radius-full: 9999px;

  /* Timing */
  --transition-fast: 150ms ease;
  --transition-normal: 250ms ease;

  /* Shadow */
  --shadow-card: 0 25px 60px rgba(0,0,0,0.6);
}
```

### 8.2 Glassmorphism

```css
.glass-card {
  background: var(--bg-glass);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border: 1px solid var(--border-glass);
  box-shadow: var(--shadow-card);
}
```

### 8.3 Aurora Background

- 3 fixed orbs, z-index 0, pointer-events none
- Orb 1: 700px, top-left, --aurora-1, blur(80px), 18s animation
- Orb 2: 600px, bottom-right, --aurora-2, blur(80px), 22s animation
- Orb 3: 500px, center, --aurora-3, blur(80px), 20s animation
- Each orb animates with different `translate` and `scale` keyframes, smooth cubic-bezier motion

### 8.4 Album Art Signature

- Circular (270px desktop, 200px mobile), border-radius 50%
- Glow: `0 0 0 4px rgba(139,92,246,0.25), 0 0 60px rgba(139,92,246,0.3), 0 20px 40px rgba(0,0,0,0.5)`
- When playing: glow intensifies, image rotates 22s linear infinite
- Fallback (no cover): gradient generated from song title hash + first letter

### 8.5 Fonts

- **Heading:** Space Grotesk 700 (song title, playlist header)
- **Body:** Inter 400/500 (metadata, labels, descriptions)
- **Mono:** JetBrains Mono 400 (time display)
- All via Google Fonts CDN, cached by Service Worker

### 8.6 Responsive Breakpoints

- **Desktop (>767px):** side-by-side layout, player 360px, playlist flex
- **Mobile (≤767px):** stacked layout, player full-width, playlist bottom sheet (72vh)
- **Touch targets:** min 44×44px
- **Safe area:** respect `env(safe-area-inset-bottom)` for notch/home indicator

### 8.7 Accessibility

- All interactive elements keyboard-accessible
- Focus outline: 2px solid --color-purple, offset 3px
- aria-labels on all buttons (play/pause, mute/unmute, etc.)
- aria-valuenow on progress bar, updated during playback
- progress-bar role="slider"
- Reduced motion: `@media (prefers-reduced-motion: reduce)` disables all animations

---

## 9. Key Decisions & Trade-offs

### Duration Extraction
**Decision:** Extract duration at upload time (not lazy)  
**Trade-off:** Slight upload delay for faster display in playlist  
**Implementation:** Temp Audio() object, read via `loadedmetadata` event

### Album Art Fallback
**Decision:** Gradient + first letter (hash-based colors consistent per song)  
**Alternative:** Generic music icon (simpler, less distinctive)  
**Chosen for:** Visual personality, consistency, no network needed

### Mobile Playlist UX
**Decision:** Bottom sheet (slide up, toggle button)  
**Alternative:** Always visible below player  
**Chosen for:** Focus on album art on small screens, progressive disclosure

### Build Approach
**Decision:** Incremental (5 layers: PWA shell → db layer → player layer → UI → glue)  
**Alternative:** Monolith (all at once)  
**Chosen for:** Testability at each layer, risk mitigation

---

## 10. Testing Strategy

| Layer | What to Test | Success Criteria |
|-------|--------------|------------------|
| Layer 1: PWA shell | Manifest valid, SW registers, start_url accessible | Lighthouse PWA checklist passes |
| Layer 2: db.js | addSong, getAllSongs, updateCover, deleteSong | Data persists after browser restart |
| Layer 3: player.js | Audio playback, Media Session, callbacks | Lock screen shows title/artist/cover, controls work |
| Layer 4: UI (HTML+CSS) | Layout responsive, glassmorphism visible, aurora animates | Desktop side-by-side, mobile bottom sheet, animations smooth |
| Layer 5: script.js | Upload, play, shuffle, repeat, seek, volume | End-to-end: upload → play → shuffle → repeat → persist |

---

## 11. P0 Feature List (MVP)

- [ ] Play / Pause / Seek / Volume / Mute
- [ ] Upload audio files (single/multiple)
- [ ] Extract & display duration
- [ ] Upload/change cover per song (with fallback)
- [ ] IndexedDB persistence (survive app restart)
- [ ] PWA manifest + installable Android
- [ ] Service Worker + offline support
- [ ] Media Session API (lock screen controls)
- [ ] Playlist with search
- [ ] Shuffle + Repeat (off/all/one)
- [ ] Glassmorphism + aurora background
- [ ] Visualizer (24 animated bars)
- [ ] Responsive mobile/desktop
- [ ] Keyboard shortcuts (Space, Arrow, M, L, Esc)

---

## 12. Deployment

**Requirements:** HTTPS (for PWA install)

**Options:**
- **GitHub Pages:** Free, HTTPS built-in
- **Netlify:** Drag-and-drop, HTTPS built-in
- **LAN:** `npx serve .` for local testing (no HTTPS, but works on WiFi)

**Install on Android:**
1. Deploy to GitHub Pages / Netlify
2. Open URL in Chrome Android
3. Tap menu (⋮) → "Add to Home Screen"
   OR wait for install banner
4. Launch from home screen → fullscreen, standalone

---

## 13. Success Criteria

- [ ] Lagu yang diupload tetap ada setelah app ditutup & dibuka lagi
- [ ] Lock screen Android menampilkan judul + artis + cover + kontrol
- [ ] App bisa di-launch dari home screen sebagai standalone
- [ ] Upload lagu baru berfungsi dari dalam app (kapan saja)
- [ ] Upload/ganti cover per lagu berfungsi
- [ ] App berjalan offline sepenuhnya
- [ ] Install banner muncul di Chrome Android
- [ ] Layout responsif: side-by-side desktop, bottom sheet mobile
- [ ] Visualizer animates saat playing
- [ ] No console errors in production code

---

**Design finalized:** 2026-08-10  
**Spec version:** 1.0  
**Next step:** Implementation plan (incremental 5-layer build)
