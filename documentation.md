# Video Guestbook — Technical Documentation

**Project:** Hakeem & Arina Wedding Video Guestbook  
**Stack:** Vanilla HTML, CSS, JavaScript (single-file application)  
**Hosting:** Netlify (static)  
**Storage:** Browser IndexedDB (local, per-device)  
**Audio:** Local MP3 file (`background-song.mp3`)

---

## Table of Contents

1. [Overview](#overview)
2. [Application Architecture](#application-architecture)
3. [Screen Flow](#screen-flow)
4. [Screens & Components](#screens--components)
5. [Design System](#design-system)
6. [Core Features](#core-features)
7. [JavaScript Modules](#javascript-modules)
8. [Data Model](#data-model)
9. [State Variables](#state-variables)
10. [Constants](#constants)
11. [Platform Detection](#platform-detection)
12. [Video Recording Pipeline](#video-recording-pipeline)
13. [Codec Strategy](#codec-strategy)
14. [Storage — IndexedDB](#storage--indexeddb)
15. [Gallery & Pagination](#gallery--pagination)
16. [Download & File Naming](#download--file-naming)
17. [PIN Authentication](#pin-authentication)
18. [Music System](#music-system)
19. [Fullscreen](#fullscreen)
20. [Custom Modal System](#custom-modal-system)
21. [Falling Petals Animation](#falling-petals-animation)
22. [Debug Tools](#debug-tools)
23. [Known Limitations](#known-limitations)

---

## Overview

A kiosk-style web application designed for use at a wedding venue. Guests approach a tablet or laptop, tap "Leave a Message", record a short video message, review it, and save it. The couple (or an operator) later views all saved messages through a PIN-protected gallery, selects videos, and downloads them to a local device.

The entire application is a single `index.html` file with no build step, no external JS dependencies, and no backend. All video data is stored locally in the browser's IndexedDB.

**Use case constraints:**
- Intended for use on a single shared device at the venue
- One-day event; storage is temporary
- Operator downloads all videos at end of event before closing the browser
- Not designed for multi-device or cloud sync

---

## Application Architecture

```
index.html
├── <style>          CSS (all styles inline)
├── <body>
│   ├── .grain       Film grain overlay (decorative)
│   ├── #petals-container   Falling petals animation layer
│   ├── .floral-corner ×4   SVG corner decorations
│   ├── #screen-welcome
│   ├── #screen-name        (exists in DOM but not used in current flow)
│   ├── #screen-record
│   ├── #screen-review
│   ├── #screen-saved
│   ├── #screen-pin
│   ├── #screen-gallery
│   ├── #db-status          Toast indicator (bottom-right)
│   ├── #device-info        Device type indicator (bottom-left)
│   └── #custom-modal       In-page modal overlay
└── <script>         All JavaScript inline
```

All screens are `position: fixed; inset: 0` and stack on top of each other. Only the screen with `.active` class is visible (`opacity: 1`). Screen transitions use CSS `opacity: 0.8s ease`.

---

## Screen Flow

```
[Welcome Screen]
    │
    ├─── "Leave a Message" ──► startRecordDirect()
    │                               │
    │                         [Record Screen]
    │                               │
    │                         stopRecording()
    │                               │
    │                         [Review Screen]
    │                          ┌───┴───┐
    │                       Save      Retake
    │                          │         │
    │                    [Saved Screen]  [Record Screen]
    │                          │
    │                    "Done" ──► [Welcome Screen]
    │
    └─── "View Messages" ──► [PIN Screen]
                                  │
                            Correct PIN
                                  │
                           [Gallery Screen]
                                  │
                            "← Back" ──► [Welcome Screen]
```

**Note:** `#screen-name` exists in the DOM (with a name input field) but is not reachable in the current flow. `startRecordDirect()` bypasses it completely. `startRecord()` (the old handler) is still present but unused.

---

## Screens & Components

### screen-welcome
**Purpose:** Landing/kiosk idle screen  
**Visible elements:**
- Gold divider lines with diamond ornament (top)
- Tagline: "The Wedding Guestbook" — dominant heading (`Cinzel`, bold, `clamp(28px, 9vw, 70px)`)
- Couple names (commented out in current version)
- Instruction text
- "Leave a Message" button → calls `startRecordDirect()`
- "View Messages" button → navigates to `screen-pin`
- Music toggle button (`🎵` / `🔇`)
- Fullscreen toggle button (`⛶`)
- Falling petals animation (visible only on this screen)

### screen-name *(inactive)*
**Purpose:** Was the name input step — now bypassed  
**Contains:** Text input `#guest-name`, Continue and Back buttons  
**Status:** Still in DOM, not navigated to in current flow

### screen-record
**Purpose:** Camera preview and recording controls  
**Layout:** Flex row on tablet (`>768px`), flex column on mobile  
**Left side — `.video-section`:**
- `.video-frame` container (`aspect-ratio: 4/3`, dark background)
- `#preview-video` — live camera feed, CSS-mirrored (`transform: scaleX(-1)`) for natural selfie view
- `.rec-indicator` — "REC" badge with blinking red dot, visible only during recording

**Right side — `.controls-section`:**
- `#timer-ring` — SVG circular countdown ring (hidden until recording starts)
  - Outer ring: `#e8d0d0` track
  - Inner ring: `#9e6b6b` progress arc (`stroke-dasharray: 213.6`, circumference of r=34 circle)
  - Centre text: `#timer-text` showing remaining seconds
- `#timer-hint` — "seconds remaining" label
- `#btn-record` — circular record button; morphs inner shape on recording state
- `#record-label` — "Tap to Record" / "Tap to Stop"
- Back button → `stopAndBack()`

**Padding:** `20px` (overrides the default `.screen` `40px 30px`)

### screen-review
**Purpose:** Playback and save/retake decision  
**Layout:** Same flex row/column as record screen  
**Left side — `.playback-section`:**
- `.playback-wrap` container — identical dimensions to `.video-frame` (`aspect-ratio: 4/3`, same border, shadow, background)
- `#playback-video` — recorded video with `controls`, fills container with `object-fit: cover`
- Autoplays on entry, loops up to 5 times then stops

**Right side — `.review-controls-section`:**
- "Save Message" button → `saveVideo()`
- "Retake" button → `retakeVideo()`

**Padding:** `20px` (shared rule with `#screen-record`)

**Transition behaviour:** On save, `screen-review` fades out first (`classList.remove('active')`), then after 800ms `goTo('screen-saved')` is called — prevents overlap during the CSS opacity transition.

### screen-saved
**Purpose:** Confirmation screen after successful save  
**Elements:**
- 🌸 icon with `pop` keyframe animation (scale 0→1)
- "Thank You!" heading
- Gold divider
- Confirmation message
- "Done" button → returns to welcome screen

### screen-pin
**Purpose:** PIN entry gate for gallery access  
**PIN:** `112233` (hardcoded constant `CORRECT_PIN`)  
**Elements:**
- 6 dot indicators (`#dot-0` to `#dot-5`) — fill rose-dark when digit entered
- 3×4 numeric keypad grid (0–9 + delete)
- Error message with shake animation on wrong PIN

**Behaviour:**
- Digits accumulated in `pinEntry` string
- Auto-submits when 6 digits entered (200ms delay)
- Wrong PIN: dots shake (`@keyframes shake`), entry resets
- Correct PIN: navigates to gallery

### screen-gallery
**Purpose:** Private video management for the couple/operator  
**Layout:** Scrollable (`overflow-y: auto`), padding-top `80px` desktop / `200px` mobile  
**Fixed action bar (`.gallery-actions`):**
- "Select All" / "Deselect All" toggle
- "Download Selected" (disabled when 0 selected)
- "Delete Selected" (disabled when 0 selected, danger style)
- Selection count badge

**Grid (`.gallery-grid`):**
- `grid-template-columns: repeat(4, 1fr)` — 4 videos per row, horizontal layout
- `VIDEOS_PER_PAGE = 4` videos per page
- Each `.gallery-item`: `aspect-ratio: 4/3`, checkbox (top-right), video element, guest label overlay

**Guest label overlay (`.guest-label`):**
- "Guest N" (bold)
- Date and time below: `DD Month YYYY, HH:MM AM/PM`

**Pagination controls** appear below grid when total > 4. Shows up to 5 page number buttons with `…` ellipsis for large ranges.

---

## Design System

### Color Palette (CSS Custom Properties)
| Variable | Hex | Usage |
|---|---|---|
| `--cream` | `#fdf8f0` | Page background |
| `--warm-white` | `#fffef9` | Card/input backgrounds |
| `--rose` | `#c9a0a0` | Petal accents |
| `--rose-light` | `#e8d0d0` | Borders, dividers |
| `--rose-dark` | `#9e6b6b` | Primary action colour |
| `--sage` | `#a8b5a0` | Leaf accents |
| `--gold` | `#c4a882` | Borders, dividers, ornaments |
| `--gold-light` | `#e8d0d0` | Video frame borders |
| `--brown` | `#7a5c4a` | Headings, primary text |
| `--text` | `#3d2b1f` | Body text |
| `--text-light` | `#7a6a60` | Subtitles, labels |

### Typography
| Font | Style | Used For |
|---|---|---|
| Cinzel | 400, 600 | Headings, buttons, labels |
| Cormorant Garamond | 300–600, italic | Body text, instructions, modal |
| IM Fell English | Regular, italic | Subtitles, hints, gallery italic text |

All loaded from Google Fonts.

### Responsive Breakpoint
Single breakpoint at `768px`:
- Above: Side-by-side flex layout (record + review screens)
- Below: Stacked column layout

### Z-Index Layers
| Z-Index | Element |
|---|---|
| 0 | `.grain`, `#petals-container`, `.floral-corner` |
| 1 | All `.screen` elements |
| 5 | `#db-status`, `#device-info` |
| 10 | `.gallery-actions`, `.btn-back-gallery` |
| 200 | `#custom-modal` |

---

## Core Features

### 1. Anonymous Recording
Guest identity is not entered manually. On clicking "Leave a Message", the system auto-generates a guest identifier:
```
guest_<index>_<DD-MMM-YYYY>_<HH-MMam/pm>
```
Example: `guest_3_01-Jun-2026_02-30PM`

The index is `existingMessageCount + 1`, queried from IndexedDB at the moment of recording.

### 2. Mirror-Corrected Recording
The preview video is CSS-mirrored (`transform: scaleX(-1)`) so guests see a natural selfie view. However, the actual recording is un-mirrored by drawing each frame onto a canvas with `ctx.scale(-1, 1)` before capturing. This means downloaded videos appear natural (not mirrored) when played back.

### 3. Timed Recording
Recording auto-stops at `MAX_RECORD_SECONDS` (currently `15`). A circular SVG ring counts down visually. The ring's `strokeDashoffset` is calculated as:
```js
circumference * (1 - remainingSeconds / MAX_RECORD_SECONDS)
```
where `circumference = 213.6` (2π × r=34).

### 4. Review with Autoplay Loop
After stopping, the recorded blob is converted to an object URL and loaded into the playback video element. It autoplays immediately and replays up to `MAX_AUTOPLAY = 5` times using the `ended` event listener, then stops.

### 5. Retake
Clears `recordedBlob`, pauses and clears `playback-video.src`, and returns to the record screen. The camera stream (`stream`) remains alive across retakes — only the canvas stream is stopped on each recording end (using `getVideoTracks().forEach(t => t.stop())`), preserving the audio track for subsequent recordings.

### 6. Save Flow
1. Playback video is paused and src cleared
2. Blob saved to IndexedDB with metadata
3. `screen-review` fades out (removes `.active`)
4. After 800ms, `screen-saved` becomes active

### 7. PIN-Protected Gallery
6-digit PIN (`112233`) gates access to the gallery. No lockout mechanism exists — unlimited attempts allowed.

### 8. Bulk Download
Selected videos are downloaded sequentially with 500ms stagger between each download to prevent browser throttling. File naming uses `formatDate()` and `formatTime()` helpers.

### 9. Bulk Delete
Selected videos are deleted in a single IndexedDB `readwrite` transaction. Each deletion fires individually; once all complete, the gallery re-renders.

### 10. In-Page Modals
All `alert()` and `confirm()` calls replaced with custom promise-based modals that do not interrupt fullscreen mode. `showAlert()` resolves on OK. `showConfirm()` resolves `true`/`false`.

---

## JavaScript Modules

All code is in a single inline `<script>` block, logically sectioned with comments:

| Section | Functions |
|---|---|
| Custom Modal | `showAlert()`, `showConfirm()` |
| Falling Petals | `createPetals()` |
| Date Formatting | `formatDate()`, `formatTime()` |
| Device Detection | Constants: `isIOS`, `isAndroid`, `isMobile` |
| IndexedDB Setup | `initDB()`, `openDatabase()`, `showDBStatus()` |
| IndexedDB CRUD | `saveMessageToDB()`, `verifyMessageSaved()`, `getAllMessages()` |
| Recording Logic | `goTo()`, `startRecordDirect()`, `startRecord()`, `initCamera()`, `toggleRecord()`, `startRecording()`, `updateTimerDisplay()`, `stopRecording()`, `handleRecordingStop()`, `retakeVideo()`, `saveVideo()`, `stopAndBack()` |
| PIN Logic | `pinPress()`, `pinDelete()`, `updatePinDots()`, `checkPin()` |
| Gallery | `renderGallery()`, `toggleVideoSelection()`, `toggleSelectAll()`, `updateSelectionUI()`, `downloadSelected()`, `deleteSelected()` |
| Music | `initMusic()`, `toggleMusic()`, `handleMusicForScreen()` |
| Fullscreen | `toggleFullscreen()` |
| Init | `window.addEventListener('load', ...)` |
| Debug | `logDatabaseStatus()`, `window.dbDebug` object |

---

## Data Model

Each video message stored in IndexedDB has this structure:

```js
{
  id: Number,          // Auto-incremented primary key
  name: String,        // "guest_1_01-Jun-2026_02-30PM"
  timestamp: String,   // ISO 8601: "2026-06-01T14:30:00.000Z"
  dateStr: String,     // "1 June 2026" (toLocaleDateString en-MY)
  blob: Blob,          // Raw video data
  mimeType: String     // "video/mp4" or "video/webm"
}
```

**Notes:**
- `name` encodes the guest number, date, and time — used as the basis for the download filename
- `dateStr` is stored as a human-readable string and used in the gallery label
- `timestamp` is the full ISO string used for sorting (newest first) and for gallery time display
- `blob` is the raw binary video — the largest field, potentially 10–50MB per entry

---

## State Variables

| Variable | Type | Description |
|---|---|---|
| `db` | IDBDatabase | Active IndexedDB connection |
| `stream` | MediaStream | Live camera/mic stream from getUserMedia |
| `mediaRecorder` | MediaRecorder | Active recorder instance |
| `recordedChunks` | Array | Accumulated Blob chunks during recording |
| `recordedBlob` | Blob | Final assembled recording |
| `timerInterval` | Number | setInterval ID for countdown |
| `remainingSeconds` | Number | Current countdown value (counts down from MAX_RECORD_SECONDS) |
| `isRecording` | Boolean | Recording state flag |
| `guestName` | String | Auto-generated identifier for current guest session |
| `canvasStream` | MediaStream | Canvas-captured stream used for un-mirrored recording |
| `currentPage` | Number | Current gallery page number |
| `selectedVideoIds` | Set | Set of IndexedDB IDs currently selected in gallery |
| `musicMuted` | Boolean | Music mute state |
| `bgMusic` | HTMLAudioElement | Reference to background audio element |
| `musicReady` | Boolean | Whether music has been successfully initialised |
| `pinEntry` | String | Accumulates PIN digits as user types |

---

## Constants

| Constant | Value | Description |
|---|---|---|
| `DB_NAME` | `'GuestbookDB'` | IndexedDB database name |
| `DB_VERSION` | `1` | IndexedDB schema version |
| `STORE_NAME` | `'messages'` | Object store name |
| `VIDEOS_PER_PAGE` | `4` | Gallery pagination size |
| `MAX_RECORD_SECONDS` | `15` | Recording time limit |
| `CORRECT_PIN` | `'112233'` | Gallery access PIN |
| `MAX_AUTOPLAY` | `5` | Max replay count on review screen |

---

## Platform Detection

Detected at script load time using `navigator.userAgent`:

```js
const isIOS    = /iPad|iPhone|iPod/.test(userAgent) && !window.MSStream;
const isAndroid = /android/i.test(userAgent);
const isMobile  = isIOS || isAndroid;
```

**iPad** is included in `isIOS` because of the `iPad` pattern match.

Used for:
- Camera constraint adjustments
- Codec selection in MediaRecorder
- Explicit `.play()` call on mobile (required due to stricter autoplay policy)
- Platform-specific camera error messages

---

## Video Recording Pipeline

```
getUserMedia()
    │
    ▼
MediaStream (stream)
    │
    ├── video track ──► <video id="preview-video"> (CSS mirrored)
    │
    └── audio track ──┐
                       │
HTMLCanvasElement ◄── drawFrame() [requestAnimationFrame loop]
  (ctx.scale(-1,1))    │   flips video horizontally
                       │
canvas.captureStream(30fps)
    │
    ├── canvas video track (un-mirrored)
    └── audio track (added from original stream)
    │
    ▼
MediaRecorder (canvasStream)
    │
    ├── ondataavailable ──► recordedChunks[]
    └── onstop ──► handleRecordingStop()
                        │
                   new Blob(recordedChunks)
                        │
                   URL.createObjectURL()
                        │
                   <video id="playback-video">
```

**Key detail:** When recording stops (`mediaRecorder.onstop`), only `canvasStream.getVideoTracks()` are stopped — NOT audio tracks. This preserves the microphone track in `stream` so it remains usable on retake without re-initialising the camera.

---

## Codec Strategy

Codec selection is platform-dependent, tried in priority order:

### iOS / iPadOS
1. `video/mp4`
2. `video/mp4;codecs=h264,aac`

### Android
1. `video/webm;codecs=vp8,opus`
2. `video/webm`

### Desktop (Chrome/Edge/Firefox)
1. `video/mp4;codecs=avc1,mp4a.40.2` — H.264 + AAC (Chrome 130+, best compatibility)
2. `video/mp4;codecs=h264,aac`
3. `video/mp4`
4. `video/webm;codecs=vp8,opus`
5. `video/webm;codecs=vp9,opus`
6. `video/webm`

**File extension** is determined by `mimeType.includes('mp4') ? 'mp4' : 'webm'`.

**Audio note:** On older Chrome (pre-130), the browser falls back to WebM + Opus audio. Opus is not natively supported by Windows Media Player. Files play correctly in Chrome/Edge browser tabs and VLC.

---

## Storage — IndexedDB

**Database:** `GuestbookDB` v1  
**Object store:** `messages` with auto-increment `id` key and a `timestamp` index

### Initialisation flow (`initDB`)
1. Open DB without version to check if object store exists
2. If store missing → delete and recreate (handles corrupted state)
3. `onupgradeneeded` → delete old store if present, create fresh store + timestamp index
4. On success → sets global `db` reference, shows status toast

### Save (`saveMessageToDB`)
- Validates `blob instanceof Blob`
- Opens `readwrite` transaction
- Stores `{name, timestamp, dateStr, blob, mimeType}`
- Calls `verifyMessageSaved()` after save to confirm blob integrity

### Fetch (`getAllMessages`)
- Opens `readonly` transaction
- Returns all records sorted by `timestamp` descending (newest first)

### Known failure scenarios
- Safari private browsing: quota = 0, save fails silently
- Safari ITP: database deleted after 7 days of no interaction
- Storage quota exceeded: write rejected
- User clears browser data: all videos lost

---

## Gallery & Pagination

`renderGallery(page)` is called each time `screen-gallery` becomes active.

**Steps:**
1. Fetch all messages from IndexedDB
2. Slice for current page: `messages.slice((page-1)*4, page*4)`
3. Show page info header ("Showing X-Y of Z messages")
4. For each message: create `.gallery-item` with checkbox, video element, guest label
5. Create object URL from blob for each video
6. Append pagination controls if `totalPages > 1`
7. Call `updateSelectionUI()` to sync button states

**Selection state** (`selectedVideoIds` Set) persists across page navigation — selecting items on page 1, navigating to page 2, and downloading will include items from both pages.

---

## Download & File Naming

### Format
```
guest_<index>_<DD-MMM-YYYY>_<HH-MMam>.ext
```
Example: `guest_1_01-Jun-2026_02-30PM.mp4`

### Helper functions

**`formatDate(date)`**
```js
// Returns: "01-Jun-2026"
const months = ['Jan','Feb',...,'Dec'];
`${dd}-${mmm}-${yyyy}`
```
Uses local timezone via `getDate()`, `getMonth()`, `getFullYear()` (not UTC).

**`formatTime(date)`**
```js
// Returns: "02-30PM"
date.toLocaleTimeString('en-MY', { hour: '2-digit', minute: '2-digit', hour12: true })
  .replace(':', '-').replace(' ', '')
```

### Download mechanism
- Creates hidden `<a>` element with `download` attribute
- Appends to body, clicks, removes
- Staggered 500ms between each file to avoid browser blocking
- Object URLs revoked after download

---

## PIN Authentication

- PIN is hardcoded as `CORRECT_PIN = '112233'`
- Stored in plain JavaScript — visible to anyone who views source
- No lockout, no rate limiting, no attempt logging
- Suitable only for low-security event use (keeping casual visitors out)
- Wrong entry triggers CSS `shake` keyframe on the dot row

---

## Music System

- Audio source: `background-song.mp3` (local file, same directory)
- Volume: `0.3` (30%)
- Element: `<audio id="bg-music" loop preload="auto">`

**Autoplay handling:**
- Browsers block autoplay until user interaction
- `initMusic()` is called on first click event (`{ once: true }`) and on page load
- Music plays on `screen-welcome` and `screen-name`, pauses on all other screens
- Toggle button switches between `🎵` (playing) and `🔇` (muted)

---

## Fullscreen

Uses the standard Fullscreen API (`requestFullscreen` / `exitFullscreen`).

**Known limitation:** Safari on iOS/iPadOS below 16.4 does not support this API. The button will silently fail or show the custom modal error message.

The `fullscreenchange` event updates the button icon. Currently both enter and exit use the same `⛶` character (icon update logic is present but uses identical strings).

---

## Custom Modal System

Replaces native `alert()` and `confirm()` to prevent fullscreen exit (browsers are required to exit fullscreen before showing OS-level dialogs).

### `showAlert(message)` → `Promise<void>`
Shows modal with message and single "OK" button. Resolves when OK clicked.

### `showConfirm(message)` → `Promise<boolean>`
Shows modal with "Cancel" and "Confirm" buttons. Resolves `true` on Confirm, `false` on Cancel.

**Styling:** Scale-in animation (`transform: scale(0.95→1)`), `z-index: 200` (above all screens), semi-transparent dark overlay. Message supports `\n` via `white-space: pre-line`.

---

## Falling Petals Animation

**Trigger:** Created once on `window.load`, visible only on `screen-welcome`.

**Implementation:**
- 18 `<div class="petal">` elements appended to `#petals-container`
- Each petal randomised: size (7–19px), position (0–100% left), duration (14–30s), delay (negative = starts mid-fall), color (from palette), horizontal drift (±90px), rotation (360/720/1080deg), border-radius orientation (two leaf shapes)
- CSS `@keyframes petalFall` animates `translateY(0 → 110vh)` with `var(--drift)` and `var(--spin)` per petal
- Negative animation delay trick: petals appear already in-progress at page load
- Visibility toggled via `.hidden` class in `goTo()` — `opacity: 0` with `transition: 0.8s ease` matching screen transitions

**Colors used:** `#c9a0a0`, `#e8d0d0`, `#a8b5a0`, `#c4a882`, `#b8c8b0`, `#9e6b6b`

---

## Debug Tools

Exposed globally as `window.dbDebug` for use in browser DevTools console:

| Command | Action |
|---|---|
| `dbDebug.status()` | Logs full DB status report: count, sizes, types |
| `dbDebug.showAllData()` | `console.table()` of all messages |
| `dbDebug.getAllMessages()` | Returns Promise of all message objects |
| `dbDebug.clearDatabase()` | Wipes all messages, reloads page |
| `dbDebug.resetDatabase()` | Deletes and recreates the database, reloads page |
| `dbDebug.exportAllVideos()` | Downloads all videos (bypasses selection UI) |

---

## Known Limitations

| Limitation | Detail |
|---|---|
| Single-device storage | IndexedDB is per-browser, per-device. Videos not accessible on other devices. |
| No cloud backup | Videos only exist in the browser until downloaded or browser data cleared. |
| Safari 7-day expiry | Safari's ITP deletes IndexedDB after 7 days of no interaction. |
| Safari private mode | Storage quota is 0 — saves will fail. |
| Fullscreen on older iOS | `requestFullscreen()` not supported on iPadOS below 16.4. |
| Autoplay on Safari review screen | `playback.play()` called outside user gesture context — may be blocked on Safari. |
| Opus audio on Windows | WebM + Opus files not playable in Windows Media Player. Workaround: drag into Chrome/Edge or use VLC. |
| PIN in source code | `CORRECT_PIN` is visible to anyone who views page source. |
| No recording on retake (audio) | Fixed: canvas stream stops only video track on recording end, preserving mic track for retake. |
| No upload to cloud | All storage is local only. AWS S3 via Netlify Functions planned as future enhancement. |
| No bot protection | No CAPTCHA or rate limiting. Planned: Cloudflare Turnstile + Netlify Bot Protection. |
