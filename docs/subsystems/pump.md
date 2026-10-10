# Pump — moving a fluid up, and the wall only some machines meet

⭐⭐ **A pump is a displacement machine set IN a source, and the law it
obeys belongs to its MECHANISM, not to it.** A suction pump pulls, so it
can lift no higher than the air where it stands will push a fluid up a
pipe; a force pump pushes and has no such wall; a bucket carried up a
shaft is not a pump at all and has no ceiling at any depth. This doc owns
the mechanism they share, the place the law lives, the seal that wears,
the two movers, and the tap that reads what feeds it.

Shipped by the pump build (2026-10-09). Before it the realm had a bailer
(drilling), a furnace's bellows behind a verb called `pump`, and the
equation for a pump's power bill on the water pack's conduit with no
caller in production. There was no pump.

---

## ⭐⭐⭐ Two families, and the ceiling belongs to the mechanism

| family | machines | how | ceiling |
|---|---|---|---|
| **A · bucket** | bailer, shadoof, noria, bucket chain, a mine cage | *carries* the fluid up in a container | **none, at any depth** |
| **B · displacement** | suction pump, force pump, sucker rod, centrifugal, a furnace's bellows | *pushes* it through a pipe past a seal | **only the ones that PULL** |

The suction limit is not a property of pumps; it is a property of pumps
that pull. Put the ceiling on the machine and the first authored noria
inherits a 10 m cap that is physically false — the lesson stops being a
law and becomes a lie. So:

- a pump row names its **mechanism** (`mechanism:
  /platform/idea/PumpMechanism/suction`) and never whether it has a
  ceiling;
- a mechanism is a **row** (`platform/idea/PumpMechanism`, `pulls:
  true|false`) — `suction` and `force` ship; a third (`centrifugal`,
  `sucker-rod`) is a row with `pulls:` set honestly and no code;
- the ceiling is **arithmetic over the place**:
  `BiomeApi.suctionHeadFor(scope, fluidDensity?)` = `P_here / (ρ_fluid·g)`
  — ≈ 10.33 m of water at sea level, less on a hill (pressure falls with
  elevation, which nobody authored), less again for brine (denser). Its
  primitive `BiomeApi.columnHeightOf(ΔP, ρ, g)` is the altimeter's too:
  altitude is `(P_sea − P_here)` over AIR, the ceiling is `P_here` over
  the FLUID. ⚠ They share the primitive, not the expression.

⛔ `LiftMixin` — slated by three slates — is **family A** and untouched.
The pump is its sibling, not its instance.

## ⭐ Nothing explains the number

The refusal at a deep well carries the **depth and nothing else**: *"You
work the handle until your arms ache, and nothing comes. The water stands
14 metres down, and the pump will not draw it up."* `analyze pump` reads
*"it will draw water from no deeper than about 10.6 m ± 0.3 m"* —
bracketed by band, a thing you can find out — and never says why. No
row, help text or reading names the air. The why is a law somebody has
to discover (the inquiry slate's first case), and a force pump on the
same well simply works.

## The shapes (`lib/pump/`)

Declared shapes, the `lib/ground/Workable.ts` category — a controller
narrows by shape; nothing composes them.

| shape | answered by | says |
|---|---|---|
| `Pumpable` (`planPump` → plan \| refusal, `completePump`) | `PumpingMixin`, **`BurnerMixin`** | *a person can work this with their hands* |
| `LiftSource` (`standingDepthM`, `standingMaterial`, `standingAvailableL`, `receivableL`, `liftInto`, `liftScope`) | `platform/thing/Well`, drilling's `Wellhead` | *a fluid stands this far below my draw point* — ⚠ adds no field anywhere |
| `PumpSource` (`pumpFitted`) | `Well`, `Wellhead`, water's `Conduit` | *a pump is SET in me* — how a pump tells set from carried |

⭐ **A furnace is not a pump.** Its bellows speaks the protocol (an
INSTANT act, `durationMs: 0`, the toggle) with the three refusals and two
scenes `PumpController` printed before this build, moved verbatim (AC 10,
pinned by `PumpController.test.ts` — the controller had no test before).

## The verb

`pump` / `work` — one view (`platform/cmd/device/pump.yaml`), `requires:
any` (a furnace, a well and a wellhead share no mixin; an alternation
would delete a check), `greedy: true` (*"pump village well"* fell through
the shape without it — found by driving). `PumpController` narrows:
protocol → work it; pump source → the pump set in it, or *"Nothing is set
in the well to work."*; anything else → *"There is nothing to pump on the chair."* A spell is an engagement on the hands with `effortW` derived from
the work (`ρ·g·h·Q/η_pump`, over the shipped `exertion.efficiency`,
floored by `pump.handFloorW`) — the body pays at completion.

## The machine — `PumpingMixin` on `platform/thing/Pump`

`Pump = PumpingMixin(StagedMixin(ContainerMixin(CraftedMixin(Good))))`.
Five facts, one of which is a class:

- **mechanism** (row), **liftM**, **throughputLps**, **strokeS**;
- **power is the mover** — a pump with no mover runs only while worked; a
  pack subclass that composes a `Powered` supply runs off it.
  ⭐ **A prime mover is a `Powered` implementer** (the steam-engine slate's
  coupling, stated as what exists): `isRunning`, `moverPowerW`,
  `deliverableM3S(head, demand)` = `min(throughput, demand, P·η/(ρ·g·h))`
  read structurally — the kernel never imports a grid.

## The seal

A packing is a `Tool` (`capabilities: [packing]`, leather) set IN the
pump; the pump holds one and vetoes everything else. It wears per spell
(`pump.wearPerStroke`) and per RUNNING hour (`pump.wearPerRunningHour`,
⚠ **integrated over `poweredTrajectory`, never sampled** — the switch has
no history, so `ElectricPump.setOn` reconciles first).

⚠⚠ **The running-hour rate ships at ZERO, until the keeper exists.** A
powered pump's packing sits two containers deep in its main and the pump is
fixed in place, so nothing a player can do reaches it to re-pack or repair.
At the rate first shipped (0.002/h) the city intake went dry for good in
about 19 game days — a failure nobody could lift, which is not a lesson.
The mechanism is tested at a set rate; the consequence waits for the
waterworks seat (Deferred). A broken packing
offers nothing (`hasCapability` is false on a broken Durable): *the
leather has gone* is capability loss, no state machine. It reads in five
words and no digit (`sound · worn · leaking · perished · gone`).

⭐ **You re-pack a pump by pulling it.** `get` reaches one container deep
(`mustBeInLocation`, the same rule `canReach` asks), so the leather comes
out of a pump standing on the ground, never out of one in a shaft. The
packing is the tanner's (`trade-tanning`, `recipes/packing.yaml`) — the
pump is the tannery's customer forever.

⚠ It is an OCCUPANT today, not a constitutive part — the assembly build's
part model absorbs it.

## The places

- **`Well`** (`platform/thing/Well`) — a shaft to standing water with a
  trough at the collar (`interior`, starts EMPTY), an inexhaustible body
  below (the finite aquifer is a non-goal). Rejection ships the village
  well (6 m, a hand pump in it) and the deep well (14 m, none — whoever
  gave up took it).
- **A bore** — drilling's `Wellhead` answers `LiftSource` from the hole's
  depth and sump; `liftInto` is bounded by the sump AND the trough;
  `lift()` (the bailer) is byte-identical and asks no pressure. A crew at
  a fitted pump earns a **continuous rate** in `reconcileRig`
  (`throughput × elapsed × hands × pump.crewDuty`) — only if the pump's
  own law lets it lift from that depth (`liftsFrom`). Rejection ships the
  old brine bore at the spring (25 m, `bodyKey` authored — authorable
  since this build).
- **A conduit** — water's `Conduit` holds its pump; see below.

## The city, and the tap that reads its main

- `/system/energy/thing/ElectricPump = GridPowered(Switchable(Pump))` —
  the energy pack's appliance; affords `switch` itself.
- The Wharfside intake holds one (`props:`). ⚠⚠ **Left overdrawn,
  deliberately**: ~98 kW to lift its 1.2 m³/s through 5 m, against the
  industrial band's 60 kW, so it lifts ~0.73. The obligation is that the
  shortfall is **legible**, and it is — `analyze water intake`: *"its pump
  would draw 98.1 kW to lift 1.20 m³/s, and the line gives it 60.0 kW — so
  it draws all the line gives it and lifts 0.73 m³/s — less than is asked
  of it."* ⛔ Do not raise the band or lower the capacity.
- `SupplyServing { supplyStateNow() }` (kernel, `lib/supply/`) is the
  SYNC subset of the six words; `Conduit` answers it (cut · closed · a
  pumped main whose pump is not running → `off`). `WaterFixture.suppliedBy`
  plumbs a tap to a main: empty is every shipped tap; set, the tap runs
  only while the main delivers, says *"Nothing comes out of it: it has
  been shut off."*, and `analyze water <tap>` reads the main. An
  unresolvable main fails OPEN and logs once. The market standpipe is the
  first tap in the realm that knows what feeds it.
- ⚠ A power cut at the pump reads `off` too — the six words have no
  *unpowered*.
- ⚠ **One tap.** Only the market standpipe is plumbed; every other tap in
  Terminus is its own source. Plumbing the city, a tap served by two mains
  (the aqueduct also serves Terminus, and is switched off), and taps that
  feel an overdraw are the water design pack's Left —
  water-design-pack.

## Gates

`lint:capabilities` (the `packing` kind declared by the tanning row and
consumed by `Pumping`), `lint:census` (reads `mechanism`, `suppliedBy`),
`lint:reachability` (`suppliedBy` is a citation), `lint:mixin-names`.
The drive: `packages/wire/tests/pump.dirty.wire.test.ts`.

## Deferred

The prime mover's coupling beyond the grid (steam, horse gin, wheel —
each a `Powered` implementer in its own pack, steam-engine slate) · a
powered pump on a wellhead (no content ships one) · the finite aquifer ·
`dry`/`frozen`/`fouled` at a tap (the async words) · a seventh supply word
for *unpowered* · the packing as a constitutive PART (assembly) · a pump
as a recipe output · the city pump's keeper (a seat the polity creates) ·
the suction limit as a discoverable law (inquiry slate) · running-hour
wear (ships at 0 until a keeper can re-pack a powered pump) · a hand
bellows you carry (pump slate) · plumbing the rest of the city · mine
dewatering and trade-mining's `drain`, which narrates *"work the pump"*
with no pump object · the gym's `pace` slot.

## History

Pump build, 2026-10-09, `4bd010214..` on `reqs/pumps`. The drive found:
a pail that could not hold water (`closure: open` means not liquid-tight;
fixed in generic-objects), `Conduit.resolveHead` with no production
caller (every live conduit's head "never surveyed"), the bank's
`aqueduct` detail shadowing the aqueduct, and a verb that fell through
the shape on two words. A browser walk then found sentences opening
lowercase on a thing's name (the platform has no sentence-initial
capitalisation, so none of this subsystem's sentences open with one) and
holder prose describing a removable pump (antipatterns.md § *A holder's
prose describing an occupant that can be REMOVED*). Review round 1 zeroed
the running-hour wear until a keeper exists.

Re-drive: `WIRE_BOOT=1 WIRE_PORT=<worktree's> npx vitest run
tests/drilling.dirty.wire.test.ts tests/pump.dirty.wire.test.ts` from
`packages/wire` — the two share one world without taking each other's kit.
