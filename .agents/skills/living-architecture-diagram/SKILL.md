---
name: living-architecture-diagram
description: Use when a project needs a diagram of its own architecture that stays true — a generated, zoomable system map built from a data file in the repo rather than hand-drawn in a diagram tool. Covers the data/layout/render split, what to draw, the zoomable SVG viewer, and the discipline that keeps the diagram from rotting.
---

# Living Architecture Diagram

Almost every codebase past a certain size has no map. The knowledge exists — in file-header comments, in the one senior engineer's head, in a `docs/` folder nobody has opened since the migration — but there is no single artifact you can hand someone that says *this is the shape of the system*.

The usual attempts fail in predictable ways. A diagram drawn in Figma or draw.io rots fastest: editing it means leaving the repo, opening another tool, and re-exporting a PNG, so nobody does it, and within two quarters it describes a system that no longer exists. A wiki page rots the same way with more words. A Mermaid block in the README survives better because it lives in the repo, but it cannot carry a dense layered map and gives the reader no way to zoom into detail.

The pattern that holds: **the diagram is a data file in the repo, its geometry is computed, and it renders inside the application itself.** Adding a box is editing a typed array — cheap enough that keeping it honest actually happens. That property is the whole point; everything else here serves it.

## When to use

- A system with real layering — infrastructure, data, domain, runtime, UI — where new contributors (human or agent) repeatedly ask "where does X live?"
- A codebase with one genuinely unusual abstraction that a file tree cannot convey. That abstraction is the reason to build this.
- An admin or operator console that already exists and has a weak landing page.

Skip it for a small service whose structure fits in a README paragraph, and skip it if nobody will own the update discipline — an unmaintained generated diagram is no better than an unmaintained drawn one, just more expensive to have built.

## The core pattern: three modules, one job each

```
diagram-data     what the system is        pure data, NO coordinates
layout           where each box goes       pure function, no rendering
renderer         paint + zoom/pan          no knowledge of content
```

The split is not architectural purity for its own sake. It exists so that the file a contributor edits — the data — contains nothing they have to be careful about. No `x`, no `y`, no "shift the other boxes down." They add a node, the drawing reflows, they are done.

### The data model

```
Node  = { id, label, detail?, paths?, kind? }   // kind: module | store | external
Group = { id, label, nodes[], via? }
Layer = { id, label, caption, accent, chain?, groups[] }
Rail  = { id, label, detail, paths? }[]         // concerns that span every layer
```

- **`detail` is one clause.** If it needs a sentence, it belongs in the prose companion, not in a box.
- **`paths` are pointers, not an inventory.** One or two per node; a directory beats listing its files.
- **`kind` changes only the outline** — solid for code you own, dashed for stored data, dotted for anything outside the process. Never let behaviour depend on it.
- **`chain`** marks a layer whose groups are steps in a sequence rather than peers; `via` labels the arrow leaving each step.

### Do not build a general edge router

The tempting next feature is arbitrary arrows between any two nodes. Resist it. Over a layout that reflows, arbitrary edges produce spaghetti and a routing problem that will consume more effort than the rest of the diagram combined. Two relationships carry nearly all the meaning and both are free:

- **Layer sits on layer** — adjacency already says it, no arrow needed.
- **Step follows step**, within one chain layer — a straight vertical arrow that the layout places, because it knows where both boxes are.

If you genuinely need a third relationship, add it as a labelled column or a rail item, not as a line.

## What to draw

**Orient it as a drill-down.** Put the surface a person touches at the top, then each layer that produces it, descending to the hardware. Reading downward is then "how is this made?", which is the question a newcomer actually has. (Bottom-up is the other defensible choice; what matters is picking one and being consistent, including inside chain layers.)

**Expand your best abstraction in full.** Most layers can be a tiled band of labelled boxes. One layer — the one where your codebase does something a file tree cannot reveal — should be drawn as an explicit step-by-step chain with arrows. This is the part of the diagram that teaches something. Pick the mechanism you have to explain to every new contributor, and draw all of it.

**Add a cross-cutting rail.** Some concerns are not a layer: tenancy, localization, observability, the compile-time seams that make an incomplete change a build error. Draw them as a column beside the stack, spanning its full height. The bar for inclusion is "appears from infrastructure to pixels", not "is important" — three or four items, no more.

**Never put volatile counts in it.** "60 tables", "124 migrations", "12 services" are wrong within a week, and a reader who catches one stale number stops trusting every other claim on the page. Describe structure — "migrations run at boot" — not tallies. Fixed sets that change only when a person changes them (five containers, three themes) are fine.

## The viewer

**Zoom the viewBox, not with a CSS transform.** A transform scales rasterized output; the viewBox re-renders vector text at the new size. The difference is the entire value proposition: zooming in must *sharpen* the fine print, not magnify a blur.

State is two things — a scale, and the diagram-space point held at the centre of the frame. Everything else derives:

```
view = { x: center.x - frameW/(2·scale),  y: center.y - frameH/(2·scale),
         w: frameW/scale,                  h: frameH/scale }
```

Keep them in **one** state value, not two. Every gesture changes both together, and splitting them invites a nested update that misbehaves under render-batching or double-invocation.

### Interactions

- **Zoom anchored at the pointer.** Zooming to the centre when the cursor is elsewhere feels broken. Compute the diagram point under the cursor before the scale change, then re-centre so it lands under the cursor again:
  ```
  d      = center - frame/(2·s₁) + cursor/s₁      // point under cursor at old scale
  center = d + (frame/2 - cursor)/s₂              // where it must be at the new scale
  ```
- **Wheel needs a non-passive listener.** Frameworks commonly register wheel handlers as passive, where `preventDefault` is a no-op and the page scrolls out from under the diagram. Attach it natively with passive explicitly disabled.
- **Drag to pan** with pointer capture, so a fast drag that leaves the element does not strand the gesture. Two simultaneous pointers become pinch-zoom, anchored at their midpoint.
- **Buttons in discrete steps**, with a live percentage readout, plus *fit width*, *fit all*, and *100%*. Keyboard: `+` / `−` / `0`, arrows to pan, and make the canvas focusable.
- **Clamp the pan** so the drawing cannot be pushed entirely off-frame, but allow some overshoot — a hard edge feels like a bug.

### Open at fit-**width**, not fit-all

Fitting the whole drawing typically lands around 30% scale, where every label is an unreadable smudge — a terrible first impression of an artifact whose job is to explain. Fit the width instead: the reader sees the full structure, all columns, legible headings. "Fit all" is a button, not the default.

### Progressive disclosure

Show file paths only above a scale threshold (~85%). At the opening view they would be sub-pixel mush; zooming in reveals them, which gives zoom an actual purpose beyond legibility and keeps the fit view uncluttered.

### Accessibility

Give the canvas a role and a label, and render the same content **again** as a plain nested list, visually hidden. It costs a dozen lines from data you already have, makes the diagram reachable by screen readers, and — underrated — makes it greppable, so a text search for a subsystem finds the diagram too.

## The layout engine

Small, pure, deterministic. Some hard-won specifics:

- **Uniform node height within a group, ragged between groups.** A grid of mismatched boxes inside one frame reads as a bug; take the max and apply it to that group's nodes.
- **Do not stretch group frames to the row height.** It seems tidier, but a stretched frame around two nodes leaves a large void that reads as content failing to render. Let bottom edges be ragged — that reads as a layout.
- **Wrap groups into rows and centre each row.** Ragged row occupancy then looks deliberate rather than left-over.
- **Estimate text width, don't measure it.** Measuring means rendering before laying out. An average-glyph-ratio estimate (~0.53 × font size for mixed-case sans) with a little slack in the box is accurate enough and keeps the layout a pure function you can unit-test and render headlessly.
- **Cap column counts by node count** (say ≤3 nodes → one column, 4–6 → two) so one large group cannot make a band absurdly tall or wide.

Because it is pure, you can call it from a script and assert on the output — see below.

## Keeping it true

This is the part that determines whether the whole exercise was worth doing.

**Write the update triggers down**, in the agent-instruction file or contributing guide, as a specific list rather than "keep docs updated". Update the data file *and* the prose companion in the same commit when you:

- add or remove a deployed service, a volume, or a stage in the delivery pipeline;
- add or remove a top-level source directory, or a subsystem within one;
- change the storage, auth, or database backend;
- add a step to the abstraction chain the diagram draws in full.

**State the failure mode, not just the rule.** "A stale diagram is worse than none — it teaches the wrong model of the system to every later reader" persuades where "please keep this updated" does not.

**Make it self-policing.** Because the layout is a pure function and `paths` are real repo paths, a CI check can assert that every path referenced by the diagram still exists. That single check catches the most common form of rot — a subsystem renamed or moved — without anyone having to look at the picture. Fail the build on a missing path.

**Verify by looking at it.** Render the layout headlessly to an image and actually open it before shipping. A diagram nobody has looked at ships broken; this is the check that catches voids, overlaps, and text escaping its box.

**Put it where it will be seen.** The landing page of the admin or operator console beats a file in `docs/`. Since the content is compiled-in data with no queries, the page also renders when the database is down — a genuinely useful property for the first screen an operator sees during an incident.

**Pair it with prose.** The diagram shows the shape; a companion document explains the *why* — the trade-offs, the things that would surprise someone, what is deliberately absent. Each should name the other, and the data file should say in its header comment that it is the thing to edit.

## Pitfalls

- **Coordinates leaking into the data file.** The first hand-placed `x` is the end of cheap updates: from then on, adding a node means nudging its neighbours. If a box is in the wrong place, the layout function is wrong — fix it there.
- **Volatile counts.** One stale number discredits the whole drawing.
- **Building the edge router.** Arbitrary arrows over a reflowing layout is a project of its own, and the result is usually less legible than adjacency.
- **Fit-all as the opening view.** Reads as an unlabelled blur; costs you the first impression.
- **CSS-transform zoom.** Magnifies pixels instead of re-rendering vectors, so zooming in reveals nothing and the feature is decorative.
- **A diagram nobody looked at.** Render it and open it. Every time.
- **Prose duplicated into the boxes.** Boxes hold a label and one clause. Paragraphs belong in the companion document; a wordy diagram is unreadable at every zoom level.
- **Drawing the aspiration.** Draw what the system *is*, including the parts you are not proud of. A diagram of the intended architecture is a third artifact to keep in sync, and it teaches newcomers something false.
- **No named owner for the update rule.** Without the trigger list written somewhere agents and humans both read, the discipline lasts exactly as long as the enthusiasm of whoever built it.

## Checklist

- [ ] Content lives in a data file with no coordinates in it; layout is a separate pure function; the renderer knows nothing about content
- [ ] Every node's `detail` is one clause; `paths` are one or two pointers, not an inventory
- [ ] No volatile counts anywhere in the data
- [ ] One layer — the codebase's least-obvious abstraction — is drawn as a full step-by-step chain
- [ ] A cross-cutting rail carries the three or four concerns that span every layer
- [ ] No general edge router; meaning is carried by adjacency and one chain direction
- [ ] Zoom drives the viewBox, anchored at the pointer; wheel listener is non-passive
- [ ] Scale and centre are a single piece of state
- [ ] Opens at fit-width; fit-all and 100% are buttons; keyboard and pinch both work
- [ ] Fine print appears only above a zoom threshold
- [ ] The same content renders as a visually-hidden nested list
- [ ] A CI check asserts every referenced repo path still exists
- [ ] Update triggers are written down as a specific list, with the failure mode stated
- [ ] Rendered headlessly and visually inspected before shipping
- [ ] Reachable from a page people actually open, and paired with a prose companion that names it
