# 🎯 Coord Finder

> Precision coordinate extraction from large-scale images.
> Built for event seating plans, floor maps, sprite sheets—anything where **every pixel matters**.

---

## What Is This?

Coord Finder is a browser-based tool for extracting precise (x, y) coordinates from images. Load an image, click (or overlay-match) the positions you need, and export the results as JSON.

It operates in two modes:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   MODE 1: Direct Click              MODE 2: Overlay Match    │
│   ─────────────────────              ─────────────────────   │
│                                                              │
│   Load an image.                     Load an image.          │
│   Hover over it.                     Load a cursor image.    │
│   Click to save coordinates.         Drag/nudge the overlay. │
│                                      Press Enter to save.    │
│                                                              │
│   Best for: quick marking            Best for: precision     │
│   of visible points.                 alignment of shapes     │
│                                      (tables, seats, icons). │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites

- [Bun](https://bun.sh) (v1.0+)

### Install & Run

```bash
cd frontend
bun install
bun run dev
```

Open `http://localhost:5173` in your browser. That's it.

---

## How It Works

### Step 1: Load Your Image

Drag & drop (or click to browse) the main image onto the landing zone. This is the image you want to extract coordinates from—a seating plan, a floor map, a game level, etc.

### Step 2 (Optional): Load a Cursor Overlay

If you need **overlay matching** (Mode 2), drag & drop a second image—the shape you want to align. For example, a single table cutout from your seating plan.

The overlay will appear on top of your main image, scaled proportionally. You then position it to match the corresponding element in the main image.

### Step 3: Extract Coordinates

**Mode 1 (No overlay):** Hover and click anywhere on the image. Each click saves the coordinate.

**Mode 2 (Overlay active):** Position the overlay using drag and/or arrow keys, then press `Enter` or `Space` to save the overlay's center coordinate.

### Step 4: Export, Import & Resume

Click the 📋 clipboard icon to copy all coordinates as JSON.

To **import** a previously exported `.json` file, click the ⬇️ import icon. If the coordinate list is empty, the data loads directly. If you already have saved coordinates, you'll be asked to confirm—importing **replaces** all existing entries (copy them first if you need to keep them).

To **clear everything**, click the 🗑️ trash icon. A confirmation dialog makes sure you don't lose unsaved work.

> **Auto-save:** Both your saved coordinates and settings are stored in `localStorage`. Close the tab, come back later, and everything is still there—no manual export needed for session continuity.

---

## Controls Reference

### Zoom

| Action | How |
|---|---|
| Zoom in/out | Scroll wheel over canvas |
| Zoom in/out (sidebar) | Click `−` / `+` buttons |
| Set exact zoom | Type a value in the zoom input (1–20) |
| Reset to 1× | Double-click the canvas |

### Navigation (No Overlay, Zoomed In)

| Action | How |
|---|---|
| Edge-pan | Move cursor near canvas edges—image auto-scrolls |
| Pan speed | Proportional to how close to the edge you are |

### Overlay Controls

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   MOVE OVERLAY         KEY              DEFAULT STEP    │
│   ─────────────────────────────────────────────────     │
│   Fine nudge           Arrow keys       1 real px       │
│   Medium nudge         Shift + Arrow    10 real px      │
│   Large jump           Ctrl+Shift+Arrow 100 real px     │
│   Drag (rough)         Click & drag     free movement   │
│                                                         │
│   SAVE COORDINATE      Enter or Space                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

All step sizes are configurable in the **Settings** panel (see below).

> **Real pixels, not screen pixels.** Arrow keys always move in the original image's coordinate space. If your image is 3000px wide but displayed at 1000px, pressing `→` once moves exactly 1 real pixel—not 1 screen pixel. This guarantees **zero rounding drift** across hundreds of captures.

### Saved Coordinates

| Action | How |
|---|---|
| Delete entry | Click 🗑️ on the row (instant, no confirmation) |
| Clear all | Click 🗑️ in the section header (with confirmation) |
| Import from JSON | Click ⬇️ in the section header, select a `.json` file |
| Copy all as JSON | Click 📋 in the section header |
| Auto-scroll | New entries scroll the list to the bottom |

### Settings Panel

The sidebar includes a collapsible **Settings** section where you can customise arrow key step sizes:

| Setting | Default | Controls |
|---|---|---|
| ↑↓ | 1 px | Arrow Up / Down step |
| ←→ | 1 px | Arrow Left / Right step |
| Shift+↑↓ | 10 px | Shift + Arrow Up / Down step |
| Shift+←→ | 10 px | Shift + Arrow Left / Right step |
| Ctrl+Shift+↑↓ | 100 px | Ctrl+Shift + Arrow Up / Down step |
| Ctrl+Shift+←→ | 100 px | Ctrl+Shift + Arrow Left / Right step |

Settings are disabled when no overlay cursor is loaded (arrow keys only apply in overlay mode). All values persist in `localStorage`.

> Both **Image Info** and **Settings** sections are collapsible—click the section title to expand or collapse.

---

## Precision Guarantees

This tool was built for a real-world scenario: mapping **500+ tables** on a large-scale event seating plan, where each table's position feeds directly into an interactive reservation system. Accuracy is not optional.

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   WHAT                     HOW                          │
│   ─────────────────────────────────────────────────     │
│   Cursor position          Stored as real-pixel         │
│                            integers (not floats)        │
│                                                         │
│   Arrow key movement       Always integer steps in      │
│                            real image space             │
│                                                         │
│   Saved coordinates        Exact—no rounding errors     │
│                            from scale factor math       │
│                                                         │
│   Duplicate prevention     Consecutive identical        │
│                            coordinates are ignored      │
│                                                         │
│   Drag → Arrow handoff     Drag snaps to nearest        │
│                            real-pixel on release        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## The Microscope Paradigm

When the overlay is active and zoom > 1×, the tool enters **microscope mode**:

- The **overlay stays centered** on screen (the focal point)
- The **image pans automatically** to keep the overlay's position visible
- You move the overlay; the world moves under it

This is the inverse of typical zoom behavior. Instead of scrolling a zoomed image and trying to find your target, you keep the target locked and the image flows to you.

---

## Output Format

Coordinates are exported as a JSON object with two arrays—**real** (original image dimensions) and **scaled** (as displayed on screen):

```json
{
  "real": [
    { "x": 1028, "y": 636 },
    { "x": 1031, "y": 729 },
    { "x": 1028, "y": 821 }
  ],
  "scaled": [
    { "x": 323, "y": 200 },
    { "x": 324, "y": 229 },
    { "x": 323, "y": 258 }
  ]
}
```

| Key | Meaning |
|---|---|
| `real` | Coordinates in the **original** image space (what you usually want) |
| `scaled` | Coordinates in the **rendered** image space (as displayed on screen) |

For most use cases, you want `real`.

### Session Persistence

Coord Finder automatically saves to `localStorage`:

- **Saved coordinates** — restored when you return, with the list auto-scrolled to the bottom
- **Settings** — your custom step sizes persist across sessions

For sharing or backup, use the clipboard export (📋) to get a portable `.json` file.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | [SvelteKit](https://kit.svelte.dev) (Svelte 5, runes) |
| Styling | Vanilla CSS |
| Icons | [Lucide](https://lucide.dev) |
| Runtime | [Bun](https://bun.sh) |
| Adapter | Static (`@sveltejs/adapter-static`) |

---

## License

MIT — see [LICENSE](LICENSE).

Built with 🎯 by [Bluegate Studio](https://github.com/bluegate-studio).
