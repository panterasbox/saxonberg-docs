# Exertion

The exertion subsystem is **what working does to a body** — one event,
every producer, four readers. It is the producer side that metabolism,
encumbrance and locomotion each half-owned before the nutrition-and-
fitness build: mining, farming, the smelt, the loaded traverse and the
limp each debited endurance in their own `%` points, and the quern, the
anvil, the loom and a spell cost nothing at all. Now every one of them
says the same sentence — `exert({ durationS, powerW })`, metabolic watts
over game-seconds, which is joules, a unit the engine already speaks —
and the body decides what it means.

Two design rules carry the whole thing:

- ⭐ **Reach is a body read, never a number.** Nothing renders a
  reserve, a rank or a fitness score (measurement.md's no-gauge rule).
  What a person gets is what they can now DO: the run that holds past
  where it broke, the climb without stopping for breath, the second
  shift — and, in the mirror, what a body looks like. `competence`
  shows a **band**; the tape shows a mass.
- ⭐ **Every rate is a per-read dial.** `body.*` and `exertion.*` in
  `AppSettingKeys`, shipped in `platform/content/settings/body.yaml`,
  read through a module-private `dial()` on every use — so `config`
  turns a season up inside one session and turns it back. The
  structure is the deliverable; the rates are playtest. The one set of
  numbers that must hold exactly are the **felt-cost pins**
  (`Exerting.felt-cost.test.ts`): at each shipped act's reference
  duration on a fresh body, the debit is what it was.

## The stocks — `lib/reserve.ts`, `lib/metabolism/Metabolic.ts`

Five biological reserves joined `endurance / satiation / hydration /
flesh` on every `Creature`, installed idempotently by the constructor
(a body that predates them gains them on next construct — no
migration):

| key | seed | what it is | moved up by | moved down by | floor |
|---|---|---|---|---|---|
| `lean` | 50 | muscle — *fat is what you ate, lean is what you did* | overload in `exert` (spends `protein`) | `partitionLean`: exact-exponential relaxation to the seed over `body.leanDetrainDays`; catabolism while satiation ≤ the flesh-deficit line | none — gaunt in the mirror, not sick |
| `protein` | 50 | the amino pool | the `protein` routing tag (it used to drain into nothing) | muscle gain; basal turnover | none |
| `wind` | 0 | conditioning — the stock the `wind` Discipline's band reads | duration at a sustainable pace in `exert` | half-life `body.windHalfLifeDays` | none |
| `vitamin-c` | 100 | the years clock | the `vitamin-c` tag (citrus, the fruit rows) | linear drain, full to empty over `body.vitaminCDrainDays` | `scurvy` — the one new condition, off the shipped cascade |
| `alcohol-tolerance` | 0 | the drinker's adaptation | `absorbToxin` for alcohol | half-life `body.toleranceHalfLifeDays` | none |

Every sink is a step in `Metabolic.integrateSlice` (steps 6–9 after
`partitionFlesh`), a no-op on a host lacking its reserve, and riding the
metabolism clock — so the linkdead freeze and the far-past guard make
*never tax absence* true of all of them with no wiring of their own.
`NutrientRoute.reserve` is an open string now, so a new stock gets a
route by adding a row, never by widening a type. ⚠ `Vitals.
getConditionBand` counts a floored reserve only when it **has** a
`floorEffect` — a stock seeded empty is untrained, not degraded.

⭐ **The stocks reach mass.** `Creature.getMass()` adds
`body.massPerFleshPct × (flesh − 55) + body.massPerLeanPct × (lean − 50)`
to the frame, reading the stored values raw (it sits on the
`wouldExceedCeiling` hot path). Every mass consumer — carry capacity,
the basal drain, thermal mass, the fist, the tailor's girth — inherits
it; a fresh body masses exactly its species figure.

## The event — `lib/exertion/Exerting.ts`

`ExertingMixin` composes on `Creature` (outer of the thermal pair and
of `Metabolic`, inner of `LoadBearing`). Three chokepoints emit, one per
producer family, and none of them knows what the body does with it:

| chokepoint | how | excludes |
|---|---|---|
| `SchedulerRegistry` at a durative activity's completion, pro-rata at a `cancelled` / `replaced` / `preconditions-changed` terminate | `DurativeActivity.effortW?` — an activity that declares none is not work | a search, a dressing, an offer, a three-day charcoal burn — by construction, not by a type test |
| `LocomotionLogic.engageAround` after a successful self-powered traverse | `actor.exertTraverse(mode)`: `walkW × costMultiplier × loadFactor` over `traverseNominalS` — the first reader `costMultiplier` ever had (walk 1.0, run 2.0, climb/swim 2.0, sneak 1.0) | riders and `forceMove`, structurally |
| `Character.onExchangeResolved` | `exertExchange()` — one combat exchange | — |

`BuildStepOptions.effortW` is **required**, so the compiler is the
census of every step verb (28 `engageStep` sites + cast 300 W, study
150 W, the mill 620 W, the smelt 425 W, hew 967 / drive 750 / shore
900). The farming acts keep their authored felt cost and convert
through `wattsForFeltCost(cost, durationS)` — body-independent, so the
plough still costs `14 / draught` and a conditioned body feels it as
less.

Inside `exert`, in order:

1. **Endurance** — `max(0, powerW − sustainableW) × durationS /
   exertion.joulesPerEndurancePct`. Only the EXCESS over what the body
   can sustain debits: the aerobic threshold. A walk is free; a run costs
   a fresh body 12 % an exit and a conditioned one nothing.
   `sustainableW = exertion.baseSustainableW × (1 + body.windSustainGain
   × wind)`.
2. **Wind** — `body.windGainPerHour × hours × min(1, powerW /
   sustainableW)` when `powerW ≥ body.windFloorFraction ×
   sustainableW`: duration at a pace you can hold; a heavy act counts for
   its duration and no more.
3. **Lean** — against the body's own ceiling `mass × body.peakWPerKg ×
   leanMargin` (`0.6 + 0.8 × lean`): when `powerW ≥ body.overloadFraction
   × ceiling`, `body.leanGainPerHour × hours × (powerW / ceiling)`,
   capped by protein at `body.proteinPerLeanPct` per point. ⭐ **A load
   you have outgrown trains nothing** — the quern (620 W) overloads a
   fresh body and stops a few points of lean later; the bar is where a
   person chooses a heavier one.
4. **Heat** — `depositWorkHeat(powerW × durationS × (1 − η))` →
   `ThermalRegulation.absorbHeatLoad`: the thermal slice sheds it by
   sweating, costing hydration, damped by what the body wears (a coat
   halves the shedding), stopped past the wet-bulb ceiling. A shift at
   the anvil in a coat is a wetter, thirstier shift with no wiring of
   its own — and cold air does not stop it (the load sheds through the
   cold branch too; you sweat shovelling snow).

Then, if the `wind` band crossed a threshold either way,
`refreshConferrals()` re-derives the verb set — so a band-gated verb on
the `wind` row (none ship; the swim is the first) needs no Transcript
append.

## ⭐⭐ Soft limits — what a spent body can still do

Every limit this subsystem imposes is a **read that narrows what you
can do next, never a wall**. The run breaks to a walk (you still move);
the climb narrates a rest (you still climb); a load is refused (pick a
lighter one); a step verb refuses at the spent line (`canExert`, *"You're
too tired for that."*) — and walking, talking, looking, buying and
resting all go on, because a walk is free and nothing but WORK asks the
body first. Standing recovery clears a refused step in seconds of wall
time.

The one wall in the game is `collapse` — endurance at 0, the faint that
`requiresConscious` reads on fifty-odd verbs, walking and eating
included. **Work is not allowed to be the thing that puts a person
there.** `exert` clamps every debit at the spent line
(`exertion.exhaustionFloorPct`, 10 %): a ladder, a set or a long fight
leaves you spent, never collapsed, and a body already under the line
(the limp) is left where it is. Collapse stays the acute conditions'
(starvation, dehydration, blood loss) and the wound's. The step verbs
refuse at the same line, so a spent body cannot start what it could not
finish; a spent body in a fight keeps exchanging at zero further cost
with its poise and tempo already reading its endurance.

## ⭐ Feedback — the body as words, at the moment it changes

No hit points, no meter — and a player still needs to know, in real
time, how their body is doing. Three surfaces, all words:

- **The shelf's `BODY` row** — `bodyState` on the `self` card
  (`Creature.subscribableFields`): breath (`fresh` · `tired` ·
  `winded` · `spent`, on the `exertion.freshPct` / `paceFloorPct` /
  `exhaustionFloorPct` lines), hunger (`full` · `fed` · `hungry` ·
  `starving`, on metabolism's own surplus/deficit lines), thirst
  (`fine` · `thirsty` · `parched`, on the recovery throttle), and the
  build phrase. The client shows the breath word and adds hunger and
  thirst only when they say something (*fresh*, not *fresh · fed ·
  fine*). Default-pinned — it joined the default the day it started
  answering.
- **The poke** — `ExertingMixin.noteBodyState()` runs after every
  `exert` and after every metabolism reconcile (`Metabolic.
  onMetabolismReconciled`, a `@hook` terminal the mixin overrides) and
  `notifyDurableSubject`s the self card **only when a band has turned
  over** — never every slice. The row changes when the word changes.
- **The cue** — on a breath crossing, one line on `self.body` in the
  register that topic already speaks (*"You're sweating."*): going down,
  *"You're winded."* / *"You're spent."*; coming back, one line for the
  whole climb — *"You've got your breath back."* — and nothing for
  spent → winded (still short of breath). The first read of a session
  is silent. Crossings only: *the deviation is the story*.

⚠ What is deliberately NOT here: a bar, a percentage, a ticking figure.
The design that makes the run *break* only works if the player is
reading the body and not the meter — with a meter you stop at 51 % and
never learn what winded feels like.

## Reach — the reads

| read | where it is asked | what it does |
|---|---|---|
| `canExert(powerW, durationS)` | `engageStep`, both pack `engageAct`s, the smelt | the double shift: refuses a step that would leave endurance under `exertion.exhaustionFloorPct`, with the mixin's one line (*"You're too tired for that."*, `too-tired`) |
| `canSustainPace(mode)` | `LocomotionControllerBase.execute`, for any mode with `speed > 1`, before the gates | the run that breaks: `capacity('locomotion') === 'full'` (reach ANDs with function — a wounded leg holds no run however conditioned) AND (power within `sustainableW`, or endurance ≥ `exertion.paceFloorPct`); otherwise the mode drops to `walk` with a `pace-broken {from, to}` note (auto-escalates to `partial`) and the winded line |
| `exertTraverse` for `climb` | inside the emit | the rest line when the climb's power exceeds `sustainableW` — a conditioned body pays nothing and says nothing |
| `leanMargin()` | `LoadBearing.getCarryCapacity` | the strength read the tape, the water and `lift` share; 1.0 at the seed |
| `conditioningBand(stock)` | `AdvancementMixin` | untrained < 20 % ≤ novice < 40 % ≤ competent < 60 % ≤ proficient < 80 % ≤ expert |

## Conditioning as a stock — `Discipline.stock`

A `conditioning` Discipline names the body stock its band is a threshold
over (`wind.yaml: stock: wind`, `alcohol-tolerance.yaml`). `Advancement`
overlays those bands on the transcript fold at every read surface
(`withConditioning`) — a stock row replaces any transcript row for the
same key, and a fresh body lists `wind` as *untrained* rather than
absent. **No Transcript rows are written**: VO₂max is a state of the
body, not a memory of your evidence. `wind.yaml` ships `conferrals: []`
on purpose — every rung of the first ladder is a verb a fresh body
already has, read through the body at the verb layer.

## The mirror — `Creature.bodyBuildPhrase`, `Character.markupAugmenters`

Flesh band × lean band, person register, twenty cells that each read
differently (*wiry*, *in good flesh, hard*, *running to fat*,
*massive*). `Character` carries a class-static augmenter that appends
the phrase as one sentence about a body to `look`, naming nobody — and
because the walk collects class statics, it reaches every person and
**no animal**: the stockman's read on a `Livestock` is byte-identical
(`trade-ranching/src/__tests__/stockman-read.test.ts` is the permanent
contract). `assess` says the BMI band in the physician's words for a
competent looker (`Creature.bodyMassIndexBand`); the number is derived
and rendered nowhere. `getBodyDensity()` — fat floats — waits for water.

⚠ The phrase table is one culture's vocabulary. The cosmetics slate's
*Beauty* section names the augmenter as the attach point a per-culture
canon replaces; the shape does not change.

## The gym — `archetypes/gym.yaml`, `LoadDevice`, `lift`

One room archetype (`industry: null`; `survey` reports it anywhere):
`load` (a thing offering the `load` capability) and `mat` (any
surface). ⚠ No `pace` slot yet — `lint:capabilities` treats an
archetype need as a requirement, and nothing offers `pace` until the
first treadmill lands (one line then). Reported, never enforced: a bar
dropped on any floor makes the room meet the load slot with no code and
no second archetype.

`platform/thing/LoadDevice extends ToolItem` — Crafted, Tool
(`capabilities: [load]`), Durable — with `loadMinKg / loadMaxKg /
wattsPerKg / setDurationS`, and `commandContributions` in the
**environment** bucket: the bar on the floor of the room affords
`lift`. `lift <load> [with <device>]` (`platform/cmd/device/lift.yaml`,
the device a declared arg — `default: "reachable:[capability.load]"`,
`requires: [ToolMixin]`) refuses on the device's range
(`load-out-of-range`), the body's strain ceiling (`too-heavy`) and the
double shift (`too-tired`), then runs one hands step at `load ×
wattsPerKg`. The exemplar is trade-smithing's `barbell` (20–160 kg,
15 W/kg, 30 s; a two-bar anvil recipe), on the Hearthworks smithy's
own slate at 14.

## Files

- `lib/exertion/Exerting.ts` — the mixin; `__tests__/Exerting.test.ts`,
  `Exerting.felt-cost.test.ts`
- `lib/reserve.ts`, `lib/metabolism/Metabolic.ts` (steps 6–9, the
  routes), `lib/creature/Creature.ts` (readers, bands, the phrase, BMI,
  density, the mass term), `lib/vitals/Vitals.ts` (the floor guard)
- `api/scheduler.ts` (`effortW`), `platform/idea/SchedulerRegistry.ts`
  (`emitExertion`), `lib/craft/ManualBuildStep.ts`, `lib/magic/
  CastActivity.ts` + `StudyActivity.ts`
- `platform/idea/cmd/crafting/ManualBuildController.ts`,
  `platform/idea/api/LocomotionLogic.ts`, `platform/idea/cmd/movement/
  LocomotionControllerBase.ts`, `lib/character/Character.ts`,
  `lib/encumbrance/LoadBearing.ts`, `lib/advancement/Advancement.ts`,
  `platform/idea/Discipline.ts` + `DisciplineCatalogue.ts`
- `platform/thing/Ladder.ts`, `platform/thing/LoadDevice.ts`,
  `platform/idea/cmd/device/LiftController.ts`,
  `platform/idea/cmd/perception/AssessController.ts`
- `lib/config/AppSettings.ts` + `platform/content/settings/body.yaml`
- content: `platform/idea/Condition/metabolism/scurvy.yaml`,
  `platform/idea/Discipline/wind.yaml`, `generic-objects/archetypes/
  gym.yaml`, `trade-smithing/…/barbell.yaml` + `recipes/barbell.yaml`,
  the ten `trade-farming` fruit rows, `trade-baking/…/wholemeal-loaf.
  yaml`, rejection's `winze-ladder` + `winze-foot`
- the drive: `packages/wire/tests/nutrition-fitness.dirty.wire.test.ts`

## History

Built as the nutrition-and-fitness build (2026-09-18/19, `design/
nutrition-fitness`, MR !269, `f4a7528d3..HEAD`) from
`docs/slates/tails/nutrition-and-fitness-slate.md` (the plan and
requirements retired at the sweep; the slate's *tail* section carries
what the plan's deferred seams named). Review added the soft-limit
clamp (work never collapses a body) and the feedback surfaces (the
`BODY` shelf row, the `self.body` cue); the browser walk removed
conditioning stocks from the *practising* figure and from the
`competence` listing when untrained. Decisions the build made that the plan did not: the Vitals
floor guard (D22), farming's felt cost converted through the body
(D23), `char` declares no effort and the mill is 620 W (D24), the
orange on the general store and the barbell on the smithy's slate
because the distributor may not point at a spoke (D25/D26), the gym
without its `pace` slot (D26).
