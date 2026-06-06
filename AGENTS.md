# AGENTS.md — Roadmap Pro

> This file provides context for AI agents working in this repository. Read all of it before touching any file.

---

## Project Overview

**Roadmap Pro** is a real-time collaborative visual roadmap builder. It is a pure browser app — no build step, no backend server, no npm. Two HTML files + Firebase Realtime Database is the entire stack.

| File | Purpose |
|---|---|
| `index.html` | Projects hub — list, create, archive, pin, and manage projects |
| `new_version.html` | Canvas editor — the actual roadmap builder with nodes, links, drawing, etc. |

Both files are deployed on **GitHub Pages** at `https://jasperzebra.github.io/ROADMAP_tool/`. Firebase is used only as a real-time database — GitHub Pages hosts the static files.

---

## Agent Rules

These are **mandatory**. Not suggestions.

### 1. Clarify Before You Code

Always achieve full understanding of the task before touching any file:

- Read the relevant source files first — do not assume based on filenames alone
- Identify all files affected by the change (often both `index.html` and `new_version.html`)
- If anything is ambiguous, ask follow-up questions until intent is clear
- Confirm expected behavior (inputs, outputs, edge cases) before proceeding

> If you are unsure about anything, ask. A wrong assumption wastes more time than a question.

### 2. Every New Feature MUST Hook Up broadcastState and Auto-Save

**This is the single most important rule for new_version.html.** Without it, changes are local-only and other collaborators never see them.

Every action that mutates canvas state (`nodes`, `links`, `strokes`, `canvasImages`, `_id`) must:

1. Call `saveState()` when the action is **committed** (mouseup, button press, modal save, etc.)
2. Call `broadcastStateLive()` during **in-progress** actions (mousemove drag, drawing in progress) if smooth live previews are wanted

`saveState()` automatically calls `broadcastState()` — so you only need to call `saveState()` for committed actions. Do not call `broadcastState()` directly on committed actions.

**Checklist for any new canvas feature:**
- [ ] Calls `saveState()` on completion → triggers `broadcastState()` automatically
- [ ] If it has a live drag/draw phase: calls `broadcastStateLive()` in mousemove
- [ ] Does NOT mutate state during mousemove without either of the above

**Examples of what happens if you forget:**
- Forgetting `saveState()`: change is visible locally but disappears on reload and never reaches collaborators
- Forgetting `broadcastStateLive()` on drag: node only teleports to final position on other users' screens instead of moving smoothly

### 3. Update AGENTS.md After Every Task

After completing any task, update this file with:
- Non-obvious patterns or gotchas discovered
- New features added and how they're wired up
- Any Firebase structure changes
- Architectural decisions made

This is mandatory. If you finish a task without updating this file, the task is incomplete.

### 4. Always Push to GitHub After Changes

After editing files, always commit and push so GitHub Pages serves the latest version:

```bash
git add index.html new_version.html AGENTS.md
git commit -m "Description of change"
git push
```

GitHub Pages takes ~30 seconds to update after a push.

---

## Firebase Architecture

### Config (same in both files)
```javascript
const FIREBASE_CONFIG = {
  apiKey:            "AIzaSyDTd8bdNHK6xIG5sd_j2_LZ6uy4aVQGMGw",
  authDomain:        "roadmap-53f16.firebaseapp.com",
  databaseURL:       "https://roadmap-53f16-default-rtdb.firebaseio.com",
  projectId:         "roadmap-53f16",
  ...
};
```

### Database Structure
```
/projects/
  {projectId}/
    name:          string
    color:         string (hex)
    created:       timestamp (ms)
    lastModified:  timestamp (ms)
    lastEditedBy:  string (user display name)
    createdBy:     string (user display name)
    nodeCount:     number
    pinned:        boolean
    completed:     boolean
    archived:      boolean

/rooms/
  {projectId}/
    nodes/{n<nodeId>}:   object (one node each; image + derived render fields stripped)
    strokes/{strokeId}:  object (one brush/eraser stroke each; has id + t order stamp)
    frames/{frameId}:    object (one image frame each; `src` stripped — see /images)
    groups/{groupId}:    object (one group box each)
    links:               array  (small — whole value)
    meta: { _id: number (next node ID counter), fav: array (favoriteColors) }
    images/
      n_{nodeId}:   string (base64 data URL for a node image)
      f_{frameId}:  string (base64 data URL for an image frame)
    cursors/
      {userId}/
        wx: number (world x)
        wy: number (world y)
    state:  (LEGACY single-blob — auto-migrated to the per-entity layout above on open, then deleted)

/presence/                    (per-room, ephemeral — kept OUT of /rooms on purpose)
  {projectId}/
    {userId}/
      name:  string
      color: string

/users/                       (site-wide roster — see below)
  {userId}/
    name:        string
    color:       string (hex)
    online:      boolean (true while a tab is connected; onDisconnect → false)
    lastActive:  timestamp (ms, ServerValue.TIMESTAMP; also set onDisconnect)
```

### Bandwidth — Keep Heavy Data Off Broadly-Watched Paths (Critical)

RTDB "Downloads" usage = bytes sent to listeners; a `.on('value')` re-downloads the **entire** watched node on **every** change under it. Two rules to avoid runaway egress (one 4 GB/day incident traced to breaking the first):

1. **`index.html` must never listen to `/rooms`.** That tree holds every project's full canvas `state` (incl. base64 images) and `cursors`; watching it re-downloaded everything on every cursor tick. The projects page now watches only `/presence` (names/colors), `/projects` (metadata, no images), and `/users` (roster) — all image-free.
2. **Presence is a top-level `/presence/{roomId}/{userId}` node, deliberately separate from `/rooms`** so the projects page can show "who's here" dots without pulling canvas state. Written/cleared by `new_version.html` (`onDisconnect().remove()`); deleting a project also removes `/presence/{id}`.

3. **Images live in `/rooms/{id}/images`, NOT in `/state`.** `/state` is rewritten on every edit, so keeping base64 images out of it means editing a node only re-sends small text/coordinate data — not every image.
   - `serializeStateForDB()` strips `img` (nodes) / `src` (frames) **and** derived render fields (`titleLines`, `catY`, `h`, …) before writing `/state`. (The local `serializeForStore()` for localStorage/undo keeps images — local, no bandwidth cost.)
   - `syncImages()` (called in `broadcastState`) uploads only **new/changed** images to `/rooms/{id}/images/{key}` (key `n_<nodeId>` / `f_<frameId>`), tracked by `_imgUploaded` so unchanged images aren't re-sent; removes images for deleted nodes/frames.
   - A `child_added`/`child_changed`/`child_removed` listener on `/images` fetches each image **once** into `_remoteImages` and `applyRemoteImage()` attaches it to its node/frame. The `/state` listener resolves images as `embedded || _remoteImages || previous` (anti-flash).
   - **Back-compat:** old projects with images embedded in `/state` still render; they auto-migrate to `/images` on the first save. `duplicateProject` (index.html) copies both `/state` and `/images`. Deleting a project removes all of `/rooms/{id}` incl. images.

4. **Strokes are shrunk on finish** — `simplifyStroke()` (called on draw mouseup) rounds points to 0.1 world units and drops points that barely moved. Strokes ride along in `/state` on every edit, so smaller strokes mean smaller broadcasts.
5. **Images are compressed on upload to AVIF** (see `compressImage`, capped ~1024px) so they're small both in storage and when fetched. See "AVIF Image Compression" below.

6. **Per-entity sync** (see "Per-Entity Sync" above) — editing one node writes/downloads only that node, not the whole board. This is the main on-going-cost lever.

7. **Per-entity payloads are gzipped** (`packVal`/`unpackVal`, using the `pako` CDN lib loaded in the head) — every `nodes`/`strokes`/`frames`/`groups`/`links`/`meta` value is gzipped before upload and inflated on read, cutting RTDB Downloads. See "Gzip Sync Layer" below. (Images are NOT gzipped — they're already-compressed AVIF; gzip wouldn't help.)

### AVIF Image Compression (replaces JPEG/PNG re-encode)

`compressImage()` (new_version.html) now encodes uploads to **AVIF** via the `@jsquash/avif` libavif WASM build, **lazily `import()`ed from esm.sh** (`getAvifEncoder()`, cached in `_avifEncode`; `_avifTried` prevents retry storms). Browsers can decode AVIF in `<img>`/`drawImage` but **cannot encode it via canvas**, hence the WASM dep.

- **CDN must bundle deps — use esm.sh, NOT raw `cdn.jsdelivr.net/npm/...`.** `@jsquash/avif`'s `encode.js` imports `'wasm-feature-detect'` as a **bare specifier**; raw unbundled jsDelivr files can't resolve that in-browser, so `import('…jsdelivr…/index.js')` throws and **silently falls back to JPEG every time** (this was a real bug — AVIF never ran). `https://esm.sh/@jsquash/avif@1.3.0` bundles the dep and rewrites the wasm URLs. The console logs `AVIF encoder ready` on success and `AVIF encoded …KB vs source …KB` per image, so you can confirm it's actually encoding.
- Settings (`AVIF_OPTS`): `cqLevel: 32` (≈**50% quality**; cqLevel is INVERTED — 0 = best/biggest, 63 = worst/smallest; ~80%→13, ~71%→18, 10%→57), `cqAlphaLevel: 32` (no lossless alpha), `speed: 6` (**medium** preset; 0 = slowest/smallest … 10 = fastest/biggest), `subsample: 3` (**4:4:4**, no chroma subsampling — 4:2:0 (`1`) would be smaller), no lossless, `tileColsLog2/tileRowsLog2: 0` (default tile size for every image). (Subsample enum: 0=4:0:0, 1=4:2:0, 2=4:2:2, 3=4:4:4.)
- Output is a `data:image/avif;base64,…` URL (base64 via the shared `u8ToB64` helper). The rest of the pipeline (cache/render/persist/`/images` store) is unchanged — it's just a data URL.
- **Fallback:** if the WASM fails to load OR encode throws OR the canvas pixels can't be read, it falls back to the **old JPEG/PNG** path (alpha-aware, JPEG q0.85 — note AVIF settings have NO effect on this path). Also still honors "never return something larger than the input."
- Old projects with JPEG/PNG/base64 images keep rendering; only *new* uploads become AVIF.
- **Caveat:** very old browsers without AVIF *decode* support won't show AVIF images. Acceptable for this tool.

### Gzip Sync Layer (`packVal` / `unpackVal`)

Lives just above `pushDiff`. `pako` is loaded as a global via a CDN `<script>` in the head (alongside Firebase).

- **Write:** `pushDiff` wraps every value (entities + `links` + `meta`) in `packVal(jsonStr)`. It gzips, base64s, and stores `{ __z: <base64> }` — **but only when that's actually smaller** than the raw JSON (tiny entities + base64/wrapper overhead can exceed raw; mirrors `compressImage`'s "never bigger" rule). Otherwise it stores the plain `JSON.parse(jsonStr)` object/array as before.
- **Read:** `unpackVal(val)` detects the `{ __z }` wrapper and inflates to a JSON string; legacy raw objects/arrays just get `JSON.stringify`'d. Used in `attachEntityListeners` (`onEntity`, `links`, `meta`) and `rebuildFromRoom` (the one-time full read inflates each child).
- **`_sync` always holds the UNCOMPRESSED JSON string**, so per-entity diffing and echo-suppression are unaffected by whether a value was stored raw or gzipped.
- **Back-compat:** old rooms with un-gzipped structured entities read fine (the `__z` check just falls through). The `room.state` legacy-blob migration is untouched (it was never gzipped).
- `index.html` does **not** need `pako` — it never reads `/rooms` entities (`duplicateProject` copies `/state`+`/images` opaquely).

Remaining lever if still high: base64 images (even compressed) live in `/rooms/{id}/images`; moving them to Firebase Storage (URL refs instead of base64) would cut storage further.

### Site-wide User Roster (`/users`)

`/presence/{id}` is **per-room and ephemeral** (removed on disconnect) — it only says who's in a given room right now. The persistent **`/users/{id}`** registry powers the **👥 People** roster (button left of "Show Archived" on `index.html`, `#roster-modal`).

- `armPresence()` (in **both** `index.html` and `new_version.html`, guarded by `_presenceArmed`) attaches a `.info/connected` listener that sets `online:true` and arms `onDisconnect()` to set `online:false` + `lastActive` (server timestamp). `syncMyUserRecord()` writes `name`/`color`/`lastActive`. Called on Firebase init in both pages and after `saveProfile()`.
- The roster splits everyone from `/users` into **🟢 Online** and **⚪ Offline** sections, each sorted **alphabetically by name** (`localeCompare`, case-insensitive). Each row shows a green/grey status dot and "Online now" or "Last active …" (`timeAgo`). Section headers are sticky within the scrollable list.
- Caveat: `online` is a single boolean per user, not a connection count — closing one of several open tabs flips you offline. Acceptable for now.

### Local Persistence Fallback (Critical)

Firebase writes use `.catch(() => {})` and fail **silently** (offline, security rules, quota). To prevent the roadmap from disappearing on revisit, `saveState()` also mirrors the full state to `localStorage['rm_state_' + getRoomId()]`.

- `loadLocalState()` seeds the canvas from this key.
- `startFirebase()` calls `loadLocalState()` immediately (online: as a pre-Firebase seed; offline: as the only restore path).
- The one-time room read (below) overrides the seed when Firebase has data; the brand-new-room branch re-uploads the local seed.

Firebase remains the source of truth when it has data. localStorage only fills the gap when Firebase is empty/unavailable.

### Per-Entity Sync (Critical — replaces the old single-blob `_sender` model)

The canvas is **not** one `/state` blob anymore. Each node/stroke/frame/group is its own child under `/rooms/{id}` (`nodes`, `strokes`, `frames`, `groups`), plus whole-value `links` and `meta`. This means **editing one node only writes/downloads that node**, not the whole board.

- **Write:** `pushDiff()` (called by `broadcastState`, debounced, and `broadcastStateLive`, throttled) diffs the in-memory arrays against `_sync` (the JSON we last saw/wrote per key) and sends only changed/added/removed entities in **one multi-path `update()`**. `_id`/`favoriteColors` go in `meta`.
- **Read:** one `.once('value')` on `/rooms/{id}` when the editor opens (you need the whole board once), handled by `rebuildFromRoom` — this also clears local "ghost" entities. After that, per-child listeners (`attachEntityListeners`) stream only deltas via `applyRemoteNode/Stroke/Frame/Group`.
- **Echo suppression (replaces `_sender`):** an incoming child event is ignored if (a) its value equals `_sync` (unchanged / our final write) **or** (b) we wrote that key in the last 2 s (`_selfWrites`) — the latter stops stale echoes of rapid live-drag writes from reverting an entity mid-drag. There is **no more `_sender` tag.**
- **Migration:** old projects with a single `/state` blob are detected on open, loaded (`applyLegacyState`), re-written in the per-entity layout (`pushDiff` + `syncImages`), and `/state` is deleted. `duplicateProject` (index.html) copies `/state` + `/images`; old duplicates still migrate on open.
- **Stroke order:** strokes carry `id` + `t` (creation time); they're sorted by `t` so eraser/draw layering survives the keyed (unordered) storage.
- Local persistence/undo (`serializeForStore`, history) still use whole-array snapshots with images embedded — unaffected by the DB split.

### Live Broadcast Is Skipped When Solo (Performance)

`broadcastStateLive()` early-returns when `Object.keys(remoteCursors).length === 0`. Live previews only matter when a collaborator is watching, and serializing the whole state (including base64 images) ~20×/sec on every draw/drag `mousemove` was the cause of laggy drawing. `saveState()` on `mouseup` still persists committed changes for everyone, so nothing is lost when solo.

### The `_sender` Pattern (REMOVED — historical)

The old single-blob sync used a `_sender` tag on live broadcasts so the sender skipped its own echo. That is **gone** — see **Per-Entity Sync** above. Echo suppression is now value-equality against `_sync` plus the `_selfWrites` recent-write window. Don't reintroduce `_sender`.

### User Identity
- Stored in `localStorage` as `rm_collab_user` — a JSON object `{ id, name, color }`
- `id` is a random string generated once, persistent across sessions
- `name` is the display name shown on cursors and presence dots
- `color` is the presence/cursor dot color, chosen from `COLLAB_COLORS`
- `initMyUser(name, color)` (both files) updates name and, if `color` is passed, the color. Both the **join overlay** (`new_version.html` `#name-overlay`, `#join-swatches`) and the **welcome overlay** (`index.html` `#name-overlay`, `#welcome-swatches`) let a new user pick name **and** dot color before entering.
- In `index.html`, the header **user chip** (`#user-chip`) opens a **Profile modal** (`#profile-modal`) on click — edit display name + dot color (`#profile-swatches`). `renderUserChip()` draws the chip as a colored ● dot + name. `renderSwatches(containerId, selectedObj, key)` is the shared swatch-picker renderer. Identity changes are local; they propagate to a room's presence when you next open a project (presence is written in `new_version.html`).
- All three dot-color pickers (`renderSwatches` in `index.html`, `renderJoinSwatches` in `new_version.html`) end with a **custom-color swatch**: a `.swatch.swatch-custom` label with a rainbow `conic-gradient` background wrapping a hidden `<input type="color">`. Picking a custom color sets the selected value (shown as a solid swatch + active ring) so users aren't limited to `COLLAB_COLORS`.

---

## new_version.html — Canvas Editor

### Core State Variables
```javascript
let nodes = [];          // milestone card objects
let links = [];          // { from: id, to: id }
let strokes = [];        // freehand brush strokes
let canvasImages = [];   // standalone image frames
let _id = 1;             // auto-increment node ID counter
let px, py, sc;          // canvas pan (px, py) and zoom (sc)
```

### Coordinate Systems
- **Screen space**: pixel coordinates on the `<canvas>` element
- **World space**: logical canvas coordinates, pan/zoom independent
- `ws(wx, wy)` → converts world → screen
- `sw(sx, sy)` → converts screen → world
- Always store positions in **world space** in state. Always render in **screen space**.

### Rendering Pipeline (`renderAll`)
Called on every `draw()`. Order matters:
1. Brush strokes (with eraser using `destination-out` compositing)
2. Standalone image frames
3. Bezier connection links (dashed if target is incomplete; `bezConn` colors line + arrowhead with `tc('rgba(255,255,255,0.85)','rgba(0,0,0,0.7)')` — white in dark mode, black in light, matching the cursor gizmo)
4. Milestone node cards
5. Remote cursors overlay (called after `renderAll` in `draw()`)

### Stroke Layer Isolation (Critical — Eraser)

Brush/eraser strokes are **not** drawn directly on the main canvas. `renderAll` step 1 draws all strokes onto a reusable offscreen canvas (`getStrokeLayer`, sized to `ctx.canvas`), then composites that layer down with `drawImage` under an identity transform.

This exists so the eraser's `globalCompositeOperation = 'destination-out'` only removes **brush ink within the layer** — it can no longer punch transparent holes through the group boxes (drawn at step 0, before the composite) or anything else. The layer copies the main canvas transform via `lctx.setTransform(ctx.getTransform())` so pan/zoom/DPR line up; it works for export too (identity transform, export-sized layer).

> **Do not** draw strokes straight onto the main `ctx` again — that reintroduces the bug where erasing eats the group boxes and reveals the page background.

The export function composites the rendered canvas onto a solid background to prevent transparent holes in exported images. **That background is theme-aware** — `exportImage` fills it with `tc('#2c2d31', '#dde1e7')` so exports match the chosen light/dark mode (the on-screen `.cw` canvas backgrounds). The on-screen cursor gizmo (a dashed preview circle shown for **both** the draw and erase tools, sized by the active tool's px input) is likewise themed: `tc('rgba(255,255,255,0.85)', 'rgba(0,0,0,0.7)')` (white in dark mode, black in light).

**Brush/eraser size is a numeric px input, not a slider.** `#brush-size` and `#erase-size` are `<input type="number" class="size-num">` (min 1, max 800, default 18). Read them via **`toolSize(t)`** (`t = 'erase'|'draw'`), which clamps to (0, 800] and falls back to 18 for a blank/invalid field, so the stroke width and the cursor-preview circle never go NaN. The eraser size input ranges to 800. `clearAllDrawings()` (🗑️ Clear button in both the brush and erase toolbars) confirms, then wipes `strokes` only — nodes, links, images and groups are kept — and calls `saveState()`.

### Node Layout: Single Source of Truth

`drawNodeOnCtx` is the **authoritative** owner of node geometry. Every frame it recomputes `catY`, `titleY`, `titleH`, `imgH`, `imgY`, `contentH`, `contentY`, and `n.h` from the (cached) wrapped line count. `updateNodeText` is only a first-pass fallback (called on load/create/edit) and must use the **same widths and constants** — it wraps at `n.w` (not the `NW` constant) so its line breaks match `drawNodeOnCtx`.

- Wrapping (`titleLines`/`descLines`) is cached per node via `n._textKey` and only recomputed when text changes; layout math runs every frame.
- `imgH` is set to `n.img ? 68 : 0` **in `drawNodeOnCtx` every frame**, so attaching/removing an image can never leave a stale offset that overlaps the text. Image-attach (the edit modal's `commitEdit`, and `compressEmbeddedImages` on import) also calls `updateNodeText` so the locally-serialized state is correct before reload.

### Tool System
`tool` can be: `'select'`, `'link'`, `'draw'`, `'erase'`

The link tool button is labeled **"Link/Unlink"** — clicking an existing link endpoint removes it; clicking a new pair creates one. The toolbar button text reflects both actions.

`Select` and `Link/Unlink` are top-level toolbar buttons. **`Draw` and `Erase` live inside the `🎨 Canvas ▾` dropdown** (`#dd-canvas`), each followed by its controls: `brush-ui` (color, numeric px size input, favorites 🎨) under Draw, and `erase-ui` (numeric px size input, 🗑️ Clear) under Erase. The 🗑️ Clear button lives **only on Erase** (it clears all strokes regardless of tool). A third row, `#canvas-bg-ui`, holds the **Background** color picker + Reset. All control rows are **always visible** inside the dropdown (CSS `.dropdown-menu .brush-ui/.erase-ui { display:flex }` overrides the default hidden state) — `setTool` still toggles `.active` on `#brush-ui`/`#erase-ui` but that no longer controls visibility here. The dropdown does **not** `closeDD()` on tool select (so you can adjust the size); it closes when you click the canvas. `setTool` also toggles `.active` on `#canvas-dd-btn` so the Canvas button stays highlighted while a draw/erase tool is active.

**Canvas background** is a per-user local setting (`localStorage['rm_canvas_bg']`), applied via `applyCanvasBg()` as an inline `background` on `#cw` (the canvas element itself is transparent). When set it overrides the theme default in both themes; `resetCanvasBg()` clears it and reverts to the CSS theme color. `applyTheme()` calls `applyCanvasBg()` so the picker stays synced to the theme default when no custom color is set. Export (`exportImage`) fills its base with `localStorage['rm_canvas_bg'] || tc('#2c2d31','#dde1e7')` so exports match the chosen background. It is **not** broadcast — it's a personal preference, not roadmap state.

The zoom +/− buttons and the `%` label were removed from the toolbar. Zoom is now **scroll-wheel only** (still multiplicative; `zB()` and the `#zl` label no longer exist).

### Edit Node Modal — Image
The node edit modal (`#modal-overlay`) has an **Image** section: a large full-width preview (`#edit-img-preview`, `max-height:220px`, `object-fit:contain` so the whole image shows) above an "Add / Change" button (triggers hidden `#edit-img-file`) and a "Remove" button. The change is **staged** in `_editImgStaged` (`undefined` = unchanged, a dataURL = new image, `''` = removed) and applied to `_et.img` only in `commitEdit`, so Cancel discards it. **This is now the only way to set a node image** — the old toolbar "Set Node Image" item (`triggerImgUpload`/`#fi-img`) was removed from the Add dropdown. The `.modal` is `max-height:90vh; overflow-y:auto` so it scrolls on short screens.

- `setTool(t)` updates the active tool and adjusts brush UI visibility. It also sets the canvas cursor per tool (`toolCursors` map): `none` for erase (the preview circle is the cursor), `ew-resize` (↔) for link so users know they're linking, default otherwise.
- WASD pan, scroll zoom, and right-click pan work regardless of active tool
- Right-click on a node opens the context menu (not pan)
- Right-click on empty canvas = pan

### History (Undo/Redo)
- `saveState()` is the single commit point: it serializes state (`serializeForStore()`), mirrors it to localStorage (`writeLocal`), pushes onto `historyStack`, and broadcasts. **Every committed canvas mutation must call `saveState()`** — that's the only way the change becomes undoable, persisted, and shared.
- `seedHistory()` establishes the as-loaded state as the undo baseline (`historyStack = [current]`, `historyIdx = 0`). It runs **once** after a project's first state load — guarded by `_historySeeded` in the Firebase state listener (both the data and `!data` branches) and in the offline `loadLocalState` path. Without this baseline the first action wasn't undoable and undo couldn't return to the opened state.
- `loadState(str)` deserializes and replaces all state variables (does **not** push history — used by undo/redo and import).
- `undo()`/`redo()` move `historyIdx`, `loadState` that snapshot, then `writeLocal(snapshot)` **and** `broadcastState()` so the reverted state is persisted locally and shared (previously they only broadcast, leaving localStorage stale).
- History is **local per user** — undo on one client broadcasts its previous state to everyone. Remote updates replace local state but are not pushed to history (documented limitation).

### Live Collaboration Flow
```
User drags node
  → mousemove fires every ~16ms
  → broadcastStateLive() throttled to 50ms (skipped when solo)
    → pushDiff(): multi-path update of ONLY the moved node(s)
    → Other users' child_changed listeners → applyRemoteNode → draw()
    → Own echo ignored (matches _sync, or _selfWrites < 2s)
  → mouseup fires
  → saveState() → broadcastState() debounced 150ms
    → syncImages() (changed images only) + pushDiff() (changed entities only)
    → Other users' child listeners apply just those entities → draw()
    → Project metadata updated (lastModified, nodeCount, lastEditedBy)
```

### Node Object Shape
```javascript
{
  id:       number,
  x:        number,   // world space top-left
  y:        number,
  w:        number,   // always NW (200)
  h:        number,   // dynamic — recomputed every frame in drawNodeOnCtx from wrapped line count
  name:     string,
  desc:     string,
  cat:      string,   // category label (uppercase)
  color:    string,   // hex
  complete: boolean,
  selected: boolean,  // LOCAL ONLY — stripped before broadcasting via serializeState()
  img:      string | undefined,  // base64 data URL
  tasks:    [{ text: string, done: boolean }]
}
```

> **Important:** `selected` is stripped by `serializeState()` before pushing to Firebase so selections stay local and don't bleed to other users' views.

### Context Menu
Right-clicking a node selects it and opens `#ctx-menu` at cursor position. Functions: `ctxEdit`, `ctxCopy`, `ctxPaste`, `ctxDupe`, `ctxDelete`. All call `closeCtxMenu()` first, then their respective function.

### Node Templates
Defined in `NODE_TEMPLATES` object. Keys: `bugfix`, `milestone`, `feature`, `task`, `note`. Called via `addFromTemplate(key)`. Each template pre-fills name, cat, color, desc, and tasks. `addFromTemplate` calls `saveState()`.

### Keyboard Shortcuts
| Key | Action |
|---|---|
| `W/A/S/D` | Pan camera |
| `Ctrl+Z` | Undo |
| `Ctrl+Shift+Z` | Redo |
| `Ctrl+C` | Copy selected node(s) |
| `Ctrl+V` | Paste at mouse cursor (world position) |
| `Ctrl+D` | Duplicate selected (+20px offset) |
| `Delete` / `Backspace` | Delete selected |
| `?` | Open shortcut overlay |
| `Escape` | Close any overlay |

Shortcuts are blocked when `e.target.tagName === 'INPUT'` or `TEXTAREA`, and when the edit modal is open.

### Copy/Paste System
- `_clipboard`: array of copied node objects (deep copies)
- `_mouseWorld`: `{ x, y }` updated on every mousemove — used by paste to place nodes at cursor
- Paste centers the clipboard group at the current mouse world position
- Duplicate places new nodes at `+20, +20` offset from originals

---

## index.html — Projects Hub

### State Variables
```javascript
let _allProjects = {};   // snapshot from Firebase /projects
let _allPresence = {};   // snapshot from Firebase /rooms/*/presence
let sortBy   = ...;      // 'lastModified' | 'created' | 'name' | 'nodes'
let viewMode = ...;      // 'grid' | 'list'
let showArchived = false;
let searchQuery = '';
```

Both `sortBy` and `viewMode` are persisted in `localStorage`.

### Project Card Features
- **Pin (two kinds)**:
  - **Global pin** — shared, stored in Firebase `projects/{id}/pinned` via `togglePin(id, cur)`. Everyone sees it; drives the gold card/row border (`.pinned` class) and the **🌐 Global Pins** section.
  - **Personal pin** — per-user, stored locally in `localStorage['rm_pinned']` (array of project IDs) via `togglePersonalPin(id)` / `getPersonalPins()` / `isPersonalPinned(id)`. Not shared, not synced across devices. Drives the **📌 My Pins** section and the gold ⋯-row 📌 button highlight.
  - The quick 📌 button (card + list row) toggles the **personal** pin; the ⋯ menu and right-click context menu offer **both** "📌 Pin (personal)" and "🌐 Pin to Global".
  - `render()` splits filtered `entries` into `globalP` / `mine` (personal & not global) / `rest`, sorts each by the current `sortBy`, and appends sectioned grids/lists into `#projects-container` (classed `projects-sections`). Section headers appear only when something is pinned (global or personal); otherwise one unlabeled list. Global takes precedence so a project never appears in two sections.
- **Complete**: shows ⭐ badge, gold border
- **Archive**: hidden by default, shown with "Show Archived" toggle, appears dimmed
- **Color**: accent dot + color picker, updates Firebase immediately
- **Rename**: double-click name → `contenteditable` inline edit → blur saves
- **Duplicate**: copies metadata + canvas state to new project ID
- **Delete**: permanent, removes both `/projects/{id}` and `/rooms/{id}`
- **Last edited by**: shown in card meta, written by `new_version.html` on every save

### Navigation
- `index.html` → Projects list
- `new_version.html?p={projectId}` → Canvas for that project
- If `new_version.html` is opened without `?p=`, it redirects to `index.html`
- The `projectId` doubles as the `roomId` for all Firebase collaboration paths

---

## Projects Hub — Both Views Must Stay in Sync

`index.html` has two render functions: `makeCard(id, proj)` for grid view and `makeListRow(id, proj)` for list view. **Any feature added to one must be added to both.**

They must always show the same data and the same ⋯ dropdown options. Current parity checklist:

| Feature | Grid card | List row |
|---|---|---|
| Completed ⭐ star | `card-complete-star` absolute positioned top-right | Inline ⭐ after name |
| Completed border | `.project-card.completed` gold border | `.list-row.completed` gold border |
| Archived badge | `archived-badge` span in card-top | `archived-badge` span inline after name |
| Archived dimming | `.project-card.archived` opacity 0.5 | `.list-row.archived` opacity 0.5 |
| Last edited by | Shown in `card-meta` | Shown in `list-meta` |
| Pin button | `card-pin-btn` | Separate icon button in `list-actions` |
| ⋯ dropdown options | Rename, Duplicate, Mark Complete/Incomplete, Archive/Unarchive, Delete | Same |

> **Rule:** If you add a new visual state or action to `makeCard`, immediately apply the same change to `makeListRow` before pushing.

## Known Gotchas

### 1. initDemo nodes are removed
The original file had demo nodes that loaded on every page open. These were removed. New projects start blank.

### 2. Scroll zoom is multiplicative
Zoom uses `sc * 1.1` or `sc * 0.9` (not additive). `SCMIN = 0.05`, `SCMAX = 100`. Zoom is scroll-wheel only — the toolbar +/− buttons and `%` label were removed.

### 3. Image uploads are base64 in state
Node images and standalone image frames are stored as base64 data URLs inside the state JSON. Large images will make Firebase writes slow. No CDN/storage is used — everything is embedded.

Both remaining image inputs (the node edit modal's `#edit-img-file`, and the canvas "Image Frame" `#fi-canvas-img`) go through **`readImageFile(file, cb)`**, which also runs **`compressImage()`**: downscales to a max ~1024px longest side and re-encodes to **AVIF** (max-compression settings — see "AVIF Image Compression" above), falling back to PNG (alpha) / JPEG q0.85 if the AVIF WASM is unavailable. Never returns something larger than the input. So a multi-MB photo is stored as a few KB. This is lossy/irreversible (the original isn't kept). For normal images it reads a data URL; for **`.dds`** it runs the built-in decoder (`decodeDDSToCanvas`) and converts to a PNG data URL first, which then goes through the same AVIF compression, so the rest of the app (cache/render/persist) is unchanged. Supported DDS formats: **DXT1/BC1, DXT3/BC2, DXT5/BC3, and uncompressed RGB/RGBA** (incl. DX10-header BC1–3). **BC4/BC5/BC7 are not supported** — those alert the user to re-export. The decoder is pure JS (no deps); `accept="image/*,.dds"` on the inputs.

**JSON import auto-compresses images.** When a roadmap JSON is imported (`#fi-json`), `compressEmbeddedImages()` scans every `nodes[].img` / `canvasImages[].src` and, for any that aren't already an AVIF data URL (`_isAvifURL`), re-encodes them to AVIF in place via `compressImage` and re-embeds (async). The import is `saveState()`d immediately (so it shows right away), then re-saved once images finish. This stops an imported uncompressed export from bloating storage/bandwidth.

### 4. The eraser punches transparent holes
Eraser strokes use `destination-out` compositing, but only on the **isolated stroke layer** (see "Stroke Layer Isolation") — never on the main canvas. The canvas background color is only painted during export (via a base canvas layer). On-screen, the canvas element is transparent and the CSS `background` of `.cw` shows through erased areas.

### 4b. JSON export must include every state array
`saveJSON()` builds its own object literal (it does **not** call `serializeState()`), so it's easy to forget a field. It must include `groups` and `favoriteColors` alongside `nodes/links/strokes/canvasImages/_id` — omitting `groups` silently drops all group boxes (name, color, size, position) on export/import. `loadState()` already reads every field with `|| []` fallbacks. Keep `saveJSON` in sync with `serializeState` whenever a new top-level state array is added.

### 5. Right-click behavior splits on node hit
`mousedown` with `button === 2`: if over a node, returns immediately (lets `contextmenu` event fire). If over empty canvas, starts panning. `contextmenu` is always `preventDefault()`'d.

### 6. Downstream drag
When dragging a node, `getDownstreamTree()` collects the dragged node AND all nodes reachable from it via links. All are dragged together. This is intentional — dragging a parent moves the whole subtree.

### 7. Firebase security rules
The Realtime Database is currently in **test mode** (`.read: true, .write: true`). This is open to anyone with the database URL. For production, add proper auth rules.

### 8. No multi-user undo coordination
Each user has their own local undo history. Pressing undo broadcasts your local previous state to everyone. If two users undo simultaneously, last write wins.

---

## Feature Implementation Template

When adding any new feature to `new_version.html`, use this checklist:

```
□ Read relevant section of new_version.html before writing any code
□ Identify if the feature mutates canvas state (nodes/links/strokes/canvasImages/_id)
□ If it mutates state on a discrete action (click, keypress, modal save):
    → Call saveState() at the end of the action
    → saveState() automatically calls broadcastState() — do not call it separately
□ If it has a continuous live phase (drag, draw):
    → Call broadcastStateLive() inside the mousemove handler (throttled to 50ms)
□ If it adds a new overlay or modal:
    → Add Escape key handler to close it
    → Block canvas shortcuts when the overlay is open
□ Add the feature to the keyboard shortcut overlay if it has a shortcut
□ Push to GitHub and verify on https://jasperzebra.github.io/ROADMAP_tool/
□ Update AGENTS.md
```
