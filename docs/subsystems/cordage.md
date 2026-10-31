# Cordage — one rope, and where its ends are tied

A rope is ONE kernel thing (`/platform/thing/Rope`), and what it is FOR is
where its two ends are: a lifeline is a rope a tender holds with a diver on
the end, a heaving line a rope thrown to a swimmer, a painter a rope on a
boat's bow, and a rope tied round a post at the lip of a pit is a way down.
The owner's two rules: *one rope, its use is where its ends are tied* and
*a rope and a hose are different things; if they share a superclass that's
fine*. Making rope is the `trade-cordage` pack's (`/trade/cordage`): the
ropewalk's jack and the one verb `twist`.

> ⭐ **Every number derives; none reaches a player.** Strength goes as the
> square of the diameter and as the fiber's toughness, a knot keeps its
> row's share of it as the tier dressed it, chafe and rot take off it — and
> what a player reads is *it would hold a man and his load*.

## The classes

| class | where | what |
|---|---|---|
| `Line` | `lib/spatial/` (substrate) | a length with two ends: `throw … to` (heave), `tug`, `pull`; mints its free end from `static END_ROW`; the haul's lift asks the `Buoy` protocol |
| `Rope` | `platform/thing/` | `Line` + Tieable, Spliceable, Climbable; length, diameter, stage, knots, chafe, rot, marks, coil, anchor, hang |
| `LineEnd` | `platform/thing/` | the free end in somebody's hands (`TetherEnd`, `Tieable`); a hose's end |
| `RopeEnd` | `platform/thing/` | a rope's end — climbable while the rope hangs (the way UP) |
| `Hose` | `trade-diving` | the other `Line`; its end is *end of an air hose* |
| `BelayEngagement` | `lib/locomotion/` | the belayer's hands + attention, holding a rope for a climber |
| `RopeJack` | `trade-cordage` | the ropewalk jack (`ropelaying`); plans `twist` |
| `LayReading` | `trade-cordage` | `analyze lay <rope>` |

Protocols (kernel, `lib/spatial/`): **`Tieable`** (`tieTo(by, load, knot)` /
`untieBy(by)` — a rope, a line's end, diving's air bag) and **`Spliceable`**
(`spliceBy(by, other | null)` — a rope, energy's `LineAccess` pole).

## The verbs

| verb | view | the thing that decides |
|---|---|---|
| `tie` (`lash`) `<gear> to <load> [with <knot>]` | `platform/cmd/inventory/tie.yaml` | `Tieable.tieTo` |
| `untie <thing>` | `platform/cmd/inventory/untie.yaml` | `Tieable.untieBy` |
| `tug <end>` | `platform/cmd/inventory/tug.yaml` | `LineEnd.tug` |
| `cut <rope> at <length>` | `platform/cmd/crafting/cut.yaml` | `Rope.cutPlan` (a `cutting` tool) |
| `splice <rope> to <rope>` / `splice` at a pole | `platform/cmd/crafting/splice.yaml` | `Spliceable.spliceBy` |
| `coil <rope>` | `platform/cmd/inventory/coil.yaml` | `Rope.coilPlan` |
| `climb down` / `climb up` | `platform/cmd/movement/climb.yaml` | the enablement walk finds the rope / its end |
| `belay <person> [with <rope>]`, `belay off` | `platform/cmd/movement/belay.yaml` | `Rope.belayFor` → `BelayEngagement` |
| `twist <yarn | strands> [--yarns n] [--length m] [--mark c]` | `trade/cordage/cmd/cordage/twist.yaml` | `RopeJack.twistPlan` |
| `turn jack` / `turn back jack` | the kernel `turn` (`TurnableMixin`) | `RopeJack.turnBy` → `WheelEngagement` |
| `dip <rope> in <tar tub>` | the kernel `dip` | `GlazeTub` lays a pitch coat |
| `analyze lay <rope>` | the kernel `analyze` | `LayReading` |

## The numbers (grain — dials in `settings/cordage.yaml`)

- **Mass**: `ρ · 0.6 · π(d/2)² · length`, × 1.15 tarred; rows author none.
- **Strength**: `toughness × cordage.tenacityMPaPerToughness (25)` MPa ×
  `π(d/2)²` mm² ÷ 9.81 → kg. Hemp (4) > flax (3) > jute (2); 12 mm hemp ≈
  1.15 t. Effective = breaking × the weakest knot (`strength × dressing`) ×
  (1 − chafe) × (1 − rot). Words: a child · a man · a man and his load · a
  horse · an ox and more.
- **Knots** are platform `Joint` rows (`bowline` .6, `clove-hitch` .6,
  `sheet-bend` .5, `reef-knot` .45, `splice` .9 and never untied). Dressing
  is the tier's `ropework` band at tie-time (untrained/novice .8, competent
  .9, proficient+ 1.0) and is credited. The reef knot fails as **`spilled`**
  (added to `JOINT_FAILURES`), the clove hitch as `slack`.
- **Twist**: tex `1653.5 / count`; a strand of `n` yarns `n·tex·1.1` g/km,
  a rope Σ strands × 1.15; `d = 2√(λ/πρφ)`; a skein is `count × 274 m/lb`.
  Never longer than the walk (the jack's room's `getLinearExtent`).
- ⭐ **Laying is a craft, real time, and a crew** (the review's lens pass):
  - *the craft* — the layer's `ropework` band sets the lay's evenness
    (untrained .8 · novice .85 · competent .92 · proficient .97 · expert 1),
    multiplying strength; a rope is as even as its worst strand and its
    layer; read as *hockles where the twist bunched up* … *hard and true*;
  - *the yarn* — the weakest yarn sets the strand's grade, the weakest
    strand the rope's; grade scales strength (poor .85 · fair 1 · fine 1.05
    · exceptional 1.08 · masterful 1.1); cut and splice carry both;
  - *the time* — a strand is length × (20 s + 0.5 s per yarn), a rope 30 s a
    meter; a full walk is hours;
  - *the crew* — a hand at the wheel (`turn jack`, a `WheelEngagement` on
    their hands) and the layer walks at full pace; alone, half. The hand is
    credited `ropework`.
- **Rot**: `cordage.rotPerWetDay (0.03) × wetness` per game day, untarred.
  Tarred = a SET coat of a `waterproofing` material outermost on the
  outside face (`Faced.coolCoats`: pitch sets as it cools below 320 K).
- **No buy-back** (yet): a house bids per row, so a coil cut into five
  would sell as five coils; the ropewalk buys no rope until retail builds
  per-instance worth (retail slate).

## The rope at a drop

`tie rope to <fixture>` in a room with a `down` exit whose media include
`vertical` hangs the rope (≥ `cordage.minHangM`): its `RopeEnd` is minted
into the bottom room. The rope's axes are `['down']`, its end's `['up']`, so
the shipped enablement walk finds each in its own room. After every climb on
it the host hears `Enablement.afterEnabledTraverse` (the ok guard carries
the host from `canTraverseExit` to `Mobile.traverse`):

- a reef knot with a climber carrying ≥ `cordage.unevenLoadKg` **spills** —
  the whole rope comes away and goes down;
- a load past the effective strength **parts** it — a 1 m stub stays tied,
  the rest falls;
- otherwise it **chafes** `cordage.chafePerUse × load / breaking`, read as
  *scuffed · fraying · a few yarns parted · about to go*.

A fall asks the climber's belays first: a belay catches when `(mass +
burden) × cordage.fallFactor` ≤ the belayer's hold — `strain ceiling ×
e^(0.3·2π·turns)` (three turns round a fixed thing in their room, else
one), never past the belay rope — and the climber is hauled into the
belayer's room unhurt. A clove hitch round the climber is not held. Else
the climber lands at the bottom, blunt at `hazard.dropFallEnergy`.

⚠ The hung end is runtime: after a restart the rope is still the way down,
but its end is re-minted only by untying and tying again.

## The places

- **The Wharfside ropewalk** (`/world/terminus/wharfside/ropewalk/`), off the
  towpath by a `ropewalk` exit: a 300 m walk, the jack, a tar tub of pitch
  (`GlazeTub`), the ropekeeper Agnes Tarrow and her counter (50 m coils,
  hemp line, hemp yarn, rigging knives, painters; she buys no rope back).
  ⭐ She takes the jack's wheel for whoever lays (the `turns` brain), and
  the ropewalk has an open **wheel-hand** seat (headcount 1) a player can
  `apply` for. The walk is open to the sky (a room's light is spread over
  its extent squared) and addressed in Terminus.
- **The Sunken Delve**: a timber post at corridor-2's lip; the pit is climbed.
- **Hemp** (`trade-farming`): *Cannabis sativa*, one species; the fiber
  variety is the plant row (`plant/hemp`); its seed is on the general store.
  The textile chain rets it (`retting-hemp`), scutches it into *hemp line*
  (`TextileStock.adoptFibre`) and spins it.

## The rogue's yarn

`twist … --mark red` lays a colored yarn in a strand's heart; every rope laid
from it carries it, a cut copies it, a splice shows both. `look` shows it
only at a cut end; `analyze lay rope` opens the lay (competent+, or any cut
end).

## Limits and seams

- Anchors never fail; drop depth vs rope length is a dial, not geometry.
- Rot is its own reconcile, not `Kinetics` (no moisture driver there yet) —
  the seam is a second driver on the kinetic rate law.
- Anchor strength, shock elasticity, block and tackle, oakum, who answers for
  a parted rope: see the cordage slate.

## History

- **2026-10-30 — the cordage build** (`reqs/cordage`, W0–W7): `Lifeline`
  became the kernel `Line`/`Rope`, diving kept `Hose`; `tie`/`untie`/`tug`/
  `splice`/`coil`/`belay` kernel verbs; knots as joints; the rope at a drop
  and the belay; tar as a coat; `trade-cordage` and the ropewalk; hemp;
  potash-on-sugar's remainder (ash percolates). Review: the buy-back
  (`Good.worthFactor`) taken out as retail's; laying made a craft, real time
  and a crew; D29 (thermal shock scales by conductivity — the potash iron
  pots had cracked since ceramics). Drive 18/18 (`cordage.dirty`); soak
  `cordage.soak.dirty`.
