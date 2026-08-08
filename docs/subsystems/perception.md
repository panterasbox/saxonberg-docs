# Perception Subsystem

The viewer-aware query pattern: a cross-cutting principle that any
property whose value depends on *who's perceiving* must follow.

This is a *pattern* document, not the description of a specific
mixin or Api. Concrete users (light, hearing, display-name
composition, language understanding, faction visibility, vendor
pricing) reference this doc rather than re-derive the rule.

## Why this doc exists

Several queries in Saxonberg return different answers depending on
who's asking:

| Query | Viewer-dependent because |
|---|---|
| `VisionModality.canSee(viewer, target)` | night-vision, blindfold, invisibility, light state |
| `DescribeApi.getDisplayName(viewer, target)` | hooded stranger reads as "a tall figure" to most, "Bob" to those who recognize him |
| `SensorApi.canHear(viewer, source)` (future) | deaf, distance, walls, magical silence |
| `understandsSpeech(viewer, speaker, msg)` (future) | does the listener speak the language? |
| `canRead(viewer, text)` (future) | literacy, language, vision |
| `factionStanding(viewer, target)` (future) | who's friendly to whom |
| `priceFor(viewer, item)` (future) | merchant haggles based on rapport |

All same shape: `f(viewer, subject [, args]) → result`. None of them
have a *single* correct answer — the answer depends on the viewer.

## The anti-pattern this replaces

Old MUDs often consulted the *current command-giver* as ambient
state. Code reaches into "who's running this command" via a global
or via a callstack walk, and renders for them. This breaks the
moment a command's effect transitively triggers a perception check
on someone else.

Concrete failure: Alice runs `tell orc to look at Bob`. The
perception check inside the orc's resolved `look at Bob` is *for
the orc*, not Alice. If `getDisplayName` infers viewer from
execution context, it returns Alice's view of Bob — wrong.

This anti-pattern also fails any time we broadcast: a sword fight
visible to N witnesses needs N independent perception checks, one
per witness. There is no single "current viewer" during a
broadcast.

**Saxonberg does not infer the viewer from execution context. The
viewer is always an explicit parameter.**

## The three layers

### 1. Pure query Apis

Functions whose answer depends on a perceiver take the perceiver as
an explicit parameter. The function is pure: same inputs, same
outputs. No reads from execution context, no globals, no
callstack-aware tricks.

```ts
VisionModality.perceivedBand(viewer: Stuff & Sensor, loc: Stuff & Container): LightBand
VisionModality.canSee(viewer: Stuff & Sensor, target: Stuff, detail?: VisibilityDetail): boolean
DescribeApi.getDisplayName(viewer: Stuff & Sensor, target: Stuff, fallback?: string): string
SensorApi.canHear(viewer: Stuff & Sensor, source: Stuff): boolean
```

Trivially testable, multi-perceiver-safe, side-effect-free.

### 2. Scene / Sensor broadcast harness

The multi-perceiver case rides on the existing messaging subsystem
(see [messaging.md](./messaging.md)). When something happens that
should be perceived, the Scene composer iterates Sensors present
and, for each Sensor, calls the query Apis with that Sensor as the
viewer. This is where one event becomes N tailored renderings.

The broadcast harness doesn't add a viewer parameter — it iterates
and applies. Each per-witness rendering is just N calls into layer
1, varying which Sensor is the viewer.

### 3. Shadow per-viewer overrides

The `Shadow` framework (see [call-security.md](./call-security.md))
is already per-instance. A shadow on a specific viewer can intercept
any query Api method called for that viewer:

- `BlindfoldShadow` on Bob → `VisionModality.canSee(Bob, …)` returns
  false.
- `NightVisionShadow` on a cat-NPC → band shifts up.
- `DarknessShadow` on a cursed avatar → `VisionModality.lightAt` from
  this viewer's perspective is capped at zero.
- `LanguageShadow` on Alice → `understandsSpeech(Alice, …, msg)`
  returns true for messages in languages Alice has learned.

> **Recognition is NOT a Shadow.** Per-viewer *identity* memory ("Bob has
> met Phil, so a hooded stranger reads as Phil") is its own subsystem —
> the `BeliefStoreMixin` on the viewer plus the explicit
> `RecognitionApi.describe(viewer, target)` naming step, not a shadow on a
> visibility query. It *consults* this perception layer only for its
> visibility gate. See [belief.md](./belief.md).

Shadow overrides are *per-viewer-per-query*. They don't fight the
contract; they're how the contract gets specialized.

## The viewer type — two orthogonal axes

Perception splits into two orthogonal mixins:

| Mixin | Layer | What it carries | Composers |
|---|---|---|---|
| `Sensor` | **channel** | `onMessage(frame)` + the shadowable `filterMessage` hook. You receive perceptual input as message frames through the messaging subsystem; vision, hearing, ESP all ride this — the frame's `meta.modality` names the modality. Modality-level filtering (deaf, blind, no-implant) lives here and is wired in the 2026-06 perception build: `Scene.modality(name)` stamps the frame; `filterMessage` drops it when the recipient's `PerceptionApi.sensorium` doesn't include the modality (actor self-frames bypass via the `audience:actor` tag). | Anything that *receives* perceptual input. |
| `Perception` | **interpretation** | `perceivedBandModifier(raw, loc)`, `canSeeOverride(target, detail, raw)`, `getVisionProfile()`, future `perceivedVolumeModifier` / `canHearOverride` / `getHearingProfile`. The seams query Apis dispatch through when the framework asks "what does this entity perceive?" Identity defaults pass the raw answer through; Shadows on the host modulate. | Anything *queryable as a viewer* — entities with subjective experience to be asked about. |

All viewer-aware query Apis take **`Stuff & Sensor & Perception`**
as the viewer parameter. Avatars and Characters compose both for
free through the Character chain. A passive recording device might
compose only Sensor (it receives, but isn't queryable for
subjective experience). Inert Stuff (rooms, items) composes
neither.

**Why split.** The original framing tried to unify both axes under
Sensor — "Sensor IS the perceiver type" — and put the
interpretation seams on Sensor too. That muddied Sensor's surface
(message reception vs. interpretation modulation are unrelated
concerns) and forced Light's perception pipeline to bypass
standard Stuff Shadow dispatch (since the seam methods didn't
naturally live anywhere a host could declare them with sensible
defaults). The split puts each axis where its methods belong:
Sensor owns the input pipeline; Perception owns the answer-
shaping.

A non-(Sensor & Perception) cannot be the viewer in a perception
query — enforced at the type level.

### Standard Stuff Shadow dispatch

PerceptionMixin's seams are real host methods with no-op identity
defaults. Shadows that intercept them follow the standard pattern:
`@Shadowing`, `callDown` for chaining, `@Unshadowable` /
`@ShadowSecurity` policies. The proxy pipeline routes
`viewer.perceivedBandModifier(...)` through the shadow stack
naturally — `VisionModality` just calls the host method.

```ts
// A blindfold curse — overrides perceivedBandModifier:
class BlindfoldShadow extends Shadow {
  @Shadowing
  perceivedBandModifier(_raw: LightBand): LightBand {
    return 'pitch-black';
  }
}

// A night-vision potion — overrides getVisionProfile, composes
// with other shadows via callDown:
class NightVisionShadow extends Shadow {
  @Shadowing
  getVisionProfile(): VisionProfile {
    return { scotopicMin: 'pitch-black', photopicMax: 'blinding', bandShift: 1 };
  }
}

// Multiple shadows on the same method compose via callDown:
class BoostShadow extends Shadow {
  @Shadowing
  perceivedBandModifier(raw: LightBand, loc: Stuff & Container): LightBand {
    const lower = this.callDown<LightBand>(raw, loc);
    return shiftUp(lower);                    // boost what's below
  }
}
```

The Light subsystem doc's "Per-Viewer Perception" section
([light.md](./light.md#per-viewer-perception)) walks the full
pipeline.

## Worked example

`Alice tells the orc to look at Bob`. End-to-end:

1. Alice's controller runs `tell`, dispatches to the orc via the
   messaging subsystem.
2. The orc receives a tell-event. Eventually (NPC behavior layer,
   currently deferred — for now, the equivalent in tests is a
   programmatic call) the orc decides to comply and runs its own
   `look at Bob`.
3. The look pipeline runs **with viewer = orc**, not Alice.
4. Inside look:
   - `VisionModality.perceivedBand(orc, Bob.getContainer())` — the
     orc's perception of band.
   - `VisionModality.canSee(orc, Bob, 'figure')` — orc's gate.
   - `DescribeApi.getDisplayName(orc, Bob)` — orc's renderer.
5. Whatever the orc reports back (say it tells Alice what it saw)
   flows through the Scene composer with orc as the source. Alice
   receives the orc's *report*, not Alice's own perception of Bob.

The command-giver (Alice) is relevant only for **attribution** —
error messages, stamina deduction, billing-style accounting.
**Perception is always answered for the perceiver.**

## Discrete-event sound push — `Audible` / `Scene.toAudible`

The multi-perceiver harness (layer 2) has a **push** variant for
one-shot sounds. Steady-state object sound already shipped —
`SoundSourceMixin` carries a stored dB level and the
`SoundModality.walkAt` field read gathers it *into* a scope on `listen`
(a pull). The discrete-event seam is the inversion: a whistle blast, a
bell, an alarm arrives **unbidden** in the source room and adjacent
rooms, attenuated and directional, blocked by closed doors.

- **`AudibleMixin.emit({ db, character, description?, timbreHook? })`**
  (`lib/perception/Audible.ts`) is the host facade — a stateless
  Thing/Location that makes a discrete sound. It carries no physics; it
  composes a `Scene`, stamps `.meta({ acousticDb: db })` +
  `.modality('hearing')`, and delegates to `.toAudible(...)`.
- **`Scene.toAudible(body, { descriptor })`** (`lib/message/Scene.ts`)
  is the delivery mode. It reads the source level from
  `meta.acousticDb` (finally making that previously-inert seam live),
  runs the audience-gather walk, and fans one frame per reached sensor.
  Same-room sensors get the full `body`; a reached-but-farther sensor
  gets a per-recipient **directional** line ("From the north, a faint
  whistle."), the adjective fading with lost level. Each arrival below
  `Sound.DEFAULT_HEARING_THRESHOLD_DB` is dropped — the same floor the
  `listen` field read gates at.
- **`AudienceGather.gather(sourceLoc, sourceDb)`**
  (`lib/perception/AudienceGather.ts`) is the walk — the push-side fork
  of `SoundModality.walkAt`. It reuses the shipped acoustic graph
  verbatim: `MAX_HOPS` + cycle guard, vacuum block, the doored-boundary
  `SoundConduit.transmissivity` branch (a closed `Door` returns 0 and
  short-circuits), doorless-exit traversal, and linear-dB attenuation
  (`sourceDb + 10*log10(cumulativeTau)`). New to the push model: a
  `PER_HOP_TAU` distance falloff and first-hop direction tracking. It
  returns every reached `(sensor, deliveredDb, direction)` and leaves
  thresholding to the caller, so different callers can gate at different
  floors.

Same-room delivery is the degenerate zero-hop case of the same walk
(`direction = null`), so `.toAudible` subsumes `.toPeers` for acoustic
events — no new physics, no new Api; it routes through the existing
`MessageApi` / `Scene` pipeline. First driver: the University Avenue
crossing's referee whistle
(`domain/eternal/university-avenue/Whistle.ts` + `BlowController.ts` —
a fixed ~110 dB blast), added in the crossing build (commit
`63a827a0`, "Audible — discrete-event cross-room sound push",
Phase 1E).

## Concealment / detection — the presence face on `PerceptionApi`

The exploration layer — **whether a thing is *there* at all**, resolved
per-viewer — is folded onto this same pair as `PerceptionApi.perceives` /
`effectivePerception` / `hasDiscovered` / `recordDiscovery` / `hintsFor` /
`resolveSearch`. **There is no `DetectionApi`**: detection *is*
concealment-gated perception, so the presence face is `PerceptionApi`, not a
fourth module — it is the real refinement the MQL `isVisible` **placeholder**
long reserved ("a real perception subsystem will refine this — light levels,
concealment, etc."). `perceives(viewer, target)` is a deterministic,
viewer-explicit query (this pattern exactly): un-concealable/`obvious`
targets short-circuit true, else `effectivePerception ≥ requirement` OR the
find is already in the viewer's `DISCOVERY` belief. It consumes the existing
per-viewer vision path (`VisionModality.perceivedBand`, and through it the
Shadow seam) as its light `conditions` term — so darkness/night-vision enter
detection with no second light read. The async `awareness` band read is kept
sync at the enumeration seams by warming it into
`PerceptionLogic.preloadForSenseGate`'s per-command snapshot. Full detail in
[concealment.md](./concealment.md).

### `resolveAmbush` — a hiding read is a perception fact

`PerceptionApi.resolveAmbush(attacker, defender)` answers "does this
opening land as an ambush?", and it lives **here**, not in combat.
`CombatLogic` asked `MixinApi.isHiding` directly when the ranged build
moved the ambush read into the initiation handshake, and
`check-combat-dynamics` refused it — correctly: whether somebody is
hidden from you is a perception question, and combat asking it does not
make it combat physics. Combat asks; perception answers. The band an
ambush buys (`close`, regardless of the arena's maximum) is combat's
call and stays in `CombatLogic` — see [ranged.md](./ranged.md).

## Where the pattern is currently used

- (existing) `PerceptionApi.perceives(viewer, target)` — the concealment /
  detection gate, threaded through the `look` / `sense` / exit / MQL / wire
  enumeration seams. See [concealment.md](./concealment.md).
- (existing) `VisionModality.canSee`, `VisionModality.perceivedBand`,
  `VisionModality.viewerVisionProfile` — Light & Boundary subsystem.
  Viewer-side overrides via Shadow seam methods
  (`perceivedBandModifier`, `canSeeOverride`, `getVisionProfile`)
  declared only on Shadows; `VisionModality` walks
  `ShadowApi.getShadows(viewer, methodName)` to invoke them. See
  [light.md § Per-Viewer Perception](./light.md#per-viewer-perception).
- (planned) `DescribeApi.getDisplayName(viewer, target)` — v2 form
  per [docs/roadmap.md](../roadmap.md)
- (existing) `Sensor` channel filtering already runs per-Sensor on
  message dispatch — see [messaging.md](./messaging.md)

## Where the pattern *will* be used

- Hearing (`SensorApi.canHear`) when sound conduit lands
- Reading / literacy / language gating
- Faction-standing queries
- Quest visibility (which quests an Avatar can see)
- Vendor pricing
- Recognition / disguise resolution
- Any future "personal" view of a shared world property

## Anti-patterns to avoid

- Reading "current command-giver" from execution context inside a
  perception query. Use the explicit viewer parameter.
- Caching per-viewer state on the *target* instead of the viewer.
  "Bob has seen this NPC, so the NPC remembers Bob" — that
  recognition state is on the *viewer* (Bob's `BeliefStoreMixin` —
  see [belief.md](./belief.md)), not on the NPC.
- Defaulting to a "current viewer" when none is passed. Refuse to
  compile instead — make the type system enforce that callers know
  who they're querying for.
- Assuming `viewer === command-issuer`. An action can transitively
  trigger checks on other agents. The viewer is whoever is
  *perceiving in this specific check*.
- Caching per-(viewer, query) results without an invalidation
  story. The state any of these queries closes over (light,
  shadows, position, faction, language) can change between
  queries; cache only with hooks that invalidate on the relevant
  events.

## Cross-references

- [messaging.md](./messaging.md) — Scene composer, Sensor routing,
  the multi-perceiver harness
- [call-security.md](./call-security.md) — Shadow framework,
  per-instance overrides
- [light.md](./light.md) — first major user of this pattern. The
  per-viewer queries (`perceivedBand`, `canSee`,
  `viewerVisionProfile`) and the Shadow seam dispatch live there.
- [roadmap.md](../roadmap.md) — Display-name composition (DescribeApi
  v2) is the second planned user

## Across the sandbox boundary

`PerceptionApi.sensorium` / `canPerceive` route through
`SecurityApi.projectAcross` (see
[call-security.md](./call-security.md)). Message delivery asks each
RECIPIENT what it can perceive, and a channel spans the boundary
routinely — sender in the field, subscriber inside a circle, or the
reverse. The walk reads the recipient's species and anatomy, so
un-apertured it denied and took the whole `chat` verb down with it.

The vision flux walk gained two guards for the same reason lighting a
room made them reachable: it skips exits answering false to
`Exit.hasSpatialDestination()` (a crossing exit names the wire, not a
room, and resolving it lands on a non-Container), and it skips a
destination that is destroyed — a reaped neighbour's proxy answers
every call with `undefined`, and `look` must not die because of it.
