# Navigation card — the Explore card's map, trail, ways and sections

> The client's card for **where you are and how to get about**: docked in
> the explore layout's panel, it shows your own map of the zone you stand
> in, the trail of areas you are in, the ways out, and what is true of the
> place and of you. Every map fact on it is **your** claim (the evidence
> firewall — [location-graph.md](./location-graph.md)); every live fact is
> the where-you-are source ([subscription.md](./subscription.md)). It is
> the first big consumer of [tooltips.md](./tooltips.md).
>
> ⭐ **Who it is for: a newcomer.** Every review ruling below comes back to
> that. A newcomer has no baseline, so the card never hides an "ordinary"
> value, never uses a word a newcomer would not know, and never tells them
> what to think of a fact — it states it.
>
> ⚠ **A redesign is coming** (owner, 2026-11-21): the card gets three
> modalities for three zoom levels, and *explore does not end at
> navigation* — when you stop moving it must stay useful, so "this room"
> and "this body" become the card's subject, not its footer. Read this
> doc as the first pass that redesign starts from.

## Where it lives

| piece | file (client) | what it is |
|---|---|---|
| the card | `components/nav/NavCard.tsx` | composes everything; the docked `map` card's body |
| the trail | `components/nav/Crumbs.tsx` | areas you are in, then the room |
| the map | `components/nav/MapArea.tsx` → `ZoneMap.tsx` | camera + overlay controls → one SVG |
| the lenses | `components/nav/LensRow.tsx` | four toggles under the map |
| the ways | `components/nav/Ways.tsx` | the compass and the named-way rows |
| the sections | `components/nav/NavSection.tsx`, `sections.tsx` | header + body; the fact rows |
| fill the height | `components/nav/useFillHeight.ts` | `fitSections` (pure) + a thin measurer |
| icon action | `components/nav/IconAction.tsx` | an icon button that runs a command (look) |
| every read | `store/navSelectors.ts` | pure views over your map copy |
| what it remembers | `store/navCardActions.ts` | the `nav.card` client-state key |

⭐ **Every read is a selector over the copy** (`zoneView`, `crumbsView`,
`boundariesView`, `wayLeadsTo`, `neighborDirection`, `routeCommandFor`,
`localityKnowledge`). A new lens, a new size, the atlas — each is a new
selector or a second view kind, never a second renderer.

## The data

- **Your map** arrives as a copy (`WireClaim` records) over the
  map-copy subscription. It holds only what you have walked, seen,
  searched or been told. The card never asks the world.
- **Where you are** arrives as `NavSections` (`zone · localities ·
  horizon · building · ways · place · you`), each replaced by name on a
  change. A field the server leaves out **does not apply here** (no sky
  concept, not a water place) — two fields are always sent: `crowd` and
  `dark`.
- **The room you stand in, before your map has it** (`withPresent`): a
  fresh player has walked nowhere, and a copy's room is never recorded,
  so the room is drawn from what you perceive now (`PRESENT_KEY`), lit
  only — dark, nothing is drawn. When the server sends no cell (a warren
  bud), `inferStanding` puts you one step back from a placed neighbor a
  way leads to. Neither is ever written to your map.
- **Guests have a map** for the length of the visit, under a session key
  — see [location-graph.md](./location-graph.md) § the Cartographer.

## The trail

Two lines: the **areas** you are in, outermost first, small, behind the
area icon; then the **room** you stand in, bold, behind the room icon. On
one line with a `›` the room read as the next step of a route walked
(owner, review); it is the innermost part of an address. A room named for
its area (the lounge, in The Lounge) shows both — they are two things.
Each area's name is YOUR claim's; one you have no claim for is *…*, and a
run of them is one *…*. A named area's click is `map <address>`; its
tooltip asks `locality-facts` (answered only for a Locality your map holds).

## The map

- **One SVG renderer** (`ZoneMap`), cell units, north up, layers bottom
  to top (`grid · edges · stubs · places · glyphs`, a layer per lens, `ticks`, `here`), every
  stroke non-scaling.
- **A map app's camera** (`MapArea`): drag to pan, wheel/pinch to zoom
  about the pointer, *find me* (follows you; on = following), *show
  everything* (fits the zone you know), zoom buttons. The camera follows
  you until you drag; moving zone resets it.
- **Overlay controls are icons**, each with a tooltip opening to the LEFT
  (`data-tip-placement` on the anchor — the tooltip layer reads it) so a
  tooltip never covers the button below.
- **Grid** toggle: gridlines with numbered axes (every 1/5/10 by zoom) and
  your `x · y · level` in the corner. Coordinates show only with the grid.
- **Legend** toggle: drawn UNDER the map (never clipped by it), each mark
  in the map's own drawing — you, a room you know, a room you have not
  been in, a way, a shut door, a locked door, a way out of the area,
  up/down, your route.
- **Every room you know is drawn the same** (owner, review): how you know
  it is its tooltip's to say. A room you have seen a way to but not been
  in is a **dashed outline** at the far cell, named when the way names it
  — never a "?".
- **Room names** at a steady 11px, beside the room when its east side is
  clear, else under it when its south side is, else not drawn (the
  tooltip names it) — never across a neighbor.
- **Levels**: a control per level you know, yours marked.
- **Click a mark**: a neighbor is `go <dir>`; a distant place draws a
  route (`route …`) and nobody moves.

## Lenses

Four toggles, each on its own drawing channel so all read at once, with
plain labels and icons: **unexplored** (rooms seen but not been in),
**route** (your last route; its least certain leg marked), **shut ways**,
**borders** (where one government's ground meets another's, from YOUR
locality claims). Which are on is remembered in `nav.card`.

## The ways

A compact **compass**: eight arms and up/down, each lit when a way goes
that way, its state (open, refuses, unseen) the server's (`Exit.wayFor`,
[boundary.md](./boundary.md)). **Named ways** (a berth, a door with a
keyword) are rows: the word you type, and where it leads. A way that
refuses says why in trying's own words, and offers the mode that works
only when it would (D18, boundary.md).

## The sections

Order: **government line** (one line, not a section: the innermost
government's name, or *none — unclaimed ground*; its tooltip lists the
ones above) · horizon · structure · **here** (the place) · **you**.

- A section header is its icon, its one-line summary, a chevron that says
  open or folded, and an optional **action** outside the toggle (the
  place's *look*, an eye icon running `look`). Open, a section whose
  summary would repeat its rows is headed by name (*Here*, *You*).
- ⭐⭐ **Every row, in every place, always** (owner, review — the rule
  this card is built on): the place shows light · overhead · ceiling ·
  weather · season · time · water · others · height · **kept** · your map; you show
  pace · posture · hands · load · your light · riding · leading · pulling
  · held back · fighting. A field the server leaves out says so in words
  (*unknown*, *none*, *nobody else here*). A fixed vocabulary teaches; a
  change shows by contrast. ⚠ The rows expose real gaps — the lounge has
  no roof or sky modelled, so *overhead* reads *unknown*. That is a
  content fact, not a card defect.
- ⭐ **The `kept` row** (persistence-model D13, 2026-11-24): *kept — what
  you leave here stays* or *not kept — what you leave here is gone at the
  next restage*, from `PlaceSection.kept` (`place.isKept()`, the server's
  `Perceiver.placeSection`; it replaced the type's `staged` field). Its
  client-source tooltip says what that means and where to leave things
  instead (a kept place, a kept locker, your hands). `look` says the same
  in its own *Kept:* / *Not kept:* line.
- **Fill the height** (`fitSections`): the card opens sections into the
  panel's spare height and folds the last ones when it overflows; a
  section the player opened or folded by hand stays as they set it
  (`nav.card`).
- The docked card shows **no card header** — the trail already says where
  you are (`HEADS_ITSELF` in `Card.tsx`).

## Tooltips

Every mark, crumb, way, lens, control and section has one, built in
`components/doc/docs.ts` (`placeDoc`, `edgeDoc`, `crumbDoc`, `levelDoc`,
`lensDoc`, `wayDoc`, `zoneDoc`, …). ⭐ **Facts, never editorial**: *"Rooms
you have seen a way to but not been in. None here."* — not advice about
what to do with it. A control's tooltip names its command, and hovering
it previews the command in the status bar.

## Decisions worth keeping (build + review)

- **No ages on the card** (D19): claims are stamped in game time and the
  client holds no game "now"; sources are listed in the order learned.
- **A far side is joined by the row path a look recorded** (D17), the
  same join the server's planner makes.
- **The lounge's origin is the room a newcomer arrives in**: the bar is
  one cell north (`bar.yaml` coords), the Warren places its host one cell
  south of the bar.
- **Lucide for now** (`iconMap.ts`); the client-wide iconography build is
  next and will replace the borrowed glyphs (riding has no horse yet).

## What is not built

- The three-modality redesign and *explore after navigation* — the next
  design (owner, 2026-11-21).
- Walking to a distant place → step-beat-slate.
- Trail, notes, pins, prices, freshness, the situational lens →
  map-slate § Lenses.
- Small and large sizes, charts, illustrated maps → map-slate § The card's
  sizes; the Explore card in other modes and on the phone →
  card-sizes-slate.
- The atlas → atlas-slate.
- A game "now" on the wire (for ages), the Locality name on the wire, the
  fields the card leaves absent → navigation-server-slate.
- The map taking the panel's leftover height (proposed at review, not
  taken: the redesign decides the layout).

## History

Built in MR !640 (2026-11-20 → 11-21), commits `58e100b5c`…the sweep.
Review rounds moved it from a dashboard of words to a map app: the camera,
icons everywhere, one trail, guest maps, every row always.

### The kept row (persistence-model, 2026-11-24)

`origin/master..HEAD` on `build/persistence-model` (MR !654) added the
place section's `kept` row and its tooltip (`nav-place-kept`).
