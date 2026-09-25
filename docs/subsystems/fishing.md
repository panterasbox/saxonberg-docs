# Fishing — every reach holds what belongs in it, unasked

The fishery as a **derived record on a reach**, the water pack's `Shore`,
and the `trade-fishing` capability pack: the tackle, the wait and the
bite and the landing contest, the traps, bait, a fish you can keep in a
bowl, six aquatic species, the fisher and the fishmonger. This doc is the
source of truth for `packages/content/water/src/idea/FisheryRegistry.ts`,
`packages/content/water/src/thing/Shore.ts`, `WatercourseCatalogue.
waterStateAt`, `Species.habitat` / `fitIn`, and everything under
`packages/content/trade-fishing/`. Built 2026-09-18 (`build/fishing`);
the plan retires at the sweep and its decisions live here.

**Read first:** [watershed.md](./watershed.md) (the reach, the flow, the
contamination this reads), [pets.md](./pets.md) (the kept fish is a kept
animal), [ranching.md](./ranching.md) (the record-not-object precedent:
*a herd MOVES; a stand does not* — and fish move), [spoilage.md](./spoilage.md)
(a landed fish dies, turns, and carries what the water carried),
[retail.md](./retail.md) (the stall is consignment, no new verb). The
design: `docs/slates/tails/fishing-slate.md` (§ 12 is the home tank).

---

## ⭐⭐ The record is DERIVED, and it is the water pack's

Nobody writes a table for the confluence or for the moor's mere. Each
`Species` row that authors a **habitat** — tolerances over the closed
water vocabulary, a role in the food web, an abundance per kilometre, a
fight rating — is asked how well it fits the water the catalogue reports
at a reach, and the answer is the population:

```
capacity = round(fit × abundance × water.fishery.reachLengthKm)   (seed 1 km)
level    = capacity − drawn                                        (the only state)
```

`FisheryRegistry` (`RegistrarMixin(Idea)` at `/system/water/idea/FisheryRegistry`,
register prefix `/system/water/fisheries`, the kernel `fishery` document
kind, `onVanish: keep`) holds **only the draw**: per species, how many
were taken and not yet recovered, and when that was reconciled. Recovery
is reconcile-on-read — `drawn` decays with a half-life
(`water.fishery.recoveryHalfLifeDays`, seed 2) — and is **written only on
a draw or a release**, never on a `look`. A reach nobody has fished has
no document at all, which is what makes *every reach holds fish* true at
zero cost. Reads verify prefix and kind (the herdbook's rule).

**Why the water pack.** It reads the catalogue, which the kernel cannot
import; two trades (fishing now, hunting later) will read it with no
common ancestor below `water`; and *a system is true whether or not
anyone participates*. The trade reads it **by path, duck-typed**
(`trade-fishing/src/lib/FisheryRead.ts` is the shape it meets it over);
the water pack imports nothing of the trade. The one word of a trade the
water pack speaks is a **setting**: `water.fishery.readDiscipline` names
the Discipline whose band gates the species read at a shore.

**`stocks:` on a Watercourse node** is the aquaculture seam: a stocked
species' capacity is the authored number regardless of fit, and the read
says *the water is stocked with…*. No shipped row authors one;
`lint:census` walks `nodes[].stocks[].species`.

## ⭐ One law for every RGO — Liebig's minimum, and it names the limiter

`Species.fitIn(state, season?)`: each authored tolerance is a factor
(`1` inside `[min, max]`, linear to `0` across `margin` beyond either
bound — a toxin is a `max` with a margin, there is no second shape), the
season is `0/1`, and **the fit is the minimum**, with the parameter at
the minimum returned as `limiting`. An unauthored parameter is factor
`1` (soil's *unmodelled is not zero*); a `null` habitat is fit `0`.
Interactions live in the derivation of the **state** (warm water holds
less oxygen), never in the combining rule.

The minimum was chosen over a product and a mean on the ground every RGO
shares: it is the law husbandry already takes (`satWater/satLight/
satRoot`), it always **names** the limiter — *too warm for trout this
month* is the whole pedagogy of the field and of the tank — stacked mild
stresses do not compound to death, fixing the limiter always moves the
number, and the mine's binary breathability is the same rule at `0/1`.
Independent mortality *events* (disease, predation) are not fit.

## ⭐ The reach reports every parameter a tank will ever hold

`WATER_PARAMETERS` (`Species.ts`, kernel, closed): `temperatureK ·
currentMps · salinityPpt · oxygenMgL · pH · hardnessDgh · nitrateMgL ·
ammoniaMgL · nitriteMgL · contamination`. `WatercourseCatalogue.
waterStateAt(reach, nowS)` reports every word:

| parameter | derivation | the seam it leaves |
|---|---|---|
| `temperatureK` | the catchment's air, floored at 274 K (water under ice) | the underwater slate's depth |
| `currentMps` | `flow / (width × depth)`; an unauthored width or depth from the flow by hydraulic geometry (`w = 5√Q`, `d = 0.4·Q^0.4`); `meanDepthM` authorable on the node | — |
| `salinityPpt` | 33 at sea level, 15 one hop above it below 50 m, 0.3 fresh (`water.fishery.salinity.*`) | the tide clock replaces this one function |
| `oxygenMgL` | saturation at temperature (14.6 @ 273 K → 7.5 @ 303 K) × turbulence by current band | — |
| `pH` · `hardnessDgh` · `nitrateMgL` | **seeded on the course row** (`water: { pH, hardnessDgh, nitrateMgL }`), every node inherits, **flow-weighted at every join** through a predecessors map | geology → hardness; soil leaching → nitrate (the RGO unification pass) |
| `ammoniaMgL` · `nitriteMgL` | 0 in a flowing river | the home tank's ledger |
| `contamination` | the shipped `contaminationAt(…).level` | — |

Every fish authors `ammoniaMgL: { max: 0.02, margin: 0.5 }` and
`nitriteMgL: { max: 0.1, margin: 1 }` — the tank's numbers, inert in a
river — so the aquarium build reads the same tolerance a reach does and
never reopens a species row. The fault lines are named, not unified: a
field's medium is *seeded* (`GroundCharacter`), a reach's is *derived*,
a tank's will be *made*.

The shipped stances: the Kestrel 7.8 / 12 / 2 (hard, alkaline, a little
fed), the Holloway 6.0 / 3 / 0.5 (the moor's peat), the Delight 7.4 / 9 / 8
(the farming valley, the richest), Cold Fell 6.8 / 5 / 0.5. Read against
the six species' habitats the confluence holds crab, mullet, eel, a few
carp and the sturgeon; the Holloway's head holds trout and nothing else;
the Delight's flats hold trout and carp. **No table anywhere.**

## The shore, and the read

`/system/water/thing/Shore` — a room-fixed feature (`fixedInPlace`)
citing a `reachRef`, the water pack's, by the `/system/` test: a
riverbank is there whether or not anyone fishes. Rows: the wharfside
`river-edge` (`kestrel:confluence` — the city's Locality declares no
reach, so this row is what makes the bank fishable) and the moor's
`heath-mere` (`holloway:head`; the moor's Locality cites the vale, the
feature wins). A verb that wants a reach takes a Shore as its declared
argument (`default: "reachable:[class.Shore]"`) and falls back to the
Locality's reach when none is bound — how Heart's Delight's millsite
fishes the Delight's flats with no row and no code.

`look <shore>` (⚠ by the shore's own keyword — `edge`, `tarn`; `water`
and `mere` are room details too and prompt): the **physical read** for
everyone (the water's name, breadth in words, still/slow/fast,
fresh/brackish/salt, cold/cool/warm, soft/hard, *an outfall discharges
into this water* as a fact about the map), then the **fishery read** at
the viewer's band in the read Discipline — nothing untrained, presence
at novice, the species in bands at competent (*plenty of eel, a few
carp*, the apex *something large*), the apex named *a royal fish* and
**the one factor that limits each species the water does not suit** at
proficient. The band words read `level / full` (the capacity at a
perfect fit), so a species the water barely suits reads *a few* and the
limiter line says why. Never a number. The shore keeps a memo refreshed
fire-and-forget (the `GristMill` shape); `settle()` awaits it.

⚠ The room's own `look` does not carry the read — a prop contributes
nothing to its room's prose (only the floor puddle has a kernel hook). A
room-level contribution hook is a finding for the sweep.

## The trade

`packages/content/trade-fishing` (root `/trade/fishing`, depends on
`water`; ⚠ a pack must be added to the **root `package.json`** — the
deployment manifest — or `PackLogic` never discovers it).

**The instrument affords the verb**, as class statics: `Rod` (`fish` /
`reel` / `slack`, `environment` = whoever holds it), `Trap` (`lay` /
`haul`, both axes; the pot and the net are rows of one class — numbers,
never a `trapKind`), `Fish` (`release`). ⚠ `haul`, not `lift`: `lift`
is the barbell's (`device/lift`, the nutrition build) and two views on
one verb shadow each other silently — `lint:verb-collisions` caught it
at the catch-up merge. ⚠ `lay`, not
`set`: `set` is a **scripting builtin** (`lib/script/builtins.ts`) and
the interpreter takes the line before dispatch; `lint:verb-collisions`
now counts the builtins as claims.

**The wait, the bite, the landing** (`lib/FishingEngagement.ts`): a
sustained engagement on the angler's hands, hosted by the rod, one tick
a game minute. Each tick accumulates **pressure** — deterministically —
from every species' `level/capacity` × twilight (the sun within ±10°) ×
weather (rain or a storm) × the bait's match to the species' role (a
worm for forage and bait, a baitfish for a predator, a bare hook 0.15,
crumbs nothing) × ⭐ **where the rig presents against where the species
feeds** (B8: `Rod.presentsAt` vs `Habitat.feedsAt` — the same layer 1,
one over 0.5, across the column 0.1; a species that authors no layer
feeds anywhere) × the rod's `showing`. At 1, **the one draw** — which
species, a seeded unit over the terms; epistemic, never resolutional.
The individual's length is seeded around the species' stature; ⭐ **the
hook selects** — a fish shorter than 8 × `Rod.hookGapeM` nibbles and is
gone, a tick that prints nothing, the bait still on. `fight =
fightRating × length/stature`. Under `fishing.contest.fighterAt` the fish
lands itself; otherwise a `LandingContest` opens. Landed, the fish is
minted alive into the hand (a body in air — it starts drowning), drawn
from the record, stamped with `e-coli` at `contamination.organic ×
fishing.contamination.loadPerUnit`, and credited to the Discipline; an
apex is a chronicle deed. **The wait ends at a landing.** A tick with no
bite prints nothing. An NPC angler releases at once and speaks one line.

**The contest** (`lib/LandingContest.ts`) is pure arithmetic over `line ·
strain · stamina`: `reel` takes line in and loads the strain by the
fish's remaining fight, `slack` unloads it and gives line, each tick the
fish tires by the load and pulls by its freshness, a line slack two
ticks running is thrown, the line parts at `breakStrain`, the fish lands
when the line is in and it is tired. Dials seeded so **two reels inside
one tick snap a full fighter** and *give when it runs, gain when it
rests* lands one in 5–8 ticks; the reel/slack prose says which it is
doing (*the line is singing · the rod is bent hard · it is resting*).

**The rig is numbers on the rod, coupled by the rows** (D26). Three
rows of one class: the *cane rod* (free line, `mid`, ordinary hook, the
default line), the *float rod* (`surface`, `showing 1.3`, `breakStrain
0.4`, a 6 mm hook — the shoal's rig, not the sturgeon's) and the
*leger rod* (`bottom`, `showing 0.8`, `breakStrain 0.75`, a 20 mm hook
— the eel, the carp, the only rod that holds a sturgeon; it takes
nothing small). Heavy line holds the big one and shows the bait worse;
fine line bites more and parts. You choose it for what you are fishing
FOR — the read tells you what is there and where, and the rod is how you
act on the sentence. ⚠ Swivels, hook numbers, poundage and a separate
line object are deliberately NOT things: an item with no decision behind
it is inventory clutter, and the line is the rod row's `breakStrain`.

**What goes on the hook is read from the thing itself**
(`FishingEngagement.baitKind()`): a `Bait` row says (`worm` · `baitfish`
· `crumbs` · `lure`); a landed `Fish` ≤ 0.25 m is a baitfish; anything
whose material carries the `bread` tag is crumbs; anything else is
nothing. **A lure is worked, not eaten** — the tin spoon draws predators
only while the angler has `reel`ed within two ticks (`work()`; a spoon
left to lie is a stone), and survives the take: one spoon, all season.
Lure fishing is active where bait fishing is patient.

**Traps** reconcile at the haul: `expected = Σ min(level, drawPerHour ×
hours × level/capacity)` over the roles the trap takes, capped by what
it holds, the fraction one seeded unit; a laid trap is fixed in place
and vetoes eviction. A net (30, 20/h) empties the confluence's ~160 in
five afternoons' lifts; a pot (2, 0.4/h) takes crabs overnight.
⭐ **A trap is a container** (`BulkableMixin(ContainerMixin(ToolItem))`),
and a laid one with an interior is **full of the water it lies in**
(`lay` fills it from `/stuff/idea/material/bulk/water`, `haul` drains it
and hands over whatever it held). That is the whole of the **keepnet**
— a Trap row with `drawPerHour: 0`, `capacity: 12`, `interiorBulk:
true`: `lay keepnet`, `put trout in keepnet`, and the trout breathes
(respiration reads the vessel a body is inside — the bowl's read, at the
bank). Keep or let go is the angler's now, not the clock's. No new
class, no guard: a pot with a crab put back in it hands the crab back
too.

**Bait is bought or caught, never dug.** Digging a worm out of a bed is
foraging's act, not fishing's; a `dig` verb shipped in the build and was
withdrawn in review (the design — the ground's own reserve as the
cooldown — went to the discovery slate).

**The kept fish.** `Fish = ContaminableMixin(KeptAnimal)` (every fish can
carry a load; nothing else kept can — the `Weapon` lesson), `lengthM`,
`sizeWords()` in bands, a *turned* line on a dead one. The bowl is a
`Feeder` row with a 4 L interior: a carp in it **breathes the water it
holds** (`Respiration` walks the containment chain for the nearest
bulk interior — immersion; a vat of ale is a liquid whether or not the
biome table names it), eats the crumbs put in it (`feeds` looks at the
vessel the animal is *inside*), and **earns its home there** — three
distinct fed days move `home` and stamp `homeEarnedDay`; `Bonded.
hasChosen()` is the naming gate (bond + followed **or** earned home and
there now), and it widens the cat's gate honestly: a stray fed at one
door three days running has chosen it. A stamped chattel is a home by
its chattel id (`homeKeyOf`), never its template path. ⭐ `peers`
affordances reach **one level into an open container standing in the
room** (`CommandLogic.applyContainmentDeltaImpl`), as the `peers` scope
already did — or a carp in a bowl could be named and afforded nothing
to name it with.

**The stall** is consignment: a `Stock` with no lines and
`staffingPolicy: self-service`; `consign <fish> --ask N`, a buyer's `buy`
splits the ask less the market's commission, `reclaim`. The one kernel
change: **`consign` refuses what has turned** — a `Provision` past
`fresh`, or a dead body whose `Postmortem.freshnessLoad()` (the
butchering arithmetic, answered by the carcass) is past `fresh`. A rule
of every shelf. The fishmonger keeps the stall on the market's roster
(committee-appointed, on a shift with an `Offstage`), replaceable through
`appoint`; the fisher fishes the confluence bare-hooked on his own
cadence and **reads the water aloud from the record** (`reads-water`, an
`engage` responder building a one-beat tree from `readFor` at the
practised band — his lines keyed *empty / thin / holds / apex*, `{{read}}`
filled by the registry; the record holds no names, so neither does he).
⚠ Once he has introduced himself his keywords are his NAME (`talk tull`).

`butcher` hands a carcass's own load onto every cut
(`transferContaminationTo`, any Contaminable body — today only a fish);
fish flesh is tagged `meat`, so the shipped cure / smoke / dry recipes
take a fillet unchanged.

## What the drive found

`packages/wire/tests/fishing.dirty.wire.test.ts` — 21 steps, green on
the merged tree (run 24, 2026-09-22; the run-by-run record lived on the
plan and is summarised on MR !268). Pre-existing content it caught: the
city's intake and outfall were **off** (`Switchable` defaults off; the
Kestrel below Terminus had never been fouled), and the baker's, miller's
and farmer's `introduces` had an unknown trigger (`witness:arrival`) and
never fired.

**Kernel defects the drive surfaced, fixed on the branch** — each in its
own doc: `set` is a scripting builtin and a verb named after one never
dispatches (`lint:verb-collisions` counts the builtins now); `buy` never
`followCustody()`'d, so a bought good vanished at restart
([retail.md](./retail.md)); `peers` affordances did not reach into an open
container on the floor ([command-routing.md](./command-routing.md));
releasing a fish mid-drain was an unhandled rejection that took the server
down ([respiration.md](./respiration.md)); ⭐⭐ the MQL keyword scorer was
INERT behind every scope seed — a bare word resolved by pool order
([mql.md](./mql.md)); ⭐⭐ the dying clock froze when unobserved
([mortality.md](./mortality.md)); wire test characters were minted naked
(testing.md). **Recorded, not fixed here:** every unfed
Cast in the world starves at room temperature — the thermal dials
(thermal-slate); a keyed pet inside a
moved vessel restores loose, and a dropped chattel restores to inventory
([pets.md](./pets.md), Risk 7 — so *the carp is in its bowl after a
restart* is the one acceptance line not met).

**The LIVE drive (a browser, 2026-09-22).** The wire drive asserts the
dispatch ENVELOPE; a browser shows what a player is shown. Driven in
Chromium against `pnpm dev` (fresh DB, `AUTH_MODE=test`): buy the whole
rig at the general store, walk south to the bank, Ambrose Tull
introduces himself, `look edge`, eat, cast, play and land an eel, lay
the keepnet, put the eel in it, haul it, release it, talk to the fisher.
It found three things wire could not:

1. ⭐ **A bare shelf refused as *"float-rod" isn't for sale here*** while
   the counter's own description lists the line — every new tackle row is
   `par: 1`, so the second run of any drive hits it. Fixed where it was
   found: `Stock.carriesLine()` + a `sold-out` refusal
   ([retail.md](./retail.md)). ⚠ The wire drive resets the database every
   run, so its shelf is never bare — this class is invisible to it.
2. ⚠ **`look edge` prints nothing in the transcript**: the read arrives
   as a card, the card column does not scroll to it, and with two older
   cards above it the card the player just asked for sits **below the
   fold** (measured: card top 794 px, viewport 720 px, the column at
   `scrollTop: 0`). The build's centrepiece read is one silent scroll
   away. The card surface's, not fishing's — recorded on
   [card-surface.md](./card-surface.md).
3. ⚠ **`look cane` prompts with two buttons both labelled *a cane rod***
   (the shelf's `par: 2`) — a choice that cannot be made. The
   `distinguishing` form exists for exactly this
   ([presentation.md](./presentation.md)); the prompt does not use it.

Also seen, already recorded: **"You're shivering."** greets a player on
arrival (the thermal dials — `thermal-slate`), and a `get` of coins in a
bank till surfaces a raw internal error with a module path
(*"Something went wrong in /platform/idea/cmd/inventory/GetController:
canRemoveContainable veto…"*).

**Acceptance observed live, and not.** Eleven of the requirements'
thirteen lines were observed by the drive. Not observed live: the carp's
bowl after a restart (above); *eaten raw it sickens* and *a fillet keeps
when smoked* — `butcher` is afforded only by the cookhouse block, so the
carcass-load transfer is a unit test; the fishmonger's replacement through
`appoint` and the sturgeon's *release* deed are unit-tested, not driven.

## Deferred

The home tank (fishing-slate § 12) · the tide (`salinityPpt`'s one
function) · geology → hardness and leaching → nitrate (the unification
pass) · the food web (`Habitat.role`) · the covenant reading
`ToolMixin.epoch` and `Trap.takesRoles` · the named apex · the barge, the
underwater regime, the spear · fish-specific recipes, skin and bone ·
the fisher's creel · a river you can fill a bowl from · a room-level
prose contribution for a prop.
