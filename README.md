# Stack Mapper

A single-file, no-build web app for mapping a company's software ecosystem: every tool you run, which layer of the stack it belongs to, and how the pieces connect to each other. View it as a tiered **Stack** diagram, an interactive **Graph**, or a rotatable **3D** layer-cake.

Everything — markup, styles, and logic — lives in one HTML file (`index.html`). There is no build step, no server, and no dependency beyond a Google Fonts stylesheet. Open the file in a browser and it works.

## Features

- **Three views of the same data**
  - **Stack view** — software grouped into horizontal tiers by layer (Presentation, Applications, Services, Data, Infrastructure, External/SaaS), rendered as cards.
  - **Graph view** — a force-directed node/edge diagram (custom physics, no library) you can pan, zoom, and drag.
  - **3D view** — the six layers rendered as stacked, semi-transparent floors (Presentation on top, External/SaaS at the base, like a building), with software as pins on their floor and relationships as lines crossing between them. Drag to rotate, scroll to zoom. It's a hand-rolled orthographic projection in plain SVG, not WebGL/Three.js — see [Architecture notes](#architecture-notes).
- **Relationships** between tools — typed (`depends on`, `integrates with`, `sends data to`, `authenticates via`, `hosted on`, `replaces`) and directional. Hovering a card or selecting a node highlights its neighborhood and dims everything else.
- **Inventory rail** — every piece of software, grouped by layer, with a live count and a relationship-count badge.
- **Add / edit / delete** software and relationships through a slide-in side panel.
- **Layer legend** that doubles as a filter — click a layer to hide/show it in both views.
- **Import / Export** — download the current map as JSON, or load one back in (with basic validation and sanitization on import).
- **Light / dark theme** — follows the OS preference by default; manual toggle persists in `localStorage`.
- **Local persistence** — the map autosaves to `localStorage` after every change. No account, no backend, no network calls (aside from the Google Fonts stylesheet).
- **Seed data** — on first run (empty storage), the app loads a sample engineering-firm stack (AutoCAD, Power BI, SQL Server, Azure, etc.) so the UI isn't empty.

## Getting started

No install needed:

```bash
# just open it
start index.html          # Windows
open index.html           # macOS
xdg-open index.html       # Linux
```

Or serve it locally if you prefer (identical behavior, useful for testing on other devices on your network):

```bash
python -m http.server 8000
# then visit http://localhost:8000
```

Because it's a static file, it also deploys as-is to **GitHub Pages**, Netlify, Vercel, or any static host with zero configuration.

## Using it

1. **Add software** — click "Add software" (top bar or rail footer), give it a name and a layer, save.
2. **Add relationships** — after the first save, the panel switches to edit mode; use the relationship builder at the bottom to link it to other software (e.g. `Power BI` → `depends on` → `SQL Server`).
3. **Switch views** — the segmented control in the top bar toggles between Stack, Graph, and 3D.
4. **Filter** — click a layer in the legend (Graph/3D view) or just look at a tier (Stack view groups by layer already) to focus on part of the stack.
5. **Explore relationships** — hover a card (Stack) or click a node (Graph/3D) to see its direct connections highlighted; everything else dims.
6. **Graph controls** — drag the background to pan, scroll to zoom, drag a node to reposition it, click a node to edit it. Use the fit-to-view and re-layout buttons (top-right of canvas) to reset.
7. **3D controls** — drag anywhere on the canvas to rotate (azimuth + elevation), scroll to zoom, click a node (or Tab to it and press Enter) to edit it. The reset button (top-right) returns to the default angle.
8. **Save your work externally** — use Export to download `stack-map.json` as a backup or to share/version the data; Import to load it back (e.g. on another machine, since data is stored per-browser).

## Data model

```
software:      { id, name, layer, vendor, notes }
relationships: { id, from (software id), type, to (software id) }
```

Layers are a fixed set of six, each with an id, display name, description, and color:

| id | Name | Purpose |
|---|---|---|
| `presentation` | Presentation & BI | Dashboards, portals, reporting |
| `application` | Applications | Desktop & productivity tools people run |
| `services` | Services & APIs | Integration, middleware, automation |
| `data` | Data & Storage | Databases, file/document management |
| `infra` | Infrastructure | Compute, identity, network, platform |
| `external` | External / SaaS | Third-party & vendor cloud services |

Relationship types are a fixed set: `depends on`, `integrates with`, `sends data to`, `authenticates via`, `hosted on`, `replaces`.

The whole state (`{software, relationships}`) is persisted as JSON under the `stackmapper.v1` key in `localStorage`, and that same shape is what Export/Import read and write.

## Architecture notes

Everything is in `index.html`, organized into clearly commented sections in the `<script>` block:

- **CONFIG** — `LAYERS` and `REL_TYPES`, the two fixed vocabularies the rest of the app is built around.
- **STATE** — the in-memory `state` object plus UI state (`selectedId`, `editingId`, `view`, `hiddenLayers`).
- **Persistence** — `save()`/`load()`, thin wrappers around `localStorage` with try/catch (so the app still works in private-browsing modes that block storage).
- **Render: Inventory / Legend / Stack view** — plain DOM building via a small `el()` helper (no virtual DOM, no framework); re-renders are full re-renders of the relevant container, not diffed.
- **Graph view** — a hand-rolled force simulation (`tick()`/`runSim()`): pairwise repulsion, spring edges, and a weak centering force, integrated with `requestAnimationFrame`. Rendering (`drawGraph`) writes SVG elements directly; interaction (`setupGraphEvents`) handles pointer-based drag/pan/zoom on the raw SVG viewBox.
- **3D view** — a "cheated" 3D: `project3(x,y,z)` does a real (orthographic, not perspective) 3D→2D projection — rotate around the vertical axis by `camAz`, tilt by `camEl`, drop the depth component for the screen position and keep it separately for sorting. Each layer is one horizontal plane at `wy = layerIndex * LAYER_SPACING`; software is placed in a simple grid on its layer's plane (`buildScene3`), recomputed fresh on every render — there's no position cache, so unlike the force graph there's no "stale until you switch views" failure mode to begin with. `renderThree()` collects every floor/edge/node into one array, sorts back-to-front by depth (painter's algorithm) and draws in that order for occlusion; node radius/opacity/font-size are scaled down by distance as a cheap parallax cue. Dragging the canvas maps pointer delta to `camAz`/`camEl`; wheel maps to `camZoom`. No WebGL, no matrix/camera library — it's about 150 lines of trigonometry.
- **Edit panel** — a single slide-in panel reused for both "add" and "edit", switching mode via `editingId`.
- **Import/Export** — `exportJSON()`/`importJSON()`, the latter sanitizing untrusted input (unknown layers fall back to `application`, unknown relationship types fall back to `integrates with`, records missing an id/name are dropped).
- **Theme** — CSS custom properties define light/dark tokens; `data-theme` on `<html>` overrides the OS preference, persisted separately from the map data under `stackmapper.theme`.

Colors are defined as CSS variables (`--L0`…`--L5`) so the same palette drives both view themes; since SVG presentation attributes can't reference `var()`, `resolveColor()` resolves them to hex/rgb at draw time for the graph view.

## Known limitations

- Single map per browser — there's no way to manage multiple named maps side by side (see [Ideas](#ideas-for-improvement)).
- Data lives only in that browser's `localStorage`; clearing site data loses it (mitigate by exporting regularly).
- Undo only covers the last delete — it's a toast-based single-step undo, not a full history.
- The graph's force layout is recomputed from scratch each time you switch to Graph view or add a node; positions aren't saved between sessions.

