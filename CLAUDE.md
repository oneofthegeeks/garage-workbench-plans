# Garage Workbench Plans

This repo contains woodworking build plans for a 9-foot garage workbench wall, published as a GitHub Pages site.

**Live site:** https://oneofthegeeks.github.io/garage-workbench-plans/
**Repo:** https://github.com/oneofthegeeks/garage-workbench-plans

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | The GitHub Pages website — fully self-contained, no build step |
| `docs/superpowers/specs/2026-03-12-garage-cabinets-design.md` | Full design spec and cut list (markdown) |

---

## Making Changes

### Updating the webpage
Edit `index.html` directly. It's a standalone HTML file — all CSS and SVG are inline, no dependencies except Google Fonts. Push to deploy.

```bash
# After editing index.html:
git add index.html
git commit -m "Update plans: <what you changed>"
git push
```

GitHub Pages deploys automatically within ~1 minute of pushing.

### Updating the spec
Edit the markdown file at `docs/superpowers/specs/2026-03-12-garage-cabinets-design.md` and push the same way.

---

## Design Summary

```
[ S1 · 24" ][ S2 · 20" ][ S3 · 20" ][ S4 · 20" ][ S5 · 24" ]
  3 drawers    shelves    miter saw    shelves     3 drawers
                         (3" lower)
|←————————————————— 9'-0" (108") ——————————————————→|
```

| Spec | Value |
|------|-------|
| Total width | 9'-0" (108") |
| Surface height | 36" |
| Depth | 30" |
| Toe kick | 4" |
| Box material | 3/4" MDF |
| Top | 1.5" (two layers 3/4" plywood, glued) |
| Construction | 5 individual frameless boxes, pocket screws |

### Section details
- **S1 & S5 (24"):** 3 drawers — 8" / 9" / 12" openings, full-extension slides
- **S2 & S4 (20"):** 2 fixed shelves, 3 open bays (~9.2" each)
- **S3 (20", center):** Ryobi miter saw recessed 3" lower (box height 27.5"), open below for dust collection

### Key dimensions for cuts
- Normal box height: **30.5"** (36" − 4" toe kick − 1.5" top)
- Miter box height: **27.5"** (verify against your saw's actual table height before cutting)
- Interior width of 24" box: **22.5"** (24" − 2 × 3/4")
- Interior width of 20" box: **18.5"** (20" − 2 × 3/4")

---

## Aesthetic / Style Notes

The webpage uses a **blueprint aesthetic**:
- Dark navy background (`#0b1929`) with a subtle grid
- Green for drawer sections, blue for shelf sections, orange for the miter bay
- Fonts: Oswald (headers), IBM Plex Mono (dimensions/tables), Lora (body)
- Print styles included — the page prints cleanly in black and white

When making changes, keep the color coding consistent and ensure any new dimensions use `class="dim"` (renders in gold) and quantities use `class="qty"` (renders bright white).
