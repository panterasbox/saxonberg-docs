# Glass — the trade, the window, and the colour that is a grade

The `trade-glass` pack (`/trade/glass`) turns three other trades'
leftovers — the quarry's sand and lime, the collier's ash — into glass:
see-through containment, chemical inertness, and the first window that
colours the room it is set in. It rides four kernel seams shipped in the
same build (see § Kernel seams) and ships a Discipline, a pack `lib/`, the
thing classes, a `Reading`, rows, dials and two archetypes — so a
second glasshouse is rows.

## The chain

1. **Win sand** at a pit (`dig`, the quarrying trade's verb). The pit's
   deposit pins the **iron grade** per face — clean in the heart, rusty at
   the fringe — and the quarry's mint stamps that assay onto the won load
   (a `CompositionMixin` good; see mining.md § the assay stamp). `analyze iron
   <sand>` reads it: a band word to an untrained eye, the figure to a
   proficient glassmaker.
2. **Mix the batch, then fire it** — ⭐ two acts, and neither is a recipe
   (ceramics W0: mixing is one rule, firing is one rule).
   - **Mix.** Put sand, ash and lime in the glasshouse's **batch pot** (a
     kernel `CompoundingVessel` row, `/trade/glass/thing/batch-pot`) and `stir
     pot`. The pot folds what was put in it **by mass** into one charge of
     batch (`/trade/glass/thing/batch`, a `Melt` piece of the batch
     material) whose `alloying` is COMPUTED from the inputs: the batch's
     iron is its sand's iron, mass-weighted, and nothing authors it. Two
     sand to one ash and one lime is the glasshouse's habit, not a law —
     a leaner or fatter batch is just what you weighed. A lump of
     charcoal in the pot is the **amber** batch (its carbon, by mass).
   - **Fire.** Put the charge in the kiln and hold it at heat (`fire`).
     The batch MATERIAL (`/trade/glass/idea/material/batch`) carries the
     heat transition: above the liquidus (1400 K — the bellows) it banks
     heat-work and, at its rung, the **same piece** becomes glass at 72 %
     of its mass, keeping everything dissolved in it. So the glass is the
     colour of its sand, derivably, and the fired charge is gatherable
     while it is hot.
3. **Work it hot** on a blowpipe through the **hot-work window** — the one
   new mechanism (see § The hot-work window): `gob` a gather (`dip pipe in melt` — `gob` is an alias of the platform's `dip`; the melt is a dip bath), `shape` it
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
work-band crossing (`workBand()`: runs · soft · stiffening · setting,
relative to the working floor) coloured by the gather's own heat on the one
incandescence scale (`FireApi.incandescenceOf` — the kiln's and the smith's
words; a gather at 1450 K glows yellow, never "white"), and when the gather goes cold converts it to cullet
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

## ⭐ Composition vs temper (storage-and-ice, 2026-10-17)

What is dissolved in a piece and what a heat treatment left in it are
two claims: `CompositionMixin` (`lib/material/Composition.ts` —
`makeup`, `fractionOf`, `setMakeup` — `alloying` until ceramics re-plan
R1) and `TemperedMixin`
(`lib/material/Tempered.ts` — `temper`). `AlloyedMixin` is the two
together, what metal stock composes; a lump of glass, wax or ice keeps
its fractions and claims no temper. So `/stuff/thing/Casting` (cullet,
the off-spec non-metal lump, salvaged glass) is `CompositionMixin`
only, and `/stuff/thing/metal-casting` (`MetalCasting`) carries both.
`Material.getCastTemplate()` derives the default — authored, else
`metal-casting` for a `metal`-tagged material, else `Casting`. ⚠ A site
that reads only the fractions asks `MixinApi.isComposition` (every glass
site, `TintedMixin`'s reads, `analyze iron`'s `subjectRequires`), one
that reads only the temper asks `isTempered`; `isAlloyed` still means
both.

## Cullet goes one way toward green

Broken glass re-melts by **phase change** (ceramics W0, ruling C — the
`remelt-cullet` recipe is gone): a bottle, a pane or a lump of cullet is
`Meltable`, and in a kiln above glass's melting point it melts and pools
into the pot standing beside it (`poolVesselIn` — never into a floorless
chamber's void), the pool carrying the piece's `alloying` in its payload.
The pool blends by volume with what is already in the pot and sets back
into a lump that reads the iron again. A gather comes from the POT (a
`Melt`): a cullet lump or an old bottle goes back in to remelt, as a
glassblower's does. The kernel `salvage` branch for meltable non-metals
(crafting.md) does the same off the fire. So mixed cullet can never make
clear ware: one green piece in the pot tints the lot.

⚠ The *pot's own* pickup — a campaign adding a little iron so even a
clean piece comes back a step greener — is not modelled since the remelt
recipe retired (the melt row's authored iron was that step, and the
`glass.pot.ironPickup` dial was never read). Recorded seam: the pot
imparting into what melts in it, on the heat-work rung.

## Kernel seams (shipped in this build's Stage A)

> (rubber build, 2026-10) The glass ROW now carries a grey
> `lightAbsorption` (α 8/m: 4 mm passes 0.97) — what a construction face
> of glass reads, by Beer–Lambert. `Tinted`'s per-channel colour term is
> unchanged and multiplies where both exist (a glazed `Window`).

- **Colour on `Light`** (light.md) — `Light.colour`, a boundary's
  `transmittanceColour`, the vision walk's chroma; a stained `Window`
  colours the room through it, and two windows' colours add.
- **The firing carry** — since ceramics W0 the batch is a `CompoundingVessel`
  blend (alloying folded by mass) and the firing a heat transition on the
  batch material (`massRetained 0.72`, the piece keeps its alloying); a
  remelt is phase change with the pool carrying `BulkPayload.alloying`.
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

### `shape` is the platform's (ceramics W4)

`shape` was promoted out of this pack (`platform/cmd/crafting/shape.yaml`):
the BLOWPIPE plans its gather's forms (`Blowpipe.shapePlanFor` — the blow and
the jacks in one move, re-validated at commit, lost to cullet if it set), and
`shape bottle` binds word for word (with no `into`, the one word is the form).
A wheel plans a pot through the same protocol.

⭐ **The forms are ROWS (ceramics review, 2026-10-28).** The pipe composes the
kernel's `FormingMixin`, as the potter's wheel does: its row lists `Form` rows
(`/trade/glass/idea/form/{bottle,cylinder}` — a closed vessel and a tall open
one), so a blown jar or flask is one more row. The gather is `Formable` (`lib/craft/Forming.ts`): the
blower's competence in `glasswork` sets the largest vessel they can blow
(`glass.blow.maxL.<band>`, untrained 2 L … expert 20 L) and the blow's grade
(untrained poor … expert exceptional). The blow records the row it will
crack off as and that grade on the gather, and `crack` mints from them — the
hardcoded form→product table and the flat "fair" are gone.

## Note — the potash build (2026-10-19)

The batch's slots are `flux` (the potash family: wood ash, potash, black
salts, pearl ash) and `stabilizer` (quicklime) — lye is alkali but no
stabilizer, so it fills no slot. The melt carries the flux's potash share
(concentrated by the yield); the melt itself stays ungraded (the trade
grades by colour). See [potash.md](potash.md).
