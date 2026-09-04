# Husbandry — the growth model

The **husbandry** substrate is the living world's engine: a thing that
**grows**, whose condition is a pure function of `(profile, clock stamp,
water, light, root room, interventions)` and is computed lazily on read.
No tick, no scheduler, no presence gate — a plant advances whether or not
anyone is logged in.

Phase 1 ships it at the smallest scale that can carry it: a **houseplant**.
You buy a pot, fill it with soil, plant a seed, and keep it alive. Tend it
and it matures, flowers, and gives you a seed to plant again; neglect it and
it dies. Nothing is harvested for money and nothing is farmed — the plant
exists to prove the growth model and the object shape the rest of the family
inherits.

Seeded by farming-slate § Houseplants
and governed by ranching-slate § The five shared
conventions; phase 1 of nine in
living-world-roadmap.md.

---

## The object model — four things you assemble

| Object | Class | What it is |
|---|---|---|
| **Pot** | `/platform/thing/PlantPot` | a `Slotted` fixture with one `plant` slot **and** a bulk interior holding soil. **Its soil volume is the root ceiling.** Two sizes ship. |
| **Soil** | `/stuff/idea/material/bulk/potting-soil` | a bulk `Material`. A sack of it is an ordinary bulk holder; you `pour` it into the pot. |
| **Seed** | `/platform/thing/Seed` | a discrete `Thing` composing `PlantableMixin`, which names the plant template it grows into. Bought, or set by a flowering plant. Consumed on planting. **The capability is the mixin** — a cutting or a tuber is plantable without inheriting `Seed`. |
| **Plant** | `/platform/thing/Plant` | a `Slottable` `Organism` carrying `GrowingMixin` — **all the state, and the persistence host.** ⚠ The class is a *composition*, not a gate: `harvest` and `repot` narrow on `MixinApi.isGrowing`, never `instanceof Plant`, and the harvest + rooting surface (`harvestTemplatePath`, `nutrientDraw`, `isHarvestable`, `getBed`, `transplantDifficulty`) lives on the mixin for that reason. |

**A pot is the density dial at N = 1, not a special case.** The farming
slate specifies the boutique density as *"a garden bed is a `Slotted`
fixture with N slots; each plant is a `Slottable`"* — which is exactly this
shape with a bigger N, so phase 2's garden bed is this code and not a
rewrite. A fused pot-and-plant object would have been thrown away.

```ts
// obj/Plant.ts — PersistableMixin OUTERMOST (the host rule)
PersistableMixin(PostRegistrationMixin(SlottableMixin(GrowingMixin(
  ReservedMixin(OrganismMixin(ThermalMixin(DetailedMixin(Thing))))))))

// obj/PlantPot.ts — the Slotted host + the soil holder. NOT a host.
PopulatesMixin(SlottedMixin(BulkableMixin(ContainerMixin(DetailedMixin(Thing)))))
```

> ### ⚠ A slotted plant lives in the pot's **contents** *and* its **slot**
>
> The `Slotted` capture slice records its occupants by **index into the
> container slice**; a non-content occupant resolves to `-1` and is
> **silently dropped on restore**. Both relations are load-bearing — this
> is the wear/equip pattern, and every code path that seats a plant
> (`plant`, `repot`, the starter pot's populate applier) does both.

`Plant` is **not** a `Creature`. It is a `Thing` composing `OrganismMixin`,
which was already written for the case ("Plant-Things … compose it on their
own concrete class", `getSex()` returns `null` "for v1 plants"). No anatomy
beyond the shipped `sessile` body plan, no vitals, no `MetabolicMixin`.

---

## The reconcile contract

`GrowingMixin.reconcileGrowth()` is **synchronous, read-triggered, and
idempotent**. Every public read (`getVigor`, `getConditionBand`,
`getGrowthStage`, `isFlowering`, `getSoilMoisture`, `getLimitingFactor`)
opens with it behind a reentry guard, so the state a caller sees is always
current as of the world clock.

1. Resolve game-seconds; bail when no world clock is running.
2. First touch (`growthClockStamp === 0`) → seed the stamp and the light
   sample, integrate nothing from epoch.
3. `elapsed <= 0` → restamp and return. Dead → return (terminal).
4. `steps = min(ceil(elapsed / husbandry.stepSec), husbandry.maxSteps)`;
   `dt = elapsed / steps`.
5. Per step: drain the moisture reserve by evapotranspiration × warmth;
   compute `satWater`, `satLight`, `satRoot`; take the **minimum**; relax
   vigor toward it; accrue maturity when the minimum clears
   `husbandry.goodAt`; advance the stage; update flowering; check the
   death floor.
6. Write back and restamp; then re-sample the light window.

**Trigger points.** Any read (above); `waterPlant`; the host's own
`onMoved`; a **carrier's** move (a pot forwards to its occupant via
`noteEnvironmentChanged` — a plant cannot see the pot it rides being
carried); and the flowering latch.

### The clock rule — and why the far-past guard is excluded

`lib/wetness/Wet.ts` is the structural template this mixin copies:
decomposed scalar persistence (a value plus a game-seconds clock stamp, no
marshaller), a reentry guard, content-derived rates, `AppSetting` dials with
seeded literal fallbacks, banded presentation via a `markupAugmenter`,
sparse defaults.

It diverges on exactly two lines, deliberately:

> **No far-past guard.** `Wet.ts` drops any interval longer than four real
> hours, because real absence must not dry you out. **A plant is not a
> body.** Owned things integrate the *full* absence — that is the entire
> point of the family clock, and inheriting the guard would silently negate
> it. A player away three real days comes back to a plant that lived those
> three days.
>
> **No linkdead freeze.** `Wet.ts` also freezes on
> `isHasInteractive() && isLinkdead()`. A plant has no interactive;
> copying that branch in is the same bug in a different hat.

Long absences are bounded by a **step cap, never a time cap**: the step
count is capped and `dt` grows to compensate, so a simulated year still
integrates fully in bounded work. The guard is per-consumer, not global —
`Metabolic.ts` uses four hours, `MechanicalMovement.ts` uses ninety days,
and husbandry uses none.

---

## Three inputs, combined by the limiting factor

Growth runs at `min(satWater, satLight, satRoot)` — Liebig's law, at the
smallest scale that can demonstrate it. This is the exact expression
phase 4's field evaluates over six inputs.

**Water** is the plant's own root-zone `moisture` reserve — a
`ReservedMixin` reserve in litres, theme `cultivation`, authored directly in
the template's `data`. It drains by evapotranspiration (accelerated by the
host's own Thermal reading) and refills through `waterPlant`.

> **⚠ REVERSED IN PHASE 2 — moisture lives in the SOIL now.** Phase 1 put
> water on the plant and rejected the physically prettier model on the
> grounds that it *"splits one checkpoint across two objects, while the
> reconcile, the clock stamp and the persistence record are all the
> plant's."*
>
> That objection was correct, and it is not waved away — it is
> **answered**. Phase 2 gave the soil a checkpoint **of its own**: a
> `CultivableMixin` host owns a `moisture` reserve, its own
> `soilClockStamp`, its own `reconcileSoil()` and its own reentry guard,
> and drains by the **summed** water demand of its occupants over its own
> elapsed window. **The plant only READS.** So there are two
> self-contained checkpoints, not one split across two objects — which is
> precisely the thing phase 1 refused.
>
> **A pot is a bed with one slot**, so the pot holds the water now. The
> shipped pot seeds author `reserves.moisture`; the plant seeds no longer
> do. Read the code and this paragraph together: the old sentence is
> preserved above only so a reader who finds it does not conclude the code
> is wrong.
>
> Three things came out better than phase 1's arrangement:
>
>   - **Water competition is emergent.** More plants in a bed drain it
>     faster, so everyone runs drier — the same shape as the shared-soil
>     root competition, from the same source of truth, with no new rule.
>   - **No read-order artifact.** The bed drains by total demand over its
>     own window rather than each plant debiting as it is read, so looking
>     at plant A before plant B gives the same world as B before A. Had
>     the plants debited, whoever read first would drink first when the
>     bed ran dry — a real fairness bug this shape avoids.
>   - **"An unpotted plant has no water" is literal.** There is no private
>     reserve left to fall back on: `soilMoisture()` returns `null` and
>     `satWater` is 0, so the sentence below is the mechanism rather than
>     a gloss.
>
> `Reserve` is a **neutral** capacity axis, not a Creature-coupled one —
> its own docstring says so, and its landscape table lists `fuel` (a
> campfire) and `air` (a Location) as non-creature consumers. Soil is now
> another.
>
> ⚠ **The recursion hazard, and its guard.** The bed's reconcile reads its
> occupants' demand and each occupant's reconcile reads the bed. Demand is
> therefore read through `waterDemandPerGameDay()` — a **pure,
> non-reconciling** read of the authored profile. Never call
> `getSoilMoisture()` from the soil side. The bed carries its own reentry
> guard as the belt to that braces.
>
> ⚠ **Both clocks start together.** The soil seeds its stamp lazily on
> first read, so a bed nobody has looked at would otherwise stamp itself
> at that moment and skip the whole elapsed window — handing its occupants
> a full reserve they should long since have drunk. Placement (`onMoved`)
> starts the soil's clock, and a plant's own first touch reads its ground.
>
> ⚠ **Occupancy changes settle the window first** — a plant seated into a
> bed must not make that bed retroactively thirsty for a month it stood
> empty, and one lifted out must leave its share drawn. But a persistence
> **re-seat is not a transplant**: settling there would swallow the
> absence the record exists to preserve, so `plant`/`repot` settle the
> destination explicitly (the caller knows which it is doing; the mixin
> cannot).

**Light** is the lux where the plant sits, read through
`PerceptionApi.modalityByName('vision').signalAt(...)`. The sample climbs
the containment chain to the nearest enclosing space, stopping at a
**closed `Sealable`** — a sealed container is an honest dark.

> **The light window is segmented at moves, not integrated from history.**
> There is no light *history* to integrate, so a window is credited at the
> single level in `_lastLux`, and a move closes the window at its true
> level before re-sampling. Carrying a plant (or its pot) into the dark
> stops crediting lit hours from that instant, exactly as watering closes
> the dry window. `getLimitingFactor()` deliberately samples **live**
> instead, so the cause line describes where the plant is *now* rather
> than the window that just ended.

**Root room** is the third factor, and one curve gives two behaviours with
no special-casing:

```
satRoot = clamp(potSoilVolume / profile.rootDemand[stage], husbandry.rootFloor, 1)
```

- The **floor** means a pot-bound plant holds at a visible band and
  **never dies of it** — honest, since real pot-bound plants look rough for
  years.
- Because the floor sits *below* `husbandry.goodAt`, maturity stops
  accruing, so a pot-bound plant also **stalls**.
- `getLimitingFactor()` returns `'root'` and the cause line reads *"It has
  outgrown its pot."* **That line is the entire tutorial for
  transplanting.**

**Unpotted reads `satRoot = 1`.** Not being in a pot is not a *root*
constraint; an unpotted plant is already in trouble via **water**, since
nothing holds moisture for it, and modelling it twice would double-punish.
The mixin gets the volume through one protected host seam,
`rootRoom(): number | null`, defaulting to `null` — `GrowingMixin` never
learns what a pot is.

---

## `GrowthProfileData` — the authored reaction norm

All plain scalars, so it round-trips as a default fields slice with no
marshaller. Authored on the **plant template's `data`**.

| field | meaning |
|---|---|
| `moistureHappyAt` | root-zone fraction at/above which water satisfaction is 1 |
| `moistureWiltAt` | fraction at/below which it is 0 |
| `litresPerGameDay` | transpiration at neutral warmth |
| `luxHappyAt` / `luxDarkAt` | the light ramp's ends |
| `rootDemand` | litres of soil each stage's roots demand |
| `daysToStage` | cumulative well-kept game-days per stage |

> **This migrates onto `Species` in phase 5.** It does not live there today
> because adding fields to `Species` is phase 5's assigned work and four
> docs want to add fields to it independently. One owner, later; the
> migration is expected.

Two species ship with genuinely inverted curves, so "one biology, many
plants" is demonstrated rather than asserted: the **peace lily**
(*Spathiphyllum wallisii*) is thirsty, light-hungry and generous at the
roots; the **snake plant** (*Dracaena trifasciata*) is drought-tolerant,
low-light-tolerant, slower to mature and modest at the roots. The same
watering gap that kills the lily leaves the snake plant alive, and the same
dim corner that stresses the lily suits it.

---

## The fruit cycle — monocarp vs polycarp

The mechanical split is botany's own taxonomy, not a "perennial flag":
**monocarpic** plants fruit once and are done (the carrot — a biennial
pulled in year one — and effectively every annual), **polycarpic** plants
fruit repeatedly (every tree, shrub, vine and herb in the produce
roster). Harvest ends a monocarp — the original shipped rule, now with
its true name and no special path — and a polycarp re-enters the cycle.

**Fruit comes from flowers, and a crop is a pulse, not a drip.** A
continuous yield-accrual model (fruit accumulating like interest) fails
the honesty test — no bloom, no season of readiness. Instead the shipped
flowering latch generalizes into the cycle:

1. **Latch** — mature + thriving latches a flowering episode (the
   shipped rule, unchanged);
2. **Set** — the episode **sets a crop** of authored count instead of
   dropping a seed (`onFloweringLatched` never fires for a polycarp; the
   houseplant's one-seed set is the degenerate case, byte-identical);
3. **Fill** — `_fruitFill` advances by `limiting × dt / (fruitFillDays ×
   DAY)` while the window is open — a starved cycle fills slower, one
   mechanism, no second neglect rule;
4. **Ripe** — fill 1; `isHarvestable` for a polycarp means mature ∧
   alive ∧ **ripe**. A pick mints `fruitSetCount` items, each graded off
   the worst limiting stretch over THIS cycle, draws the full authored
   nitrogen once, and `settleCycle()` re-opens the window — the plant
   survives;
5. **Re-enter or end** — a polycarp's next thriving reconcile latches a
   new episode; a monocarp is done.

**The verdict window re-seeds at the SET, not at harvest**:
`_worstLimiting` returns to 1 the moment the crop sets, so the fruit on
the tree is graded by the keeping that made it — a seedling drought
marks the tree's first crop only through its slower arrival, and last
cycle's neglect never follows the tree. A monocarp never re-seeds: its
one crop is its one life.

The profile's `fruitSetCount` + `fruitFillDays` **are the polycarp
marker** — both > 0, no flag; a profile authoring neither takes no new
branch anywhere. Persistent state: `_fruitFill` (with the shipped
`_flowering`/`_seedSet` reused as the window latch — a mid-fill dip
clears `_flowering` but the SET crop survives it, precisely so the bad
stretch grades the crop). Death zeroes the cycle. Presentation rides the
size line: a filling crop reads the flowering phrase, a ripe one reads
*"heavy with fruit"*.

Supply therefore arrives in **pulses**: with no global season, every
plant's cycle free-runs — a many-tree farm desyncs into smooth aggregate
supply while a single backyard tree gives its owner a real harvest day.

**Refusals name the state**: a mature polycarp between crops refuses
with *nothing ripe on it yet* (`nothing-ripe`), never "still young".
**Harvest accepts the ground**: the spec admits
`GrowingMixin|CultivableMixin`, and a Cultivable target resolves to its
first harvestable occupant, else its first growing one (so the refusal
names the stage), else *nothing is growing* — the `water the pot`
convention; `pick` is an alias on the same spec.

**Dials with homes, shipped OFF** (comment-documented seams on
`GrowthProfileData`, deliberately no fields): thinning (set count vs
fill), alternate bearing (a mast-year modulation), per-stage stress
sensitivities (phase 4), and the over-ripe/drop window (the
preservation-family seam — ripe fruit waits patiently for v1).

---

## Legibility — size, condition, and the cause read separately

`look` carries three lines, appended by a `markupAugmenter` in this order —
**how big it is, how it is doing, and why**. All three are prose and never a
number, and they are deliberately different *kinds* of statement.

```
It is well established.
It looks healthy.
The soil is dry.
```

**Size** is a physical fact, one phrase per growth stage. It is the only
place growth is observable, so it has to reach the player — and **flowering
rides this line** (`It is fully grown, and in flower.`) rather than adding a
fourth kind of statement, because a flowering plant sets a seed into its pot
and that seed would otherwise appear from nowhere.

**The condition band** describes *state*, never cause:

> `thriving` · `healthy` · `stressed` · `failing` · `dead`

**The cause line** names the limiting factor in plain language, and is
omitted when nothing is limiting.

Symptom banded, cause inferable — a player observes a symptom and infers a
reason. That split is what **phase 7's diagnosis surface generalizes**, and
it is why the band vocabulary describes state rather than cause. **Nothing
ever gates on a band** (banding is presentation, never security).

A **dead** plant reports only `It is dead.` — no size worth reporting and no
cause left to infer.

### Growth stages

`seedling → young → established → mature`, advanced by accumulated **good**
time rather than wall-clock age — so a badly-kept plant stalls rather than
growing sickly. Each stage carries a root demand, which is what makes the
pot matter.

**Flowering** latches at `mature` in thriving condition and clears if vigor
falls back. On a fresh latch the plant **sets one seed** into its pot's
contents (a plant is not a container), once per flowering episode. The
player takes it with the shipped `get`: keep it well → it flowers → you get
a seed → pot it → two plants. **Propagation needs no store, no currency and
no new verb.**

**Death** latches on vigor *crossing* below `husbandry.deathAt`
(exponential relaxation never reaches zero on its own, so the crossing is
what counts, checked every sub-step). It sets `lifecycleState: 'dead'`,
makes `reconcileGrowth` a no-op, and cannot be watered back. No
auto-destruct, no respawn — the dorm spine restores captured state rather
than re-running `seedBornWith`, so a dead plant stays dead until the player
discards it.

---

## Calibration — every number is a dial

Under the 12× clock one real day is 12 game days, so **every threshold is
calibrated against the login, not the game-day**. All values are
`AppSetting` keys under `husbandry.*` with seeded literal fallbacks, and all
of them are **placeholders for a running game**.

| dial | ships | governs |
|---|---|---|
| `husbandry.stepSec` | 43200 | integration step (half a game-day) |
| `husbandry.maxSteps` | 2000 | the step cap (never a time cap) |
| `husbandry.vigorTauSec` | 3801600 | the neglect fuse's slope |
| `husbandry.goodAt` | 0.6 | satisfaction at which maturity accrues |
| `husbandry.deathAt` | 0.12 | the terminal floor |
| `husbandry.rootFloor` | 0.4 | how far root-binding can depress condition |
| `husbandry.band.{thriving,healthy,stressed}At` | 0.8 / 0.55 / 0.3 | the band ladder |
| `husbandry.warmth{Factor,ReferenceK}` | 0.03 / 295 | transpiration accelerant |

Behaviour the shipped literals produce, with the peace lily's profile. The
neglect figures are measured from a **`healthy`** plant (the band a fresh one
starts in) and run from a *full* pot, so they include the ~1 real day the
moisture reserve buffers before satisfaction starts to dip at all; a
`thriving` plant takes roughly half again as long at every rung, because the
decay is exponential from wherever it started.

| behaviour | lands at |
|---|---|
| a full pot → `stressed` | ~1.5–2 real days of total neglect |
| → `failing` | ~4 real days |
| → `dead` | ~7 real days |
| `seedling` → `mature`, well kept | ~2 real weeks |

Two consequences worth keeping in view. **A daily player who waters at each
login never leaves `healthy`** — the reserve is sized against the login, not
the game-day. And **one skipped day is recoverable at every rung above
`dead`**: farming's forgiveness contract governs the slope, not the floor.

---

## Durability — growing ⇒ cultivated ⇒ durable

> **A `Plant` owns its own persistence record**, keyed per instance, and
> carries its own location — so it keeps growing wherever you take it, and
> nothing about its durability is wired to dorm code. **Decorative greenery
> is scenery**, an ordinary `Thing` with a description, and needs none of
> this. A cultivated plant left loose in a **transient** room is
> **abandoned**, the same rule chattel.md already applies to owned goods.

There is no "ambient plant" class and no rescue registry: one rule across
all owned things. `shouldPersist()` remains the per-instance hatch for a
wizard's throwaway clone.

**The key is minted lazily, on first demand, never at register** — a keyed
restore stamps the real key *after* register, so minting at register would
race it. `Plant` overrides `getPersistenceKey()` because the capture path is
the only caller that asks, both directly and through an ancestor's
`{ref, key}` entry.

So a pot in a dorm room restores as **room → pot (nested state) → plant (its
own record)**. That required lifting a stated limitation in the spine —
nested hosts were assumed singletons — which is documented in
[persistence.md](./persistence.md) § Keyed nested hosts.

**Capture is event-driven, not periodic, and not at shutdown.** Autosave is
Avatar-only, and `AppBootstrap.shutdown()` persists the world clock and
nothing else. That is survivable *because* reconcile-on-read derives state
from a clock stamp rather than accumulating it: a rolled-back checkpoint
re-derives the elapsed time on the next read, where a tick-based model
would lose it outright. What a rollback **does** lose is the player's
**interventions** — so a mutating act captures its host, through
`PersistableApi.captureHostOf`.

> **For phase 5: pets and livestock are this same shape** — many instances
> of one template, each its own keyed host, carrying its own location. That
> is what the `{ref, key}` work unlocked, and it needs no further spine
> change.

---

## The verb surface

| verb | shape | notes |
|---|---|---|
| `pour <sack> into <pot>` | shipped | filling a pot needs **no new verb** |
| `plant <seed> in <pot>` | new, `inventory` | requires soil + a free slot; mints the seed's named plant, consumes the seed |
| `repot <plant> into <pot>` | new, `inventory` | requires soil + a free slot + `fitsSlot`; preserves every scrap of state |
| `water <plant> [with <src>]` | new, `bulk` | **tool-afforded** (below) |
| `look` | shipped | the band + cause lines ride the augmenter |

No generic "put a `Slottable` into a `Slotted`" verb ships — `wear`, `wield`
and `mount` are each bespoke controllers with their own occupy call — so
`plant` and `repot` are their own controllers too. Both live under
`inventory`, where "put this thing in that thing" already lives.

**`fitsSlot` is the candidate-side gate, and needs no new `SlotSpec`
field.** The *plant* decides, comparing its current stage's root demand
against the pot's soil volume, so a mature plant refuses a thimble.

> ⚠ **`fitsSlot` must not veto a restore.** The sizing rule is a
> *placement* policy — "may an actor put this plant in that pot" — and both
> verbs consult it *before* they move anything. A plant already inside the
> pot's contents always fits it, because re-seating it is the persistence
> restore re-establishing an arrangement that already existed. Refusing
> that would make a **root-bound plant unrestorable**, and root-bound is
> the ordinary designed state the whole transplanting lesson depends on.

### `water` is tool-afforded

`water` is conferred by a watering can **in your inventory**, through
`ToolMixin.getInstanceContributions` over the can row's own capability
entry (the vocabulary is open; the row names the verb):

```yaml
capabilities:
  - { kind: watering, verbs: [platform/cmd/bulk/water.yaml], placement: carried }
```

`placement: 'carried'` is the whetstone's personal-capital rule as data —
the verb appears only while you hold a can, and one on the floor confers
nothing. This is the standing "instruments confer working verbs" rule
getting its **first non-crafting consumer**; `pour` remains the manual path.

`WaterController` is `DrinkController` with the plant's moisture reserve in
place of `ingest`, and two deliberate differences: it passes an **explicit
measure** sized to the plant's own headroom rather than `{ kind: 'all' }`
(so a 2 L can does not vanish into a teacup of soil), and a `PlantPot`
target **resolves to its occupant**, because a player will type `water the
pot` as readily as `water the lily`.

---

## Content

- **Acquisition.** Every dorm room is seeded with a soil-filled starter pot
  holding a peace lily on the desk, a watering can and a corner tap — all
  `props:` data. Nobody has to shop before meeting the growth model.
  The starter pot's 0.5 L carries a lily to `young` and then binds, so the
  free plant is also what teaches transplanting.
- **Commerce.** The Terminus general store stocks a small pot, a large pot,
  a sack of potting soil and a snake-plant seed — **content only**, since
  `itemTemplatePath` takes any path and the pots and seed are stocked
  straight from their `/platform/… + /stuff/` templates rather than duplicated under
  `goods/`. Only the sack is new. **The large pot is the first thing a
  player has a reason to buy**, and the reason is legible before the
  purchase.
- **Light.** `AmbientLitMixin` shipped, but no location class composed it
  and no seed authored `ambientIntensity`, so every room read pitch-black.
  It now composes on `Location` — inert by default, since the perception
  walk skips a zero-flux ambient — and Duncan Hall carries the tree's first
  authored values: lobby `bright`, dorm room `lit`, corridor `dim`, front
  steps outdoor daylight, cistern unauthored (dark).

> ⚠ **A pot must retain its soil.** `closure: open` does not mean "has a
> drainage hole" — an open interior does not **retain** bulk at all, and a
> transfer into it drains straight through to the floor. Pots and the soil
> sack use the default `liquidTight`.

> ⚠ **The footlocker is not a dark place.** A `Footlocker` is a plain
> `Vessel` with no lid to close, so the light sample climbs past it and it
> reads exactly as bright as the room. **Room-to-room placement carries the
> light axis** (the dim corridor is the shipped darker place); a closed
> `Sealable` container would also work, and the model supports it.

---

## Advancement — `horticulture`, and how it earns

Two Discipline rows ship, opening the Catalog's **first agricultural branch**
(39 shipped before them and none were agricultural):

| key | channel | ISCED-F | role |
|---|---|---|---|
| `agriculture` | knowledge | `081` | the narrow-field spine node; nothing practises it |
| `horticulture` | skill | `0812` | the practised leaf — `specializes: [agriculture]` |

`agriculture` is minted ahead of its siblings on purpose: it is the home
farming's crop rows, ranching's animal husbandry and stewardship's land care
will all need, and giving them a parent that already exists is cheaper than
re-parenting later. (Re-parenting is safe regardless — edges join on `key`,
never on templatePath.) The broad-field parent (ISCED-F `08`) stays absent,
per advancement.md's deferred-spine rule and `medicine`'s precedent.

**`horticulture` is the plant side only.** Animals are a different practice
with a different anchor, so phase 5's pets and livestock get their own leaf
under the same parent rather than overloading this one — which is exactly what
naming it `horticulture` instead of `husbandry` preserves.

### Evidence is outcome-shaped, not an action count

All three husbandry verbs mint a `deed`, and **difficulty is read off the
world** at the moment of the act — never tagged on the verb:

| act | difficulty | from |
|---|---|---|
| `plant` | `trivial` | pressing a seed into soil is the entry act |
| `water` | `trivial` → `hard` | `Plant.careDifficulty()` — the condition band, so a top-up is trivial and a rescue is hard |
| `repot` | `trivial` → `hard` | `Growing.transplantDifficulty()` — the growth stage, since root disturbance scales with what there is to disturb |

Two properties make this a practice rather than an odometer:

- **A refused act earns nothing.** `water` credits only when the soil actually
  had headroom — going through the motions on a wet plant is rejected before
  any deed is minted — and a planting into a soil-less pot credits nothing.
- **The easy end cannot be farmed, by the estimator's own maths.** A trivial
  success is *unsurprising* and barely moves the BKT estimate, and the
  transit rate is an inverted-U peaking at `hard`. So spamming waterings is a
  levelling-mill the model already forbids; there is no bespoke guard here
  because none is needed.

> ⚠ **One tension worth naming.** Because difficulty tracks how far gone the
> plant is, the `hard` grade is only reachable by letting one get to
> `failing` — so the model does, in principle, reward brinkmanship. It is
> left standing because the cost is real (days of game-time, and the plant may
> simply die, which credits nothing and is terminal) and because the
> alternative — grading by *how hard the judgment was* — is not measurable
> until phase 7's diagnosis surface exists. Revisit it there.

### No conferrals, deliberately

`horticulture` confers no verbs. The natural knowing→doing seam for growing
things is **diagnosis** — reading a plant closely enough to tell a thirsty one
from a root-bound one, with instruments and error bars behind it — and that is
phase 7's build. A conferral now would mean inventing the content the band is
supposed to open.

The row is still live rather than decorative: it accrues real evidence today,
and the shipped `competence` verb reports its band.

---

## Deferred seams — attach points, not stubs

- **`_vigor` is the condition score.** Phase 5 gives it a general home
  across livestock and pets; **phase 6 reads it as the disease-resistance
  factor — good husbandry is immunity.** Do not generalize it here.
- **`Organism.age`** is stamped as time passes but nothing drives lifecycle
  from it. The general maturation driver is phase 5's.
- **The growth profile → `Species`** (above). Phase 5 owns that schema.
- **No `PathogenBehavior`, no growth term, no `ContagionSpec`.** Phase 3/6.
- **No soil *quality*.** Soil is a volume and a prerequisite — no N/P/K, no
  pH, no tilth, no fertilizer. The six-reserve soil is phase 2/4.
- **No genetics.** A seed grows into its parent's species, full stop, and
  carries no inherited variation. `Globbable` seed *lots* arrive with
  genetics, where fixed-vs-segregating lines make stacking meaningful.
- **The switchable desk lamp.** `obj/Lamp.ts` and `device/switch.yaml` both
  ship, so a *player-controlled* in-room light lever is cheap — but wiring one
  means dimming a room every player lives in, a blast radius phase 1 declined.
  The light axis is exercisable without it (room-to-room placement), so this
  is an ergonomics upgrade rather than a gap.
- **No sun→light driver.** Room ambient light is authored data here;
  deriving it from `CelestialApi` is phase 4's, and the front steps' static
  value is a placeholder awaiting it.
- **No general autosave mixin.** `Avatar.startAutoSave` is still the only
  periodic capture in the tree; `captureHostOf` after a mutating act covers
  this build, and the general version should be designed against several
  hosts.
- **Persistence for owned *ground*.** A plant is a movable host and solves
  its own case. A garden bed or a Warren-budded field-room is a **room** —
  the `DormWarren` keyed-holder pattern — which phase 2 and phase 4 pay for.
- **The finite-but-regenerating bulk source.** The tap is infinite, as
  `lib/bulk/UnboundedSource.ts`'s own docstring anticipates.
- **No competence GATE, no instruments, no error bars.** Science is the skill
  ceiling, not the entry fee; v1 is the entry. The `horticulture` Discipline
  ships and accrues evidence (below), but nothing reads its band yet.
- **No bond, no attention need, no regard.** A plant cannot hold an opinion
  of you — the family's stated asymmetry, and
  pets' whole divergence.
- **No automation ladder.** No hired hand, no self-watering pot. Needs the
  employment engine and a production brain (phase 8).

---

## Cross-references

[persistence.md](./persistence.md) · [residence.md](./residence.md) ·
[slot.md](./slot.md) · [bulk.md](./bulk.md) · [light.md](./light.md) ·
[race.md](./race.md) · [crafting.md](./crafting.md) ·
[retail.md](./retail.md) · [time.md](./time.md) ·
[quantities.md](./quantities.md) ·
[command-spec.md](./command-spec.md) ·
farming-slate ·
ranching-slate ·
living-world-roadmap
