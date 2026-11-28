# Perception Subsystem

The viewer-aware query pattern: a cross-cutting principle that any
property whose value depends on *who's perceiving* must follow.

This is a *pattern* document, not the description of a specific
mixin or Api. Concrete users (light, hearing, display-name
composition, language understanding, faction visibility, vendor
pricing) reference this doc rather than re-derive the rule.

## ⭐ `canReach` — the ONE definition of reach, ONE rule

An actor reaches (size-and-carrying D17, 2026-11-11 — the rungs `top` ·
`open` · `deep` are gone): itself; its **whole carried tree**, never into
another body, nearest first by tier — *hand* · *ready* (worn, hung from
what is worn, a pocket, a worn container anywhere but the back — nothing
is carried loose since the carrying flip) · *packed* (inside a worn
container in the `back` slot). A thing behind a shut lid or packed is IN
the list; the act's `canReach` refuses it by the object's own
`reachRefusal` (`enclosed-shut {by, region?}` · `packed {by}`), naming the
fix
(`Carrying.tierOf`, the carrier's own read); what stands
in its place and in the vessel it sits in (`getEnclosingPlace` — in a
boat on the river, both) and **one level** into the open things there (the
glass rack's coupe, a crate's limes); a door attached to one of those
places' exits, the way itself, and the fixtures.

⭐⭐ **`canReach` is membership AND the object's own refusal.** A thing
behind a shut lid, or in the pack on your back, IS in the list — the
binder binds it (it may be the nearest thing you perceive) — and
`Containable.reachRefusal(actor)` refuses it: `enclosed-shut` (*"The box
is shut."*) or `packed` (*"The rucksack is on your back — take it off
first."*, checked before the lid at the same holder because opening it
would not help). The `canReach` validator words both, after binding.

⭐⭐⭐ **It is ONE list — `PerceptionApi.reach(actor, {location,
pools})`** (controller-liquidity W20; the owner: *"I just don't want two
different algos for the same thing"*). It returns everything to hand,
each tagged with its pool — `self` · `attached` (slot occupants, the apps
an implant hosts) · `inventory` · `around` (the place and the vessel that
hold the actor — MQL's `here` names them; the one pool that is NOT reach,
since a hand is in the room, not on it) · `here` (the place and the vessel's
surface: the place itself, the ways the actor can see and their doors,
the buildings whose landing it is, open ground seen across a way) ·
`peers` (what stands there, fixtures included) — in MQL's tie-break
order. `canReach` / `reachableAmong` are membership in it (every pool
but `around`) plus the refusal; MQL's `person` · `inventory` · `here` ·
`peers` · `reachable` pools are **filters** over it.
`Container.getReachableContents()` is the per-holder step for the place
(top level + one level into open things). `pools` computes only the
sections asked for; `reachableAmong` is one walk for the whole list.
`reach.sameSet.test.ts` and `reach.oneRule.test.ts` pin it.

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

**A lid — or a pack on your back — is a refusal with a name.**
`Containable.reachRefusal(actor)` returns `{reason: 'packed' |
'enclosed-shut', facts: {by}}` for the innermost holder that stands in the
way (never a lid that also holds the actor, never another person's
pockets). MQL names through shut things (`me:I:apple`); the verb asks this
and says why.

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

### ⭐⭐⭐ The holder rules, and the one filter every surface applies (size-and-carrying D23–D25, 2026-11-11)

**MQL names things the same for every viewer; the acts perceive.**
`PerceptionApi.perceives(viewer, target)` decides, in order:

0. **the builder lens** — `lens on` memoized the extents the viewer holds
   (`AuthorMixin.lensOn` ← `AccessApi.heldExtents`; never a wizard check); a
   thing whose enclosing place (an exit: its source) lies under one is
   perceived. ⛔ Never a player's body or what is on one.
1. **your own body** and everything on it, at any depth.
2. **the holders** — walking up through what encloses the target, to the
   viewer's place: a SHUT container light does not pass hides it
   (Beer–Lambert over the holder's material, `e^(−α·t) ≥ 0.3` passes —
   glass, ice; `t` from the authored box's wall or a 3 mm jar; no
   `lightAbsorption` is opaque); a placement region that is `covered` (an
   inner pocket) or `closes` (a buttoned one) hides it; **another body**
   (anything `Attired` that is not the viewer — a corpse included) shows
   only what is in its hands, a worn layer nothing covers
   (`Attired.coveringAt`), what hangs `from` something it wears, and what
   THIS viewer found on it by searching while it still holds it (D24a —
   `BeliefPayload.foundOn`); never its loose pockets or an implant. An
   open pocket of an outer coat shows; under a cloak it does not (the coat
   is no longer outermost). (The pocket rule reads the pair's host, which
   for a pocketed thing is the container itself — the coat — since W9.) What holds the viewer is the viewer's side of
   its lid.
3. concealment, discovery and light (unchanged).

⭐ **Searching a body** (D24a) is the shipped `search` on a named body
that **cannot resist** — a `Corpse`, or an unconscious body
(`Vitals.searchRefusal(by)`; a conscious one refuses `resists`: *"Bob
won't let you."*). When the activity completes, each thing its pockets
cover is recorded as found **by the searcher on that holder** (the
discovery realm's `BeliefPayload.foundOn`, the holder's identity path);
rule 2 admits it only while the thing's holder is still that one — a
move to another holder makes the find inert, checked on read, no sweep.
A thing also concealed must still clear `resolveSearch`'s awareness
check. Witnesses hear *"Mara goes through the dead man's pockets."*

⭐ **The builder lens** is a verb (`lens` reports, `lens on`, `lens off`
— `author` category, afforded by `AuthorMixin`) and a memo on the author shell
(`AuthorMixin.lensOn()` / `lensOff()`, no setter — nothing can hand a
character a lens it does not hold), never a wizard check.

**`PerceptionApi.perceived(viewer, list, ordinal?)`** — the members of a
list the viewer perceives, in order, then the head ordinal — is the one
filter every surface applies (`lint:perceived-surfaces` holds the count at
0): the binder, `query`, `find`, the script island, the subscriptions and
cards, `look`, `sense`. **`perceivedHow`** answers `plain` ·
`builder` · `null`; `look`, `query` and `find` mark a line
`[builder]` when only the lens shows it. `visibleContentsOf` is now a
GROUPING (worn · held · carried) of what `perceives` admits.

### ⭐ The room listing is one query, and the orphan guard stays

`look`, `sense` and the inspection card list a place (or a holder) by ONE
MQL query since controllers-are-io W2 —
`here:i:visible:[not mixin.AdornmentMixin] - me` (the card's
`#<host>:i:visible:carried:…`) — where three verbatim hand-rolled filters
stood. ⚠ Since D23 the query only NAMES (`:visible` is a class word, `:i`
the raw contents); the listing is that list through `perceives`.

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
it lists like any container. ⚠ What of another body is IN VIEW is now
`perceives`' rule 2 (above — its hands, its outermost layers, what hangs
from them, an open uncovered pocket; never what its clothes cover), read
the same by every surface; `isAnotherPerson` still names the weight rule
below.

The card surface follows the same rule, so the wire cannot leak what the
prose withholds:

- **`contents`** on a card (`Container`'s subscribable read): the
  children the viewer `perceived` (worn stays the `worn` projection's),
  so a stranger's pocket never reaches a client — and the client no
  longer hides an agent's contents list itself (`CardBodies`' `HereList`
  suppression went with D23: the wire carries only what is perceived).
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
Which of their things are in view at all is `perceives`' holder rule,
not this one.

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


## ⭐⭐ The room's reading holds the world still (crowded-rooms D9)

What the room reads for everyone may never change the world, or the world
comes to depend on who looked. Four walks open
`ExecutionContextApi.holdWorldStill(walk)` for their own synchronous duration:
the light walk (`VisionModality.walkLight`), the sound gather
(`SoundModality.signalAt`), the smell fold (`SmellModality.signalAt`) and
a live card's projection (`projectStuffInto`). A deliberate act — `look`
at a thing, `get … from` — is NOT inside a hold and may still bring what it
reads up to date.

- **Inside the hold, a reconcile-on-read getter answers as banked** (owner
  ruling R1): `reconcileThermal`, `Bulkable`'s seeper and face-exchange
  reconcilers, the mould's bank and the heap's percolation stand off.
  `Thermal.getTemperature()` answers its banked curve read forward (toward
  the ambient it last saw), so a cooling pot still smells cooler with
  nothing stamped. There is no second "peek" getter — one mechanism.
- **The window lives on `ExecutionContextApi`**, as a property of the
  synchronous frame like the call stack: the write chokepoints (`StuffApi`,
  `EventApi`) sit below every subsystem Api in the import graph — an
  import of `api/perception` from them is a load-order cycle — and the
  class is undecorated, so the flag `ExecutionContextApi.worldHeldStill` (a
  static accessor) costs a hot getter no security frame (D15).
- **The tripwire.** Every write chokepoint calls
  `ExecutionContextApi.wouldWrite(what)`: a field fire (`fireFieldChange`,
  `EventApi.fire` of a `FieldChangedEvent`), `StuffApi.clone` / `create` /
  `destruct`, `ContainmentApi.move`, `PersistableApi.capture`. Outside a
  hold it is a no-op. Inside one, a **test world throws `WalkWriteError`**
  (naming the walk and the write; `lib/security/errors.ts`); any other
  world warns once per walk × write, counts it (`walkWriteCount`, on
  `/stats`) and lets the write proceed — a player's `say` is not broken by a pack's defect.
  `createSync` is not tripped: it mints the logic singletons.
- **The census is behavioral**: `lib/perception/__tests__/walk-purity.test.ts`
  stages a hoop past its set, a pan below freezing, a warm good and a
  leaching loaf, runs the three walks and asserts no fire, clone, move or
  destruct and every stamp unchanged. A new walk-time write fails the
  first test or wire drive that walks over it; there is no method-name lint.

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
- **Size and carrying W3 (2026-11-11)** — reach is ONE rule (D17): the
  rungs, `depth:`, `--top/--open/--deep` and `refuseIfShutAway` deleted;
  the person's whole tree in tier order; `canReach` = membership + the
  object's refusal, which grew `packed`; `mustNotBeShutAway` folded into
  the `canReach` validator. MQL made universal (D23) and perception moved
  to the surfaces through `perceived`; `perceives` gained the holder rules
  (D24), the body search (D24a) and the builder lens (D25).
- **Size and carrying W7, the flip (2026-11-12)** — `39d4ba23e..50f562ed4`
  for the build. Nothing sits loose on a body, so the *ready* tier no
  longer holds loose things; the person's carried tree is
  `Container.getCarriedContents` (breadth-first, never into another body),
  so `PerceptionLogic.reachOf` walks no frontier of its own; `held` is a
  hand slot everywhere. See [carrying.md](./carrying.md).

- **2026-11 (crowded rooms, MR !598)** — the room's reading holds the
  world still (§ above). The hold was first placed on `PerceptionApi`; it
  moved to `ExecutionContextApi` in the same build because the write
  chokepoints in `api/stuff` and `api/event` sit below `api/perception` in
  the import graph (a load-order cycle).
