# Locomotion

Mode-aware movement: `LocomotionMode` singletons, per-mode enablement
mixins (`Climbable` / `Swimmable` / `Flyable`), the
`LocomotionControllerBase` verb suite (`walk` / `climb` / `swim` /
`fly` / `ride` / `drive`), and the engagement lifecycle that hangs
`engagedMode` on the actor for the duration of a traversal.

## The Cast

| Name | Location | Role |
|---|---|---|
| `LocomotionMode` | `lib/locomotion/LocomotionMode.ts` | Singleton Idea — one per mode. Author-data: speed / noise / body-profile / ground-contact / cost / passthrough / conveyance + enablement mixin names / medium |
| `Enablement` | `lib/locomotion/Enablement.ts` | Shared interface (axes + difficulty + capability gate) implemented by all three per-mode enablement mixins |
| `Climbable` / `Swimmable` / `Flyable` | `lib/locomotion/{Climbable,Swimmable,Flyable}.ts` | Host capability mixins. Each exports its own `*_CAPABILITY_PROP` for the per-mode skill gate |
| `LocomotionApi` | `api/locomotion.ts` | Mode resolution, eligibility, engagement lifecycle, passthrough chain, emission walk, default-mode resolution |
| `LocomotionControllerBase` | `platform/idea/cmd/movement/LocomotionControllerBase.ts` | Abstract base for the six per-mode verbs and refactored `go` |
| `Walk` / `Climb` / `Swim` / `Fly` / `Ride` / `DriveController` | `platform/idea/cmd/*.ts` | Concrete controllers — override `modeName()` and (optionally) `composeRejection()` for verb-templated prose |

`LocomotionMode` extends `SingletonMixin(PropertiedMixin(Idea))`. The
nine v1 modes live at `/platform/idea/LocomotionMode/<name>`.

## Modes (v1)

| Name | Medium | Passthrough | Enablement | Required body-plan |
|---|---|---|---|---|
| walk | ground | — | — | `['walk']` |
| sneak | ground | — | — | `['walk']` |
| run | ground | — | — | `['walk']` |
| climb | vertical | — | `ClimbableMixin` | `['climb']` |
| swim | water | — | `SwimmableMixin` | `['swim']` |
| fly | air | — | `FlyableMixin` | `['fly']` |
| ride | — (null) | ✓ MountableMixin | — | — |
| drive | — (null) | ✓ DrivableMixin | — | — |
| wheeled | ground | — | — | — |
| sailed | water | — | — | — |
| aerial | air | — | — | — |

**The care↔speed axis** — `sneak` / `walk` / `run` are the three ground
*paces*, bracketing `walk` (sneak `speed 0.5` / `noiseLevel quiet` /
`costMultiplier 1.5`; run `speed 2.0` / `loud` / `2.0`). They share `walk`'s
`['walk']` body-plan requirement (any ground mover can sneak or run) and the
ground medium, so every `media: ['ground']` exit admits them with no
re-authoring — an **empty `media` list now admits the whole ground-pace
family** (`walk`/`sneak`/`run`), widened from walk-only. Their detection
edge is **not** a mode field: it's the `movement.attention.{sneak,run}`
AppSettings dials read by `PerceptionLogic.modeAttention`, so sneaking spots
a trap a walk would spring and running blunders into one a walk would notice
(see [concealment.md](./concealment.md)). The `noiseLevel` self-hiding side
is authored-as-data but its observer-detection is the reserved stealth
consumer. Thin `sneak` / `run` verbs (`SneakController` / `RunController`
override `modeName()`); the **persistent pace default rides the existing
`movement.defaultMode` setting** (`set movement.defaultMode sneak`) — there
is no separate `movement.pace` knob (it was redundant with `defaultMode`,
which already selects the mode `go` dispatches under and now accepts
`sneak`/`run`).

Walk / climb / swim / fly are organism modes — actors engage directly.
Ride / drive are passthrough — the actor's slot occupancy on a
Mountable / Drivable host stands in for direct engagement; the host
traverses, the actor rides along via the conveyance ripple. Wheeled /
sailed / aerial are vehicular modes — hosts declare them on
`Drivable.vehicularMode` and engage them when driven.

A mode's `medium` is also a [respiration](./respiration.md) reassess
input: while swimming, the engaged-mode medium (`water`) wins over the
ambient atmosphere — you breathe (or drown in) the medium you move
through — so surfacing/diving re-resolves the air-exchange state.

## Two parallel mode vocabularies

The substrate juggles two string vocabularies for the same conceptual
mode:

- **Full templatePath** (`/platform/idea/LocomotionMode/walk`) — what storage holds
  (`Mobile._engagedModePath`, `Drivable._vehicularModePath`).
- **Short name** (`walk`) — what authoring uses (`Exit.media` →
  medium lookup, `BodyPlan.locomotionModes`, `BodyPlan.defaultLocomotionMode`,
  `LocomotionMode.requiresBodyPlanMode`, Mml prose).

`LocomotionApi.modeOf(nameOrPath)` accepts either form. Internally the
Api normalizes short names to `/platform/idea/LocomotionMode/<name>` and looks up
the singleton via `StuffApi.findByTemplatePath`. `Mobile.isEngagedIn`
accepts both forms too.

## Reference shapes

The substrate exercises every reference pattern from
[ref-shapes.md](../ref-shapes.md):

- **`_engagedModePath`** on Mobile — an identity ref (string-by-path).
  Runtime-only (NOT in `fieldMeta`'s persistent entries): a reloaded actor wakes up
  unengaged. The mode singleton itself persists by templatePath in the
  domain collection.
- **`_vehicularModePath`** on Drivable — an identity ref (string-by-path).
  Persistent: a horse-drawn wagon ships with `vehicularMode: wheeled`
  in its template and round-trips through saves.
- **`defaultLocomotionMode`** on BodyPlan — short name string
  (an identity ref with no leading slash). Resolved by
  `LocomotionApi.defaultModeFor` when no explicit setting is set.

## LocomotionApi surface

```
modeOf(nameOrPath)           → LocomotionMode | null
modeOfOrThrow(nameOrPath)    → LocomotionMode (throws if not loaded)
allModes()                   → readonly LocomotionMode[]
resolveHostMode(host)        → LocomotionMode (engagedMode → vehicularMode → walk)
bodyPlanAllows(actor, mode)  → boolean
postureAllows(actor, mode)   → boolean
exitAllowsMode(exit, mode)   → boolean
canEngage(actor, mode)       → boolean (bodyPlan + posture only)
canTraverseExit(actor, exit, mode, direction) → TraversalGuard
checkEnablement(actor, mode, direction)        → TraversalGuard
findConveyanceHost(actor, mode) → Stuff | null (throws on non-passthrough)
emissionAt(mover)            → EmissionData | null
eligibleModes(actor)         → readonly LocomotionMode[]
engagedMode(actor)           → LocomotionMode | null (non-Mobile-safe)
isTransientEngagement(mode, exit) → boolean
engageAround(actor, mode, exit, action) → Promise<T>
defaultModeFor(actor)        → string (chain: setting → bodyplan → 'walk')
traverseWithDefault(actor, exit) → Promise<void>
```

## Mode-gate cascade

`canTraverseExit` runs four gates in order; first failure surfaces:

1. **body-plan** — `BodyPlan.locomotionModes` must include at least
   one of the mode's `requiresBodyPlanMode`. Non-Organism actors pass
   trivially (no anatomy constraint applies). Sessile plants ARE
   organisms with empty `locomotionModes` — rejected here.
2. **posture** — actor's `Posed.getPosture()` must be in the mode's
   `requiresPosture`. Empty set means no posture gate.
3. **exit.canTraverse** — checks `blocked`, `door`, and `media` (mode
   medium must match the exit's medium list, or empty media → walk-
   only legacy default).
4. **enablement** — for non-passthrough modes with an `enablementMixin`,
   walk the actor's scope for a host that composes that mixin AND
   accepts the direction AND can be engaged (capability ≥ difficulty).
   For passthrough modes, walk slot occupancy for a host composing
   the mode's `conveyanceMixin`.

Each rejection carries `gate` / `mode` / optional `context` so
controllers can render verb-templated prose without re-parsing
`reason`.

### Rejection emission

`LocomotionControllerBase.emitRejection(guard, mode, model, context)`
is the centralized fix point for all six per-mode controllers and the
refactored `go`. It does two things:

1. Fires `Scene.send` at `shell.result` with the
   verb-templated rejection prose (per the concrete controller's
   `composeRejection` override).
2. Emits a `locomotion-gate-failed { gate, mode }` note onto the
   dispatch context.

The note's `gate` vocabulary is the closed eight-value union
`'exit-mode' | 'posture' | 'body-plan' | 'enablement' | 'capability' | 'no-conveyance' | 'blocked' | 'door'`.
The `mode` is the short LocomotionMode name (`'walk'`, `'climb'`, …).

The gate names are kebab-case on the wire even though
`TraversalGuard.gate` uses camelCase internally (`bodyPlan`,
`exitMode`, `noConveyance`); `emitRejection` does the mapping. See
[response-envelope.md § Notes](./response-envelope.md) for the full
note shape and auto-escalation rule.

## Engagement lifecycle

`engageAround(actor, mode, exit, action)` sets `actor.engagedMode = mode`,
runs the action, then conditionally clears engagedMode:

- **Transient** (cleared): walk + the vehicular modes (no enablement-
  mixin to remain "in" at the destination), OR climb/swim/fly when
  the destination doesn't compose the enablement mixin AND contains
  no Containable that does.
- **Persistent** (kept): passthrough modes — the rider stays engaged
  in `ride` while occupying the mount slot; `Slotted.vacate` fires
  the `Slottable.onSlotReleased` witness which `Mobile.onSlotReleased`
  uses to clear engagedMode on dismount. Or climb/swim/fly when the
  destination is also enabled (climbing from one face to another).

Errors from `action` propagate; the `finally` clause still clears
engagement for transient modes, so a failed traversal doesn't leave
a stale `engagedMode` behind.

## Verb dispatch

`go <target>` dispatches under `LocomotionApi.defaultModeFor(actor)`
— a three-tier chain:

1. Actor's explicit `movement.defaultMode` setting (`EnvironmentMixin`
   composers only).
2. Actor's `Species → BodyPlan → defaultLocomotionMode` (Organism
   composers).
3. Universe default `'walk'`.

Birds get `fly` for free; fish get `swim`. Players override per-
character via `set movement.defaultMode <name>`.

Literal mode verbs (`walk`, `climb`, `swim`, `fly`, `ride`, `drive`)
extend `LocomotionControllerBase` with a one-line `modeName()`
override and (optionally) verb-templated `composeRejection` prose.

## Exit.media

The exit-side gate. Authors declare which media (`'ground'` /
`'water'` / `'air'` / `'vertical'`, open vocab) the exit admits;
`allowsMode(modeName)` resolves the mode singleton and matches its
`medium` against the exit's set.

| Exit shape | Admits |
|---|---|
| `media: []` (default) | walk only (legacy backcompat) |
| `media: ['ground']` | walk + wheeled + (future run/crawl/sneak) |
| `media: ['vertical']` | climb |
| `media: ['water']` | swim + sailed |
| `media: ['air']` | fly + aerial |
| `media: ['ground', 'water']` | mixed (beach shore) |

Passthrough modes (`ride`, `drive`) have `medium: null` and are never
admitted directly — the controller substitutes the host's mode at
the gate call site (see `LocomotionControllerBase.execute`).

## Mobile.traverse mode-gate

`Mobile.traverse(exit, mode)` now enforces `exit.canTraverse(this, mode)`
before announcement and throws `ContainmentError` on rejection.
Programmatic-violation policy: player-input paths always pre-check via
`LocomotionApi.canTraverseExit`, so the throw is reached only by
misbehaving programmatic callers (admin tools, scripted NPC AI). The
throw payload carries the structured `TraversalGuard` on
`cause.traversalGuard` for debug surfaces.

## Drivable.vehicularMode — fail-loud

`Drivable.vehicularMode === null` for a host that's about to be driven
is a content-author bug: `LocomotionApi.resolveHostMode` throws. The
template-time authoring contract is: a cart declares
`vehicularMode: wheeled`, a rowboat `sailed`. The previous version
silently fell back to walk, which masked the bug; the new contract
surfaces it loudly in dev.

The companion fix: `Drivable.controllerSlot` defaults to `'driver:1'`
(not `'mount:1'`) so a Stuff composing both `Mountable` and `Drivable`
doesn't collide rider and driver on the same slot name.

## Slot-release witness

`Slottable.onSlotReleased?(host, slotName)` is an optional method on
the `Slottable` interface. `Slotted.vacate` and `Slotted.vacateSole`
invoke it synchronously after removing the candidate. `Mobile`
implements it to clear `engagedMode` when the vacated host composes
the engaged mode's `conveyanceMixin` — a dismounting rider's
engagement clears automatically without any controller-side
bookkeeping.

## Emission

`LocomotionApi.emissionAt(mover)` walks the passthrough chain (depth
capped at 16) to the host whose engaged mode is non-passthrough and
returns its emission data (noise / body-profile / ground-contact +
the resolved host chain). Future trap / detection / sound consumers
read this to ask "what does this mover sound / look / feel like right
now?" — a rider on a galloping horse reads the horse's wheeled-style
emission, not the rider's own walk.

## Authoring locomotion content

Adding a new mode (e.g., `slither`):

1. Author the singleton seed at `seeds/lib/locomotion/slither.yaml`
   with `class: /platform/idea/LocomotionMode/LocomotionMode`, the property values,
   and the appropriate `medium`.
2. If the mode needs an enablement scope, create a `*ableMixin` that
   implements `Enablement`, add it to the `Mixins` registry, and set
   the seed's `enablementMixin` field to the registry constant.
3. Add a verb YAML view (`content/platform/cmd/movement/slither.yaml`) and a controller
   that extends `LocomotionControllerBase` with `modeName()` returning
   `'slither'`. Author a controller seed at
   `platform/content/platform/idea/cmd/movement/SlitherController.yaml`.
4. Update body-plan seeds that should permit the mode (add to
   `locomotionModes`) and optionally bump `defaultLocomotionMode`
   for species whose default movement is the new mode.

## Future work

Forward-looking design surface salvaged from the retired
locomotion slate. None of these are scheduled; each waits for a
content slate that pulls on it.

- **Trap subsystem.** Mode properties drive trap predicates —
  pressure plates fire on `groundContact !== 'none'`, tripwires
  on `bodyProfile === 'upright'`, pit traps the same as pressure
  plates, snares on the combination. Passthrough modes resolve
  through the conveyance chain (a rider trips a tripwire because
  the *horse* is `upright` / `full` ground contact). The trap
  subsystem itself is its own slate when content earns it; the
  locomotion substrate exposes everything traps need to read
  (`LocomotionApi.emissionAt` walks the passthrough chain).
- **Pathfinder.** Each `(location, target, mode)` edge has cost
  derived from mode + per-exit override (today the substrate
  carries `speed` and `defaultDurationMs` is the natural ETA
  source — see
  locomotion-as-activity-slate.md
  for the time-based duration model). Mode transitions in a
  multi-segment path are zero-distance "transition edges." The
  pathfinder is its own slate when route-planning content earns
  it. Stealth-aware planning (cost weighted by `noiseLevel`)
  composes naturally.
- **Auditory detection.** Mover emits a signal; observer compares
  against a threshold. Same shape as `VisionModality.canSee` — viewer-
  aware, modulated by distance / obstruction / acuity. Mode
  contributes the emission level via `noiseLevel`; observer's
  species hearing profile sets the threshold. The channel
  infrastructure belongs to the hearing/perception substrate
  ([senses.md](./senses.md)); locomotion's
  contribution is the `noiseLevel` data and the passthrough
  walk on `LocomotionApi.emissionAt`.
- **`run` / `sprint` / `tiptoe` as separate modes.** Verb-as-mode
  pattern extends naturally — each new singleton at
  `/platform/idea/LocomotionMode/<name>` with shared `requiresBodyPlanMode:
  ['walk']`. Adding the verb is mechanical. Open question whether
  every walk-variant deserves its own mode or whether some
  collapse into a `modifier` parameter on the walk verb; lean
  separate modes (each one is one new template, no framework
  churn).
- **`Crawlable` mixin + `crawl` mode.** Paired with
  locomotion-as-activity-slate.md;
  crawl was deferred alongside sneak when locomotion stayed
  synchronous. Size constraints (a fat character may not fit
  through a vent) live on the `Crawlable` host.
- **`LocomotionMode.extends` for shared defaults.** Run, sneak,
  crawl all share most of `walk`'s property values. A
  template-level `extends` field would cut duplication;
  open question whether the savings justify the
  template-hierarchy complexity. Defer until enough modes ship
  to make duplication painful.
- **`fly` direction vocabulary in open-air zones.** Most rooms
  are 2D; an aerial zone in a Spherical coordinate system wants
  full 3D (compass + up/down + 3D diagonals). Resolve at the
  zone level when aerial content lands.
- **Passthrough emission amplification.** A steel-shod horse is
  louder than a barefoot one. Per-host emission modifier
  (worn-equipment composition? species attribute?) lands when a
  content case needs the differentiation; framework supports
  the seam via the existing `emissionAt` walk.


> ⚠ **Deliberately untagged.** `speed`, `noiseLevel` and `costMultiplier` are numbers, but a locomotion mode is the **player's own control surface** — `sneak` and `run` are things they do. Hiding how your own legs work teaches nothing: you learn sneaking is slower by sneaking, once. `CombatFormation`'s roles and coup rights are untagged for the same reason. The sweep that put `spoiler: 1` on
> `Material`'s and `Biome`'s measurements stopped here on purpose —
> *collapse what the WORLD measures, never what the PLAYER operates.*
> See [wiki.md](./wiki.md) § the reveal model.

---

## Cross-references

- [response-envelope.md](./response-envelope.md) —
  `locomotion-gate-failed` note shape and the gate vocabulary.
- [conveyance.md](./conveyance.md) — Mountable / Drivable hosts,
  conveyance ripple, vehicle design space.
- [spatial.md](./spatial.md) — `Mobile.traverse`, conveyance ripple
  integration, movement messaging.
- [boundary.md](./boundary.md) — Exit substrate, `TraversalGuard`
  shape.
- [slot.md](./slot.md) — slot universe, `Slottable.onSlotReleased`
  witness shape.
- [race.md](./race.md) — `BodyPlan.locomotionModes` +
  `defaultLocomotionMode`.
- [ref-shapes.md](../ref-shapes.md) — identity refs (path-by-string) for
  mode references on actor / drivable.
