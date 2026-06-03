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
    state/
      nodes:        array
      links:        array
      strokes:      array
      canvasImages: array
      _id:          number (next node ID counter)
      _sender:      string | undefined (only present on live mid-drag broadcasts)
    presence/
      {userId}/
        name:  string
        color: string
    cursors/
      {userId}/
        wx: number (world x)
        wy: number (world y)
```

### Local Persistence Fallback (Critical)

Firebase writes use `.catch(() => {})` and fail **silently** (offline, security rules, quota). To prevent the roadmap from disappearing on revisit, `saveState()` also mirrors the full state to `localStorage['rm_state_' + getRoomId()]`.

- `loadLocalState()` seeds the canvas from this key.
- `startFirebase()` calls `loadLocalState()` immediately (online: as a pre-Firebase seed that real Firebase state overrides; offline: as the only restore path).
- The state listener's `if (!data) { broadcastState(); … }` branch re-uploads the seeded local state when Firebase has no saved state — this is the recovery path.

Firebase remains the source of truth when it has data (its listener overrides the local seed). localStorage only fills the gap when Firebase is empty/unavailable.

### Live Broadcast Is Skipped When Solo (Performance)

`broadcastStateLive()` early-returns when `Object.keys(remoteCursors).length === 0`. Live previews only matter when a collaborator is watching, and serializing the whole state (including base64 images) ~20×/sec on every draw/drag `mousemove` was the cause of laggy drawing. `saveState()` on `mouseup` still persists committed changes for everyone, so nothing is lost when solo.

### The `_sender` Pattern (Critical)

This is the mechanism that prevents live drag/draw broadcasts from breaking the local user's active state:

- **`broadcastState()`** (committed saves): pushes state **without** `_sender`. All listeners including the sender apply it. This is what persists data and reloads correctly when reopening a project.
- **`broadcastStateLive()`** (mid-drag/draw): pushes state **with** `_sender: myUser.id`. The sender's own listener detects `data._sender === myUser.id` and skips applying it, preventing the incoming JSON from destroying the live `_curStroke` or `_dragGroup` references.

**Do not add `_sender` to `broadcastState()`** — it will cause the user's own previously-saved data to not reload when reopening a project.

### User Identity
- Stored in `localStorage` as `rm_collab_user` — a JSON object `{ id, name, color }`
- `id` is a random string generated once, persistent across sessions
- `name` is the display name shown on cursors and presence dots
- `color` is randomly assigned from `COLLAB_COLORS` on first use

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
3. Bezier connection links (dashed if target is incomplete)
4. Milestone node cards
5. Remote cursors overlay (called after `renderAll` in `draw()`)

### Stroke Layer Isolation (Critical — Eraser)

Brush/eraser strokes are **not** drawn directly on the main canvas. `renderAll` step 1 draws all strokes onto a reusable offscreen canvas (`getStrokeLayer`, sized to `ctx.canvas`), then composites that layer down with `drawImage` under an identity transform.

This exists so the eraser's `globalCompositeOperation = 'destination-out'` only removes **brush ink within the layer** — it can no longer punch transparent holes through the group boxes (drawn at step 0, before the composite) or anything else. The layer copies the main canvas transform via `lctx.setTransform(ctx.getTransform())` so pan/zoom/DPR line up; it works for export too (identity transform, export-sized layer).

> **Do not** draw strokes straight onto the main `ctx` again — that reintroduces the bug where erasing eats the group boxes and reveals the page background.

The export function composites the rendered canvas onto a solid background to prevent transparent holes in exported images. **That background is theme-aware** — `exportImage` fills it with `tc('#2c2d31', '#dde1e7')` so exports match the chosen light/dark mode (the on-screen `.cw` canvas backgrounds). The on-screen eraser cursor gizmo is likewise themed: `tc('rgba(255,255,255,0.85)', 'rgba(0,0,0,0.7)')` (white in dark mode, black in light).

The eraser size slider (`#erase-size`) ranges 4–800. `clearAllDrawings()` (🗑️ Clear button in both the brush and erase toolbars) confirms, then wipes `strokes` only — nodes, links, images and groups are kept — and calls `saveState()`.

### Node Layout: Single Source of Truth

`drawNodeOnCtx` is the **authoritative** owner of node geometry. Every frame it recomputes `catY`, `titleY`, `titleH`, `imgH`, `imgY`, `contentH`, `contentY`, and `n.h` from the (cached) wrapped line count. `updateNodeText` is only a first-pass fallback (called on load/create/edit) and must use the **same widths and constants** — it wraps at `n.w` (not the `NW` constant) so its line breaks match `drawNodeOnCtx`.

- Wrapping (`titleLines`/`descLines`) is cached per node via `n._textKey` and only recomputed when text changes; layout math runs every frame.
- `imgH` is set to `n.img ? 68 : 0` **in `drawNodeOnCtx` every frame**, so attaching/removing an image can never leave a stale offset that overlaps the text. Image-attach (`fi-img`) also calls `updateNodeText` so the locally-serialized state is correct before reload.

### Tool System
`tool` can be: `'select'`, `'link'`, `'draw'`, `'erase'`

The link tool button is labeled **"Link/Unlink"** — clicking an existing link endpoint removes it; clicking a new pair creates one. The toolbar button text reflects both actions.

- `setTool(t)` updates the active tool and adjusts brush UI visibility
- WASD pan, scroll zoom, and right-click pan work regardless of active tool
- Right-click on a node opens the context menu (not pan)
- Right-click on empty canvas = pan

### History (Undo/Redo)
- `saveState()` serializes all state to JSON and pushes onto `historyStack`
- `loadState(str)` deserializes and replaces all state variables
- `undo()` and `redo()` both call `broadcastState()` directly (they bypass `saveState`)
- History is **local per user** — undo on one client undoes for everyone (it broadcasts)

### Live Collaboration Flow
```
User drags node
  → mousemove fires every ~16ms
  → broadcastStateLive() throttled to 50ms
    → Firebase write with _sender tag
    → Other users' listeners receive it → draw()
    → Own listener receives it → skips (sender check)
  → mouseup fires
  → saveState() → broadcastState() debounced 150ms
    → Firebase write without _sender
    → ALL listeners apply it → draw()
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
- **Pin**: stored in Firebase, always sorts to top
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
Zoom uses `sc * 1.1` or `sc * 0.9` (not additive). `SCMIN = 0.05`, `SCMAX = 100`. The zoom label shows `Math.round(sc * 100) + '%'`.

### 3. Image uploads are base64 in state
Node images and standalone image frames are stored as base64 data URLs inside the state JSON. Large images will make Firebase writes slow. No CDN/storage is used — everything is embedded.

### 4. The eraser punches transparent holes
Eraser strokes use `destination-out` compositing. The canvas background color is only painted during export (via a base canvas layer). On-screen, the canvas element is transparent and the CSS `background` of `.cw` shows through erased areas.

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
