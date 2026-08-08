# Electricity — the `shock` channel + conduction-spread

Electricity adds **`shock` as a materials-response channel** and the
**conduction-spread** that makes it electricity and not a reskinned damage
type: an honest Ohm's-law model (`I = V/R`, *current* is what harms)
resolved over a conductive-contact graph with a ground path, so a shock
flows through water, betrays metal armor, and spares the insulated. Built
for its own sake (hazards, stun weapons, electrocution); it is also the
prerequisite substrate for the **Lightning** frontier noun in the magic
build. Magic is deferred — a spell is just another source imposing a
potential into the same graph.

Consumes: [materials-response](./materials-response.md) (the channel vocab +
the response-fn home), [harm](./harm.md) / [vitals](./vitals.md) (the
`inflict` door, the `heartRate` death seam, conditions),
[perception](./perception.md) (the `Audible` gather-walk shape),
[bulk](./bulk.md) (the `Floor` surface-pool), [quantities](./quantities.md)
(new units), [app-settings](./app-settings.md) (dials).

## The model at a glance

- **`shock` is a `Channel`** (`lib/material/Channel.ts`), but it does **not**
  resolve through the mechanical energy-attenuate covering fold. The three
  mechanical channels (edge / point / blunt) are their own closed subtype
  `MECHANICAL_CHANNELS` — the `Construction` shape tables and the
  materials-response fold key on *that*. `shock` resolves by **circuit**
  (`I = V/R`), reading **conductivity**, not hardness. `Channels.isChannel`
  accepts it; `Channels.isMechanicalChannel` does not.
- **Conductivity is a real `Material` property** —
  `electricalConductivity: Quantity<'S/m'>`, a strict-typed sibling of
  `hardness` / `toughness`, marshalled, authored in the base-library content
  pack (copper ≈ 6×10⁷, salt water ≈ 5, flesh ≈ 0.2, fresh water ≈ 0.01,
  wood ≈ 1×10⁻⁴, rubber ≈ 1×10⁻¹³ S/m). New quantity units `V` / `A` / `Ω` /
  `S/m` (+ `kV` / `mV` / `mA` / `kΩ` / `MΩ` coursework scales with converters
  to the SI base).
- **The Ohm's-law primitives live on `MaterialApi`** (the materials-response
  home): `ohmsCurrent` (`I = V/R`), `jouleHeat` (`P = I²R`, the loss term —
  the deferred Joule→fire seam, shipped-but-unread), `bodyResistance` (flesh
  σ + nominal geometry → dry ≈ 100 kΩ; a `wet` flag divides by ~100),
  `contactResistance` (a material's σ → a series resistance at a node — a
  conductor ≈ 0, an insulator ≈ ∞), `seriesResistanceOfCoveringStack`, and
  `resolveShock` (current → a local contact `burn`). Pure, dial-parameterized;
  the SHAPE is code, every MAGNITUDE is an `electricity.*` AppSetting.
- **`inflict` branches on channel.** `ConditionApi.inflict` intercepts
  `mechanism:'shock'` FIRST and routes to a shock path that **skips the
  covering-stack fold** — the path resistance was resolved upstream, so it
  maps the **current through the victim** straight to a contact burn. The
  mechanical fold (`inflictThroughStack`) and the thermal/tearing passthrough
  stay byte-identical. `InflictSpec` is a discriminated union:
  `EnergyInflictSpec` (magnitude = `energy`) vs `ShockInflictSpec`
  (magnitude = `current: Quantity<'A'>`).
- **The conduction walk** is the gated `ElectricityApi` / `ElectricityLogic`
  pair (`/obj/api/electricity`). `conduct(source)` builds the conductive
  contact graph of the source's location, resolves per-body potentials
  (live contact + ground path), divides current toward the ground sink by
  Ohm's law, and inflicts each bridged body — the `Audible.emit` /
  `AudienceGather` precedent (a source pushes an effect out over a graph and
  collects one arrival per affected body), but the graph is conductive
  contact and it divides toward ground. Faction-blind — everyone bridged
  takes it.

## The source — `EnergizedMixin`

A source is **a node held at a potential** (`lib/electricity/Energized.ts`):
`getVoltage(): Quantity<'V'>`, marshalled. A downed live wire, a stun baton,
the deferred wall socket, and the magic Lightning bolt all compose this same
mixin — a source is never a bespoke special case. That is the
**generalizable seam** the unified-physics bar demands: the grid plugs in
additively because a fixture-held-at-potential is just another `Energized`
node in the same graph. Composing `SwitchableMixin` gates it live/dead (a
switched-off wire imposes 0 V). `MixinApi.isEnergized` narrows it. Since
the combat-hooks build, `EnergizedMixin` also composes
`CombatReactiveMixin` — an energized wielded weapon (or shock-innate
creature) delivers its shock through the combat instrument seam's
`augmentInflict`, not an engine branch (see
[combat-hooks.md](./combat-hooks.md); harmless for a non-instrument
source like a `LiveWire`, which is never resolved as an instrument).

## The graph — grounding, insulation, potential difference

`ElectricityLogic` (module-private free functions) builds a
`ConductiveGraph` from a room:

- **Ground** is the room's modeled `Floor` (an `Adornment` fixture carrying a
  surface-bulk slot). Its conductive surface **pool** (salt water) bridges
  everything standing in it; a metal floor's own material bridges too.
- **Live contact** — a body is live iff it directly holds/touches a source,
  OR is co-immersed in a conductive pool that also holds a live source.
- **Ground path** — a body is grounded iff it stands in the conductive
  medium and is **not insulated**: rubber boots (an insulating covering on
  the *foot* parts specifically — a plate breastplate does not insulate the
  feet), a raised dry step, or an insulating floor break it.
- A body is shocked iff it bridges a live potential **and** a ground path.
  So **bird-on-a-wire** (touching only the live node, no ground), the
  **one-hand rule**, and **insulated positioning** are true and emergent, not
  scripted. Potential is **computed at resolution time** (walk from source:
  connected-to-source = live, connected-to-ground = 0 V), never stored — so
  live-ness is invisible except via the sensory cue / `analyze`.

Current division is **simple conductance-to-ground division with dials**
(D1), not a full Kirchhoff mesh; v1 divides each bridged body independently
(the faction-blind pool hazard hits everyone). `divideCurrent` is the single
seam a future mesh solver replaces without changing callers. The edge set
(D2) is **containment / resting-on-surface / co-immersion in a conductive
pool**; hand-chains, damp-not-pooled floors, and humidity are deferred.

## The armor inversion (emergent) + wet-skin

`R_path = pathResistance(bodyResistance(wet), worn covering)`, with **no**
`isElectrical` narrowing:

- A **conductive** worn layer (metal plate/mail) spreads the current over the
  body and bypasses the high-resistance skin contact — multiplying the body
  resistance by a `<1` factor (`electricity.armor.conductiveSkinFactor`), so
  a **plate-armored body takes MORE current than a bare one**. This is the
  "metal spreads current → lower protection" distribution effect rendered as
  through-current (the observable criterion); multi-site entry/exit
  distribution is the deferred seam.
- An **insulating** layer (rubber / leather / wool) instead ADDS its series
  contact resistance, collapsing the current — real protection.

**Wet skin** lowers body resistance ~100× (the real reason water is deadly),
derived at resolution time (nothing stored): co-immersion in a conductive
pool, OR standing in a SkyExposed scope under active **rain** (`WeatherApi`,
a sync/defensive read). A stored drying gauge is the flagged upgrade.

## The temporal model — event + reconcile-on-read sustain

Contact is an **event** (one conduction-walk resolution). A *persisting*
closed circuit becomes a **`SustainedShock`** condition — a third
`ActiveCondition` kind alongside `Trauma` / `AfflictionRecord` — that
integrates current × time **lazily on read** (the harm-bleed idiom,
presence-frozen), reusing `VitalsMixin.reconcileConditions`'s
first-touch / linkdead / far-past machinery verbatim. It accrues a contact
burn, re-verifies the circuit each read (a cheap `currentThrough` probe),
and is relieved the moment the circuit breaks — **unless tetany holds it
closed**. `ElectricityApi.conduct` mints/upserts it; `isBeingShocked()` /
`isTetanized()` are the predicates.

**Tetany** (`SustainedShock.tetany`, latched at/above the tetanic band)
holds the circuit closed regardless of volition ("can't let go") — the
sustain keeps accruing even after the body would otherwise let go. The
volition-verb gate (block `release`/`drop`/`move` while tetanized) is
exposed via `isTetanized()`; wiring it into every verb is a light follow-up.

## The vitals coupling — the electrocution death seam

Current bands drive outcomes (`electricity.letGoAmps` ≈ 10 mA,
`electricity.fibrillationAmps` ≈ 100 mA):

- perception (tingle, below the burn threshold) →
- **let-go / tetany** (a paralysis that prevents release — a weaponizable
  disarm) →
- **fibrillation** — a sufficient sustained current drives `heartRate` toward
  arrest (`electricity.heartRate.arrestDrivePerSec`); when it floors below
  the survivable band, the body dies with **`cause = 'electrocution'`**,
  stamped via the vitals seam (`setCauseOfDeath` + `setLifecycleState`),
  **never** `StuffApi.destruct`. This **drives the previously-undriven
  `heartRate` death seam** — a `getVitalSign('heartRate')` read arms it,
  mirroring the `bloodVolume` bleed read. `getConsciousness()` already reads a
  failing heart as unconscious, so the waypoint falls out for free.

Local `burn` trauma lands at contact sites (the covering fold is skipped —
metal armor does NOT protect against a shock).

## Sources + the demonstrator

- **`LiveWire`** (`lib/electricity/LiveWire.ts`) — a `Thing` + `Energized` +
  `Switchable`, ~120–240 V. The downed cable.
- **`FloodedCell`** (`domain/substation/FloodedCell.ts`) — the primary
  demonstrator (the duncan-hall cistern precedent: a flooded hazard room a
  body ENTERS by walking). A proper **`CartesianLocation`** with coordinates,
  living in its own self-contained **`CartesianZone`** — *The Drowned
  Substation* (`/domain/substation`), electricity's own home (and where the
  deferred power-grid content grows), so it never pollutes another themed
  area. Provisions a salt-water-pooled `Floor` + a `LiveWire` at standup;
  `onEntered` → `ElectricityApi.conduct`. Teaches the whole model with no
  magic — barefoot → shocked, rubber boots / dry step → unharmed, two allies →
  both shocked. **Reachability**: seeded with coords in its zone + an
  end-to-end integration test; no cross-area inbound exit is wired this cycle
  (the treeline precedent — keeps content-area standup clean; reachable by
  teleport / a future travel node).
- **`StunBaton`** (`lib/electricity/StunBaton.ts`, authored as a template at
  `/domain/substation/stun-baton` and `populates:`-placed in the cell) — a
  `Weapon` + `Energized` + `Switchable` (the combat toe-hold). A landed hit
  routes through **`ElectricityApi.shockContact`** — a **direct two-terminal
  contact** (a taser/baton completes its own circuit through its electrodes,
  so it needs no ground path and no conductive medium) into the same
  `ConditionApi.inflict({mechanism:'shock'})` door, never the mechanical fold.
  The shock rides the **combat instrument seam** (the combat-hooks build —
  see [combat-hooks.md](./combat-hooks.md)): `EnergizedMixin` composes
  `CombatReactiveMixin`, and its `augmentInflict` queues
  `ctx.deliverShock(this)` — the engine drains it right after the mechanical
  inflict (the former hard-coded `isEnergized` branch in
  `CombatLogic.commitInflict` is deleted, the migration pinned
  byte-identical; a switched-off baton still truthfully delivers nothing via
  the `effectiveVoltage ≤ 0` guard, unmoved). The same override serves the
  **shock innate**: an `Energized`-composing creature with a `shock` natural
  attack (the electric eel case, formerly a code comment — now **shipped**
  through `commitShockInflict`'s delivery split, no mechanical primary,
  single-fire by construction). Ships **safed** (`on: false`) — it
  only becomes a source when armed (an armed baton dropped in the pool would
  electrify it — correct physics; safe because it ships off).

## Legibility

- **`analyze electrical <target>`** (the multimeter,
  `AnalyzeElectricalController`) — conductivity / contact-resistance /
  path-to-ground **bands** and the **raw** `Quantity<'V'|'A'|'Ω'|'S/m'>`
  numbers on measurement (banding is presentation, never security). A source's
  potential; a body's grounded-vs-insulated read.
- **Pips** — `ConstructedMixin`'s `responsePipsAugmenter` renders a **shock
  column** on armor: the inversion made legible (a steel breastplate near-
  empty — "conducts, barely protects"; a rubber layer full). Reads the same
  `electricalConductivity` axis the model resolves against.
- Banding tag tables for `A` (current ladder) / `Ω` / `S/m` in the pack's
  `quantity-tags.yaml`.

## The dials

All `electricity.*` AppSettings with real-world-grounded seeded literals —
`electricity.body.dryResistanceOhms`/`wetFactor`/`geometryFactor`,
`electricity.contact.geometryFactor`/`maxOhms`,
`electricity.resistanceFloorOhms`, `electricity.letGoAmps`/`tetanicAmps`/
`fibrillationAmps`, `electricity.burnThresholdAmps`/`burnSeverityPerAmp`,
`electricity.pool.minConductivity`, `electricity.insulator.maxConductivity`,
`electricity.armor.conductiveSkinFactor`,
`electricity.sustain.burnSeverityPerAmpSec`,
`electricity.heartRate.arrestDrivePerSec`.

## Deferred seams (named, not built)

- **Magic `Create·Lightning`** — a spell is another `Energized` source into
  the same graph; chain lightning + caster-in-the-graph + magic friendly-fire
  arrive free.
- **Power / the grid** — this same Ohm's-law physics **scaled up and
  aggregated** (one law, two scales), never a second "power" abstraction. The
  v1 seams (source imposes a potential, a wire = a high-conductivity edge, a
  fixture held at potential) make it plug in additively.
- **AC vs DC** — a real distinction (AC fibrillates, DC throws); v1 ships one
  `shock`.
- **Joule → fire** — the first post-v1 coupling (`jouleHeat` ships now); the
  bridge to the Fire noun.
- Full Kirchhoff mesh division, hand-chains, humidity, multi-site entry/exit
  distribution, the stored drying gauge, the tetany volition-verb wiring, the
  `check-does-nothing` shock-inert lint, a live-ness hum/ozone `Audible` cue.
