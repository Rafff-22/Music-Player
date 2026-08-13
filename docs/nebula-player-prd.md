# 🎵 PRD — Nebula Player (PWA Edition)
**Versi:** 2.0 | **Platform:** Android PWA | **Status:** Ready for OpenCode

> Skills aktif: `brainstorming` · `grill-me` · `honey` · `frontend-design` · `impeccable` · `codebase-design`

---

## Yang Baru di v2.0
| Fitur | Keterangan |
|-------|-----------|
| 📱 PWA Android | Installable dari Chrome, fullscreen, ikon di home screen |
| 💾 IndexedDB | Lagu & cover tersimpan permanen di device |
| 🔒 Media Session API | Kontrol play/pause/prev/next dari lock screen & notif bar |
| 🧩 Module split | `db.js` + `player.js` + `script.js` (deep modules) |
| 🚀 Skill-aware prompts | Setiap prompt invoke skill yang tepat |

---

## 1. Vision & Goals

**Vision:** Music player pribadi yang di-install di Android sebagai app — berjalan offline, simpan lagu kapan saja, kontrol dari lock screen, tampilan memukau.

| # | Tujuan | Ukuran Sukses |
|---|--------|--------------|
| 1 | Installable di Android | "Add to Home Screen" muncul di Chrome |
| 2 | Offline support | Berjalan tanpa internet setelah install |
| 3 | Tambah lagu kapan saja | File picker + IndexedDB persist antar session |
| 4 | Kontrol lock screen | Media Session API: judul + artis + artwork + controls |
| 5 | UI memukau | Glassmorphism + aurora, bukan design template |

---

## 2. Fitur

### P0 — MVP (harus ada)
- Play / Pause / Next / Prev / Seek / Volume / Mute
- Upload lagu via file picker (multiple) atau drag & drop
- Upload cover per lagu (kapan saja, bisa diubah)
- **IndexedDB:** lagu & cover tersimpan setelah app ditutup
- **PWA:** manifest.json + service worker = installable + offline
- **Media Session:** kontrol dari notification bar & lock screen Android
- Playlist dengan album art thumbnail
- Responsive: mobile-first (390px) + desktop

### P1 — Segera setelah MVP
- Shuffle + Repeat (off / all / one)
- Search playlist (filter title/artist)
- Animated equalizer visualizer
- Aurora background animation
- Install prompt banner ("Install sebagai app")

### P2 — Nanti
- Edit metadata lagu (judul / artis)
- Hapus lagu dari library
- Sort playlist (by title / date added)

---

## 3. Arsitektur Teknis

### 3.1 File Structure

```
music-player/
├── index.html          ← Entry point, markup, SW registration
├── style.css           ← Semua styling & animasi
├── script.js           ← UI coordinator (imports player + db)
├── player.js           ← Deep module: HTMLAudioElement + MediaSession
├── db.js               ← Deep module: IndexedDB (song & cover storage)
├── manifest.json       ← PWA manifest
├── sw.js               ← Service Worker (cache-first, offline)
└── icons/
    ├── icon-192.png    ← PWA icon (dihasilkan via Canvas)
    └── icon-512.png    ← PWA icon maskable
```

### 3.2 Module Design (codebase-design: deep modules)

**`db.js` — SongDB** *(interface kecil, implementasi tersembunyi)*
```
Interface:
  addSong(audioFile, meta)     → Promise<{id, audioURL}>
  updateCover(id, imageFile)   → Promise<coverURL>
  getAllSongs()                 → Promise<Song[]>
  deleteSong(id)               → Promise<void>

Tersembunyi di dalam:
  openDB(), IDBRequest, objectStore, transaction,
  onupgradeneeded, URL.createObjectURL
```

**`player.js` — Player** *(interface kecil, implementasi tersembunyi)*
```
Interface:
  init(callbacks)              → void
  load(src, meta)              → void
  play()                       → Promise<void>
  pause()                      → void
  toggle()                     → void
  seek(percent)                → void
  setVolume(0-1)               → void
  get currentTime              → number
  get duration                 → number
  get isPlaying                → boolean

Tersembunyi di dalam:
  HTMLAudioElement, navigator.mediaSession,
  MediaMetadata, setActionHandler
```

**`script.js` — UI Coordinator** *(tidak sentuh IndexedDB atau Audio langsung)*
```
Orchestrates:
  SongDB ← → state ← → DOM
  Player ← → state ← → DOM

Handles:
  Events, render, state, routing, toasts, install prompt
```

### 3.3 State (script.js)

```javascript
const state = {
  songs: [],           // Song[] dari SongDB.getAllSongs()
  currentIndex: 0,
  isShuffle: false,
  repeatMode: 'none',  // 'none' | 'all' | 'one'
  liked: new Set(),    // Set<id>
  shuffleOrder: [],
}
// Volume, repeatMode, isShuffle → localStorage (primitif, bukan Blob)
```

### 3.4 IndexedDB Schema

```
Database: "NebulaDB"  version: 1
Store: "songs"
  id         → autoIncrement (keyPath)
  title      → string
  artist     → string
  duration   → string ("3:24")
  audioBlob  → Blob    ← file MP3/WAV/OGG
  coverBlob  → Blob | null
  dateAdded  → number  (Date.now())
```

### 3.5 PWA Spec

**manifest.json**
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

**Service Worker — Cache Strategy: Cache First**
```
Install  → cache semua static assets
Fetch    → cacheFirst (cache → fallback network)
Activate → hapus cache versi lama
```

**Media Session (player.js)**
```javascript
navigator.mediaSession.metadata = new MediaMetadata({
  title, artist, artwork: [{ src: coverURL, sizes: '512x512' }]
})
// handlers: play, pause, previoustrack, nexttrack, seekbackward, seekforward
```

---

## 4. Design System — Nebula

| Token | Value |
|-------|-------|
| `--bg-primary` | `#080810` |
| `--bg-glass` | `rgba(255,255,255,0.04)` |
| `--border-glass` | `rgba(255,255,255,0.08)` |
| `--color-purple` | `#8b5cf6` |
| `--color-cyan` | `#06b6d4` |
| `--color-pink` | `#f472b6` |
| `--text-primary` | `#f1f5f9` |
| `--text-secondary` | `#94a3b8` |
| Heading | Space Grotesk 700 |
| Body | Inter 400/500 |
| Timer/mono | JetBrains Mono 400 |

**Signature visual:** Album art circular yang bercahaya (glow purple) berputar pelan saat playing, dikelilingi SVG progress ring gradient — efek seperti planet di nebula.

---

## 5. Deployment ke Android

| Opsi | Cara | Kelebihan |
|------|------|-----------|
| **GitHub Pages** | Push repo, aktifkan Pages | Gratis, HTTPS otomatis |
| **Netlify Drop** | Drag folder ke netlify.com/drop | Paling cepat, gratis |
| **LAN Server** | `npx serve .` di PC | Tanpa internet, lewat WiFi |

> PWA memerlukan HTTPS untuk bisa diinstall. GitHub Pages & Netlify sudah HTTPS.
> Setelah deploy: buka URL di Chrome Android → tiga titik → "Add to Home Screen"

---

## 6. Acceptance Criteria

- [ ] Lagu yang diupload tetap ada setelah app ditutup & dibuka lagi
- [ ] Lock screen Android menampilkan judul + artis + cover + kontrol
- [ ] App bisa di-launch dari home screen sebagai standalone (tanpa browser UI)
- [ ] Upload lagu baru berfungsi dari dalam app (bisa kapan saja)
- [ ] Upload cover per lagu berfungsi (ganti cover kapan saja)
- [ ] App berjalan offline sepenuhnya
- [ ] Install banner muncul di Chrome Android
- [ ] Layout responsif: side-by-side di desktop, stack + bottom sheet di mobile

---

---

# 🤖 VIBE CODING PROMPTS untuk OpenCode

> **Urutan wajib — jangan skip, jangan gabung**
> Setiap prompt memanfaatkan skill yang tepat pada waktu yang tepat.

```
Skills yang aktif:
  brainstorming  → HARD-GATE: tidak ada code sebelum design disetujui
  grill-me       → interview untuk tajamkan keputusan
  honey          → auto-aktif di semua coding prompt (terse, no bloat)
  frontend-design → design direction yang tidak templated
  impeccable     → award-winning UI craft (shape → build → polish → adapt → audit)
  codebase-design → deep modules, minimal interface, clean seam
```

---

## 📋 PROMPT 0 — Brainstorming (WAJIB PERTAMA)

> *Skill: `brainstorming` — HARD-GATE aktif, tidak ada kode sampai design disetujui*

```
/brainstorming

Saya ingin membangun Nebula Player: personal music player sebagai PWA yang
di-install di Android.

Kebutuhan utama:
- Simpan lagu & cover permanen di device (IndexedDB)
- Bisa tambah lagu dan ganti cover kapan saja dari dalam app
- Kontrol dari lock screen Android (Media Session API)
- Installable dari Chrome Android (PWA: manifest + service worker)
- Berjalan offline sepenuhnya
- Desain glassmorphism + aurora animated background, dark theme

Stack: Vanilla HTML/CSS/JavaScript, ES Modules, zero framework, zero build tool.
File utama: index.html · style.css · script.js · player.js · db.js · sw.js · manifest.json

Bantu saya plan ini sebelum menulis satu baris kode pun.
```

*→ Jawab pertanyaan AI satu per satu sampai design disetujui dan spec ditulis.*

---

## 🔥 PROMPT 0.5 — Grill Me (Opsional, sangat disarankan)

> *Skill: `grill-me` — interview keras untuk temukan kelemahan sebelum build*

```
/grill-me

Tajamkan keputusan ini sebelum saya mulai build Nebula Player:

1. IndexedDB: apakah Blob storage cukup handal untuk MP3 besar di Android Chrome?
   Edge case: storage limit, eviction policy, private mode.

2. Service Worker cache-first: kapan strategi ini bisa backfire?
   Apakah ada asset yang tidak boleh di-cache?

3. Module split (player.js / db.js / script.js): apakah seam-nya sudah tepat?
   Apa yang paling mungkin bocor ke module lain?

4. Media Session API: browser support di Android? Apa yang tidak didukung?

5. Apa satu hal yang paling sering salah di PWA music player pertama kali?
```

*→ Selesaikan semua jawaban sebelum lanjut ke PROMPT 1.*

---

## 🎨 PROMPT 1 — Shape & Design Direction

> *Skill: `impeccable shape` + `frontend-design` — plan UI/UX dulu, belum ada kode*

```
$impeccable shape

Saya akan membangun Nebula Player — personal music player PWA untuk Android.
Mode: Operate (user menyelesaikan task: memutar dan mengelola musik pribadi).

Brief desain:
- Tema: Glassmorphism di atas deep space dark (#080810)
- Aurora animated background: 3 orb bergerak lambat (purple/cyan/pink nebula)
- Album art: circular, bercahaya (glow purple), berputar saat playing
- SVG progress ring mengelilingi album art
- Animated equalizer visualizer (24 bar)
- Font: Space Grotesk (heading) + Inter (body) + JetBrains Mono (timer)
- Mobile-first portrait 390px, juga menarik di desktop

Target pengguna: satu orang (saya sendiri), personal use.
Signature visual: album art seperti planet berputar di nebula.

Tolong:
1. Tentukan layout desktop vs mobile (ASCII wireframe)
2. Tentukan komponen dan hierarki
3. Tentukan flow interaksi (upload lagu → play → kontrol)
4. Identifikasi satu risiko desain yang perlu diambil agar tidak templated

Belum perlu kode. Shape dulu.
```

*→ Review shape output, minta revisi jika ada yang kurang pas, baru lanjut.*

---

## 🚀 PROMPT 2 — Project Foundation (Scaffolding + PWA + Modules)

> *Skill: `honey full` + `codebase-design` — minimum code, deep modules*

```
[honey full]

Buat seluruh project foundation Nebula Player sekaligus.
Semua file di bawah harus LENGKAP dan RUNNABLE.

── STRUKTUR FOLDER ──
music-player/
├── index.html
├── style.css
├── script.js
├── player.js
├── db.js
├── manifest.json
├── sw.js
└── icons/  (kosong dulu, kita generate di langkah berikutnya)

── FILE 1: manifest.json ──
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

── FILE 2: sw.js (Service Worker) ──
Cache name: "nebula-v1"
Assets to cache: ['./','./index.html','./style.css','./script.js',
  './player.js','./db.js','./manifest.json',
  './icons/icon-192.png','./icons/icon-512.png']
Strategy: Cache First (cache → fallback fetch → cache new response)
Activate: delete cache entries with key !== "nebula-v1"
Tulis lengkap: install + fetch + activate handlers.

── FILE 3: db.js (Deep Module — SongDB) ──
[codebase-design]

IndexedDB wrapper sebagai deep module.
Database "NebulaDB" v1, store "songs":
  keyPath: "id" (autoIncrement)
  fields: title, artist, duration, audioBlob, coverBlob, dateAdded

Interface yang diexport (hanya ini yang boleh dipakai dari luar):
  export async function addSong(audioFile, meta)
    → { id, audioURL }    ← buat objectURL dari audioBlob yang disimpan
  export async function updateCover(id, imageFile)
    → coverURL            ← objectURL dari coverBlob
  export async function getAllSongs()
    → Song[]              ← tiap song include fresh audioURL + coverURL
  export async function deleteSong(id)
    → void
  export async function clearAll()
    → void

Sembunyikan di dalam: openDB(), IDBRequest, transaction, cursor, onupgradeneeded.
openDB() dipanggil lazily (pertama kali ada operasi).
Gunakan satu promise cached untuk DB instance agar tidak buka berkali-kali.

── FILE 4: player.js (Deep Module — Player) ──
[codebase-design]

HTMLAudioElement + MediaSession wrapper sebagai deep module.

export const Player = {
  init({ onTimeUpdate, onEnded, onError, onLoading, onReady })
    → void    ← setup audio events, expose di module scope

  load(src, { title, artist, coverSrc })
    → void    ← set audio.src, update MediaSession metadata + action handlers

  play()      → Promise<void>
  pause()     → void
  toggle()    → void    ← play jika pause, pause jika play
  seek(pct)   → void    ← pct: 0-100, set audio.currentTime
  setVolume(v)→ void    ← v: 0-1

  get currentTime()  → number
  get duration()     → number
  get isPlaying()    → boolean
}

MediaSession: di dalam load(), set:
  navigator.mediaSession.metadata = new MediaMetadata({ title, artist,
    artwork: coverSrc ? [{ src: coverSrc, sizes: '512x512', type: 'image/jpeg' }] : [] })
  setActionHandler untuk: play, pause, previoustrack, nexttrack,
    seekbackward (15s), seekforward (15s)
  Action handlers memanggil callbacks yang di-pass ke init().
  Pastikan: wrap di if ('mediaSession' in navigator) untuk safety.

── FILE 5: index.html (Boilerplate) ──
HTML5, charset UTF-8, viewport responsive.
<meta name="theme-color" content="#080810">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<link rel="manifest" href="manifest.json">
Google Fonts: Space Grotesk (400,700), Inter (400,500), JetBrains Mono (400)
Lucide Icons: https://unpkg.com/lucide@latest/dist/umd/lucide.min.js
<link rel="stylesheet" href="style.css">
<script type="module" src="script.js">
SW registration di <script> non-module terpisah:
  if ('serviceWorker' in navigator)
    window.addEventListener('load', () =>
      navigator.serviceWorker.register('./sw.js').catch(console.error))
<body> kosong (markup diisi prompt berikutnya).

── FILE 6: style.css (CSS Foundation) ──
:root dengan semua design tokens:
  --bg-primary: #080810
  --bg-secondary: #0f0f1a
  --bg-glass: rgba(255,255,255,0.04)
  --bg-glass-hover: rgba(255,255,255,0.07)
  --border-glass: rgba(255,255,255,0.08)
  --border-glass-strong: rgba(255,255,255,0.15)
  --color-purple: #8b5cf6
  --color-purple-glow: rgba(139,92,246,0.4)
  --color-cyan: #06b6d4
  --color-pink: #f472b6
  --text-primary: #f1f5f9
  --text-secondary: #94a3b8
  --text-muted: #475569
  --text-accent: #c4b5fd
  --aurora-1: rgba(76,29,149,0.6)
  --aurora-2: rgba(14,116,144,0.5)
  --aurora-3: rgba(131,24,67,0.4)
  --radius-sm: 8px
  --radius-md: 16px
  --radius-lg: 24px
  --radius-xl: 32px
  --radius-full: 9999px
  --transition-fast: 150ms ease
  --transition-normal: 250ms ease
  --shadow-card: 0 25px 60px rgba(0,0,0,0.6)

CSS Reset: *, box-sizing border-box, margin/padding 0, smooth scrolling.
body: flex center, Inter font, --bg-primary background, overflow-x hidden.

── FILE 7: script.js (Stub) ──
import { SongDB } from './db.js'
import { Player } from './player.js'

const state = {
  songs: [], currentIndex: 0,
  isShuffle: false, repeatMode: 'none',
  liked: new Set(), shuffleOrder: []
}

document.addEventListener('DOMContentLoaded', async () => {
  // TODO: init() dipanggil setelah HTML markup selesai di PROMPT 3
  console.log('Nebula Player booting...')
})

── ICON GENERATOR ──
Tulis juga script Node.js satu file (generate-icons.mjs) yang membuat
icon-192.png dan icon-512.png menggunakan Canvas API atau jimp/sharp
(pilih yang paling minimal). Icon: background #8b5cf6, teks "♫" putih di tengah.
Jalankan: node generate-icons.mjs
Lalu icons/ siap.

Tampilkan semua 8 file lengkap dan runnable.
```

---

## 🏗️ PROMPT 3 — HTML Markup + CSS Lengkap

> *Skill: `impeccable` + `frontend-design` — production UI, tidak templated*

```
$impeccable

Bangun UI Nebula Player. Mode: Operate. Mobile-first: 390px portrait.
Gunakan frontend-design principles: design harus distinctive, tidak membaca
seperti template. Signature element: album art seperti planet di nebula.

── HTML (tambahkan ke <body> di index.html) ──

<!-- Aurora Background -->
<div class="aurora-bg" aria-hidden="true">
  <div class="aurora-orb aurora-orb--1"></div>
  <div class="aurora-orb aurora-orb--2"></div>
  <div class="aurora-orb aurora-orb--3"></div>
</div>

<!-- Main -->
<main class="player-wrapper">

  <!-- Player Card (kiri di desktop, atas di mobile) -->
  <section class="player-card" id="player-card">

    <!-- Album Art + SVG Ring -->
    <div class="album-art-container">
      <svg class="progress-ring" viewBox="0 0 300 300" aria-hidden="true">
        <defs>
          <linearGradient id="ringGrad" x1="0%" y1="0%" x2="100%" y2="100%">
            <stop offset="0%" style="stop-color:#8b5cf6"/>
            <stop offset="100%" style="stop-color:#06b6d4"/>
          </linearGradient>
        </defs>
        <circle class="ring-track" cx="150" cy="150" r="140"/>
        <circle class="ring-fill" id="ring-fill" cx="150" cy="150" r="140"/>
      </svg>
      <div class="album-art-wrap">
        <img id="album-art" alt="Album cover" src=""
          onerror="this.style.display='none'; this.nextElementSibling.style.display='flex'">
        <div class="album-art-fallback" id="album-art-fallback" style="display:none"></div>
        <div class="loading-spinner" id="loading-spinner" aria-label="Loading"></div>
      </div>
    </div>

    <!-- Info + Actions -->
    <div class="song-info">
      <div class="song-meta">
        <h2 class="song-title" id="song-title">Pilih lagu</h2>
        <p class="artist-name" id="artist-name">Nebula Player</p>
      </div>
      <div class="song-actions">
        <button id="like-btn" class="action-btn" aria-label="Like">
          <i data-lucide="heart"></i>
        </button>
        <button id="share-btn" class="action-btn" aria-label="Salin info lagu">
          <i data-lucide="share-2"></i>
        </button>
      </div>
    </div>

    <!-- Progress -->
    <div class="progress-wrapper">
      <div class="progress-bar" id="progress-bar" role="slider"
        aria-label="Posisi lagu" aria-valuemin="0" aria-valuemax="100"
        aria-valuenow="0" tabindex="0">
        <div class="progress-fill" id="progress-fill"></div>
        <div class="progress-thumb"></div>
      </div>
      <div class="time-display">
        <span id="current-time">0:00</span>
        <span id="total-time">0:00</span>
      </div>
    </div>

    <!-- Controls -->
    <div class="controls">
      <button id="shuffle-btn" class="ctrl-btn ctrl-sm" aria-label="Shuffle Off">
        <i data-lucide="shuffle"></i>
      </button>
      <button id="prev-btn" class="ctrl-btn ctrl-md" aria-label="Lagu sebelumnya">
        <i data-lucide="skip-back"></i>
      </button>
      <button id="play-btn" class="ctrl-btn ctrl-play" aria-label="Play">
        <i data-lucide="play" id="play-icon"></i>
      </button>
      <button id="next-btn" class="ctrl-btn ctrl-md" aria-label="Lagu berikutnya">
        <i data-lucide="skip-forward"></i>
      </button>
      <button id="repeat-btn" class="ctrl-btn ctrl-sm" aria-label="Repeat Off">
        <i data-lucide="repeat" id="repeat-icon"></i>
      </button>
    </div>

    <!-- Volume -->
    <div class="volume-wrapper">
      <button id="mute-btn" class="ctrl-btn ctrl-sm" aria-label="Mute">
        <i data-lucide="volume-2" id="volume-icon"></i>
      </button>
      <input type="range" id="volume-slider" min="0" max="100" value="80"
        aria-label="Volume" class="volume-slider">
      <span id="volume-display" class="volume-display">80%</span>
    </div>

    <!-- Visualizer (bars diisi JS) -->
    <div class="visualizer" id="visualizer" aria-hidden="true"></div>

    <!-- Mobile: toggle playlist -->
    <button class="playlist-toggle-mobile" id="playlist-toggle-mobile"
      aria-label="Buka playlist" aria-expanded="false">
      <i data-lucide="list-music"></i>
      <span>Playlist</span>
      <span id="now-playing-mini" class="now-playing-mini"></span>
    </button>

  </section>

  <!-- Playlist Card (kanan di desktop, bottom sheet di mobile) -->
  <aside class="playlist-card" id="playlist-card" aria-label="Playlist">

    <!-- Header -->
    <div class="playlist-header">
      <div class="playlist-title">
        <i data-lucide="music-2"></i>
        <h3>Library</h3>
      </div>
      <span id="track-count" class="track-count">0 lagu</span>
    </div>

    <!-- Search -->
    <div class="search-wrapper">
      <i data-lucide="search" class="search-icon" aria-hidden="true"></i>
      <input type="text" id="search-input" class="search-input"
        placeholder="Cari judul atau artis..." autocomplete="off" spellcheck="false">
      <button id="search-clear" class="search-clear" aria-label="Hapus pencarian" hidden>
        <i data-lucide="x"></i>
      </button>
    </div>

    <!-- Track List -->
    <div class="playlist-body" id="playlist-body">
      <ul id="playlist-list" class="playlist-list" role="listbox"
        aria-label="Daftar lagu"></ul>
      <div id="empty-state" class="empty-state" hidden>
        <i data-lucide="music-2"></i>
        <p>Library kosong</p>
        <small>Upload MP3 untuk mulai</small>
      </div>
      <div id="no-results" class="empty-state" hidden>
        <i data-lucide="search-x"></i>
        <p>Tidak ditemukan</p>
      </div>
    </div>

    <!-- Upload Zone -->
    <div class="upload-zone" id="upload-zone">
      <i data-lucide="upload-cloud"></i>
      <p>Drag & drop file audio</p>
      <span>atau</span>
      <button id="add-songs-btn" class="btn-upload">
        <i data-lucide="plus"></i>
        Tambah Lagu
      </button>
      <input type="file" id="file-input"
        accept="audio/*,.mp3,.wav,.ogg,.flac,.aac,.m4a"
        multiple hidden>
    </div>

    <!-- PWA Install Banner -->
    <div class="install-banner" id="install-banner" hidden>
      <i data-lucide="smartphone"></i>
      <div class="install-text">
        <strong>Install sebagai app</strong>
        <small>Akses cepat dari home screen</small>
      </div>
      <button id="install-btn" class="btn-install">Install</button>
      <button id="install-dismiss" class="btn-dismiss" aria-label="Tutup">
        <i data-lucide="x"></i>
      </button>
    </div>

  </aside>

</main>

<!-- Mobile overlay (di balik bottom sheet) -->
<div class="mobile-overlay" id="mobile-overlay" aria-hidden="true"></div>

<!-- Cover upload picker (shared, hidden) -->
<input type="file" id="cover-input" accept="image/*" hidden>

<!-- Toast container -->
<div id="toast-container" class="toast-container" aria-live="polite" role="status"></div>

── CSS (style.css) — LENGKAP ──

[frontend-design: dua-pass]
Pass 1 (planning): Desain ini tidak templated karena:
- Background aurora bukan gradient statis tapi 3 orb mengambang dengan physics berbeda
- Album art circular berputar seperti planet — bukan cover art biasa
- Progress ring SVG melingkari album art, bukan bar horizontal terpisah
- Warna tidak menggunakan near-black + acid-green (default AI) tapi deep space purple + cyan

Pass 2 (implementation): Tulis semua CSS sesuai plan di atas.

Bagian yang HARUS ada:

1. AURORA (position:fixed, z-index:0, pointer-events:none)
   - 3 orb: position absolute, border-radius 50%, filter blur(80px)
   - Orb 1: 700px, top-left, aurora-1, anim aurora-float-1 18s ease-in-out infinite
   - Orb 2: 600px, bottom-right, aurora-2, anim aurora-float-2 22s
   - Orb 3: 500px, center, aurora-3, anim aurora-float-3 20s
   - @keyframes: translate + scale berbeda tiap orb, smooth looping

2. PLAYER WRAPPER
   display:flex gap:24px max-width:900px width:100% padding:24px
   align-items:flex-start z-index:1 position:relative

3. PLAYER CARD
   bg-glass + backdrop-filter:blur(24px) -webkit-backdrop-filter:blur(24px)
   border:1px solid --border-glass border-radius:--radius-xl
   padding:32px 28px width:360px flex-shrink:0 --shadow-card
   display:flex flex-direction:column

4. ALBUM ART CONTAINER + RING
   .album-art-container: relative 270x270 margin:0 auto 24px
   .progress-ring: absolute inset:-16px width:calc(100%+32px) height:calc(100%+32px)
     transform:rotate(-90deg) z-index:2 pointer-events:none
   .ring-track: fill:none stroke:rgba(255,255,255,0.06) stroke-width:3
   .ring-fill: fill:none stroke:url(#ringGrad) stroke-width:3 stroke-linecap:round
     transition:stroke-dashoffset 0.3s ease
   .album-art-wrap: position:relative width:270px height:270px border-radius:50% overflow:hidden
   #album-art: width:100% height:100% object-fit:cover border-radius:50%
     box-shadow: 0 0 0 4px rgba(139,92,246,0.25), 0 0 60px rgba(139,92,246,0.3), 0 20px 40px rgba(0,0,0,0.5)
     animation: album-spin 22s linear infinite; animation-play-state:paused
   .album-art-fallback: position:absolute inset:0 border-radius:50% display:flex
     align-items:center justify-content:center font-size:80px
   .loading-spinner: position:absolute top:50% left:50% transform:translate(-50%,-50%)
     width:40px height:40px border:3px solid rgba(255,255,255,0.1)
     border-top-color:--color-purple border-radius:50% display:none
     animation:spin-loader 0.8s linear infinite z-index:3
   @keyframes album-spin: from 0deg to 360deg
   @keyframes spin-loader: to rotate(360deg)
   .player-card.playing #album-art: animation-play-state:running
     box-shadow: 0 0 0 4px rgba(139,92,246,0.4), 0 0 80px rgba(139,92,246,0.5), 0 20px 50px rgba(0,0,0,0.6)

5. SONG INFO
   display:flex align-items:center gap:12px margin-bottom:20px
   .song-meta: flex:1 min-width:0
   .song-title: Space Grotesk 700 18px --text-primary whitespace:nowrap overflow:hidden text-overflow:ellipsis
   .song-title.marquee: animation marquee 10s linear infinite
   @keyframes marquee: 0%{transform:translateX(0)} 40%{transform:translateX(-60%)} 60%{transform:translateX(-60%)} 100%{transform:translateX(0)}
   .artist-name: 13px --text-secondary mt:4px
   .song-actions: display:flex gap:8px flex-shrink:0
   .action-btn: 36px circle --text-secondary hover:--color-pink transition
   .like-btn.liked svg: fill:--color-pink stroke:--color-pink

6. PROGRESS BAR
   .progress-wrapper: margin-bottom:20px
   .progress-bar: relative height:4px bg:rgba(255,255,255,0.1) border-radius:full cursor:pointer
     transition height 0.15s on hover → height:6px
   .progress-fill: absolute height:100% width:0 gradient purple→cyan border-radius:full transition:width 0.1s
   .progress-thumb: absolute top:50% right:calc(var(--fill,0)*1%) width:14px height:14px
     bg:white border-radius:50% transform:translate(50%,-50%) scale(0) → scale(1) saat hover
     box-shadow glow purple
   .time-display: flex space-between JetBrains Mono 11px --text-secondary mt:8px

7. CONTROLS
   display:flex align-items:center justify-content:center gap:12px mb:16px
   .ctrl-btn: display:flex center border-radius:50% --text-secondary transition
   .ctrl-sm: 36px · .ctrl-md: 44px
   .ctrl-sm svg, .ctrl-md svg: 16px/22px
   .ctrl-play: 64px bg:linear-gradient(135deg,--color-purple,--color-cyan) color:white
     box-shadow:0 0 24px rgba(139,92,246,0.5) hover:scale(1.08) + glow active:scale(0.96)
   .ctrl-sm:hover, .ctrl-md:hover: --text-primary scale(1.1)
   .ctrl-btn.active: color:--color-purple

8. VOLUME
   display:flex align-items:center gap:10px mb:20px
   .volume-slider: custom range: flex:1 height:4px border-radius:full
     -webkit-appearance:none thumb:14px white circle
   .volume-display: 11px --text-muted min-width:34px text-align:right

9. VISUALIZER
   display:flex align-items:flex-end justify-content:center gap:3px height:52px
   .visualizer-bar: width:4px height:4px bg:gradient(to top,purple,cyan) border-radius:2px
   @keyframes viz-pulse: 0%,100% height:4px 50% height:var(--max-h,30px)
   .player-card.playing .visualizer-bar: animation viz-pulse var(--dur,0.8s) infinite var(--delay,0s)

10. MOBILE PLAYLIST TOGGLE
    display:none (desktop); mt:16px width:100% padding:12px
    bg:--bg-glass-hover border:1px solid --border-glass border-radius:--radius-md
    --text-secondary flex center gap:8px font-size:14px
    .now-playing-mini: ml:auto font-size:11px --text-muted truncate max-width:120px

11. PLAYLIST CARD
    bg:--bg-glass backdrop-filter:blur(24px) border:1px solid --border-glass
    border-radius:--radius-xl padding:28px 24px flex:1 min-width:0
    display:flex flex-direction:column gap:16px --shadow-card

12. PLAYLIST HEADER
    display:flex space-between pb:16px border-bottom:1px solid --border-glass
    .playlist-title: flex center gap:10px font-weight:600 --text-primary
    .playlist-title svg: 18px --color-purple
    .track-count: 12px --text-secondary

13. SEARCH
    .search-wrapper: relative display:flex align-items:center
    .search-icon: absolute left:12px 16px --text-muted pointer-events:none
    .search-input: width:100% padding:10px 36px 10px 38px bg:rgba(255,255,255,0.04)
      border:1px solid --border-glass border-radius:--radius-md --text-primary
      font 13px outline:none focus:border-color:--color-purple
    .search-clear: absolute right:10px 28px --text-muted hover:--text-primary

14. PLAYLIST LIST & ITEMS
    .playlist-body: flex:1 overflow-y:auto
    .playlist-list: display:flex flex-direction:column gap:2px
    .playlist-item: display:flex align-items:center gap:10px padding:10px 8px
      border-radius:--radius-md cursor:pointer border-left:3px solid transparent transition
    .playlist-item:hover: bg:rgba(139,92,246,0.08) border-left-color:rgba(139,92,246,0.4)
    .playlist-item.active: bg:rgba(139,92,246,0.12) border-left-color:--color-purple
    .playlist-item.active .track-title: color:--text-accent
    
    Thumbnail kecil (40px circle) di tiap item:
    .track-thumb: 40px 40px border-radius:50% object-fit:cover flex-shrink:0
    .track-thumb-fallback: 40px circle gradient berdasarkan generateGradient()
    
    .track-info: flex:1 min-width:0
    .track-title: 13px --text-primary truncate mb:2px
    .track-artist: 11px --text-secondary
    .track-duration: 11px --text-muted monospace flex-shrink:0
    
    Equalizer icon (tiap item aktif + playing):
    .eq-icon: 3 bar kecil animasi, position:absolute atau mengganti nomor
    
    Cover upload button (muncul saat hover item):
    .cover-upload-btn: 24px 24px absolute bottom-right item, opacity:0 hover:opacity:1
      bg:rgba(0,0,0,0.6) border-radius:50% --text-primary

    Delete button (muncul saat hover):
    .delete-btn: 24px di sebelah duration, opacity:0 → 1 saat hover

15. UPLOAD ZONE
    border-bottom:1px solid --border-glass pt:16px
    .upload-zone: border:2px dashed --border-glass border-radius:--radius-lg
      padding:20px text-align:center transition
    .upload-zone.drag-over: border-color:--color-purple bg:rgba(139,92,246,0.06)
    .btn-upload: inline-flex center gap:6px padding:8px 16px
      bg:gradient(135deg,--color-purple,--color-cyan) color:white
      border-radius:full font:13px/500 hover:translateY(-1px) + glow

16. INSTALL BANNER
    display:flex align-items:center gap:12px padding:14px
    bg:rgba(139,92,246,0.08) border:1px solid rgba(139,92,246,0.2)
    border-radius:--radius-md mt:auto
    .install-text: flex:1 · strong: 13px --text-primary · small: 11px --text-secondary
    .btn-install: padding:6px 14px bg:--color-purple color:white border-radius:full 12px font
    .btn-dismiss: 24px --text-muted hover:--text-primary

17. TOAST
    .toast-container: fixed bottom:24px right:24px z-index:1000 flex flex-col gap:8px
    .toast: padding:12px 16px bg:rgba(10,10,20,0.92) backdrop-filter:blur(16px)
      border:1px solid --border-glass border-radius:--radius-md 13px
      animation: toast-in 0.3s · .hiding: toast-out 0.3s
    .toast.info: border-left:3px solid --color-purple
    .toast.success: border-left:3px solid --color-cyan
    .toast.error: border-left:3px solid --color-pink
    @keyframes toast-in: translateX(20px)→0, opacity 0→1
    @keyframes toast-out: reverse

18. SCROLLBAR
    ::-webkit-scrollbar: 4px
    ::-webkit-scrollbar-thumb: --border-glass border-radius:2px
    Firefox: scrollbar-width:thin scrollbar-color:rgba(255,255,255,0.08) transparent

19. RESPONSIVE MOBILE (max-width: 767px)
    .player-wrapper: flex-direction:column padding:16px
      padding-bottom:env(safe-area-inset-bottom,16px) align-items:stretch
    .player-card: width:100% padding:24px 20px
    .album-art-container, .album-art-wrap, #album-art: 200px
    .ctrl-play: 56px
    .playlist-toggle-mobile: display:flex
    .playlist-card: position:fixed bottom:0 left:0 right:0 height:72vh z-index:200
      border-radius:24px 24px 0 0 transform:translateY(100%)
      transition:transform 0.35s cubic-bezier(0.32,0.72,0,1)
      padding-bottom:env(safe-area-inset-bottom,20px)
    .playlist-card.open: transform:translateY(0)
    .mobile-overlay: display:block position:fixed inset:0 bg:rgba(0,0,0,0.5)
      z-index:199 display:none → .visible: display:block
    Semua touch target min 44x44px
    .progress-thumb: 20px saat mobile (lebih mudah disentuh)

20. REDUCED MOTION
    @media (prefers-reduced-motion: reduce)
      .aurora-orb: animation:none
      #album-art: animation:none !important
      .visualizer-bar: animation:none !important height:4px !important
      *: transition-duration:0.01ms !important

Tulis CSS lengkap. Tidak boleh ada class yang tidak dipakai, tidak ada duplikasi.
Perhatikan specificity — jangan buat selector yang saling membatalkan.
```

---

## ⚙️ PROMPT 4 — JavaScript: script.js Lengkap

> *Skill: `honey full` + `codebase-design` — script.js hanya sebagai orchestrator*

```
[honey full] [codebase-design]

Tulis script.js lengkap sebagai UI coordinator.
Aturan: script.js TIDAK boleh langsung sentuh IndexedDB atau HTMLAudioElement.
Semua audio → lewat Player, semua persistence → lewat SongDB.

import { SongDB } from './db.js'
import { Player } from './player.js'

── STATE ──
const state = {
  songs: [],             // Song[] (dari SongDB.getAllSongs())
  currentIndex: -1,      // -1 = belum ada lagu
  isShuffle: false,
  repeatMode: 'none',    // 'none' | 'all' | 'one'
  liked: new Set(),
  shuffleOrder: [],
  deferredInstallPrompt: null,
}

── DOM REFS ──
Ambil semua elemen via getElementById/querySelector di bagian atas.

── FUNGSI UTAMA ──

async init()
  state.songs = await SongDB.getAllSongs()
  loadPrefs() ← dari localStorage: volume, repeatMode, isShuffle
  Player.init({
    onTimeUpdate: (ct, dur) => { updateProgress(ct, dur); updateProgressRing(ct/dur) },
    onEnded: () => handleEnded(),
    onError: (e) => showToast('Gagal memuat audio', 'error'),
    onLoading: () => showSpinner(true),
    onReady: () => showSpinner(false),
  })
  buildVisualizer(24)   ← append 24 .visualizer-bar dengan CSS vars random
  renderPlaylist()
  if (state.songs.length > 0) loadSong(0)
  setupInstallPrompt()
  lucide.createIcons()

buildVisualizer(n)
  Buat n div.visualizer-bar, tiap bar set:
    style.setProperty('--max-h', random(12,52)+'px')
    style.setProperty('--dur', random(0.5,1.2).toFixed(2)+'s')
    style.setProperty('--delay', (i * 0.05).toFixed(2)+'s')
  Append ke #visualizer

loadSong(index)
  state.currentIndex = index
  const song = state.songs[index]
  Player.load(song.audioURL, { title: song.title, artist: song.artist, coverSrc: song.coverURL || '' })
  updateAlbumArt(song)
  updateSongInfo(song.title, song.artist)
  resetProgress()
  setActiveItem(index)
  document.title = `${song.title} – Nebula`
  updateNowPlayingMini(song.title)

updateAlbumArt(song)
  const img = document.getElementById('album-art')
  const fallback = document.getElementById('album-art-fallback')
  if (song.coverURL) {
    img.src = song.coverURL
    img.style.display = ''
    fallback.style.display = 'none'
  } else {
    img.style.display = 'none'
    fallback.style.display = 'flex'
    fallback.style.background = generateGradient(song.title)
    fallback.textContent = song.title.charAt(0).toUpperCase()
  }

togglePlay()
  if (state.currentIndex < 0) return
  Player.toggle()
  const playing = Player.isPlaying
  document.getElementById('player-card').classList.toggle('playing', playing)
  const icon = document.getElementById('play-icon')
  icon.setAttribute('data-lucide', playing ? 'pause' : 'play')
  document.getElementById('play-btn').setAttribute('aria-label', playing ? 'Pause' : 'Play')
  lucide.createIcons({ el: document.getElementById('play-btn') })

playNext()
  const n = state.songs.length
  if (n === 0) return
  if (state.repeatMode === 'one') { Player.seek(0); Player.play(); return }
  let next
  if (state.isShuffle) {
    const pos = state.shuffleOrder.indexOf(state.currentIndex)
    next = state.shuffleOrder[(pos + 1) % n]
  } else {
    next = state.currentIndex + 1
  }
  if (next >= n) {
    if (state.repeatMode === 'all') next = 0
    else { stopPlayer(); return }
  }
  loadSong(next); Player.play()

playPrev()
  if (Player.currentTime > 3) { Player.seek(0); return }
  const prev = (state.currentIndex - 1 + state.songs.length) % state.songs.length
  loadSong(prev); Player.play()

stopPlayer()
  Player.pause()
  document.getElementById('player-card').classList.remove('playing')
  updatePlayIcon(false)

handleEnded() → panggil playNext()

updateProgress(currentTime, duration)
  const pct = duration ? (currentTime / duration) * 100 : 0
  document.getElementById('progress-fill').style.width = pct + '%'
  document.getElementById('current-time').textContent = formatTime(currentTime)

updateProgressRing(ratio)
  const r = 140, c = 2 * Math.PI * r
  const fill = document.getElementById('ring-fill')
  fill.style.strokeDasharray = c
  fill.style.strokeDashoffset = c * (1 - (ratio || 0))

resetProgress()
  document.getElementById('progress-fill').style.width = '0%'
  document.getElementById('current-time').textContent = '0:00'
  document.getElementById('total-time').textContent = '0:00'
  updateProgressRing(0)

seekTo(e)
  const bar = document.getElementById('progress-bar')
  const rect = bar.getBoundingClientRect()
  const ratio = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width))
  Player.seek(ratio * 100)

updateVolume(e)
  const v = e.target.value / 100
  Player.setVolume(v)
  document.getElementById('volume-display').textContent = Math.round(v * 100) + '%'
  updateVolumeIcon(v)
  updateVolumeSliderStyle(v)
  localStorage.setItem('nebula-volume', v)

updateVolumeSliderStyle(v)
  const s = document.getElementById('volume-slider')
  const pct = v * 100
  s.style.background = `linear-gradient(to right, var(--color-purple) ${pct}%, rgba(255,255,255,0.1) ${pct}%)`

toggleMute()
  ← implementasi via Player (buat property isMuted di player.js jika perlu)
  updateVolumeIcon(muted ? 0 : Player.volume)
  document.getElementById('mute-btn').setAttribute('aria-label', muted ? 'Unmute' : 'Mute')

updateVolumeIcon(v)
  const icon = document.getElementById('volume-icon')
  const name = v === 0 ? 'volume-x' : v < 0.5 ? 'volume-1' : 'volume-2'
  icon.setAttribute('data-lucide', name)
  lucide.createIcons({ el: document.getElementById('mute-btn') })

toggleShuffle()
  state.isShuffle = !state.isShuffle
  if (state.isShuffle) {
    state.shuffleOrder = [...Array(state.songs.length).keys()]
    // Fisher-Yates shuffle
    for (let i = state.shuffleOrder.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [state.shuffleOrder[i], state.shuffleOrder[j]] = [state.shuffleOrder[j], state.shuffleOrder[i]]
    }
  }
  document.getElementById('shuffle-btn').classList.toggle('active', state.isShuffle)
  document.getElementById('shuffle-btn').setAttribute('aria-label', state.isShuffle ? 'Shuffle On' : 'Shuffle Off')
  localStorage.setItem('nebula-shuffle', state.isShuffle)
  showToast(state.isShuffle ? '🔀 Shuffle aktif' : '🔀 Shuffle mati')

toggleRepeat()
  const modes = ['none', 'all', 'one']
  state.repeatMode = modes[(modes.indexOf(state.repeatMode) + 1) % 3]
  const icon = document.getElementById('repeat-icon')
  icon.setAttribute('data-lucide', state.repeatMode === 'one' ? 'repeat-1' : 'repeat')
  document.getElementById('repeat-btn').classList.toggle('active', state.repeatMode !== 'none')
  document.getElementById('repeat-btn').setAttribute('aria-label', 'Repeat ' + state.repeatMode)
  lucide.createIcons({ el: document.getElementById('repeat-btn') })
  localStorage.setItem('nebula-repeat', state.repeatMode)
  const labels = { none: '▷ Repeat mati', all: '🔁 Repeat semua', one: '🔂 Repeat satu' }
  showToast(labels[state.repeatMode])

toggleLike()
  const song = state.songs[state.currentIndex]
  if (!song) return
  const liked = state.liked.has(song.id)
  liked ? state.liked.delete(song.id) : state.liked.add(song.id)
  document.getElementById('like-btn').classList.toggle('liked', !liked)
  showToast(!liked ? '❤️ Disukai' : '♡ Dihapus dari suka')

formatTime(sec)
  if (!isFinite(sec) || isNaN(sec)) return '0:00'
  const m = Math.floor(sec / 60), s = Math.floor(sec % 60)
  return `${m}:${s.toString().padStart(2, '0')}`

generateGradient(str)
  let hash = 0
  for (const c of str) hash = c.charCodeAt(0) + ((hash << 5) - hash)
  const h1 = Math.abs(hash) % 360, h2 = (h1 + 140) % 360
  return `linear-gradient(135deg, hsl(${h1},70%,35%), hsl(${h2},60%,25%))`

renderPlaylist(filter)
  const list = document.getElementById('playlist-list')
  const items = filter ?? state.songs
  list.innerHTML = ''
  document.getElementById('empty-state').hidden = state.songs.length > 0
  document.getElementById('no-results').hidden = !(filter && filter.length === 0 && state.songs.length > 0)
  document.getElementById('track-count').textContent = `${state.songs.length} lagu`
  
  items.forEach((song, i) => {
    const realIndex = state.songs.indexOf(song)
    const li = document.createElement('li')
    li.className = 'playlist-item' + (realIndex === state.currentIndex ? ' active' : '')
    li.setAttribute('role', 'option')
    li.setAttribute('aria-selected', realIndex === state.currentIndex)
    li.dataset.id = song.id
    li.dataset.index = realIndex
    
    // Thumbnail
    const thumb = song.coverURL
      ? `<img class="track-thumb" src="${song.coverURL}" alt="" loading="lazy">`
      : `<div class="track-thumb track-thumb-fallback" style="background:${generateGradient(song.title)}">${song.title.charAt(0)}</div>`
    
    li.innerHTML = `
      ${thumb}
      <div class="track-info">
        <div class="track-title">${song.title}</div>
        <div class="track-artist">${song.artist}</div>
      </div>
      <span class="track-duration">${song.duration || '–:––'}</span>
      <button class="cover-upload-btn" data-id="${song.id}" aria-label="Ganti cover" title="Ganti cover">📷</button>
      <button class="delete-btn" data-id="${song.id}" aria-label="Hapus lagu">
        <i data-lucide="trash-2"></i>
      </button>
    `
    list.appendChild(li)
  })
  lucide.createIcons()
  setActiveItem(state.currentIndex)

setActiveItem(index)
  document.querySelectorAll('.playlist-item').forEach(el => {
    const active = parseInt(el.dataset.index) === index
    el.classList.toggle('active', active)
    el.setAttribute('aria-selected', active)
  })

updateNowPlayingMini(title)
  const el = document.getElementById('now-playing-mini')
  if (el) el.textContent = title || ''

async handleFileUpload(files)
  const audioFiles = [...files].filter(f => f.type.startsWith('audio/') ||
    /\.(mp3|wav|ogg|flac|aac|m4a)$/i.test(f.name))
  if (!audioFiles.length) { showToast('Tidak ada file audio', 'error'); return }
  let added = 0
  for (const file of audioFiles) {
    const title = file.name.replace(/\.[^.]+$/, '').replace(/[-_]/g, ' ')
    const meta = { title, artist: 'Unknown', duration: '0:00' }
    try {
      const { id, audioURL } = await SongDB.addSong(file, meta)
      state.songs.push({ id, ...meta, audioURL, coverURL: null, dateAdded: Date.now() })
      added++
    } catch (e) { showToast(`Gagal upload: ${file.name}`, 'error') }
  }
  renderPlaylist()
  if (added > 0) {
    showToast(`${added} lagu ditambahkan`, 'success')
    if (state.currentIndex < 0) { loadSong(0); Player.play() }
  }

async handleCoverUpload(songId, file)
  try {
    const coverURL = await SongDB.updateCover(songId, file)
    const song = state.songs.find(s => s.id === songId)
    if (song) song.coverURL = coverURL
    if (state.songs[state.currentIndex]?.id === songId) updateAlbumArt(song)
    renderPlaylist()
    showToast('Cover diperbarui', 'success')
  } catch (e) { showToast('Gagal upload cover', 'error') }

async handleDeleteSong(songId)
  try {
    await SongDB.deleteSong(songId)
    const idx = state.songs.findIndex(s => s.id === songId)
    state.songs.splice(idx, 1)
    if (state.currentIndex >= state.songs.length) state.currentIndex = state.songs.length - 1
    renderPlaylist()
    if (state.songs.length > 0) loadSong(state.currentIndex)
    else { stopPlayer(); resetProgress() }
    showToast('Lagu dihapus')
  } catch (e) { showToast('Gagal menghapus', 'error') }

showToast(msg, type = 'info', duration = 3000)
  const el = document.createElement('div')
  el.className = `toast ${type}`
  el.textContent = msg
  document.getElementById('toast-container').appendChild(el)
  setTimeout(() => {
    el.classList.add('hiding')
    el.addEventListener('animationend', () => el.remove(), { once: true })
  }, duration)

setupInstallPrompt()
  window.addEventListener('beforeinstallprompt', e => {
    e.preventDefault()
    state.deferredInstallPrompt = e
    document.getElementById('install-banner').hidden = false
  })
  window.addEventListener('appinstalled', () => {
    document.getElementById('install-banner').hidden = true
    state.deferredInstallPrompt = null
    showToast('✅ App berhasil diinstall!', 'success')
  })

loadPrefs()
  const v = parseFloat(localStorage.getItem('nebula-volume') || '0.8')
  Player.setVolume(v)
  document.getElementById('volume-slider').value = v * 100
  updateVolumeSliderStyle(v)
  updateVolumeIcon(v)
  state.repeatMode = localStorage.getItem('nebula-repeat') || 'none'
  state.isShuffle = localStorage.getItem('nebula-shuffle') === 'true'
  if (state.repeatMode !== 'none') document.getElementById('repeat-btn').classList.add('active')
  if (state.isShuffle) document.getElementById('shuffle-btn').classList.add('active')

showSpinner(show)
  document.getElementById('loading-spinner').style.display = show ? 'block' : 'none'

── EVENT LISTENERS ──

progress-bar: click → seekTo(e)
  mousedown → isDragging=true
  window mousemove saat isDragging → update fill visual
  window mouseup → seekTo + isDragging=false
  (sama untuk touch: touchstart/touchmove/touchend)

volume-slider: input → updateVolume

buttons: play-btn click→togglePlay, prev-btn→playPrev, next-btn→playNext
  shuffle-btn→toggleShuffle, repeat-btn→toggleRepeat
  like-btn→toggleLike
  mute-btn→toggleMute
  add-songs-btn click → document.getElementById('file-input').click()
  file-input change → handleFileUpload(e.target.files)
  install-btn click → state.deferredInstallPrompt.prompt()
  install-dismiss click → document.getElementById('install-banner').hidden=true
  share-btn click → copyTrackInfo()

upload-zone:
  dragover → e.preventDefault() + add .drag-over
  dragleave → remove .drag-over
  drop → e.preventDefault() + remove .drag-over + handleFileUpload(e.dataTransfer.files)

playlist delegation (click pada #playlist-list):
  .playlist-item click → loadSong(parseInt(el.dataset.index)); Player.play()
  .cover-upload-btn click → e.stopPropagation(); triggerCoverUpload(parseInt(el.dataset.id))
  .delete-btn click → e.stopPropagation(); handleDeleteSong(parseInt(el.dataset.id))

triggerCoverUpload(songId):
  const input = document.getElementById('cover-input')
  input.value = ''
  input.onchange = e => { if (e.target.files[0]) handleCoverUpload(songId, e.target.files[0]) }
  input.click()

search:
  #search-input input → filter + renderPlaylist(filtered)
  #search-clear click → clear input + renderPlaylist()
  show/hide #search-clear based on input value

mobile:
  #playlist-toggle-mobile click → toggle .open + .visible
  #mobile-overlay click → close

keyboard:
  keydown: Space→togglePlay, ArrowLeft→playPrev, ArrowRight→playNext,
    M→toggleMute, L→toggleLike, Escape→closeMobilePlaylist
  (skip if focus on input)

── AUDIO DURATION ──
Player.init → di onTimeUpdate, jika total-time masih '0:00' dan Player.duration:
  document.getElementById('total-time').textContent = formatTime(Player.duration)

── CALL ──
document.addEventListener('DOMContentLoaded', init)

Tulis script.js lengkap. Komentar ringkas per fungsi saja.
```

---

## 💅 PROMPT 5 — Polish, Animate & Harden

> *Skill: `$impeccable polish` + `$impeccable animate` + `$impeccable harden`*

```
$impeccable polish index.html style.css script.js player.js db.js

Quality pass Nebula Player. Tiga area sekaligus:

[animate]
1. Transisi antar lagu: .album-art-wrap + .song-info fade out (opacity:0 translateY:-8px 250ms)
   → update konten → fade in. Implementasi di loadSong() sebelum update DOM.

2. Play button ripple: saat klik, buat span.ripple yang append ke button, scale 0→4 opacity 1→0,
   hapus setelah animasi. CSS: position:absolute border-radius:50% bg:rgba(255,255,255,0.3)
   pointer-events:none animation ripple-out 0.6s.

3. Playlist item: saat renderPlaylist(), tiap li dapat animation-delay (i * 30ms) dan
   @keyframes fade-slide-in (opacity:0 translateY(8px) → opacity:1 translateY(0)).

4. Double-tap album art = like: dblclick event pada #album-art-wrap → toggleLike().
   Flash animasi heart di tengah cover sesaat.

[harden]
5. player.js: wrap audio.play() dengan try/catch, callback onError jika gagal.
   Safari khusus: perlu user gesture untuk pertama kali play.

6. db.js: semua operasi IDB dalam try/catch. Jika openDB() gagal (private mode/storage full),
   throw error yang meaningful. getAllSongs() — jika audio/coverBlob null → skip createObjectURL.

7. Audio duration: saat audio 'loadedmetadata' event di player.js → panggil callback baru
   onMetadata({ duration }) → script.js update #total-time.
   Jangan di onTimeUpdate (NaN pada awal).

8. Playlist item duration: saat handleFileUpload, buat Audio() sementara untuk dapat duration:
   const a = new Audio(URL.createObjectURL(file)); a.onloadedmetadata = () => duration=formatTime(a.duration)
   Lalu update song di SongDB dan state.

[harden - cover]
9. Jika img.onerror pada .track-thumb → replace dengan .track-thumb-fallback gradient.
10. Jika #album-art onerror → sudah ada di HTML (onerror handler), pastikan fallback muncul.

[accessibility]
11. Semua ctrl-btn: aria-label dinamis (play/pause, mute/unmute, shuffle on/off, repeat mode)
12. progress-bar: aria-valuenow update di updateProgress()
13. Focus outline visible: .ctrl-btn:focus-visible { outline: 2px solid var(--color-purple); outline-offset: 3px }

[copy track info]
14. share-btn click → navigator.clipboard.writeText(`${title} — ${artist}`)
    → showToast('📋 Info lagu disalin')
    Fallback jika clipboard API tidak tersedia: prompt() dengan teks.

Tampilkan semua perubahan per file. Jangan ubah yang sudah berjalan.
```

---

## ✅ PROMPT 6 — Adapt, Audit & README

> *Skill: `$impeccable adapt` + `$impeccable audit` — production ready*

```
$impeccable adapt index.html style.css
$impeccable audit index.html

ADAPT — pastikan bekerja sempurna di:
- Android Chrome (390px portrait) — prioritas utama
- Desktop Chrome/Firefox/Safari (1280px+)
- iPad (768px landscape)
- iOS Safari: -webkit-overflow-scrolling, safe-area, backdrop-filter prefix

AUDIT — cek dan fix semua:
1. PWA installability: manifest.json valid? SW terdaftar? start_url accessible?
   Gunakan Lighthouse checklist (auditable secara manual dari kode).
2. Accessibility: semua interaktif elemen keyboard-accessible? aria labels lengkap?
3. Performance: animasi pakai transform/opacity (bukan layout properties)?
   aurora orbs: will-change:transform
   pointer-events:none pada non-interaktif elements
4. Cross-browser:
   - -webkit-backdrop-filter semua backdrop-filter
   - Custom range input: Chrome + Firefox + Safari style
   - ES Modules: sudah di-support semua target browser (fine)
5. Media Session: hanya dijalankan jika 'mediaSession' in navigator (sudah ada di player.js?)
6. IndexedDB: sudah ada fallback jika tidak tersupport? (sangat jarang, tapi good practice)
7. No console.log di production code

Buat juga README.md:

# 🎵 Nebula Player

Personal music player PWA — install di Android, simpan lagu selamanya.

## Install di Android
1. Deploy ke GitHub Pages / Netlify (wajib HTTPS)
2. Buka URL di Chrome Android
3. Tap ikon menu (⋮) → "Add to Home Screen"
   atau tunggu banner install muncul otomatis
4. Buka dari home screen → fullscreen, tanpa browser UI

## Deploy

### GitHub Pages (gratis)
\`\`\`bash
git init && git add . && git commit -m "Nebula Player v1.0"
git branch -M main
gh repo create nebula-player --public --push --source=.
# Settings → Pages → Deploy from main branch
# URL: https://[username].github.io/nebula-player
\`\`\`

### Netlify (paling cepat)
Drag folder `music-player/` ke **netlify.com/drop** → URL langsung jadi

### LAN (tanpa internet)
\`\`\`bash
npx serve . --listen 8080
# Buka http://[IP-PC]:8080 di Chrome Android (sambungkan ke WiFi yang sama)
\`\`\`

## Cara Pakai

### Tambah Lagu
Playlist → **Tambah Lagu** → pilih file MP3/WAV/OGG  
Atau drag & drop file ke zona upload  
Lagu tersimpan permanen di device, tidak hilang meski app ditutup

### Ganti Cover Album
Hover/tap item di playlist → tap ikon **📷** → pilih gambar

## Keyboard Shortcuts (Desktop)
| Tombol | Aksi |
|--------|------|
| `Space` | Play / Pause |
| `←` | Lagu sebelumnya |
| `→` | Lagu berikutnya |
| `M` | Mute / Unmute |
| `L` | Like lagu |
| `Esc` | Tutup playlist (mobile) |

## Fitur
- 🎵 Putar MP3, WAV, OGG, FLAC, AAC, M4A
- 💾 Lagu & cover tersimpan permanen (IndexedDB)
- 📱 Installable di Android sebagai PWA
- 🔒 Kontrol dari lock screen & notification bar Android
- 🌐 Berjalan offline sepenuhnya
- 🔀 Shuffle + Repeat (off/all/one)
- 🔍 Cari lagu di playlist
- 🎨 Upload cover per lagu
- 🌌 Glassmorphism + aurora animated background

## Tech Stack
- HTML5 · CSS3 · Vanilla JavaScript (ES Modules)
- IndexedDB (lagu & cover storage)
- HTMLAudioElement + Web Audio API
- Media Session API (lock screen)
- Service Worker (offline + caching)
- Zero framework · Zero build tool · Zero dependencies

## License
MIT

Tampilkan README.md final + laporan audit (temuan + fix yang dilakukan).
```

---

## 🐛 PROMPT DEBUG (pakai jika ada error)

```
[honey full]

Error di [nama file]:
[paste pesan error]

Konteks: [jelaskan apa yang sedang terjadi]

Fix minimal. Jangan ubah bagian lain yang sudah berjalan.
Jelaskan penyebab dan fix-nya dalam 2–3 kalimat.
```

---

## 📋 Panduan Testing per Prompt

| Prompt | Yang Harus Ditest |
|--------|------------------|
| P0 brainstorm | Design disetujui, spec ditulis |
| P0.5 grill-me | Semua kelemahan teridentifikasi |
| P1 shape | Wireframe + flow masuk akal |
| P2 setup | `node generate-icons.mjs` berhasil, semua file ada |
| P3 HTML+CSS | Aurora terlihat, glassmorphism OK, layout responsif |
| P4 script.js | Upload MP3, play, IndexedDB tersimpan, lock screen |
| P5 polish | Animasi halus, cover upload, durasi terbaca, error handled |
| P6 audit | Lighthouse PWA score, audit bersih, README ada |

## 🚢 Checklist Sebelum Deploy ke Android

```
[ ] node generate-icons.mjs → icons/icon-192.png dan icon-512.png ada
[ ] Buka index.html di browser → tidak ada error di console
[ ] Upload 1 lagu → play → tutup browser → buka lagi → lagu masih ada
[ ] Play lagu → kunci layar Android → kontrol muncul di lock screen
[ ] Deploy ke GitHub Pages / Netlify
[ ] Buka URL di Chrome Android → install banner muncul
[ ] Install → launch dari home screen → fullscreen tanpa browser bar
```

---

*Nebula Player PWA v2.0*
*Skills: brainstorming · grill-me · honey · frontend-design · impeccable · codebase-design*
