# Concealment & detection

The exploration/perception layer: **whether a thing is *there* at all**,
resolved per-viewer. Where belief/recognition hides *identity* (who a
perceived thing is, resolved only *after* you can perceive it — see
[belief.md](./belief.md)), this build hides **presence**. Every
perceivable carries one `concealment` level; a viewer's *effective
perception* resolves it deterministically; a find lands as a permanent,
per-viewer **discovery** in the belief store. It reuses shipped substrate
end to end — the perception viewer-aware pattern, the belief store, the
activity framework, the light pipeline — and adds no parallel "searching
system." **Traps** ([hazard.md](./hazard.md)) are the flagship first
consumer; a **care↔speed movement axis** (`sneak`/`walk`/`run`) is the
risk-dial at the trap seam.

## The gate — `ConcealableMixin` + `ConcealmentLevel`

`lib/concealment/`. Two pieces: a vocabulary value-object and a dumb
carrier mixin.

**`ConcealmentLevel`** (`ConcealmentLevel.ts`) is the monotone band
vocabulary — `'obvious' | 'subtle' | 'hidden' | 'deep' | 'buried'`
(`CONCEALMENT_LEVELS`), ordered weakest→strongest. `obvious` (index 0) is
"not concealed, fully present in every viewer's world"; each higher band
demands more of a viewer's effective perception before the thing resolves.
The `ConcealmentLevels` static holder (the `Channel` / `Grade` /
`WeatherType` precedent — a closed tuple + a thin static surface, no
free-floating predicate functions) exposes `isLevel` / `isConcealed`
(`!== 'obvious'`) / `rankOf` (0…4) / `requirementFor(level)` /
`hiddenDefault()`.

**Shape-vs-magnitude.** The band *names* and their ordering live in code;
the *magnitude* each band demands is an AppSettings dial
(`concealment.level.<band>`), so a designer tunes the detection curve
without a code edit. `obvious` is a hardcoded 0. `requirementFor` reads the
dial with a **seeded-literal fallback** (the harm/electricity dial idiom —
`subtle 2`, `hidden 4`, `deep 7`, `buried 11`) so a pre-warm / unit-test
read is safe. The fallbacks are monotone by construction, as are the
authored dials.

**`ConcealableMixin`** (`Concealable.ts`, `_mixinName =
'ConcealableMixin'`) is the carrier: one authored durable field
`concealment: ConcealmentLevel` (default `'obvious'` — an un-authored thing
is inert, backcompat by construction) plus an authored `concealmentHint`
(the "tell" a nearly-perceiving viewer notices — "a draft," "the bookshelf
sits oddly"; never names the concealed thing's identity — honest fog). The
inter-Stuff method surface is `getConcealment` / `setConcealment` (validated
on set) / `isConcealed` / `getConcealmentHint` / `setConcealmentHint` /
`getDiscoveryKey`. Both fields are `@authorable`; `persistentFields =
['concealment', 'concealmentHint']`.

**Composed onto `Thing`, `Creature`, and `Exit`** — the three perceivable
bases that cover the loose perceivables (a stashed item, a lurking
creature, a secret door). `Character`/`Avatar` inherit it through
`Creature`; a `Trap` inherits it through `Thing`.

> **Divergence from the plan: `Container` does *not* compose
> `ConcealableMixin`.** The plan listed `Container` as a composition point;
> as shipped, only `Thing`/`Creature`/`Exit` carry the mixin. A **hidden
> cache** is therefore realized as a concealed **`Thing`** (a stashed
> pouch — the Sunken Delve's `hidden-cache` seed is a `Thing`), not a
> concealed container. The wire-fog gate still runs on a container's
> *contents* (below), so a concealed item *inside* a plain container is
> hidden correctly; it's the container-as-secret case that a concealed
> Thing stands in for.

### `Exit.hidden` is subsumed, not paralleled

`Exit extends ConcealableMixin(Idea)` (mixin-on-`Idea`, the
`LocomotionMode` precedent). The legacy `Exit.hidden` boolean — a dead-end
"never displayed" flag with **no reveal path** (`setHidden` had zero
non-test callers) — is now a thin view over the band:

- `isHidden()` re-derives as `isConcealed()`.
- `setHidden(true)` raises the band to the migration default
  (`ConcealmentLevels.hiddenDefault()` → the `concealment.hiddenDefaultLevel`
  dial, defaulting to `'hidden'`); `setHidden(false)` clears it back to
  `'obvious'`.
- At construction, `ExitOptions.hidden: true` maps onto the same default;
  an explicit `concealment` band on the seed **wins** over the legacy flag
  (`concealment: 'deep'` buries it further), and an authored
  `concealmentHint` rides the carrier.
- `getObviousExits()` **still drops a hidden exit** (it reads `isHidden()`)
  — the difference is that the reveal path now exists (the bar's sealed
  `north → office` door, a documented dead-end, becomes discoverable).

Because an `Exit` is a runtime instance with **no `templatePath` of its
own**, it overrides `getDiscoveryKey()` to a durable synthetic handle —
`` `${source.templatePath}#exit:${direction}` `` — so a discovered secret
door stays discovered across re-clones. It returns `undefined` when the
source room has no durable templatePath (a shared multi-clone room — the
deferred player-placed-concealment case).

## The detection surface — on `PerceptionApi`, not a new Api

**There is NO `DetectionApi`.** Detection *is* concealment-gated
perception, and the presence face is `PerceptionApi` — a fourth module
would gate an operation the perception face already owns. The surface is
added to the existing **`PerceptionApi`** → **`PerceptionLogic`** pair
(`api/perception.ts` / `obj/api/PerceptionLogic.ts`), siblings of
`canPerceive` / `perceiveAt`:

| Method | Meaning |
|---|---|
| `perceives(viewer, target, attention?)` | The core gate — `true` iff the target is un-concealable, `obvious`, already discovered in belief, or `effectivePerception ≥ requirement`. |
| `effectivePerception(viewer, target, attention)` | `capacity + attention + conditions` (a number). |
| `hasDiscovered(viewer, target)` / `recordDiscovery(viewer, target)` | The `DISCOVERY`-realm read/write sink (below). |
| `hintsFor(viewer, scope)` | The nearly-perceived candidates (`requirement − effectivePerception ≤ concealment.hintCutoff`). |
| `resolveSearch(viewer, scope, depth)` | The active-search resolver — records + returns each newly-cleared candidate. |
| `modeAttention(mode)` | The care↔speed attention a locomotion mode brings to a traverse check. |

**The resolution is pure and deterministic** (no `Math.random`), monotone
in attention to a ceiling set by capacity vs. concealment — the
poker-not-slots discipline the combat fog established. `perceives`
short-circuits `true` for a non-concealable or `obvious` target (the common
path — everything currently visible stays visible) and for anything already
discovered (the per-viewer belief sticks — you never re-resolve a found
secret).

`effectivePerception` sums three terms:

- **`capacity`** = the viewer's `awareness` competence-band rank ×
  `detection.capacityPerBand`. The band read (`AdvancementApi.bandFor`) is
  `async`, but the enumeration seams are sync — so
  `PerceptionLogic.preloadForSenseGate(actor)` (already called per-command
  to warm anatomy + modalities) is **extended to warm the actor's
  `awareness` band into a snapshot** (`awarenessBandCache`, keyed on
  `stuffId`). A cache miss or an unseeded `awareness` Discipline both read
  as the floor band (rank 0 → capacity 0) — the Phase-2-self-contained
  degrade. This is the one place the detection-on-`PerceptionApi` fold pays
  off directly: the sync boundary was already solved by the perception
  face's own preload.
- **`attention`** = the passive baseline (`concealment.passiveBaseline`),
  or an active-search bonus (below), or a care↔speed mode modifier.
- **`conditions`** = a light term. It reuses the per-viewer vision path
  (`VisionModality.perceivedBand`, which threads the viewer's Shadow seam —
  so night-vision / blindness enter here with no second light read):
  neutral (0) at `lit` and brighter, a negative penalty in dimmer bands. It
  degrades to 0 whenever the viewer can't run vision queries or the vision
  singleton isn't loaded (unit fixtures), so detection stays deterministic
  and testable without a light substrate.

### The `DISCOVERY` belief realm

A find is a **per-viewer world-fact**, so it lands in the belief store as a
new realm alongside recognition / identification / regard — see
[belief.md](./belief.md). `BeliefStore.ts` exports `const DISCOVERY =
'discovery'` and adds a bare `found?: boolean` flag to `BeliefPayload`
(flag-by-default). `PerceptionLogic.recordDiscovery` calls `viewer.know(
DISCOVERY, referent, { found: true })`; `hasDiscovered` reads
`viewer.recall(DISCOVERY, referent)?.payload.found`. The referent key comes
from `target.getDiscoveryKey()` (default = `templatePath`; the `Exit`
synthetic handle above). Per-viewer isolation, no-inherit, and persistence
(`beliefs` collection) fall out unchanged. **The `DISCOVERY` realm is exempt
from the liveness-GC** — its referent can be an `Exit`'s synthetic key with
no live Stuff to reach, so the GC must never reap a valid discovery.

## Honest fog — the enumeration seams

A concealed-and-undiscovered thing must be **absent from the viewer's world
at the server** — never "present but flagged hidden," and never sent to the
client (`nothing-is-pure-client`, no metagaming). One predicate —
`PerceptionApi.perceives(viewer, target)` — is threaded through every seam
that builds a viewer's world:

1. **Room contents** — `LookController.visibleContents`
   (`obj/command/perception/LookController.ts`) filters on `perceives`.
2. **Arrival sense** — `SenseController.visibleContents` filters on
   `perceives` (so a new room's fog is honest on entry too).
3. **Exits** — `Exitable.obviousExitsFor(viewer)` is the viewer-aware exit
   listing consulted by the perception paths (a concealed exit shows iff
   `perceives`). The viewer-blind `getObviousExits()` (`!isHidden()`) stays
   the seam for the physics/propagation walks that have no viewer.
4. **MQL targeting** — the `isVisible` predicate (`api/mql/predicates.ts`)
   now calls `perceives` (the refinement its placeholder comment reserved),
   and `scope-walk.pushDirect` gates every candidate on `perceives`.
5. **The client wire** — `Container.contents` (the `subscribableFields`
   projection on `lib/spatial/Container.ts`) filters concealed-undiscovered
   children out of the MQL projection. *(This is the shipped seam — the
   plan named `MqlSubscriptionLogic.projectFields`; as built, the fog lands
   at the Container-contents `read` projector, which is what the
   subscription serializes.)*

A **wire-shape leak test** constructs a concealed item + a low-perception
viewer, runs the `look` + projection paths, and asserts the concealed
thing's `stuffId` / `displayName` appear in **no** emitted frame or
projection payload — then `recordDiscovery` and asserts it now appears.

### Deliberate v1 boundaries

- **Acoustic / olfactory propagation is not concealment-gated.** The
  `SoundModality` / `SmellModality` / `AudienceGather` walks still iterate
  the viewer-blind physics seams — a concealed thing that makes a noise is
  a *hint channel*, not a leak. Cross-modal concealment (a silence
  discipline) is the reserved actor-face stealth consumer.
- **MQL deep-chain anchoring is viewer-blind past the direct scope.** The
  direct-scope enumeration (`pushDirect`) is gated, but a deep MQL chain
  that anchors on an object through other operators does not re-run the
  per-viewer gate at every hop. Named so a reviewer reads it as a scoped v1
  boundary, not a gap.

## Passive hints + active search

**Passive noticing** — `LookController` renders `PerceptionApi.hintsFor`
lines in the room description (the authored `concealmentHint`, else a
generic nudge), so attention is *directed*, not pixel-hunted.

**Active `search`** — `search [<target>]` (`cmd/perception/search.yaml` +
`SearchController`, perception category) is a **costed engaged act**: a
`SearchActivity` (`lib/concealment/SearchActivity.ts`, `implements
DurativeActivity` — the `Coup` / `ManualBuildStep` precedent) holds the
searcher's **`hands`** slot for `concealment.searchSeconds` game-time and
resolves `PerceptionApi.resolveSearch` **at completion**, so a barge-in
mid-rummage aborts it and finds nothing (the ambushable requirement). The
`hands` slot leaves `voice` free — you can still speak while you search.
No arg = a **broad-shallow** room scan (`'broad'` depth →
`concealment.searchBonus`); `search <container>` = a **narrow-deep** scope
(`'narrow'` → `searchBonus + searchDepthBonus`). Competence grades the
outcome via the `awareness` band.

**`examine` is folded into `look`** — there is **no separate `examine`
verb or controller**. `look` / `l` / `examine` / `exa` are all aliases of
`LookController`, and `look <container>` peers in for half-concealed
contents via a cheap instantaneous **`'glance'`** depth
(`concealment.examineBonus`, weaker than `search`), which ties up nothing
and folds into the same `resolveSearch`. *(The plan specified a distinct
`ExamineController`; as shipped it collapsed onto `look`.)*

### The `awareness` Discipline

`seeds/lib/advancement/Discipline/awareness.yaml` (`key: awareness`,
`channel: skill`, `iscedf: "0288"`) is the perception/attention Discipline,
seeded as **pure data** (the bartending / melee-combat precedent — the
`DisciplineCatalogue` harvests it at boot). It grades `effectivePerception`
capacity, `search`, and `disarm`. **No `conferrals`** — `search` / `look` /
`disarm` are universally afforded; competence only *grades* them, it doesn't
gate the menu. See [advancement.md](./advancement.md).

## The care↔speed detection axis

Two new `LocomotionMode` singletons (`seeds/lib/locomotion/sneak.yaml`,
`run.yaml`) bracket `walk` on the ground medium (sneak `speed 0.5`,
`noiseLevel: quiet`, `costMultiplier 1.5`; run `speed 2.0`, `loud`,
`costMultiplier 2.0`), plus thin `sneak` / `run` verbs. The **detection
modifier is not a mode field** — it's an AppSettings dial map read by
`PerceptionLogic.modeAttention(mode)`: `movement.attention.sneak` (a
positive delta — careful, notices more), `movement.attention.run` (a
negative delta — careless, notices less), `walk` and every other mode = 0
(so a walk crossing reads byte-identically to the passive baseline).
`HazardMixin.resolveTraversal` passes `modeAttention(mode)` into
`perceives`, so over the *same* concealed trap a sneaker steps around what a
walker is baseline on and a runner springs. See
[locomotion.md](./locomotion.md). The `noiseLevel` self-concealment side is
authored-as-data but its observer-side detection is the reserved stealth
consumer — no test asserts others-can't-see-a-sneaker.

## AppSettings dials

| Group | Keys |
|---|---|
| Level requirements | `concealment.level.{subtle,hidden,deep,buried}`, `concealment.hiddenDefaultLevel` |
| Detection curve | `concealment.passiveBaseline`, `detection.capacityPerBand` |
| Search / hints | `concealment.searchBonus`, `concealment.searchDepthBonus`, `concealment.searchSeconds`, `concealment.hintCutoff`, `concealment.examineBonus` |
| Care↔speed | `movement.attention.sneak`, `movement.attention.run` |

(The `hazard.*` dials live in [hazard.md](./hazard.md).)

## Deferred

- **The *hiding* half of stealth (the actor face)** — ✅ **shipped** (see
  [stealth.md](./stealth.md)): `HidingMixin` gives an actor a dynamic
  `hide` state whose *level* derives on `PerceptionApi.hideLevelFor`;
  motion-degrades-concealment lights the observer-side of the `sneak`/`run`
  axis (`movement.concealment.*` → `motionExposure`); the `wary` brain is
  the NPC detection/response; ambush (`CombatOpenOptions.ambush`) is the
  surprise-attack initiation; and the player-trapper (`TrapKit`/`arm`)
  places concealed traps at the placer's `stealth`. Consent/blame for
  ambush + traps rides the unified [accountability.md](./accountability.md)
  ledger.
- **The knowledge economy** — sharing / selling / transferring found
  secrets, maps as social currency. Only the *personal* per-viewer
  found-memory ships.
- **`frisk` + player-placed concealment** — searching a person/downed body,
  `stash`/`conceal`/plant against a `hide` competence. (The `getDiscoveryKey`
  `undefined` return is the seam: a generic multi-clone has no durable key.)
- **Ranged / remote traps** — the `HazardDelivery.range` reserved seam
  (see [hazard.md](./hazard.md)).
- **Resettable / rearming traps** — one-shot in v1.
- **Disguise-as-concealment unification** — identity-disguise stays its own
  axis (they meet only at the naming step — see [belief.md](./belief.md)).
- **Threat reads** (`assess <person>` out of combat).

## Cross-references

- [hazard.md](./hazard.md) — traps, the flagship consumer of this gate.
- [belief.md](./belief.md) — the belief store + the `DISCOVERY` world-fact
  realm this build added.
- [perception.md](./perception.md) — the viewer-aware query pattern; the
  detection surface is now the real refinement of the `isVisible`
  placeholder.
- [boundary.md](./boundary.md) — `Exit.hidden` subsumption + the reveal
  path; `allowsMode` and the ground-pace family.
- [locomotion.md](./locomotion.md) — the `sneak` / `run` modes + the
  care↔speed wiring.
- [advancement.md](./advancement.md) — the `awareness` Discipline.
- [activity.md](./activity.md) — `SearchActivity` as a costed engaged act.
- [light.md](./light.md) — `VisionModality.perceivedBand`, the light
  `conditions` term.
