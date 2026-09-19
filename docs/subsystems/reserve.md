# Reserve subsystem

A **Reserve** is a depletable-and-replenishing capacity axis. The
substrate is broader than biology: the body's biological reserves
(endurance / satiation / hydration) and, deferred, magic-side reserves
(a guild's "charge", a tradition's "essence") are all instances of the
same axis — they differ only in **what drains them, what replenishes
them, and their theme**. So the engine ships the *axis* and content
names the instances.

Source: `lib/reserve.ts` — a **top-level cross-cutting substrate** (the
value shape + decomposed persistence + the biological-key constant + the
`ReservedMixin`), sitting at `lib/` root next to `lib/quantity.ts`
because a reserve is a neutral capacity primitive that consumers (body,
later magic) name and drive, not biology and not a game system. It is a
substrate module + a mixin — **no Api, no registry** (the
`lib/quantity.ts` precedent). Parking note: it lives here until the RPG
layer reveals a better organizing principle (game systems over a common
physics substrate).

This build ships the substrate + the biological instances + the
authored-thematic seam. The reserve *producers* (consumption refilling
satiation, locomotion exertion draining endurance, any recharge rule)
are deferred.

## "Reserve" is the engine word

`Reserve` is the engine identifier. "Mana", "charge", "essence", "favor"
are **content** — they ride the value's `theme` and `key`, never the
engine surface. The engine has reserves; a magic system names its pools
over them. This is the affliction-vs-trauma authored-content move applied
to reserves: the substrate ships the axis + the seam; the thematic pools
are content. (Magic itself is deferred — see
capability-magic-slate.md,
where mana is documented as endurance's mirror riding this axis.)

## The value shape

```ts
interface Reserve {
  key: string;              // identity within a host (= Record map key)
  capacity: Quantity<Unit>; // maximum (typically Quantity<'%'>)
  current: Quantity<Unit>;  // clamped to [0, capacity]
  theme: string;            // 'biological' | a content theme
  floorEffect: string | null; // named effect at the floor — driven by metabolism
}
```

Persistence is by **decomposition to scalars** (the `AmbientLit`
precedent): a `Reserve` holds `Quantity` capacity/current, but the stored
form is a plain `ReserveStored` (`{ capacityValue, currentValue, unit,
theme, floorEffect }`) in the host's keyed `Record<key, ReserveStored>`,
so it hydrates free with **no per-element marshaller**. The `Reserve`
value objects are reconstructed on read. (A `Map`/array of `Quantity`
value-objects has no clean marshaller; decomposing to scalars is the
pattern.)

## `ReservedMixin` surface

Collections.md Shape B (keyed Map). Composed onto every `Creature`.

```ts
interface Reserved {
  getReserve(key: string): Reserve | undefined;
  getReserves(): ReadonlyMap<string, Reserve>;
  setReserve(reserve: Reserve): void;          // add/replace; clamps + unit-checks
  adjustReserve(key: string, delta: Quantity<Unit>): void; // clamps [0, capacity]
  hasReserve(key: string): boolean;
  removeReserve(key: string): boolean;
  installBiologicalReserves(): void;           // idempotent
  reserves: Record<string, ReserveStored>;     // public for the Hydrator
}
```

Per-field invariants live on the setter: `setReserve` / `adjustReserve`
clamp `current` to `[0, capacity]` and reject a unit mismatch.

## The biological instances

`BIOLOGICAL_RESERVE_KEYS = ['endurance', 'satiation', 'hydration']`, each
a `Quantity<'%'>` with `theme: 'biological'` and a `floorEffect`
(`collapse` / `starvation` / `dehydration`). Every living body installs
them at full capacity in the `Creature` constructor
(`installBiologicalReserves()`, idempotent — hydration overwrites from
stored values afterward).

The `floorEffect` strings are no longer a dangling seam:
[metabolism](./metabolism.md) is their consumer. Its lazy reconcile
drives these reserves (basal drain on satiation/hydration, coupled
recovery rebuilding endurance) and, when one floors, spawns the
`floorEffect`-named `Condition` (`starvation` / `dehydration` /
`collapse`) — clearing it on recovery. Encumbrance drains `endurance`;
metabolism replenishes it; both layer on the one reserve.

The "what's biological" set is a substrate constant in `Reserve.ts`, not
auto-installed by `ReservedMixin` itself — so a future non-biological
reserve host (a magic-only entity) can compose `ReservedMixin` without
acquiring biology.

## The non-biological instances

The axis is genuinely neutral, and the shipped consumers prove it. Beyond
`fuel` (a `Combustible`, theme `combustion`), `air` (an enclosed scope's
`Location`) and `mana` (a `Caster`, theme `arcane`):

| key | unit | owner (installs/drives) | read it via |
|---|---|---|---|
| `moisture` | `L` | `Plant` + `GrowingMixin` (theme `cultivation`, floor effect `wilting`) | `getSoilMoisture()` — the raw keyed read skips the growth reconcile, so never use it outside `lib/husbandry` |

`moisture` is the first reserve **authored entirely in a template's `data`**
rather than installed by a constructor — the whole keyed Record is one
`fieldMeta`'s persistent entries entry, so a plant's root-zone capacity is content. It is
also the first with a real-units capacity that is neither `%` nor `pt`.

> ⚠ **`Wet.ts` used to claim this axis is "biological (Creature-coupled)"**
> and cited that as its reason for not reusing it. That was wrong, and the
> houseplant build corrected the comment. Wetness skips `Reserved` because a
> single unbounded `[0,1]` saturation needs no keyed collection — not because
> the axis is biological. See [husbandry.md](./husbandry.md).

## The band feed

A floored biological reserve degrades the body: `VitalsMixin.getConditionBand`
reads the host's reserve surface (if it composes `ReservedMixin`) and
adds load for each biological reserve at/below its floor — a derived
reading, exactly like consciousness reading vitals. See
[vitals.md](./vitals.md).

## The authored-thematic seam

`setReserve({ key, capacity, current, theme, floorEffect })` works for
any non-biological reserve. No magic content ships; the seam is
demonstrated (a `theme: 'arcane'` reserve can be defined and round-trips)
and waits for the magic subsystem.

## Cross-references

- [vitals.md](./vitals.md) — the band feed; reserves are body-state
- [quantities.md](./quantities.md) — `Quantity` capacity/current
- [collections.md](./collections.md) — the keyed-Map surface
- capability-magic-slate.md
  — mana as a magic-side reserve on this axis
