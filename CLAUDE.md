# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Terrassenhelfer is a single-file, dependency-free web app: a calculator for planning a paver-tile
patio (Terrasse). Given the patio dimensions, tile size, joint width, and border-stone/substrate
specs, it computes how many tiles and border stones to buy, generates a cut list, and renders a
laying plan (floor plan) and a cross-section of the substrate layers — all in German.

## Running / developing

There is no build step, package manager, or test suite. The entire app is `index.html`
(HTML + CSS + vanilla JS in one file).

- Open `index.html` directly in a browser to run it, or `open index.html` from the shell.
- To iterate: edit `index.html`, reload the browser. No compilation, bundling, or linting is set up.
- Deployment is GitHub Pages serving `main` from the repo root (see README.md) — pushing to `main`
  is the deploy.

## Architecture

Everything lives in `index.html`: markup, `<style>`, and a single IIFE `<script>` at the bottom.
The JS follows a straightforward read-inputs → calc → render-outputs cycle, re-run on every input
event (`update()` at the bottom wires this up):

1. **`readValues()`** — reads all form fields into a `v` object, normalizing units to centimeters
   (inputs are in m/cm/mm as labeled). Returns `null` on invalid input.
2. **`calc(v)`** — pure function computing everything derivable from `v`: tile grid dimensions,
   border stone counts, substrate volumes/weights, slope drop, and costs. Returns a result object
   `r` (or `{error}` if border stones don't fit).
3. **`renderResults(v, r)`, `renderViz(v, r)`, `renderSection(v, r)`** — turn `r` into DOM: the
   numeric results panel, the top-down laying-plan SVG, and the cross-section SVG.

### Two laying patterns

`v.pattern` is either `"grid"` (simple rectangular grid, sized via the `axis()` helper) or
`"offset"` (1/3-brick-bond / Läuferverband, the default) — the offset case is the complex path:

- **`axis(inner, tile, joint)`** — for the simple grid: how many tiles fit an axis and whether the
  last one needs cutting.
- **`offsetRows(v, innerW, ay)`** — builds the actual row-by-row width layout for the staggered
  pattern. Cycles three offset classes (0, 1/3, 2/3 of tile width) so cut edges never land next to
  each other, and avoids slivers below `minPiece` by rebuilding a row's end pieces or merging the
  last two rows in depth when needed. This is the trickiest function in the file — read its inline
  comments before changing tile-layout behavior.
- **`packPieces(lengths, cap)`** — first-fit-decreasing bin packing, used to work out how many
  full tiles are needed to cut all the required offcuts from (reuse of scrap pieces).
- Both patterns feed into the same downstream rendering and cost logic in `calc()`.

### Suggested cut-free dimensions

`suggest()` in `calc()` proposes the nearest smaller/larger patio dimension (width and/or depth)
that lays out with zero cut tiles. These render as clickable buttons (`.mini`, `data-target`/
`data-value`) that write straight back into the `terraceW`/`terraceD` inputs and re-trigger
`update()` — see the click handler on `#results` and the `#noCutBtn` handler near the bottom of the
script.

### SVG rendering

Both `renderViz` (floor plan) and `renderSection` (cross-section) hand-build SVG strings via small
helpers (`svgRect`, `band`, `dim`) and set `innerHTML` — there's no charting library. The
cross-section vertically exaggerates layer heights (factor `k`) so thin substrate layers stay
visible; the floor plan is roughly to scale. Both derive their layout from `r.off.rows` when in
offset-pattern mode, or from `r.ax`/`r.ay` grid data otherwise — keep new geometry logic consistent
across both branches.

### Units convention

Internal calculation and rendering code works entirely in **centimeters**; only `readValues()`
(m/mm → cm on the way in) and display formatting (`fmt`, `fmtM`, cm → m on the way out) cross unit
boundaries. Keep it that way — don't let m or mm leak into `calc()`.
