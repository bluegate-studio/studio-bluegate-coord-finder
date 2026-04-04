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

### Step 4: Export & Resume

Click the 📋 clipboard icon to copy all coordinates as JSON. Save this to a file to preserve your progress.

To **continue where you left off**, click the ⬇️ import icon (left of clipboard) and load a previously exported `.json` file. The imported coordinates are appended to any existing entries, so you can merge sessions.

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
│   MOVE OVERLAY         KEY              STEP SIZE       │
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

> **Real pixels, not screen pixels.** Arrow keys always move in the original image's coordinate space. If your image is 3000px wide but displayed at 1000px, pressing `→` once moves exactly 1 real pixel—not 1 screen pixel. This guarantees **zero rounding drift** across hundreds of captures.

### Saved Coordinates

| Action | How |
|---|---|
| Delete entry | Click 🗑️ on the row (instant, no confirmation) |
| Import from JSON | Click ⬇️ in the section header, select a `.json` file |
| Copy all as JSON | Click 📋 in the section header |
| Auto-scroll | New entries scroll the list to the bottom |

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

### Resuming a Session

Coord Finder is client-side only—there's no database or auto-save. To preserve your work:

1. **Copy** coordinates to clipboard (📋) and paste into a `.json` file
2. Next time, **import** that file (⬇️) to pick up where you left off

Imported coordinates are appended, so you can also merge results from multiple sessions.

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
