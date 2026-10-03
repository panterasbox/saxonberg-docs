# energy — the realm's power economy, end to end

The `/system/energy` pack: turning a source into power, moving it to where
people are, and letting a place consume it — with someone paying. It ships two
epochs at once, and the fiction it is about is that **the same civic service
(street lighting) burns lamp oil in a frontier town and draws grid power in the
city**, the visible sign of a place developing being which one it is.

See the seeding design in
grid-slate § Part 5 and
power-utility-slate.

## The two epochs

- **Combustion** — a town funds street lighting, buys real **lamp oil**
  (`trade-fuel`: a `bulk` material + an `oil-cask`/`lamp-oil-cask` good, made by
  an **oil works** outfit in the Terminus goods yards), and burns it a
  street-night at a time out of a **`FuelStore`**. A store run dry darkens the
  junior streets first, the way a short treasury already does.
- **Electric** — a public **grid**: generation (the shipped Wharfside
  hydro `ControlStructure`), a **feeder** network over the streets, and a
  **meter** at each premises. A place is powered because a line reaches it.

## The meter is the boundary

Consumption resolves to the **covering parcel**, never by walking the interior
(`ParcelApi.powerOf(path)` → `{ band, feeder, parcel }`, longest-prefix then
inherit). The kernel carries two citations on `ParcelRecord` (the `reach`
shape):

- **`feeder`** — the grid node that meters this ground (`terminus-main:avenue`),
  or `''` for ground no line reaches;
- **`powerBand`** — the electric posture (`off-grid · domestic · commercial ·
  industrial`, kernel `lib/parcel/PowerBand.ts`), or `null` to inherit.

Both are `TitleClaim` fields authored in `pack.yaml`, indexed by
`ParcelRegistry.byFeeder`. `off-grid` is a first-class declaration — a shack is
off the grid *on purpose* — and `lint:power-posture` catches a premises that
declares nothing (a build error) while a *wrong* declaration is a reviewer's
disagreement.

## The grid is a compiled reachability set

`Feeder` is a data Idea (`/stuff/idea/Feeder`, the commons — a realm edits its
own grid, the `Watercourse` rule): a `source`, an ordered chain of `nodes`
`{name, at, buried}` down the streets, and `branchesFrom` for a spur.
`GridCatalogue` (singleton Idea) compiles every feeder once into:

- the **downstream set** per node (a cut's blast radius, spanning spurs);
- the **trace** chain up to the source;
- the street→node map;
- the resolved source instances.

A node is **energized iff** its source `isGenerating()` **and** no cut lies on
the path from source to it — all sync and live, so a cut is felt the same
second. The **cut is the one piece of state** (`sever`/`splice` on a
`LineAccess` pole write it). ⚠ Cuts are **in-memory/transient** today — a reboot
re-energizes the grid; durable cuts matter only to the Tier-deferred lineman
work-order economy. A consecutive node pair whose streets no exit joins is a
**compile problem** recorded (never a throw) and its downstream goes unreachable
— *a line may not leave the road*.

⚠ **The compile is lazy and must run post-install** — warming it from a
`onCreate` races the street hydration and caches a broken grid (found by the
drive). It runs on the first real read (the dusk settle, or `analyze grid`).

## The consumer, the light, the pole

- **`GridPoweredMixin`** (pack `lib/`) — composed only by things that DRAW
  (`ElectricLight`, the `ColdStore`/`ColdRoom`; never on `Thing`/
  `LightSource`/`BurnerMixin`). Resolves its premises' band + feeder once;
  `isPowered()` is sync. ⭐ **It implements the kernel `Powered` shape**
  (`isPowered` · `availablePowerW` · `poweredTrajectory`) structurally, so
  `ClimateControlMixin` (kernel) drives off it without the kernel importing
  the pack. A host that is not Containable (a `ColdRoom` Location) is metered
  against ITSELF (`resolveRoomPath` returns its own path).
- **`ElectricLight`** — `GridPowered(LightSource(Switchable(…)))`: emits only
  while on AND powered.
- **`ColdStore`** (`/system/energy/thing/ColdStore`) + **`ColdRoom`**
  (`/system/energy/location/ColdRoom`) — ⭐ the cold-storage build's active
  cooler, a Thing and a Location composing the one `ClimateControlMixin`
  (the Thing≡Location proof; see [thermal.md § The active twin](./thermal.md)).
  A fridge/freezer is a `ColdStore` row (it ships a colder freezer
  compartment via `props:`); a walk-in is a `ColdRoom` row. Both gate their
  cooling on `isPowered()` and warm when the feeder is cut.
- **The cut log** — `GridCatalogue` records each `sever`/`splice` with a
  game-time stamp (`cutSince` + a bounded `outages` ring) and publishes
  `poweredTrajectory(node, fromS, toS)`, a 0/1 trajectory an appliance's
  envelope integrates — so a cut mid-outage and a splice after it warm and
  re-cool the store correctly even if nobody watched. (Source generation is
  read as-now, not logged — the cut is the state a fridge cares about.)
- **`LineAccess`** — a pole/manhole on a node; affords `sever`/`splice`; an
  overhead one faults in a storm (`onStormExposure` → `energy.stormFaultRate`),
  a buried one is storm-safe. ⭐ **A `LineAccess` is SPARSE, not one-per-street**:
  it is the physical access point where a lineman actually works the line, so
  content props one only where that interaction is wanted. The line *reaching* a
  street is not an object — it is a parcel fact (`powerOf` → `feeder`, read by
  `analyze grid` in any room) plus an authored `line` room Detail (`look wires`).
  Terminus ships exactly two: an overhead **pole** at the avenue and a buried
  **manhole** on the Mayfield spur (the overhead/buried contrast); the other
  grid streets carry the line as a Detail and no object.
- **`analyze grid`** — a `Reading` channel: bare = the premises + the locality's
  **derived epoch** (electric if a feeder reaches it, gas-lit if it burns oil,
  off-grid if neither — no "tech level"); on a pole = the trace naming the first
  cut.

## The goods leg and who pays

A `Locality`'s `_publicLighting` names a `supply` — a `FuelStore` (gas-lit) or
the `GridCatalogue` (electric) — implementing the kernel's `StreetLightingSupply`
shape; the settle asks it `lightStreets` and lights exactly what it covers.
Terminus migrated from the general-store placeholder to the grid; Heart's Delight
is gas-lit, with a **parish government** (a treasury, a public-works department,
the Warden of the Ways) that buys oil from the oil works through the shipped
procurement loop. The sales tax splits a `banking.localTaxShare` to a sale's
covering locality so a town's budget fills from its own trade; `appropriate`
sources from a locality's own treasury when it has one.

## Deferred seams

- **Tier B — the interior** (circuits, breakers): the meter is the attach point
  (a premises-side `Energized` fixture held at potential iff `isPowered()`). →
  `electricity.md`, grid-slate Part 5.
- **Tier C — metered billing**: `availablePowerW` × time as a payment leg. →
  power-utility-slate.
- **Durable cuts + the lineman work-order market**, **gas as a second piped
  commodity**, **the retort** (crafted lamp oil), **overdrawn brownout**. →
  power-utility-slate / destructive-distillation-slate.
