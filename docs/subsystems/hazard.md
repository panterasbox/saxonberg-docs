# Hazards & traps

The flagship consumer of the [concealment](./concealment.md) gate: a
`HazardMixin` (`lib/hazard/`) that generalizes the `GlassAlley` one-off
(`onEntered → inflict`, explicitly *"NOT a `HazardMixin`"* — see
[harm.md](./harm.md)) into a real **armed→sprung** hazard substrate. A
trap's *delivery* is the weapon grammar (armor mitigates a trap exactly as
it does a blow); its *trigger* is the locomotion traverse; its
*concealment* is the detection gate (you can spot and disarm one). The
taxonomy is **compositional axes → authored data**, with **zero
per-archetype subclass**.

## The self-resolving `HazardMixin` — no `HazardApi`

**There is NO `HazardApi` / `HazardLogic`.** A hazard is **self-contained**:
it holds its own state, its trigger/delivery/consequence descriptors, and —
crucially — **its own resolution**. There is no shared state, no registry,
no cross-trap coordination for a `*Logic` singleton to own; and the
*powerful* steps a spring orchestrates (`ConditionApi.inflict`,
`PerceptionApi.perceives`, the metabolism / locomotion / posture Apis) are
each **already gated**, so wrapping them in one more gate buys nothing.
Gating an orchestrator over already-gated primitives is the redundant
layer. This is exactly `GlassAlley.onEntered` — a plain Stuff method
calling the gated `inflict` seam — now made reusable content. A mover
reaches a trap through its `resolveTraversal` **method**, honoring the
inter-Stuff methods-only contract.

`HazardMixin` (`lib/hazard/Hazard.ts`, `_mixinName = 'HazardMixin'`)
carries the authored descriptor fields (all `@authorable`;
`fieldMeta`'s persistent entries = `hazardState`, `trigger`, `delivery`,
`traverseConsequence`, `groundTriggered`, `dropDestination`,
`springMessage`):

| Field | Axis | Values (v1 · reserved) |
|---|---|---|
| `hazardState` | **State** | `armed` → `sprung` (one-shot) · `disarmed` (third terminal) |
| `trigger` | **Trigger** | `traversal` (fired at the move), `interact` (open/pull/grab) · `proximity` / `timer` / `remote` (authored-representable, unhandled — no code fires them) |
| `delivery` | **Delivery** | a `HazardDelivery` value-object (below) |
| `traverseConsequence` | movement veto/redirect | `none` / `pin` / `trip` / `drop` |
| `groundTriggered` | modality clear | `true` (default) → a fly/swim crossing clears it; `false` catches every medium (a ceiling deadfall) |
| `dropDestination` | the `drop` "below" | an authored template path — *not* computed geometry |
| `springMessage` | flavor | an optional authored victim line; blank → a generic spring line |

The **Locus** axis is free — implicit in the host type (a deployable
`Trap`, or a `Location`/`Exit`/`Container` composing the mixin). The
**Concealment** axis rides `ConcealableMixin` (a `Trap` composes both).

### Method surface + resolution

The outward `Hazard` interface: `isArmed` / `getHazardState` /
`getDelivery` / `setDelivery` / `getTrigger` / `getTraverseConsequence` /
`isGroundTriggered` / `getDropDestination` (accessors), `disarm()` (state
flip, armed → disarmed; no-op once sprung), and the interaction/resolution
methods. `spring()` is `protected` and sets `sprung` **first** (the
drop-reentrancy guard).

- **`resolveTraversal(mover, mode)`** — the mover entered this hazard's
  locus. Returns early if not armed or not a `traversal` trigger. The mover
  **avoids** it (`moverAvoids`) iff they've already discovered it
  (`PerceptionApi.hasDiscovered`), actively perceive it now
  (`PerceptionApi.perceives(mover, self, PerceptionApi.modeAttention(mode))`
  — this is where the care↔speed axis enters), or a ground hazard is cleared
  by their air/water medium (`clearedByMedium` → fly/swim over it). Else it
  springs: `spring()`, then `narrateSpring`, then `deliverHarm`, then
  `applyTraverseConsequence`.
- **`resolveInteract(actor)`** — the interact-touch trigger. Springs an
  armed `interact` hazard (spring → narrate → deliverHarm) but applies **no**
  traverse veto (you weren't moving; you touched it).
- **`disarmBy(actor)`** — defuse on behalf of `actor`: flip to `disarmed`
  and narrate. **The disarm *interaction* (state flip + narration) lives
  here on the mixin**, symmetric with `narrateSpring` — the command
  controller keeps only the command machinery (below). No-op once
  sprung/disarmed.

**`narrateSpring`** is the player-facing beat. `ConditionApi.inflict` is
**silent** (it only mutates body state), so without this a player walks into
a trap, takes the wound, and sees nothing. The victim gets the authored
`springMessage` if set, else a generic line ("Something gives way beneath
you — a hidden trap springs!"); witnesses always get a generic
"…springs a hidden trap." on the `act.deed` topic.

**`deliverHarm`** runs `ConditionApi.inflict(mover,
delivery.toInflictSpec(mover))` (the covering stack mitigates armor for
free), then, if the delivery carries a toxin and the mover is `Metabolic`,
`mover.introduceToxin(type, amount)` (the metabolism seam, below).

**`applyTraverseConsequence`** routes through the existing locomotion /
posture ladder, never a bespoke hazard-movement path:

- `pin` — a `HazardActivity` holds the mover's **`body`** slot for
  `hazard.pinSeconds` (a snare; the body-slot hold is the substrate's honest
  representation of being locked in place, the `Coup` precedent).
- `trip` — `mover.setPosture(Postures.Lie)` (knocked prone).
- `drop` — fall into the authored `dropDestination`, then take the fall
  blunt on landing (`hazard.dropFallEnergy`). See the caveat below.

## `HazardDelivery` — the delivery value-object

`lib/hazard/HazardDelivery.ts` is the pure `Channel × energy × site →
InflictSpec` producer (the `WeaponProfile` / `Construction` category — a
data carrier with one projection method, no Stuff, no state, deterministic).
Fields: `channel` (a materials-response `Channel` — a spike's `point`, a
scythe's `edge`, a deadfall's `blunt`, an electrified floor's `shock`),
`energy`, `siteSelector` (ordered `body.*` part keys), optional `toxin`
(`ToxinTag`), and the reserved `range`.

- **`resolveSite(mover)`** picks the first `siteSelector` entry the mover
  actually has (the `GlassAlley` FOOT_SITES first-match scan, gated on
  `isVitals`); `null` for a non-woundable mover or a non-matching anatomy —
  the graceful "no cut" no-op (a non-biped over a foot-spike takes no wound).
- **`toInflictSpec(mover)`** returns the exact discriminated `InflictSpec`
  `ConditionApi.inflict` consumes: a `ShockInflictSpec` for `shock` (the
  `energy` scalar read as amperes, the current skips the covering fold — the
  circuit was already resolved by the [electricity](./electricity.md)
  substrate), else an `EnergyInflictSpec` for a mechanical channel (the
  covering stack resolves severity + type). `null` when no site resolves.
- **The `range` field — seam ADOPTED.** `'contact'` means the trap harms
  whoever physically meets it; **`'ranged'`** means the delivery crossed a
  band gap (a thrown flask, a dart from the far wall) and had already
  found its victim when it resolved. `resolveTraversal` therefore skips a
  ranged delivery: walking through where it landed is not what springs
  it. See [ranged.md](./ranged.md).

  *(A LINGERING residue that does harm the next person through is a real
  feature — it needs a lifetime/expiry model hazards do not have yet, and
  it defers with cover.)*
- `from()` / `toJSON()` round-trip the value-object through persistence
  capture (`HazardMixin.setDelivery` normalizes a raw seed object or an
  instance).

## `Trap` + the trigger hook

**`Trap = HazardMixin(DetailedMixin(Thing))`** (`lib/hazard/Trap.ts`, the
`Bandage = DressingMixin(Thing)` / `Coin = GlobbableMixin(Thing)` precedent
— a Thing plus its capability mixin). Because `Thing` already composes
`ConcealableMixin`, a `Trap` is concealable out of the box.

**Traversal trigger** — from `Mobile.traverse` at the post-move `onEntered`
site (mode in hand), a `MixinApi.isHazard` scan over the entry's candidates
(the destination if it's a hazard host, its deployable hazard contents, or
the traversed `exit` itself) calls `hazard.resolveTraversal(mover, mode)` on
each. It fires on **walked/ridden/conveyed** entry only — teleport arrival
uses `autoSenseOnArrival`, a different path (the `GlassAlley` `onEntered`
semantics, generalized). A `hazardSeen` set dedups the scan.

**Interact trigger** — `OpenController` calls `opened.resolveInteract(actor)`
when opening a container; a plain (non-hazard) chest is a no-op, an armed
`interact` hazard springs (the trapped-chest case).

**`HazardActivity`** (`lib/hazard/HazardActivity.ts`) is one
`DurativeActivity` with two registered types — `HAZARD_PIN_TYPE` (holds the
victim's `body` for `hazard.pinSeconds`) and `HAZARD_DISARM_TYPE` (holds the
disarmer's `hands` for `hazard.disarmSeconds`, barge-in-abortable so an
interrupted disarm defuses nothing). The semantics differ only in slot +
completion, so a single configurable activity avoids two near-identical
files.

**Player-placed traps + accountability (stealth & deployables build).** A
`HazardMixin` carries a durable `placedBy` field — empty (`''`) for an
authored/environmental hazard (a dungeon pit — no culpable placer), the
placer's `templatePath` when a player deployed it via `arm` (see
[stealth.md](./stealth.md)). When set, `deliverHarm` appends a single
`harm` row to the unified accountability ledger (`AccountabilityApi.record`,
co-located with the `inflict` chokepoint — the producer that knows the
consent: a snare is never agreed to), from which `crime` derives on a
non-consenting sentient. An authored trap (`placedBy === ''`) appends
nothing — an unlucky pit is not a crime. See
[accountability.md](./accountability.md).

## `disarm` — verb machinery on the controller, interaction on the object

`disarm <trap>` (`cmd/device/disarm.yaml` + `DisarmController`, the
**`device`** category = "operating a mechanism"). The controller keeps
**only** the command machinery: target resolution, the **found gate**
(`PerceptionApi.hasDiscovered` — you must perceive a trap to defuse it, so
`search`/`examine` it first), the armed gate, the costed `HazardActivity`
(`HAZARD_DISARM_TYPE`, warming the `awareness` band so the completion reads
a live snapshot), and the disarmer's `awareness`-graded `ActSignature` deed.
The **interaction itself** — the state flip and the narration — is
`hazard.disarmBy(actor)` on the mixin.

## The `introduceToxin` metabolism seam

`MetabolicMixin.introduceToxin(type, amount)` (`lib/metabolism/Metabolic.ts`)
is the bloodstream seam a poisoned dart / needle uses — it injects a toxin
dose **directly onto the body burden, past digestion**. Unlike `ingest`,
there is no digestion pool + absorption curve: the dose lands immediately,
and the banded `Condition` then reads live off the burden
(reconcile-on-read), exactly as an eaten toxin's does. It is *how the dose
arrives*, not a new toxin model. See [metabolism.md](./metabolism.md).

## The `/obj/traps/` generics + the Sunken Delve demonstrator

The **trap taxonomy is authored data, not subclasses** — a spike pit, a
poisoned dart, and a pressure-blade are all one `Trap` with different
`delivery` / `trigger` / `traverseConsequence` field combinations. Three
**generic trap objects** live in `/obj/traps/` (`spike-pit`, `step-dart`,
`pressure-blade`) and are cloned into rooms via `populates`:

- **`spike-pit`** — `trigger: traversal`, `delivery: { channel: point }`,
  `traverseConsequence: drop` to an authored `pit-below`, `groundTriggered`
  (fly/swim clears). `concealment: hidden`.
- **`step-dart`** — `trigger: traversal`, `delivery: { channel: point,
  toxin: { type: venom } }`, no movement consequence. A puncture through the
  covering stack (a boot mitigates) plus a toxin dose past digestion.
- **`pressure-blade`** — `trigger: traversal`, `delivery: { channel: edge }`,
  `traverseConsequence: trip`.

The **demonstrator is the "Sunken Delve"** — a **sub-area of the
newbie-wilds locality** (`seeds/domain/newbie-wilds/delve/`), its own
`CartesianZone`, hung off the newbie-wilds treeline by a **single ordinary
ground exit** (vestibule → trapped corridor → vault; the traps clone in via
`populates`, plus a concealed shortcut `Exit` and a `hidden-cache` concealed
`Thing`). Every secret has an obvious alternative — the far room is reachable
by a plainly-visible path; traps wound/redirect but never hard-gate.

> **Not a `domain/traps` sphere.** The plan proposed a fresh `domain/traps`
> sphere; in review that was rejected — *"traps isn't a domain."* The
> generic trap *objects* are content-agnostic `/obj/traps/` generics; the
> *demonstrator* is content, homed under the newbie-wilds locality it
> belongs to.

The delve host was chosen the way [harm.md](./harm.md)'s GlassAlley failure
teaches: the treeline stands up via the char-gen spawn override, not a boot
cascade, and asserts no exit count, so an added exit pulls this sphere into
**no** standup/fast-travel test (the three ways GlassAlley broke — cross-
domain standup cascade, the exit-count assertion, the fast-travel cascade —
all avoided by construction; the delve composes no `FastTravelMixin` and
appears in no TPA manifest).

### The `dropDestination` placement-coupling caveat

The `drop` consequence's "below" is an **authored template-path redirect**,
not computed geometry — a designer names where down is. A fall is modeled as
a **containment move** to that location (`ContainmentApi.move`), *not* a
door-traverse, so it **never re-enters `Mobile.traverse`** and can't trigger
another trap on the way down. `sprung` is already set before the relocation
(the reentrancy guard), so even a trapped below-room can't re-fire *this*
hazard. The Sunken Delve's `pit-below` climbs back up, so a fall wounds and
delays but never strands. Left unset, a `drop` inflicts the fall blunt but
relocates no one.

## AppSettings dials

`hazard.pinSeconds`, `hazard.dropFallEnergy`, `hazard.disarmSeconds`. Per-
trap **wound energies** are authored content data (a trap's energy is
authored like a weapon's), **not** dials — only the global consequence
constants are dials. (The concealment/detection dials — including the
`awareness`-graded gate a trap resolves against — live in
[concealment.md](./concealment.md).)

## Deferred

- **Ranged / remote delivery** — the `HazardDelivery.range` reserved seam;
  the ranged build grafts distance onto the same spec for weapons *and*
  traps at once.
- **`proximity` / `timer` / `remote` triggers** — authored-representable
  descriptor values, unhandled.
- **Resettable / rearming traps + a trap crafting economy** — one-shot v1.
- **Alarm/signal traps** (effect = an NPC alert; wants the deferred
  detection/response wave), **area/gas traps** (the reserved ranged/AoE
  seam), and **magic traps** (the magic build).
- **Migrating `GlassAlley` onto `HazardMixin`** — an optional, skippable
  proof the one-off is retired; left as a class + integration fixture (an
  *obvious* underfoot hazard is always perceived → always avoided, which is
  a different case from a *spottable* trap, so unavoidable-glass ≠
  spottable-trap and the two coexist).

## Cross-references

- [concealment.md](./concealment.md) — the detection gate a trap resolves
  against; `search`/`disarm`, the `awareness` Discipline, the care↔speed
  axis.
- [harm.md](./harm.md) — `ConditionApi.inflict`, the covering stack, and
  the `GlassAlley` one-off this substrate generalizes.
- [materials-response.md](./materials-response.md) — `Channel`, the
  `InflictSpec` shapes, the covering-stack mitigation.
- [metabolism.md](./metabolism.md) — `introduceToxin`, the bloodstream seam.
- [electricity.md](./electricity.md) — the `shock` channel a hazard can
  deliver.
- [locomotion.md](./locomotion.md) — the traverse seam the trigger fires at;
  the `sneak`/`run` care↔speed modes.
- [activity.md](./activity.md) — `HazardActivity` (pin / disarm) and
  `SearchActivity`.
