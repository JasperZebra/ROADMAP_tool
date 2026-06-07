# Roadmap Pro

A real-time collaborative visual roadmap builder. Create milestone nodes, connect them with arrows, draw freehand annotations, and see your teammates' cursors live — no installs, no accounts, just open the link.

**Live app:** https://jasperzebra.github.io/ROADMAP_tool/

---

## Local Testing

This app is static HTML, so local testing only needs a simple file server:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000/` for the Projects Hub, or `http://localhost:8000/new_version.html?p=local-test` for the canvas editor. Firebase is still the shared live database, so use a throwaway project ID when testing.

---

## Getting Started

1. Open the link above — you land on the **Projects Hub**
2. Enter your display name (shown to collaborators on the canvas)
3. Click **+ New Project** and give it a name
4. You're dropped straight into the canvas editor
5. To invite someone: copy the URL from your browser and send it — anyone who opens it joins the same live session automatically

---

## Projects Hub

The landing page lists all your projects with live metadata.

### Project Cards
Each card shows the project name, node count, last edited time, who last edited it, and avatar dots of anyone currently inside the project.

| Control | Action |
|---|---|
| Click card | Open project |
| Double-click name | Rename inline |
| **⋯** menu | Rename, Duplicate, Mark Complete, Archive, Delete |
| Color dot | Click to change the card accent color |
| **📌** icon | Pin project to the top |
| **⭐ Mark Complete** | Adds a gold star and gold border to the card |
| **📦 Archive** | Hides the project from the main view |

### Toolbar
| Control | Action |
|---|---|
| Search bar | Filter projects by name as you type |
| **Show Archived** | Toggle to show/hide archived projects |
| Sort dropdown | Sort by Last Modified, Date Created, Name A–Z, or Node Count |
| **⊞ / ☰** | Switch between grid and list view |

---

## Canvas Editor

### Toolbar

| Button | Action |
|---|---|
| **← Projects** | Return to the Projects Hub |
| **Edit ▾** | Undo, Redo, Delete Selected |
| **+ Add ▾** | Add a blank node, a node from a template, or an image |
| **👆 Select** | Select and move nodes |
| **🔗 Link** | Click one node then drag to another to connect them. Click the same pair again to disconnect |
| **🖍️ Draw** | Freehand brush — pick color and size in the brush panel |
| **🧽 Erase** | Erase brush strokes |
| **File ▾** | Open JSON, Save JSON, Export PNG, Export JPEG |
| **🔗 Share** | Copies the session URL to your clipboard |

---

## Nodes

Nodes are the milestone cards that make up your roadmap.

### Creating Nodes

**Blank node:** Add ▾ → Blank Node

**From a template:**

| Template | Pre-filled with |
|---|---|
| 🐛 Bug Fix | 3 tasks: Identify root cause, Implement fix, Write regression test |
| 🎯 Milestone | Title placeholder, description prompt |
| ✨ Feature | 3 tasks: Design spec, Implementation, Testing & QA |
| 📋 Task | 1 task: ADD TASK HERE |
| 📝 Note | Description field ready to fill |

### Editing a Node
Double-click any node (or right-click → Edit) to open the editor:

- **Topic Title** — the large heading on the card
- **Description** — shown at the bottom when no tasks exist
- **Category Label** — the colored pill at the top (e.g. BUG FIX, MILESTONE)
- **Color** — changes the pill and accent color
- **Mark as Complete** — adds a green ✓ COMPLETE tab and green border
- **Tasks Checklist** — add/remove tasks, check them off; shows a progress bar on the card

### Moving Nodes
Click and drag any node. Dragging a node also moves all nodes connected downstream from it (the whole subtree moves together).

### Connecting Nodes
1. Select the **🔗 Link** tool
2. Click the source node and drag to the target node
3. Release to create the arrow
4. Repeat on the same pair to remove the connection

Connections to incomplete nodes render as dashed arrows. Connections to complete nodes are solid.

### Node Images
Select a node, then use **Add ▾ → Set Node Image** to embed a photo inside the card.

---

## Drawing Tools

| Tool | How to use |
|---|---|
| **🖍️ Draw** | Hold and drag to paint freehand strokes. Adjust color and size in the brush panel |
| **🧽 Erase** | Hold and drag to erase brush strokes |

Single-click with either tool places a dot.

---

## Image Frames

Standalone photo frames that float on the canvas independently of nodes.

1. **Add ▾ → Image Frame** — pick any image file
2. The frame appears centered on screen at a sensible size
3. **Move**: drag the frame
4. **Resize**: drag the bottom-right handle
5. **Rotate**: drag the top-center handle

---

## Navigation

| Action | How |
|---|---|
| Pan | Two-finger trackpad scroll, hold **right-click** and drag, or hold **W / A / S / D** |
| Zoom | Trackpad pinch, or **Ctrl / ⌘ + scroll wheel** |

---

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `W` `A` `S` `D` | Pan camera |
| `Trackpad scroll` | Pan camera |
| `Trackpad pinch` | Zoom in / out |
| `Ctrl / ⌘ + scroll wheel` | Zoom in / out |
| `Ctrl + Z` | Undo |
| `Ctrl + Shift + Z` | Redo |
| `Ctrl + C` | Copy selected node(s) |
| `Ctrl + V` | Paste at mouse cursor |
| `Ctrl + D` | Duplicate selected (+20px offset) |
| `Delete` | Delete selected |
| `?` | Open keyboard shortcut reference |
| `Escape` | Close any open overlay |

Press **?** at any time inside the canvas to open the full shortcut reference.

---

## Right-Click Context Menu

Right-click any node to open a quick-action menu:

- ✏️ Edit
- ⎘ Copy
- 📋 Paste Here
- ⧉ Duplicate
- 🗑️ Delete

Right-clicking empty canvas space still pans the view.

---

## Real-Time Collaboration

Every change — moving nodes, drawing, editing, deleting — is pushed to all connected users automatically. There is no save button for collaboration; it is always live.

- **Cursors**: see your collaborators' mouse positions labeled with their names in real time
- **Presence bar**: colored avatar dots in the toolbar show who is currently in the session
- **Live drag**: when someone drags a node you see it move as they drag it
- **Live drawing**: brush strokes appear stroke-by-stroke as they are drawn

### How to share a session
Just send anyone the URL of the canvas page. The URL includes the project ID — anyone who opens it joins automatically.

### Offline behavior
Changes are saved to Firebase as you work. If a collaborator makes changes while you are offline, you will see their changes the next time you open the project.

---

## Saving & Exporting

| Method | What it does |
|---|---|
| **Auto-save** | Every action is saved to Firebase automatically — no button needed |
| **File ▾ → Save JSON** | Downloads a `.json` backup of the canvas to your computer |
| **File ▾ → Open JSON** | Loads a previously saved `.json` file onto the canvas |
| **File ▾ → Export PNG** | Downloads a high-resolution PNG of the full canvas (2× resolution) |
| **File ▾ → Export JPEG** | Downloads a JPEG version |

Exports are auto-cropped to the content with padding — only the area containing nodes and strokes is included.
