# Water supply — a tap gives what its supply gives

The running-water and irrigation build (water-supply, 2026-11). The
watershed ([watershed.md](watershed.md)) knew where water ran; this is how
it gets to a tap, a field and a well, and what goes wrong on the way.

## The premises citation

A premises says where its water comes from on its **parcel**, the twin of
energy's feeder: `waterSource` on `ParcelRecord`/`TitleClaim` is a supply's
template path, `none` (a declared dry premises), or `''` (inherit). A pack
declares it on a title claim (`waterSource:`). `ParcelApi.waterOf(path)` is
the longest-prefix-then-parent walk. `ParcelApi.citeWaterSource(extent,
path)` is the runtime write, made by a plumber's completed service run.
`PlatBook.lotWaterSource` stamps a sold lot. `lint:water-posture` (ceiling 0): every
residential/commercial/industrial/civic claim declares or inherits a
supply. ⚠ An inherited `none` does not count — a premises is dry only if it
says so itself. A `kept` grant fills an empty record from its claim, and a
record that already cites (a service run's write) is never overwritten.

## The fixture reads it, and fails closed

`WaterFixture` resolves its premises: the room at the end of its
containment walk, or a keyed holding room by its **persistence key** (the
lot, not the shared row). It asks `waterOf`, loads the supply on use
(`StuffApi.singleton` — a supply composes `SingletonMixin`, so the room that
props it adopts the same instance), and reads it. **Before it resolves it
is empty and says nothing**; it never runs endlessly because a main was
not loaded. `natural: true` is a source (a leat, a pond, a spring trough).
`suppliedBy` names a supply outright (a hydrant on a main).

The supply is the kernel **`WaterSupply`** shape (`lib/supply/SupplyState.ts`):
`supplyStateNow`, `supplyHeadM` (async), `drawL`, `availableL`,
`shareFor(landUse)`, `serves(path)`, optional `readWaterChemistry`.
Implemented by the water pack's `Conduit` and `StorageNode`, and the
kernel's `WaterStore` (+ `SingletonWaterStore`, `PersistentWaterStore`).

**The seventh word.** `SupplyState` gained `unplumbed` (*no pipe brings
water here*), ahead of `cut` in precedence.

**Pressure is head.** Head is the supply's water surface minus the outlet's
height (its zone's elevation plus its storey × `construction.storeyHeightM`).
At or below 0 the outlet reads `dry` (*"the water stands below here"*).
Below `water.weakHeadM` the tap runs weak (a draw gives at most
`water.weakDrawL`). A pumped main holds its line at
`water.mainPressureHeadM` over its served ground.

**A shortage is shared by rank** (`Conduit.shares`): `priority` serves
civic outlets first, then the trades, then houses, weighted by an
authored `demandSplit`; `proportion` gives everybody the same fraction. At
the tap a share of 0 reads `overdrawn` and a partial share runs weak. A
continuous outlet (the fountain) asks as civic. `ConduitReading.carriesM3S`
is what the main physically carries, whatever words its reading says.

**Intermittent supply.** A `StorageNode` fills on a `ShiftEntry` schedule
while its main delivers (the main's 0/1 `deliveryLog`), and is drawn at the
district rate. It is reconciled hour by hour on read. A `WaterStore` (butt,
cistern) fills from the rain on its roof or from what is poured in. Its
level is its interior bulk, primed at its authored first level from its
first read.

## The pipe in the room

**A room has water because a pipe brings it**: a supply-reading fixture
needs a sound `PipeRun` in its room. The exemptions are `natural`,
`suppliedBy`, a `Street` (a public outlet's pipe is under the street), and a
`WaterStore` supply standing in the same room.

⭐⭐ **A tap implies its pipe; nobody props one.** Plumbing is a layer of
the room's fabric, said once on the room by `PlumbedMixin`
(`lib/supply/Plumbed.ts`, composed on the ROOM rung — `Room`, `Hall` and
their kept twins — and nowhere else): `plumbing: lead`,
`plumbing: none` (not plumbed — waiting for a plumber), or nothing at all,
which is the default (`plumbing.defaultRun`, `iron`). A plumbed room with no
run in it stands up the run its word names
(`/stuff/thing/fixture/<word>-run`) the first time a fixture needs one, and
keeps it. A run an author or a plumber put in the room is the room's run.
Not `Enclosed` (a pipe bounds nothing); not a `Structure` (a structure is
never made of anything, and a lone hut is plumbed with no building). Not
every `Place`: a street, a yard or a field never carries the field.
`lint:water-plumbed` keeps it true: every word names a run row, `plumbing:`
is said only on a room, every tap that needs a pipe stands in a room (a tap
on a named source, in a street, beside its own store or on natural water
needs none), and no room props a pre-laid run by hand.

⚠ History (review, 2026-11-06 → 11-07): sixteen rooms were forced to prop a
pipe; the first fix composed `PlumbedMixin` on every `Place`, which the
owner struck as a field nearly every place would never use; the second
moved the word onto the premises' title claim (`pipe:`), which only
relocated the problem and lost the per-room pin. ⭐ The root cause was the
place taxonomy, which sorted places by addressing and persistence but never
by KIND — so the room rung was added (`location.md` § The room rung) and
the mixin went there.

`PipeRun` (kernel) = `AdornmentMixin(AssembledMixin(DurableMixin(ThermalMixin(BulkableMixin(Good)))))`.
It is out of the room's loose listing (part of the room's structure, not
concealment). It **bursts** off its room's temperature trajectory (lead
bulges to 0.4, `brittle` iron to 0) and **leaks** onto the floor on the
thaw, once; `repair` mends it. Its standing water carries the supply's
chemistry, and a lead run leaches into it by the face law, so the first
draw of the morning carries the lead. One piece without a bill; a
plumber's run (`trade-plumbing`) is laid empty and filled by `fit`. A
**service run** refuses its last length where the main it names does not
serve the premises, and cites the main on completion (the `Assembled`
hooks `canTakeMembers` / `afterMembersFitted`).

## Chemistry, health, the kit

`WaterChemistry` (pH, hardness, salt) rides the payload. The lead law
reads the WATER's pH and softness: hard water scales a lead pipe shut. A
dose tag is the Condition row's **file leaf** (`lint:toxin-keys`, ceiling
0); `dysentery` now exists. **The kit, not the eye**, says whether water is
fit: `analyze water` prints a fouled verdict and the lead only with the
water-test kit in hand.

## Readings

These are under the new `hydrology` Discipline: `analyze flow <weir>` (with a
competent reader's account of what the gates above take, and what no gate
accounts for), `analyze stage <staff gauge>` (`0.4·Q^0.4`), and `analyze
water-table <observation well>`, the only reader of an aquifer's level.
Units: `m³/s`, `L/s`, `miner-in` (0.7373 L/s).

## Gates, the ditch, the rota

`Gate` (water pack) is a `ControlStructure` that is Sealable, Lockable and
Throttleable (kernel `ThrottleableMixin`, `lib/spatial/Throttleable.ts`: `opening`, an opening trajectory,
and a work log of who set it). `open … to <setting>` — the thing answers
(`Sealable.openPlanFor(by, setting)`, which `ThrottleableMixin` overrides to
work the amount; controller-liquidity W7c). `canOpen` refuses a
padlocked gate to somebody with no key, and a locked `Door` to everybody.
The catalogue **credits diversions** into the course a control names, less
seepage by `lining`. An artificial channel carries only what it is given:
no catchment, none of the river's draws above its headgate, and its works
draw on what it carries. The **rota is the turn list**:
`Gate.reconcileScheduled` opens and shuts a turnout for its position's
roster windows, and records a refused turn when it is padlocked.

## Soil, stores and wells

**Soil**: `SoilMixin.inlets` integrate a gate's opening × capacity ×
method efficiency (`settleInlets`, async, on the read path). What the soil
can't take is ponded; ponds drain and leach salt. `soilSalinityDsm()` reads
the salt left behind. Maas–Hoffman salt is folded into the plant's water
term (salt is physiological drought). A ponded bed drowns roots.
`PlantPot.drained`.

**Wells**: a `Well` cites an `Aquifer` (the kernel `GroundWater` shape) or
keeps a lone, recovering body. `collarM` stands a hillside well above its
zone.

## Sprinklers and hoses — rain in a circle

⭐⭐ **A sprinkler is rain in a circle** (the review, 2026-11-06:
sprinklers are core to farming). What the rain reaches, it reaches.

- **`Sprinkler`** (kernel `platform/thing/Sprinkler.ts` =
  `IrrigatingMixin(ThrottleableMixin(SealableMixin(Good)))`, row
  `/stuff/thing/gear/sprinkler`) is a good you set down and `open`, by an
  amount like a gate.
  - **Flow and throw are the tap's head:** an orifice, `Cd·A·√(2gh)`
    (`nozzleMm`), thrown half its head round, up to `maxThrowM`. Twenty
    metres of main throws eight metres; a three-metre tank dribbles; a ditch
    drives nothing.
  - **It draws the tap's supply** at its rank's share, through
    `WaterFixture.pressureNow()`.
  - **What lands is the weather's** (`Soil.landedOpenSeconds`, the overhead
    inlet's own law).
  - **What it reaches:** the ground it stands on over the circle it throws,
    and the beds beside it by the sky each catches. Never a pot, which
    catches no rain. What the soil can't take stands on it, and whatever is
    in the spray gets wet.
- **`Irrigating`** (kernel protocol, `lib/husbandry/Irrigating.ts`): the soil
  asks what stands on it, or beside it, to bring its water current. The
  irrigator integrates on its own clock, so two grounds asking count it once.
  The soil's sync reconcile kicks the settle in the background. That also
  closed the review's finding that authored inlets were integrated only when
  someone surveyed. ⭐ **The tap asks too:** a hose screwed onto an outlet
  registers with it, so any settle at the tap (a draw, a fill,
  `analyze water`) brings the sprinkler at the far end current first, and
  its supply never reads as if it had not run. Walked: an hour open took
  1.3 m³ more from the District tank than an hour shut (the arithmetic says
  1.2: 0.33 L/s at 18 m of head).
- **`WaterHose`** (kernel `platform/thing/WaterHose.ts`; trade-rubber's
  `hose` row). Its use is where its two ends are coupled:
  - `tie hose to tap` and `tie hose to sprinkler` screw on each end. Coupled
    at both ends, it is laid down where you stand.
  - Carried and coupled at the tap only, it is a hose in your hand you can
    `water` a bed from.
  - Not a `Line`: a rope and a hose are different things. Its couplings are
    runtime, as a line's end is.
- **The ladder:** the can (a pot, by hand) → the hose by hand (a bed) → a
  sprinkler (a garden, a small field, unattended, a circle, the weather's
  share) → several sprinklers or the ditch's furrow by turn (a big field:
  pressure, settings, the rota). The general store stocks the hose and the
  sprinkler.
- ⚠ **Open** (irrigation slate § 13):
  - On a tract divided into fields, a sprinkler waters the whole tract, its
    fields by area. Setting it on one field needs fields that hold things.
  - The hose's couplings do not survive a restart.
  - The leather hose.
  - The bench's authored sprinkler-valve as a placed sprinkler on a ram.

## Staged or live

Staged or live is the content's choice (D30). Each stateful kind ships
plain (staged) plus a `Persistent…` twin used where a row needs live state:
`PersistentConduit` (the city intake, the aqueduct), `PersistentGate` (the
Delight's gates), `PersistentWaterStore` (a sold roof cistern),
`PersistentAquifer` (the Delight valley), `PersistentWell` (a dug well).

## Deferred

Each with where it goes (salvaged from the plan at the sweep, 2026-11-06):

- **The loft** (a tap on an upper storey reading its head) — paint's
  storeys landed; the head rule is unit-tested, the content is not built.
- **A plumbing line on `analyze rooms`** — `look run` gives the state in words today.
- **The irrigates / rides brains** (the ditch rider riding, a farmer
  irrigating unwatched by choice) → irrigation-slate.
- **Water rights as documents**, the turn book → irrigation-slate § 12.
- **Rota → calendar**, the bank breach, `ditch with tile`, deep percolation
  into an aquifer, the drilled bore's strike → irrigation-slate.
- **`dig well`** and every other hole (graves, pits, trenches, cellars,
  machines, blasting, backfill) → digging-slate. `HoleMixin` is the seam.
- **Construction proper** — a pipe run is its first worked element (made of
  parts by a trade, laid with `fit`) → construction-slate.
- **`break <padlock>`**, the ditch's padlocks as goods on hasps, the key
  board → lock-slate § 8.
- **An NPC plumber on a gig** (the `job fulfill` shape over `fit`) →
  freight-slate / contract.
- **Metered billing** (`drawL` × time as a payment leg, with power) →
  power-utility-slate.
- **Sewer, WC, septic** — a drain is a service run reversed; the outfall is
  the sink → sanitation-and-pollution-slate.
- **Bored elm pipe** (a pipe length of wood) → forestry-slate.
- **A powered pump on the bench** feeding a cistern; the cone of
  depression and subsidence (`Aquifer.level`) → irrigation-slate.
- **Sprinklers: one field of a tract, couplings across a restart, the
  leather hose, the bench on a ram, drip** → irrigation-slate § 13.
- **Flood crops** (`floodToleranceDays`); pots nobody sells → farming-slate.
- **`reach:` on a title claim** (riparian derivation finds nobody: `PackLogic`
  does not parse it) → a water-rights follow-up, one parse line and a test.
- **Durable cuts on the grid** (`Conduit.cutLog` is the water twin) →
  power-utility-slate.
- **Watering ships; a ship's discharge as a load** → running-water-slate § 7.
