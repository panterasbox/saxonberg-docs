# Glass — the trade, the window, and the colour that is a grade

The `trade-glass` pack (`/trade/glass`) turns three other trades'
leftovers — the quarry's sand and lime, the collier's ash — into glass:
see-through containment, chemical inertness, and the first window that
colours the room it is set in. It rides four kernel seams shipped in the
same build (see § Kernel seams) and ships a Discipline, a pack `lib/`, the
thing classes, a `Reading`, rows, recipes, dials and two archetypes — so a
second glasshouse is rows.

## The chain

1. **Win sand** at a pit (`dig`, the quarrying trade's verb). The pit's
   deposit pins the **iron grade** per face — clean in the heart, rusty at
   the fringe — and the quarry's mint stamps that assay onto the won load
   (an `AlloyedMixin` good; see mining.md § the assay stamp). `analyze iron
   <sand>` reads it: a band word to an untrained eye, the figure to a
   proficient glassmaker.
2. **Fire the batch** (`fire <kiln>`, the platform verb). The recipe
   `glass-batch` takes sand ×2 + ash + lime (matched on the material tags
   `silica` / `potash` / `alkali`), needs 1400 K (above the liquidus — the
   bellows), and yields a `Melt` at `massYield 0.72`. The firing **carries
   the charge** (kernel D6): the melt's mass is the consumed mass × yield,
   and the sand's iron is merged onto it — so the glass is the colour of
   its sand, derivably. `glass-batch-amber` adds charcoal for the brown
   (beer) glass; `remelt-cullet` returns broken glass at 1200 K and full
   mass.
3. **Work it hot** on a blowpipe through the **hot-work window** — the one
   new mechanism (see § The hot-work window): `gob` a gather, `shape` it
   (bottle or cylinder), `reheat` to buy time, `crack` it off. Dawdle and
   the gather cools past working and is lost to cullet.
4. **Work it cold** at the bench: `scribe` a line, `snap` a scored sheet,
   `groze` an edge, `flatten` a scored cylinder into a pane at the furnace.
5. **Glaze** a pane into a window (`glaze <pane> in <window>`): the pane's
   derived colour becomes the window's, and the window colours the light
   it lets into the room.

## Colour is a grade, never a choice (the firewall)

A glass good's colour is **derived, never stored**. `TintedMixin`
(`/trade/glass/lib/Tinted`, on glass goods only — never kernel `Bottle`)
reads the piece's `AlloyedMixin` iron (and carbon, for amber) and answers
`lightTransmittance(): Colour` by Beer–Lambert per channel: iron absorbs
red and blue but barely green (→ green), carbon absorbs blue hard (→
amber). More iron is greener and darker, monotonically. A bottle is green
*because of* its sand, and no act at the furnace takes the colour out —
which is the immersion firewall (no decorative colour a player could catch
as a lie). The dials `glass.colour.*` tune the absorption and the band
edges.

## The hot-work window

A `Gather` (`Good` + `ThermalMixin` + `AlloyedMixin`) cools by the kernel's
own Newton drift the moment it leaves the pot — no new timing machinery.
`isWorkable()` is `getTemperature() ≥ workingFloorK()` (0.75 × the glass's
melting point). `HotWorkWatch` (a `SustainedEngagement` on `attention`,
hosted by the blowpipe) ticks every `glass.hotwork.tickGameS`, narrates a
glow-band crossing, and when the gather goes cold converts it to cullet
(`Gather.loseToCullet` — the verb on the object: full mass back, its iron
kept, one step greener). `shape`/`crack` are `hands` steps whose
`onComplete` re-validates `isWorkable()` at the commit point — the
framework's lazy-revalidation doctrine applied to a precondition that
decays continuously. `cancel glasswork` stops the watch, not the cooling.

⭐ `Gather.effectiveR()` is overridden by `glass.hotwork.gatherRFactor`
(×10) so the window lasts ~40 real seconds: the lumped thermal `R` is
tuned for an open mug, a compact blob on a pipe loses heat slower per
kilogram, and that factor is the one playtest knob dressed as physics — a
dial, and the one place the build's "derivable throughout" bends.

## Cullet goes one way toward green

Broken glass re-melts (`remelt-cullet`, or the kernel `salvage` branch for
meltable non-metals — crafting.md) at full mass but carrying its iron, and
the pot adds a little iron each campaign (`glass.pot.ironPickup`). So glass
only ever gets greener: mixed cullet can never make clear ware, and even a
single clean piece re-melted alone comes back a step greener. Recycling
that is physically truthful and teaches the opposite lesson from the
aluminium can.

## Kernel seams (shipped in this build's Stage A)

- **Colour on `Light`** (light.md) — `Light.colour`, a boundary's
  `transmittanceColour`, the vision walk's chroma; a stained `Window`
  colours the room through it, and two windows' colours add.
- **The firing carry** (`Recipe.massYield` + `FireController`) — a firing
  carries its charge's mass and alloying onto the output.
- **The salvage branch** (crafting.md) — a meltable non-metal salvages
  whole, carrying its alloying.
- **Light-strike** (spoilage.md) — a `light-sensitive` material (beer)
  spoils under the blue light its vessel lets in; a brown bottle protects,
  a clear one does not.

## The venue

Rejection's **glasshouse** sits in a clearing of the Hanging Wood (fuel-
bound, a LULU outside the walls); the **sand pit** is its own small zone
with a clean face and a dirty face; the **glazier's cold bench** is a hut
in town, lit through two windows — the game's first authored windows. A
second glasshouse anywhere is a deposit row, a business and a location,
with no pack code (the second-instance test).

## Deferred

Optics (lenses, acuity) → `optics-slate`; molds → the dairy build; the
sell counter and the returns/empties loop → retail then a glass follow-on;
the converter/instrument bench → a glass epoch follow-on; decolorisers
(manganese) and the composed stained panel → glass follow-ons. See the
glass plan's deferred-seams table.
