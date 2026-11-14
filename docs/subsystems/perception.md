# Perception Subsystem

The viewer-aware query pattern: a cross-cutting principle that any
property whose value depends on *who's perceiving* must follow.

This is a *pattern* document, not the description of a specific
mixin or Api. Concrete users (light, hearing, display-name
composition, language understanding, faction visibility, vendor
pricing) reference this doc rather than re-derive the rule.

## ⭐ `canReach` — the ONE definition of reach, at a rung

An actor reaches: itself; what it carries; what stands in its place and
in the vessel it sits in (`getEnclosingPlace` — in a boat on the river,
both); a door attached to one of those places' exits, the way itself,
and the fixtures; and what stands inside **open containers** — on its
own body all the way down (the coin in the open pouch in your open
pack), anywhere else **one level** (the glass rack's coupe, a crate's
limes). That is the `open` rung; `top` stops at the holders' own
contents and `deep` goes through every lid but never into another body
(controller-liquidity D4/D5).

⭐⭐⭐ **It is ONE list — `PerceptionApi.reach(actor, {depth, location,
pools})`** (controller-liquidity W20; the owner: *"I just don't want two
different algos for the same thing"*). It returns everything to hand,
each tagged with its pool — `self` · `attached` (slot occupants, the apps
an implant hosts) · `inventory` · `around` (the place and the vessel that
hold the actor — MQL's `here` names them; the one pool that is NOT reach,
since a hand is in the room, not on it) · `here` (the place and the vessel's
surface: the place itself, the ways the actor can see and their doors,
the buildings whose landing it is, open ground seen across a way) ·
`peers` (what stands there, fixtures included) — in MQL's tie-break
order. `canReach` / `reachableAmong` are **membership** in it (every pool
but `around`); MQL's
`person` · `inventory` · `here` · `peers` · `reachable` pools are
**filters** over it (`api/mql/scope-walk.ts` only builds and scores the
candidates). `Container.getReachableContents(rung)` is the per-container
step underneath. `pools` computes only the sections asked for, so an
`inventory` scope never walks the room; `reachableAmong` is one walk for
the whole list. `reach.sameSet.test.ts` pins that every MQL `reachable`
result but the `around` holders is `canReach`-true and vice versa, scene by
scene and rung by rung.

⚠ History, twice over: before W2 the seed reached the person's top level
only while `canReach` went one level into an open pouch; after W2 the walk
was one but the ASSEMBLY (which holders, the fixtures, the ways and their
doors) was written twice — in the scope walk and in a `reachIds` helper —
and they disagreed at the edges (the room you stand in, the doors of ways
you cannot see, hosted apps, the buildings whose landing it is). The
consequences of the merge (W20): the place you stand in is explicit in the
list as `around` (named, not reached), and a door is in reach only on a way
the actor can see.

**One level in the room, deliberately.** Reach into the open crate, yes;
a box *inside* that crate must be opened or taken out first. Without a
bound, "reachable" would mean "anywhere in the room's transitive
contents", which is not what a hand does. What you carry is different:
an open bag in an open pack is to hand.

**A lid is a refusal with a name.** `Containable.reachRefusal(actor)`
returns `{reason: 'enclosed-shut', facts: {by}}` for the innermost shut
thing between a thing and the actor (never a lid that also holds the
actor, never another person's pockets). MQL *addresses* through shut
things (`me:I:apple`); the verb asks this and says *the box is shut*.

The openness test is **`MixinApi.isOpenContainer`**, and it is the single
rule four things ask — `canReach`, the MQL `peers` scope walk,
`mustBeInLocation`, and `VisionModality` — so what you can **name**, what
you can **see** and what you can **touch** cannot drift apart. Its two
exclusions are load-bearing: a shut `Sealable` is opaque, and a container
that is **somebody** (a `CommandGiver`, a `HasInteractive`, or any
`Organism`) is never transparent — you cannot reach into another actor's
pockets, and `Organism` is in that list because keying only on
commandability left every non-commandable creature's inventory public.

⚠ **How this went wrong once already, recorded so it does not repeat.**
`canReach` was flat — me, my contents, the room's contents — so a thing
standing in an open container was unreachable by every verb in the game.
The libations build hit it in a live drive and patched it at **four
symptom sites** (the scope walk, the validator, vision, and
`GetController`'s own pool) instead of at the concept, leaving four
encodings of reach that could diverge while the predicate actually
*named* reach still did not know the rule. The fix was to give the rule
one owner and let the callers ask.

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
> `target.describeFor(viewer)` naming step (the recognition face on the
> `Stuff` base), not a shadow on a
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
as the viewer parameter. ⭐ Both are composed on
**`lib/creature/Actor`** since the base-class narrowing (2026-09-30),
so every animate body is a viewer — a person, a wolf, a horse, a cat —
rather than only the ones that are somebody. That pair is one of the
three readers the rung was derived from: `PerceptionLogic` asks for
`Sensor && Perception` and has never asked whether the viewer is a
person. A passive recording device might
compose only Sensor (it receives, but isn't queryable for
subjective experience). Inert Stuff (rooms, items) composes
neither, and so does a **corpse** — it is a body, not a body that
acts.

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

### ⭐ The room listing is one query, and the orphan guard stays

`look`, `sense` and the inspection card list a place (or a holder) by ONE
MQL query since controllers-are-io W2 —
`here:i:visible:[not mixin.AdornmentMixin] - me` (the card's
`#<host>:i:visible:carried:…`) — where three verbatim hand-rolled filters
stood. `:visible` is this face (`perceives`), `:i` is
`visibleContentsOf`, and ⚠ `visibleContentsOf` now asks `perceives` per
viewer too: a bare `isConcealed()` there hid a thing the viewer had
already DISCOVERED the moment it sat in any listing.

⚠ **The orphan guard stays.** The listing then calls
`Container.getLooseContents(visible)`, whose `ids.has(host)` clause is
set-relative: an item placed on a host the viewer does NOT perceive is
listed loose rather than vanishing with it. The mql-predicate-parity
slate filed it as a question for this doc — *should anything ever hide
a host and show what is on it?* — and the answer kept is **yes, until a
build decides otherwise**: concealment is per object, a hidden shelf
does not hide the bottle someone left on it, and removing the clause
would be a behavior change no test proves safe. So `:loose` (the
predicate, typeable) reads the holder's FULL list, and the listing keeps
the guard by calling the method on what it can see.

### ⭐⭐ `observe` — a look's world writes are ONE moment (controller-liquidity W8)

`PerceptionApi.observe(actor, target)` is everything a deliberate `look`
writes, called once, so a player rewriting the verb copies none of it:
the warm (`preloadForSenseGate` — the boot link below), a thing's
`reconcileAiring` (cleanliness D21), a place's light band, listing and
`Perceiver.learnSurroundings` (the `seen` channel; in the dark the ways
out only, and nothing at all for a place with nothing to perceive), a
person's `learnIdentityOf`, and a container's close look — what it holds
read FIRST, then the `glance` `resolveSearch` that records each
discovery, then the hints. It returns an `Observation` the controller
only words. ⛔ A glance records DISCOVERY, never a `searched` claim —
only `search`'s resolve writes `searched`, because only a deliberate
going-over makes absences evidence ([location-graph.md](./location-graph.md) § 4).

⚠ `look`'s per-class self-reports (the floor puddle, the help-wanted
notices, a placing host's members, a good's owner, a screen) stay in the
controller: the `markupAugmenters` seam is synchronous, yields only
`state` segments folded into the long before `look`'s sections, and also
feeds the card projection and every sense verb — moving them would
change words and order. → `one-resolution-rule-slate` § 2.

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

## ⭐⭐ Another person — what a look may not tell you

*Their pockets and their weight are not in view* (client-surface W0,
#103/#104). One predicate decides it, named once:

```
PerceptionApi.isAnotherPerson(viewer, target)
  = viewer !== target && target.isAgent() && MixinApi.isSlotted(target)
```

A PERSON is an agent with slots; a rack is slotted and wears nothing, so
it lists like any container. `PerceptionApi.visibleContentsOf(viewer,
target)` — the one rule `look`, `inventory` and MQL's `:i` read (see
[perceiver.md](./perceiver.md)) — reads it: for another person the
answer is what they **wear** and **hold**, never what they **carry**; for
yourself it is all three; for a container it is everything inside, once
open.

The card surface follows the same rule, so the wire cannot leak what the
prose withholds:

- **`contents`** on a card (`Container`'s subscribable read): for
  another person, only the children `visibleContentsOf` says they
  **hold** survive the filter (worn stays the `worn` projection's), so a
  stranger's pocket never reaches a client.
- **`mass`** (`Tangible`'s read) returns nothing for another person:
  *another person's weight is read by a scale, never by a look.* ⚠ The
  read states `isAnotherPerson`'s three clauses inline rather than
  importing the Api, because that import closes a cycle through `Thing`
  (B6); `mql-subscription.privacy.test.ts` pins that the two agree, and a
  change to the predicate must change both.

### A person's things are seen in their room's light

`VisionModality`'s detail gate lifts the light environment for a
person: when the thing being looked at sits inside an agent with slots,
the light it is seen in is the light **its wearer stands in**
(`env = env.getContainer() ?? env`), beside the existing lift for a
thing inside a containable. The light inside a body is no light at all,
so before this every worn hat on a stranger read *something* (#90).
Which of their things are in view at all stays `visibleContentsOf`'s
rule, not this one.

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


## ⚠⚠ Modalities are warmed LAZILY, and the warming is a reachability link

`PerceptionApi.modalityByName('vision')` reads a cache that
`loadCaches()` fills by path-glob over live modality singletons. Those
singletons are cloned lazily, and **the only thing that warms them is
`PerceptionApi.preloadForSenseGate(actor)`** — which the verbs that
needed it called, and nothing else did.

⭐ That was fine for as long as nothing in a room-level render asked a
modality anything. The envelope build made `look` read the light band
before describing a place, and on a fresh world it threw *"no modality
'vision' loaded"* — caught, degraded, and **every room described itself
at midnight**. `analyze light` failed outright for the same reason.

⚠ **No unit test could see it**: a unit fixture calls
`buildAllModalities()` in `beforeEach`, so the cache is always warm.
Only a live boot with a cold cache can tell, which is the **boot** link
of the five and the one this project has now paid for four times. Any
new sync read of a modality must warm it first.


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

## History

- **Client surface (2026-10-31)** — `PerceptionApi.isAnotherPerson`
  named the person rule that `visibleContentsOf` reads; a card's
  `contents` and `mass` follow it; `VisionModality` lifts a person's worn
  and held things to their room's light. See
  `docs/plans/client-surface-plan.md` (W0, B6) until it is retired at the
  sweep.
- **Controller liquidity (MR !561, 2026-11)** — reach is defined once
  (W20, `985558c4d`): `PerceptionApi.reach` (`PerceptionLogic.reachOf`)
  lists what an actor can lay hands on, tagged by MQL's pools;
  `canReach`/`reachableAmong` are membership in it and MQL's seeds filter
  it. The `reachIds` helper and scope-walk's own holder/fixture/way/door
  assembly were deleted — they had disagreed on the room itself (now the
  `around` pool: named, not reached) and on doors of unseen ways. Earlier
  in the build (W2, `fd81f87e2`) `here` came to mean the enclosing place
  (`Containable.getEnclosingPlace`), MQL ADDRESSES a thing wherever it is,
  and a shut lid is the verb's gate (`mustNotBeShutAway` →
  `Containable.reachRefusal`), not the query's. `look` observes once (W8).
