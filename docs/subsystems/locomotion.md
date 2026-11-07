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
| `Climbable` / `Swimmable` / `Flyable` | `lib/locomotion/{Climbable,Swimmable,Flyable}.ts` | Host capability mixins. Each exports its own `*_CAPABILITY_PROP` for the per-mode skill gate. ⚠ Nothing composed `Climbable` until the nutrition-and-fitness build shipped `platform/thing/Ladder` (`ClimbableMixin(Good)`) and rejection's winze ladder — the first climb in the game's history. `fly.yaml` is held on `base-class-narrowing-slate` (no `media: ['air']` exit, no flying species, so the refusal would point at nothing). ⭐ `swim` is LIVE since the maritime build: a `platform/location/Pool` is the first composer — the place IS the water (review, 2026-10-13, which retired the interim `StillWater` thing: the reachability sweep had already reversed one water-thing, `OpenWater`, and maritime reintroduced the shape) — and `ShoreMixin` affords the verb beside any water, the far-side rule letting you swim from a shore INTO a pool. `Flyable` still composes nothing |
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

⚠⚠ **`walk` was afforded by NOTHING until the reachability sweep**, and
this paragraph is where it should have been caught. `MobileMixin.self`
carried `go`, `sneak` and `run` and not the middle one, so a player who
took `sneak.yaml`'s own advice (`set movement.defaultMode sneak`) had no
verb to walk a single exit back — `go` was the only way out of the
setting. Two of the three paces worked and the one this paragraph is
named for did not.

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
input: while swimming you are at the SURFACE, and breathe whichever of
the water and the air over it your body can (a fish the water, a person
the air). Going under — and drowning while swimming — is the underwater
build's.

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

⭐ **`allModes()` is a path glob, not a world scan.** Every mode is
declared under one branch (`/platform/idea/LocomotionMode/*` — eleven
rows), so the registry's existing path trie already knows them and no
new index is needed. It used to be `world:[class.LocomotionMode]`, which
read every object in the realm to find eleven singletons. ⭐ **The rung
matters more than the mechanism**: when a roster's content lives under
one branch, the path index is the cheapest honest answer, and it is the
first thing to reach for before an index is even considered. The
`instanceof` narrowing stays, so a stray row under the branch cannot
widen the vocabulary.


```
modeOf(nameOrPath)           → LocomotionMode | null
modeOfOrThrow(nameOrPath)    → LocomotionMode (throws if not loaded)
allModes()                   → readonly LocomotionMode[]  (a PATH GLOB — see below)
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
engageAround(actor, mode, exit, action) → Promise<TraversalGuard>  (internal to Mobile.traverse)
defaultModeFor(actor)        → string (chain: setting → bodyplan → 'walk')
traverseWithDefault(actor, exit) → Promise<TraversalGuard>  (= actor.traverse(exit, defaultModeFor(actor)))
```

## ⭐⭐ The one crossing — `Mobile.traverse`

Every self-powered move — a player's `go`/`climb`/`ride`, an NPC brain,
a `Journey` leg — is one call, `actor.traverse(exit, mode)`, and it owns
the whole sequence. Callers choose the exit and the mode; the crossing
decides whether, and does it. A refusal is **returned** as a
`TraversalGuard`, never thrown.

| step | what | who can hook it |
|---|---|---|
| 1 gate | `LocomotionApi.canTraverseExit` — the cascade below; the exit's own `canTraverse` is **async**, so a ford reads its river, a pass its snow and an ice crossing its sheet *as it is asked* | the exit's `canTraverse` (`@hook`); the rows (modes, body plans, enablement hosts) |
| 2 the mover's say | `beforeTraverse(via)` → `TraversalGuard` — may act and refuse | any mover mixin; **combat's flee** is the first (a parting shot, a pin that holds you, gate `breakaway`) |
| 3 passthrough | a ride/drive mode hands the crossing to the conveyance host, which runs 1–7 under its own mode | — |
| 4 engage | `engageAround`: engaged mode, then exertion + limp **only if the crossing happened** | — |
| 5 apply | `exit.applyTraversal(mover)` — an exit that handles the crossing itself (the wardrobe) | the exit (`@hook`) |
| 6 the places' say | `canExit` on the source, `canEnter` on the destination (`VetoResult` → gate `veto`) | the rooms |
| 7 the move | announce, `ContainmentApi.move`, the conveyance ripple + the tow, `onExited`/`onEntered`/`onTraversed`, hazards, concealment, the arrival sense | the rooms + the mover (`on*`) |

⭐ **One question, one hook.** *May this mover pass, now?* is
`Exit.canTraverse`, and it is async so it can read whatever moves. A
reader with no mover in hand — fire spread, the scope walk, inverse
wiring at load — asks `isBlocked()`, which stays sync and is the exit's
last-known summary (a ford's last read of its river). (A separate
`prepareTraversal` step lived here briefly while `canTraverse` was sync;
it was the same question asked in two halves, and it is gone.)

**A carried rider** (`opts.carriedBy`, the ripple's) skips 1–4: it is
carried, not walking — only the exit's own `canTraverse` under the
host's mode, and it never exerts.

⚠ **Why one place (the climate build, 2026-10-10).** The controller
used to run the gate and the flee itself and then call `traverse`, which re-ran the
exit's gate as a backstop — two layers each deciding, and an NPC brain
or a `Journey` leg skipped the controller's half entirely (no body-plan,
enablement or haulage gate, and no flee: an NPC walked out of a fight
free). The ford refreshed its state in `applyTraversal`, after the gate,
so the first crossing after the river rose went through. Now the
controller resolves, calls, and renders.

⭐ **The host hears of it** (cordage D11): on an ok crossing in a mode with
an enablement mixin, the guard carries the host the enablement walk found
(`TraversalGuard.host`, threaded past haulage in `canTraverseExit`), and
`traverse` calls `host.afterEnabledTraverse(actor, direction, exit)` inside
the engagement, after the move and its witnesses. Climbable, Swimmable and
Flyable default it to nothing; a rope at a drop takes its wear, parts or
spills there. The belay (`BelayEngagement`, `belay <person>`) catches the
fall a rope causes — see [cordage.md](cordage.md).

## Mode-gate cascade

`canTraverseExit` runs five gates in order; first failure surfaces:

1. **body-plan** — `BodyPlan.locomotionModes` must include at least
   one of the mode's `requiresBodyPlanMode`. A body with no anatomy to
   consult passes (a non-Organism, or an Organism with no body plan —
   since every NPC now runs this gate, a species-less NPC would
   otherwise stop walking). Sessile plants have a body plan with empty
   `locomotionModes` — rejected here.
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
5. **haulage** — a hitched cart must be able to follow: the cart's
   passage mode must be admitted by the exit's `media` and the exit
   wheel-passable (`terrain`), and the draft within the hauler's strain
   ceiling (`breakaway`). For a ridden move the HOST hauls, and since the
   host now runs its own crossing this gate reads the horse, not the
   rider.

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

The note's `gate` vocabulary is the closed union
`'exit-mode' | 'posture' | 'body-plan' | 'enablement' | 'capability' | 'no-conveyance' | 'blocked' | 'door' | 'terrain' | 'breakaway' | 'veto' | 'encumbrance' | 'locked' | 'unbuilt' | 'unpublished'`.
⚠ The last four were guards with no word of their own that fell to the
mapper's `exit-mode` default (storage-and-ice, 2026-10-17): a laden
swimmer refused *"You're carrying too much to swim."* read for three
drive runs as a way that does not swim. `mapGate`'s default is still
`exit-mode`, so a new guard must add its word to the union AND the
mapper.
A `blocked` refusal renders the exit's own `reason` when it has one (a
ford, a snowbound pass), else *"The way is blocked."* ⭐ An `exitMode`
refusal names the way that works, from the exit's `media`
(*"You can't walk that way — you'll have to climb."*) — the card's exit
link sends `go <dir>`, and out of the river that used to say only *"You
can't walk that way"* beside steps you have to climb (underwater walk).
The `mode` is the short LocomotionMode name (`'walk'`, `'climb'`, …).

The gate names are kebab-case on the wire even though
`TraversalGuard.gate` uses camelCase internally (`bodyPlan`,
`exitMode`, `noConveyance`); `emitRejection` does the mapping. See
[response-envelope.md § Notes](./response-envelope.md) for the full
note shape and auto-escalation rule.

### ⭐ `swim` with no target — *swim here* (storage-and-ice, 2026-10-17)

`swim.yaml`'s `target` is optional. Bare `swim` lets go of the floor and
swims where you are: the room's water surface decides —
`WaterSurface.goIn(actor)` ([ground.md](./ground.md) § The water's
surface) needs open water (`openWaterAreaM2() > 0` — a cut channel, a
hole, an open pool) and you on the surface, then calls `giveWay(actor,
true)` — the deliberate drop, the gentler scene, nothing broken in the
ice — and returns its refusal otherwise (*"There is no open water here to
swim in."*, *"You are already in the water."*); `SwimController` only
renders. ⚠ Bare `stand` back out stays the posture verb's own act:
`transferPosture` is gated to the posture verbs, so no floor can stand
you up. A **typed** way that binds nothing
(`swim north` at a bank with no north) is the crossing, refused *"You
can't swim that way."* — not the bare branch. In the water you are
immersed by the place's say (thermal.md § Immersion); getting out is
bare `stand` (posture.md).

## Engagement lifecycle

⭐ **A traverse is work** (nutrition-and-fitness W1). After a successful
self-powered traverse `engageAround` calls `actor.exertTraverse(mode)`
(`lib/exertion/Exerting.ts`): the walk's watts × the mode's
`costMultiplier` (the first reader that knob has ever had — walk 1.0, run
2.0, climb/swim 2.0, sneak 1.0) × the load factor. The body debits only
the excess over what it can sustain, so a walk is free and a run costs a
fresh body 12 % an exit; `canSustainPace(mode)` is what the verb layer
asks to break a run to a walk (W2).

`engageAround(actor, mode, exit, action)` sets `actor.engagedMode = mode`,
runs the action, then conditionally clears engagedMode:

⭐ **The far side** (maritime review, 2026-10-13). When the near side has
no enablement host, `canTraverseExit` asks the exit's DESTINATION: a
destination that itself composes the mode's enablement mixin (a `Pool`)
admits the mode, with its own axes, difficulty and the load veto. Only
the place counts, never its contents — you swim INTO water, but a ladder
at the foot of a shaft does not make the top climbable.

- **Transient** (cleared): walk + the vehicular modes (no enablement-
  mixin to remain "in" at the destination), OR climb/swim/fly when
  the destination doesn't compose the enablement mixin AND contains
  no Containable that does.
- **Persistent** (kept): passthrough modes — the rider stays engaged
  in `ride` while occupying the mount slot; `Slotted.vacate` fires
  the `Slottable.onSlotReleased` witness which `Mobile.onSlotReleased`
  uses to clear engagedMode on dismount. Or climb/swim/fly when the
  destination is also enabled (climbing from one face to another).

A refused crossing (the guard `ok: false`) exerts nothing. A throw
propagates; the `finally` clause still clears engagement for transient
modes, so a failed traversal doesn't leave a stale `engagedMode` behind.

## Duration lives in the Journey, not the traverse

A traverse has no duration in the kernel. `go north` is instantaneous
and must stay so — a duration on ordinary movement would put a real-time
toll on every step in the game. Where movement *does* take time (a wagon
to the next town) the time belongs to the transport pack's Journey
([logistics.md § The Journey](./logistics.md)): a sustained engagement on
the driver whose every beat issues the same `traverse` a player's `go`
does, spending `Exit.edgeMinutes × 1/LocomotionMode.speed × loadFactor`
game minutes per leg. `Exit.edgeMinutes` is authored per EDGE — an event
budget, not a metric: no `Exit.length`, nothing derived from `coords` —
and nothing in the kernel reads it. `speed` is the relative multiplier
vs walk (walk 1.0 · run 2.0 · sneak 0.5), not m/s, and the Journey is its
one duration consumer. The `defaultDurationMs` / `durationOverrideMs`
fields the locomotion-as-activity slate designed were never built;
pedestrian durative movement stays open there.

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

## Mobile.traverse returns its refusal

`Mobile.traverse` RETURNS the first failing step's `TraversalGuard`
(see **The one crossing** above); it no longer throws `ContainmentError`
on a gate, and a `canExit`/`canEnter` veto comes back as gate `veto`.
Brains read `guard.ok` (`follows` remembers a leader only on a step that
happened; `stocks` reports a stranded keeper).

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
4. ⭐⭐⭐ **CONFER THE VERB.** A view no `commandContributions` static
   names is dead YAML and the verb is simply unknown to every player
   (`command-routing.md:378-381`). There are two shapes, and which one
   you want follows from whether the mode needs an enablement host:
   - a mode anybody can use anywhere is a **`self` contribution of the
     MOVER** — `walk`, `sneak` and `run` are `MobileMixin.self`;
   - a mode that needs something in the room is an
     **`environment`/`peers` contribution of the ENABLEMENT HOST** —
     `climb` on `platform/thing/Ladder` is the shipped exemplar. *The
     instrument affords the verb.* ⚠ `swim` and `fly` have no host and
     their views are declared `unreachable:` rather than conferred; the
     enablement host for water is `navigable-water-slate`'s to choose.

   ⚠⚠ **This step is new because it was MISSING, and the omission cost
   four verbs.** `climb`, `swim`, `walk` and `dismount` each shipped with
   a view, a controller, a mode row and a body-plan entry, and were
   afforded by nothing — `walk` for the whole life of the subsystem,
   while `sneak` and `run` beside it worked. `lint:reachability` refuses
   a view in that state now, so step 4 is enforced rather than
   remembered.
5. Update body-plan seeds that should permit the mode (add to
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

---

- **2026-10-18 (storage-and-ice, MR !378).** Bare `swim` (*swim here*);
  the gate vocabulary gained `encumbrance` · `locked` · `unbuilt` ·
  `unpublished`.

## River build (2026-10-21)

In moving water the direction words are the river's — `swim upstream`,
`swim downstream`, `swim left`/`right` to a bank — ways worked out when
taken (`ReachExit`). Finding your feet in the shallows drops a water mode for
walk (`WaterSurface.occupy`), so the way out is waded. See
[river.md](./river.md).
