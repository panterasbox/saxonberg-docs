# Fractionation — a batch that yields in ordered fractions as it is drawn

⭐⭐ **A pot of wash under a fire does not give up one liquid.** It gives up
a sequence of them: the first drops are the poisonous ones, then the harsh
ones, then the good ones, then the dull ones, and what is left in the pot
is not a spirit at all. Nothing in the engine could say that.

This subsystem is the shape that says it, and it says nothing about
whiskey. A schedule row names what comes off, in what order, and the host
that composes `FractionatingMixin` yields it as it is drawn. The first
consumer is `trade-distilling`'s pot still; the second, whenever the
drilling slate lands, is a refinery column, and that is rows.

Shipped by the whiskey build (2026-10-04). Siblings worth reading first:
[maturation.md](./maturation.md) (the *clock*-driven transform this is the
*volume*-driven twin of) and [bulk.md](./bulk.md) (the slot the run rides).

---

## The one idea: the clock is the VOLUME DRAWN

There is no time integral here, deliberately. A charged still that nobody
is pouring from makes no progress, which is the whole reason the skill is
*where you stop* rather than *when you look*. It also decides the shape:
`MaturingMixin` could not have been reused, because

| | maturation | fractionation |
|---|---|---|
| driven by | game-time | litres drawn |
| outputs | **one** product material | one material, **many payloads** |
| the act | wait, then draw | draw, and the drawing *is* the run |
| the skill | keeping the band | knowing when to stop |

⭐ The mechanism generalises the **lees split** — `maturation.md`'s
rack-floor rule, which is amount-triggered rather than time-integrated —
from one boundary to many.

## One material, many payloads

⚠⚠ The fractions do **not** differ by `Material`, and that is forced
rather than chosen: `BulkableApi.transfer` declines a cross-material pour,
so fractions that were different materials could never be **recombined** —
and recombining them is most of what a distiller does (the heads back into
the next charge, the hearts into one cask).

So a schedule names ONE `productMaterial`, and the fractions differ by
what rides on the payload:

- **the dose** — `dissolvedToxins`, mg per litre (see *The harm* below);
- **the quality** — a `gradeBand` per fraction;
- **the prose** — a `character` per fraction, which is the only thing a
  player ever reads.

One `residueMaterial` names what is left in the pot once the last
fraction is drawn — the lees-style swap.

## The pieces

| piece | where | what it is |
|---|---|---|
| `FractionSchedule` | `lib/fractionation/FractionSchedule.ts`, concrete twin at `platform/idea/fractionation/` | the row. A singleton `Idea` per schedule, under any root's `idea/fractionation/` subtree, matched to a charge by the charge material's TAGS against `inputCategory`. Statics `byKey` / `forMaterial`; `all()` private. |
| `FractionScheduleCatalogue` | `platform/idea/`, + a platform-pack row and a `boot:` entry at role `sync-read` | the roster warm. ⚠ A schedule row nothing stands up matches nothing, **silently** — a charged still would simply never start. The reference-Ideas-inert-at-boot rule, which this repo has broken three separate times. |
| `FractionatingMixin` | `lib/fractionation/Fractionating.ts` | the host face. Requires `BulkableMixin`; probes `ThermalMixin`. |
| `BulkPayload.dissolvedToxins` | declared from `lib/metabolism/DissolvedToxins.ts` | the dose a draw carries, per litre. |

### The row

```yaml
class: /platform/idea/fractionation/FractionSchedule
data:
  key: wash
  inputCategory: wash          # a TAG on the charge material
  discipline: distilling       # what a draw credits
  requiresHeatK: 351           # 0 = runs the moment it is charged
  productMaterial: …/new-make
  residueMaterial: …/stillage
  readBlur: 0.02               # how late an untrained nose reads it
  gradeStretch: 0.0075         # how much a poorer charge grows the head
  fractions:
    - { key: foreshots, upTo: 0.005, gradeBand: poor, character: "…",
        toxins: [{ type: methanol, amount: 12000 }], aromaticCarry: 0 }
    - { key: heads,     upTo: 0.03,  gradeBand: fair, character: "…",
        aromaticCarry: 0.5 }
    # … strictly ascending in (0, 1]; the last one's complement is residue
```

`setFractions` validates the whole schedule and **refuses** an incoherent
one at the row: boundaries out of order, a band the grade vocabulary has
never heard, an empty character, a last fraction that claims the whole
charge (a pot that gives up everything it holds is not a still) — and
⭐ **a character containing a digit**, because the cut is a judgement and
not a gauge.

### The state machine

Four words, all persistent runtime state on the host:

- **idle** — nothing in it, or matter no schedule matches (a still full of
  water, which is not an error).
- **charged** — the charge is in, the heat is not. ⭐ A cold charge pours
  straight back out **as what it is**: nothing refuses it and nothing has
  happened to it.
- **running** — the heat arrived, the interior material swapped to the
  product, and every draw advances `drawnL`.
- **spent** — the last fraction is gone and what remains is the residue.

⚠ The run ends on **`drawnL`**, never on comparing the interior amount
against the floor: those are two floating-point routes to the same number
(`charge × residue` against `charge − drawn`) and they do not always land
on the same side of it. A pour of exactly `available()` once left a still
one part in 10⁻¹⁰ above its own floor and the run never ended.

### The three policy seams it overrides

`Bulkable` exposes `getBulkAvailable` / `isBulkEmpty` / `debitBulk` as
host policy (the `UnboundedSource` precedent). This mixin overrides three
of them plus a new one:

- **`getBulkAvailable`** — a running host gives up everything down to the
  residue floor and no further. ⭐ And it clamps at the first fraction
  whose own `requiresHeatK` the host has not reached: the heavy ends do
  not come over until the pot is hot enough for them. That is the seam a
  refinery column is built on and a pot still never exercises.
- **`debitBulk`** — advances `drawnL`, which is the only clock there is.
- **`getBulkPayloadForDraw(affordance, litres)`** — **new in `Bulkable`**,
  base impl returns the slot's own payload so nothing changes for any
  other composer. Answers with the **span** `[drawnL, drawnL + litres]`
  blended by volume: the dose volume-weighted, the grade the **worst band
  in the span** (a pour with tails in it is a pour with tails in it), the
  maker the hand that charged the pot.
  ⚠ It takes the litres because a draw that straddles a boundary is two
  things at once. Without the argument a single pour started at the first
  drop is stamped wholly as its first fraction — three quarters of a litre
  of foreshots instead of the honest thirty millilitres of them.
- **`getBulkMaterialPath`** — reconciles the run first, and this is not
  tidiness. ⚠⚠ `transfer` captures `from.getMaterial()` at step 1 and
  stamps the destination with it at step 5, but the run does not start
  until something asks a *policy* seam, and the first thing that does is
  the clamp at step 4. So the first draw after the fire came up stamped
  the receiving vessel with **wash** while handing it a payload full of
  new-make's foreshots: a bottle of wash that poisons you. Found by a
  world test; every unit test heated the host before pouring, which is the
  one ordering that hides it.

## ⭐⭐ The read: blurred, optimistic, and silent

`readFraction(viewer)` reports the **best-graded fraction within
`readBlur × blurForBand(band)`** of where the run actually is. At `expert`
the window is zero and the read is exact; at `untrained` it is wide, and
the nose hears what it wants to hear.

```
BLUR_BY_BAND = { untrained: 1, novice: 0.75, competent: 0.5,
                 proficient: 0.25, expert: 0 }
```

⚠⚠ **It was a LAG for an afternoon, and a lag is the wrong error.** The
plan reasoned that a late nose *"keeps some heads and some tails"*, and
the first half is impossible: you start collecting when you believe the
hearts have begun, so a nose that notices boundaries late starts
collecting **late**, throws good spirit away and makes a *cleaner* bottle.
It taught the opposite of the lesson — and the two world-test distillers,
an "expert" and an "untrained", made the identical cut.

⭐ An optimistic window produces exactly the two errors the design wants,
from one sentence instead of two rules: the untrained distiller believes
the hearts have started while the heads are still coming over, and
believes they are still running after the tails begin. Both errors enlarge
the cut and make it worse — which is also the true pressure on a real
novice, because yield feels like money.

**Competence resolves DETAIL and never ACCESS.** The untrained distiller
can draw every drop in the pot; the refusal they get is from the fire, not
from their transcript.

⚠ **And nothing announces a boundary.** No note, no scene line, no push.
The character changes on the next `smell` and that is the whole signal; a
message would turn a judgement into a prompt. A test asserts the pour's
notes are byte-identical either side of a boundary.

The augmenter is filter-gated (`smell` / `taste` only). `look` gets the
*state* — cold, charged, running, spent — and never the fraction: a glance
at a running still tells you it is running, not where in the run it is.
⚠ Do not copy `maturationAugmenter`, which takes no `opts` and lands on
every channel; that is how a cellar line came to be read out over a field
of linen.

## ⭐⭐⭐ `aromaticCarry` — where a choice at ANOTHER rung lands

A fraction may carry the **charge's own aromatics**
(`BulkPayload.dissolvedAromatics`, `metabolism.md`) at its own multiple.
Absent ⇒ `1`, which is *comes over unchanged* and is what every schedule
did before this existed; `0` means none of it reaches this fraction.

⭐ **This is the seam that makes a vertical a vertical rather than six
corridors into one decision.** Phenols are high-boiling, so a peated wash
comes over clean at the start and leaves its heaviest smoke at the end.
The shipped wash schedule:

| fraction | span | carry | a 30 mg/L malt reads |
|---|---|---|---|
| foreshots | 0.005 | **0** | nothing |
| heads | 0.025 | 0.5 | 15 mg/L — *clearly* |
| hearts | 0.15 | 2.5 | 75 mg/L — *strongly* |
| tails | 0.05 | **6** | 180 mg/L — *overpoweringly* |

So a peated wash and a clean one **do not want the same cut**:

- take the hearts short and you have thrown away the character you spent
  a day of turf on;
- run them long to keep it and you are taking tails into the spirit, and
  the grade falls to the worst band you drew.

⚠ Neither answer is right, and that is the point — the tails smell more
of what you made the malt for than the hearts do, and they are `poor`. A
distiller who follows their nose ruins the cut; one who follows the grade
leaves character in the stillage. **The maltster's decision has rewritten
the distiller's correct answer**, which is the thing the cuts build could
not say about any of its six rungs.

⚠ There is no mass-balance enforcement across a schedule and there should
not be: the residue is where a remainder goes. The shipped figures put
about 69 % of the charge's phenol over and leave the rest in the
stillage. A straddling draw volume-weights the two carries exactly as the
dose is weighted — anything else would let a distiller take the tails'
smoke at the hearts' grade.

## The charge's grade moves the boundaries

`gradeStretch` grows the toxic head of the run by that fraction of the
charge **per grade band below `masterful`**, and shifts every boundary but
the last by the same absolute amount. A `poor` wash has roughly twice the
head of a `masterful` one, its hearts start later and run shorter, and
there is no litre count anybody can memorise. Seeded from the charge,
never drawn (uncertainty.md).

⚠ The figure is small on purpose. It was `0.02` for an afternoon: the
shift is **absolute** and the foreshots are five thousandths of the
charge, so 0.02 per band made a merely-`fine` wash nine-tenths foreshots
and the grade swamped the schedule entirely. ⭐ *A dial whose smallest step
is larger than the thing it adjusts is not a dial.*

## The harm

The dose a draw carries is `BulkPayload.dissolvedToxins` — **mg per
litre**, declared onto the payload from `lib/metabolism/DissolvedToxins.ts`
the way `formedToxins` is. It is the third toxin shape and the first that
blends:

| shape | per | blends on a pour? | scales with litres drunk? |
|---|---|---|---|
| `Material.toxicity` | serving, per **substance** | n/a | no |
| `BulkPayload.formedToxins` | serving, per **instance** | **no** (a ptomaine dose is a dose) | no |
| `BulkPayload.dissolvedToxins` | **litre** | **yes**, volume-weighted | **yes** |

⭐ The scaling is the whole distinction and it is one line in
`routeIntake`: a dissolved tag contributes `amount × litres`. That is why
a badly-cut bottle hurts the person who finishes it and not the person who
tastes it.

### Who the ledger names

`Metabolic.noteConsumptionHarm` is **one function with two call moments**
(see [accountability.md](./accountability.md)):

- the **pathogen** arm calls it at the infection, as it always did;
- the **toxin** arm calls it at the **first band crossing** — because
  attributing at the swallow would name a maker for the trace congeners in
  every honest bottle in the realm.

`Metabolic.toxinMakers` remembers who made the doses a body is still
carrying, recorded at ingest for the two **per-instance** shapes only.
⭐⭐ **A Material's own authored toxicity records nobody**: the ledger names
the maker for what the *making* put in it, never for what the thing *is*,
so a bartender is not a poisoner for every drunk patron. The makers clear
with the burden.

⭐ Side effect, accepted deliberately: the shipped staph/botulinum
`intoxicate` arm is now attributed too. It never was.

## Composing it

```ts
FractionatingMixin(                 // outermost — its seams shadow Bulkable's
  BurnerMixin(LightSourceMixin(ReservedMixin(
    ThermalMixin(CraftedMixin(BulkableMixin(ToolMixin(Good))))))))
```

Composing it claims: **this host's interior yields in ordered fractions as
it is drawn, under a schedule matched to its charge.** `Vat` does not
(its lees are one boundary and a material swap, which `MaturingMixin`
already says); a bottle pours what it holds; a forge holds no bulk.
⭐ No guard re-narrows the host set — the second consumer composes it
itself.

⚠ **`Still` is the first `Burner + Bulkable` composition in the tree.** It
needed `BulkableMixin` because a pot still is a pot with a fire under it
and the class held no pot, and `CraftedMixin` for the `Vat`'s reason: the
charge's grade and the charging hand arrive through the transfer seam and
leave again on the draw, so the bottle names a person and a band.

## ⛔ And it had never run

Worth recording, because the failure was invisible for the whole life of
the `trade-distilling` pack. The `distil` / `brandy` / `grappa` recipes
named the still's capability and asked for 351 K — and **no still row
authored a fuel reserve**, so `FireLogic` refused to ignite the burner
(`not-flammable`), `reachableHeatForImpl` counted no heat, and every
`order distil` declined `insufficient-heat`. Nothing observable depended
on any of it: the counter's gin and spirit came off Veshko's faucet rows,
not off a run. All three recipes retired for schedule rows, and both still
rows author fuel now.

## Adding a feedstock

Rows, and nothing in `lib/`:

1. a material for what comes off, and one for the residue;
2. a schedule row whose `inputCategory` is a tag the charge carries;
3. that is all — `pour`, `ignite`, `smell` and `taste` are already there,
   and the draw credits the schedule's own `discipline`.

⚠ One constraint: **a material tag may belong to only one matcher of a
given kind.** A `MaturationProfile` whose `inputCategory` is also a
schedule's would re-key every vessel that so much as received the charge —
which is why distilling's wash profile keys on `distillers-wort` and the
schedule keys on `wash`.

The crude sketch, for whenever drilling lands: `inputCategory: crude`,
`requiresHeatK` per fraction (naphtha 350, kerosene 450, gas oil 550),
residue `bitumen`, no toxins anywhere, characters by smell. A six-fraction
version of exactly that runs in
`lib/fractionation/__tests__/Fractionating.test.ts` today, on unchanged
code, as the proof that a different product is rows.

## Deferred seams

- **`drawTemperatureK` on a schedule** — the condenser is not modelled, so
  a draw lands at the pot's working heat (~358 K) and cools. Autoignition
  is far away; a bottle of hot spirit next to a hearth is the only odd
  case.
- **Burner refuelling** — no refuel verb exists for *any* burner in the
  game. A still burns its authored reserve down once, which at its
  authored rate is a long working day. → the fire/energy slate.
- **The transfer participant hook** — `BulkableApi.transfer` is at
  **eight** domain insertions now (freshness, water, pathogens, blood,
  thermal, the payload copy, the identity carry, and this build's
  dissolved blend). `bulk.md` and `maturation.md` both named poisons as
  the moment to generalise to a host-side hook. Filed with the count
  rather than done, per *census then ratchet*: refactoring four shipped
  blends inside a feature build would have hidden the feature.

---

## History

**2026-10-05 — `aromaticCarry` (the whiskey-styles build).** The schedule
gained one optional number per fraction and it is what answers the charge
the cuts build's own lens pass recorded and left open: *"if the cut
decides everything and the other five are corridors, then five rungs are
ceremony."* See the table above.

⭐ The grain line arrived the same day as rows only — a second schedule
(`grain-wash`) keyed on a second wash material, with a shorter head and a
longer, more neutral heart. **No code knew there were two kinds of
whisky.** ⚠ Its `aromaticCarry` is flat at 1 deliberately: unpeated wheat
carries nothing to redistribute, and authoring a late carry there would
assert a chemistry that is not in the charge.

⚠ A tag may belong to **one matcher of a given kind**. `grain-wash` must
not carry `wash` or two schedules claim one charge, and `forMaterial`
resolves a double match by taking the lowest key — silently. A test now
scans the whole content tree for that clash.

## ⭐⭐⭐ `separation` — grades of one thing, or different things (the drilling build, 2026-10)

A schedule now declares what KIND of separation it performs, and the
distinction is not a matter of degree.

- `'cuts'` (the default, so every row authored before this is
  byte-identical) — **grades of one substance**. A pot still's
  foreshots, heads, hearts and tails are all new-make spirit; they
  differ in character and in what they will poison you with, and the
  distiller's art is deciding where to cut and then **blending what they
  kept**. One `productMaterial`, several qualities of it.
- `'fractions'` — **different substances**. A refinery column's gasoline
  is not a grade of kerosene and no amount of blending makes it one.
  Every `FractionSpec` names its own `material`, `productMaterial` must
  be empty, and ⚠ **`getBulkAvailable` clamps at the boundary**: the
  column will not hand you two substances in one cask. You draw until
  the character changes and you change casks.

⭐ That clamp is the mechanism behind *you cannot distil crude and choose
not to make the light ends* — joint production, and the honest reason a
player ends up holding something nobody will buy.

⚠ The pair is validated in **`onCreate`**, not in `setFractions`:
`separation` and `fractions` are two authored keys and the applier
dispatches them in whatever order the row lists them, so a setter check
reads whatever `separation` happened to be at that moment. A perfectly
good column would throw and reordering the YAML would fix it — a
validation that depends on authoring whitespace. The convention already
says a cross-field rule lives in the host's `onCreate`.

⭐ And `FractionSpec.requiresHeatK` is exercised for the first time by
`/trade/fuel/idea/fractionation/crude`: naphtha comes over at 350 K and
gas oil does not, so a cold column gives the light ends and stops. That
is the seam this doc's own header predicted *a refinery column is built
on and a pot still never exercises.*
