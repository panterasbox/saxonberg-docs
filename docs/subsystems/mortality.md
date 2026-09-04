# Mortality — the dying arc

The other half of every risk system in the game. Nine subsystems could
kill a character; nothing could bring one back. This is the driver
`vitals.md` § Layer 6 deferred and the consequence
mortal-vessel-slate
designed: **dying as a rescuable state, death as one transition, the corpse
as a forensic object, and the shade as a participant who kept everything
except a body.**

Read [vitals.md](./vitals.md) and [harm.md](./harm.md) first — this drives
their substrate. [sandbox.md](./sandbox.md) matters more than it looks:
death reuses its crossing machinery, backwards.

## The shape

```
        threshold crossed              window expires
 alive ─────────────────▶ dying ──────────────────────▶ dead
                           │                              │
              treat ───────┘                    ┌─────────┴─────────┐
           (rescued, not healed)                │                   │
                                             corpse               shade
                                       (material half,      (identity half,
                                        stays, decays)       walks away)
                                                                    │
                                                    reembody ───────┘
                                                   (content's call)
```

Two objects come out of a player's death because two different things have
to happen to them: a body has to lie there and rot so it can be examined,
and an identity has to keep playing.

## The dying clock

Crossing a lethal threshold no longer kills — it opens a window, and the
**clock** kills. That interval is the whole point: it is the only moment
anyone can intervene, and it turns nine independent "you are now dead"
flips into a state a medic can act on.

`DyingRecord` (condition Kind E, `lib/vitals/Condition.ts`) with the
`dying` band between `critical` and `dead`. It persists, so a dying body
evicted and restored is still dying with its accrued time intact.

**The window comes from the driver**, not a central table — bleeding out
and freezing to death are not the same length of story, and the producer
that knows the mechanism knows the number:

| cause | window | home |
|---|---|---|
| asphyxiation | 90 s | `RESPIRATION_DEFAULTS` |
| electrocution | 90 s | `HARM_DEFAULTS` |
| exsanguination | 120 s | `HARM_DEFAULTS` |
| hypo/hyperthermia | 300 s | `THERMAL_DEFAULTS` |
| starvation / dehydration / toxin | 600 s | `METABOLIC_DEFAULTS` |

Slow deaths leave the longest chance that someone reaches you.

### The clock runs while you are disconnected

**The single most load-bearing divergence in this build.** Every other arm
of `reconcileConditions` freezes while a player is linkdead and drops any
gap longer than `MAX_REASONABLE_GAP_SEC`, so that being away never costs
you anything. Applied to the dying clock, that same kindness makes Alt-F4 a
cure for death.

So the dying arm opts out of **both**, commented loudly at the site.
`Vitals.dying-disconnect.test.ts` pins it, and deliberately exercises a
*bleeding* body in the same fixture: if someone ever "fixes" the divergence
by copying the adjacent `if (linkdead)` block, the freeze assertions still
pass and only the dying ones fail, naming the reason.

The far-past guard is skipped for the same reason. Elsewhere a huge elapsed
gap produces an absurd result, so it is dropped; here it produces the
*correct* one — you were dying, nobody came, and the reading resolves that
when someone finally looks.

### Dying is incapacitation

`getConsciousness()` reads a dying body as `unconscious`. Without this, six
of the nine drivers left a dying body walking and talking: that readout
only knows blood volume, SpO₂ and head trauma, so cold, heat, hunger,
thirst, toxin and electrocution all read `conscious` right up to death.

## Stabilization

`treat` on a dying body pulls it out — the first time a non-combat
Discipline decides a life-or-death outcome, and it rides entirely on the
shipped medic loop.

- A dying body is treatable with **no wound to dress** (cold or a toxin
  leaves nothing to bandage; the controller used to refuse outright).
- The graded outcome gates the rescue; a failed attempt spends the dressing
  without holding them.
- **Rescued, not healed.** `stabilize()` drops the dying record and touches
  nothing else — the wound is still open, the blood is still gone, the cold
  is still cold. A body still under its threshold falls back into the
  window on the next reconcile. Stabilizing someone in a snowdrift buys
  them time, not a life.

`assess` reports the clock, and **competence buys information, not
outcomes**: a novice sees that someone is going, a competent medic names
what is taking them, a proficient one says how long there is to work with.
No roll, no effect on the clock.

## One transition

`ConditionApi.die(host, cause, spec?)` is the only way anything dies. It
replaced seven scattered sites, three of them byte-identical copy-pasted
`applyDeath` helpers. A scan for `setLifecycleState('dead')` outside tests
returns the transition itself and the explicitly-excepted plant path.

**Ordering:** everything a reader can observe happens in the synchronous
prefix, before the first `await`. The accountability append stays there too
— it is sync fire-and-forget, and combat's own coup choreography reads the
ledger in the same turn as the killing blow. Only the chronicle deed (prose
+ Mongo) is deferred.

### The ledgers never infer

Death is the accountability ledger's biggest producer, and used to be its
biggest gap: only combat wrote there, so eight of the nine ways to die left
no record that anything had happened to anybody.

One call now writes for all of them **without** collapsing accountability's
producers-not-a-chokepoint rule, because consent, killer and terms are
supplied by the producer that knows them. Combat builds its row
(`buildDeathRow`); hypothermia supplies nothing, and `die` writes an
environmental row that **omits `lethality`** — structurally incapable of
deriving as a crime, which is a stronger and more honest statement than
claiming the victim consented to freezing to death.

Attribution rides the dying record, so a bleed-out that finishes long after
the fight resolves still names who struck: by then the session is gone and
nothing else knows.

## Death is the sandbox crossing, backwards

The sandbox forks the **person** out of a body and discards the body.
Death forks the **body** out of a person and discards the person's claim on
it. One protocol (`ForkableMixin`), complementary slice families:

| | sandbox | death |
|---|---|---|
| slices | `Presentation`, `Embodiment`, `ClientState`, `Environment`, `Alias` — the **shell** | `Vitals`, `Trauma`, `CauseOfDeath`, `Anatomy` — the **material record** |
| merge path | epistemic only (`Contacts`) | **none** |

### Un-reanimatable by protocol, not policy

`ForkableMixin.applyForkedState` applies a slice by calling
`mergeSlice_<Name>` **on the target**. So if material body-state travelled
that route, every `VitalsMixin` host would need a `mergeSlice_Vitals` —
corpse and living body alike — and that method would be exactly the
"trusted mixin escape" the design forbids. One call, and a corpse walks
again.

Instead the apply side is `adoptMaterialState`: gated to the death
choreography, and deliberately **not** named `mergeSlice_`. The guarantee
falls out of the absence — `forkRuntimeState(corpse, freshBody)` is a
structural no-op, because there is no applier to find. Nobody has to
remember the rule; there is nothing to call.

`material-slices.no-merge.test.ts` walks the prototype chains and fails if
anyone ever adds one. This is the single edit that would silently undo the
design — see [antipatterns.md](../antipatterns.md).

## The corpse

The corpse is cloned from the authored template `/stuff/agent/Corpse`, backed by
**`obj/Corpse.ts`** — a thin concrete subclass of `lib/creature/Creature`.
The template used to name `Creature` itself, but `Creature` is mid-spine
substrate (`Agent → Creature → Character`) and the lib/obj taxonomy rule
is that nothing instances `/lib/`. Giving the corpse its own class also
says the truer thing: a corpse is a game object with a name, not a
generic creature. Behavior is unchanged — the forensic-Creature role
below is exactly as it was.

The clone is then configured from the body — the `split` shape. What a
corpse *is* is authored; whose it *was* is poured in through the gated
`adoptMaterialState`. A missing template **throws**: a body failing to
appear where someone died would leave a death with no evidence, no loot and
nothing to examine, and forensics simply would not work in that world.

`PostmortemMixin` on **`Creature`** — the tier whose own doc already names
"a corpse" as a valid bare Creature, and which already brings `Container`
(the loadout), `Vitals` + `BodyPlanSlots` (the wound map) and `Thermal`
(algor mortis, free, as passive drift toward ambient). So the mixin adds a
clock and an eviction opinion, nothing else.

Its clock runs **unguarded**, unlike every living-body clock: those freeze
while a player is linkdead because absence should never cost a living
player anything, but a corpse has no player to protect, and a body left for
a week should be a week gone when someone finds it.

**Decay degrades evidence while the cause stamp stays ground truth.** That
gap is the whole pedagogy of forensics: the examiner reads signs and
concludes, and can be wrong; the stamp is the answer key. Stages run
`fresh → stale → decomposed → spent`, with forensic readability falling
`1 → 0.6 → 0.25 → 0`.

At `spent` the corpse **stops vetoing `canEvict`** rather than destructing
— withdrawing an objection, so goods on it evacuate through shipped
container behaviour instead of dying with it.

### ⭐⭐ `sinceDeath()` is public, and butchering reads it

The food-safety build made the clock a second reader's input, and the
reason is worth stating: **a knife must not reset a clock that has been
running since the animal died.** If the cuts' spoilage gauge started at
the butchering, a player could kill a boar, leave it lying three days,
come back, and get *fresh meat* — a free lunch of exactly the shape the
cooking build closed when it made the kill step deposit the dose the
population had already earned. The failure is silent and reads as
generosity.

So `butcher` derives the cuts' initial microbial load from `sinceDeath()`
at the carcass's own temperature. Field dressing is time-critical, and the
cellar earns its keep from the first kill.

⚠ **The two clocks stay separate.** This one stays forensic and keeps its
own stages; spoilage reads it as an *input*. Nothing fuses them, and
`Creature` composes no `FreshnessMixin` — see [spoilage.md](./spoilage.md).

⚠⚠ **And a corpse's species must be WARMED before it is asked anything.**
A `Species` Idea is not resident at boot, and `SpeciesApi.isSentient`
answers **false** for one that is not — so the "you cannot butcher a
person" gate failed *open* on any corpse whose species nobody had touched
yet. The live drive is what found it. `ButcherController` calls
`SpeciesApi.preloadAnatomy` first and refuses outright on a species it
cannot resolve; anything else reading a corpse's species should do the
same. (The reference-Ideas-inert-at-boot trap, third recurrence.)

## The doctrinal split

race.md's **death ≠ destruction** holds, and the axis is whether an
identity has to leave. Both paths end with a persistent Creature-tier body
in the world.

- **NPCs, creatures, beasts** — the same Stuff stops. Unchanged, zero new
  machinery.
- **PCs** — the body divides, because only a player has an identity waiting
  to re-enter the world.

`ConditionApi.die` branches on whether the host carries a player identity,
detected structurally (never `instanceof Avatar` — that would be an import
cycle, and a future player-bearing class should behave identically).

### Ordering is the substance

`divideBody` is written against specific failures, each commented at its
step:

1. **stop the autosave** — or a periodic capture writes a half-drained body
   over a good snapshot;
2. **take the material slices** — draining is what destroys them;
3. **record the arc on the identity** — never a dead lifecycle on the body;
4. **drain to a clean baseline** — the avatar is *never* flipped to `dead`;
5. **capture** — before anything is destructed;
6. **mint the corpse**, hand it the gear;
7. **revert-and-destruct the old body** — before any new one registers,
   since `PlayerApi` warns-and-returns on a taken slot and `byTemplatePath`
   throws on two live objects at one path;
8. **only now** may the shade take the slot and the sockets — and it is
   **stood up in the room before it is handed them**. `Avatar.enter`
   refuses a body with no container (the spawn contract), and a shade is
   an Avatar, so a shade minted in mid-air throws straight out of the
   dying clock's expiry. Live, that surfaced as the client dropping to
   *"Disconnected — reconnecting…"* at the exact moment of death: the
   rejection took the socket down with the command, and nothing past the
   first step of the arc was reachable. The room is therefore captured at
   step (3) as **both** a durable path (`whereTemplatePath`, for a shade
   that returns at a later login) and a **live ref**, because by (8) the
   body has been destructed and there is nothing left to ask. The login
   path (`embodyForSessionImpl`) always placed its shade; this one did
   not, and no test caught it because none drove death with a live
   connection attached.

**Accepted risk:** between (3) and (8) the drained body is briefly alive at
baseline and still holds its Interactives. Per-socket command
serialization makes an interleaved command unlikely and harmless (a healthy
body doing an ordinary thing). Closing it fully would need a socket-parking
step.

## The two ways to persist death

These look alike and behave oppositely, and confusing them is the defect
this build exists to close.

| | behaviour |
|---|---|
| `lifecycleState: 'dead'` on a persisted **body** | a dead end. `requiresAnimate` refuses every verb, and the state is itself restored, so the next login restores it again — forever. **This was the live bricking defect.** |
| `Avatar.mortalArc` on the **identity** | always has re-embodiment as an exit. **This is the design.** |

The arc must *also* be durable: without it, logging out would mint a fresh
body and death would cost nothing.

`Avatar.reconcileMortalState` is a permanent backstop that heals a snapshot
which restores a body as dead — the *record*, not just the instance.
Nothing writes that state any more, which is exactly why it is kept: it
holds against code that has not been written yet.

## The shade

An `Avatar` subclass — the whole verb surface has to survive losing a body,
and re-deriving it as a parallel stack would start missing verbs
immediately. **Activations differ; composition does not.**

| | Avatar | Shade |
|---|---|---|
| persistence | captures | `shouldPersist() → false` |
| `PlayerApi` slot | held | **held** |
| identity | `/platform/agent/Avatar/<id>` | same |
| material verbs | allowed | refused (`requiresEmbodied`) |
| lifecycle | `alive` | `undead` |
| attunement | implant | intrinsic |

**`undead` is animate without being alive.** `SpeciesLogic.isAnimate`
already admits it, so a shade walks and speaks; `isLivingBody()` excludes
it, so it does not starve, suffocate, freeze, or die again. race.md shipped
the state unused; this is its first consumer.

**It holds the registry slot**, unlike `WireBody` — whose rationale is that
the parked field avatar keeps it. In death there is no field avatar. Every
`getAllAvatars()` consumer (channels, presence, social, bulletin, stream)
is a platform-half surface that must keep reaching a dead player, so
registration is not merely safe: it is the point.

**Attuned with no hardware.** A ghost wearing a cranial implant would be
silly. The shade confers `AetherMixin` directly through the per-host
conferral seam (`Species.innateMixins` is shared reference data and cannot
be used). It is perceptible because *being dead does not log you off* — the
aether is the internet, and a shade is still on it. It is **not** a spirit
field, and a mystical reading of it is a standing correction.

**Transient, but it lingers like a body.** A shade does **not** override
`onLinkdead`: a deliberate sign-out fires `PlayerLoggedOut`, a bare drop
fires `PlayerDisconnected`, and either way the shade stays in the world
holding the `PlayerApi` slot — so reconnecting returns you to the *same*
shade in the *same* room, exactly as reconnecting to a living body does.

### Logging out as a shade

Three separate mechanisms have to agree, and they do:

| you | what happens |
|---|---|
| **drop / sign out** | shade lingers, holding the registry slot |
| **reconnect** | `loadAvatarsForUser` reuses the in-world shade; `embodyForSession` sees no arc on it and passes it straight through |
| **log in after a restart** | the shade is gone (it persists nothing); the identity's snapshot carries the arc, so `embodyForSession` mints a fresh one |

**You are still a shade either way** — that is what makes quitting useless
as an escape from death. And there is no living body kept in reserve to
fall back into: death *destructs* the drained avatar rather than parking
it, so nothing can reconnect you into the person you used to be.

The shade is a *view*; the arc on the identity is the state.

## Function over form, made mechanical

`requiresEmbodied` is the fourth validator sibling (`requiresAnimate`,
`requiresConscious`). The verbs carrying it *are* the embodied half of what
a participant can do; everything else rides the participant and keeps
working no matter what happened to the body. A dead player still talks,
walks the commons, reads the forums and shows up in `who`.

Tagging is by **acts on matter**, not by folder: reading verbs in material
categories (`inventory`, `wallet`, `menu`, `spells`) stay untagged, because
a shade checking its empty pockets touches nothing. `tpa` stays untagged
too — a shade holds no travel credential, so the shipped gate already
refuses it, and a second refusal would just be a worse error message.

`embodied-tagging.test.ts` pins the inventory, so a new verb in a material
category fails until someone decides which half it belongs to.

### What confines a shade

Nothing bespoke. A shade is a baseline vessel with **no keys, no credential
wallet, no gear**, so the shipped `Lockable` / credential machinery keeps it
in the commons by itself. **The shade walks; it never phases.** It goes
anywhere an ordinary member of the public could walk, and nowhere else —
which needs no new access model and never touches parcels.

## Coming back

`ConditionApi.reembody(shade, container)` is the **entire** engine
involvement, and it is content-facing: a temple, a clinic, a quest, someone
owed a favour — anything that wants to offer a way back calls it when its
own terms are satisfied.

**No route type, no terms vocabulary, no registry.** An earlier design had
all three; they were cut as a schema for content that does not exist yet.
Being a ghost is an authoring space, and a vocabulary written today would
constrain it rather than serve it. Everything a passage might charge or
restore is already expressible: banking charges, containment gives and
takes, a quest gates however it likes. The caller decides **where** (the
argument), **what you keep** (move the items), and **what it costs**.

**`reembody` never reads the corpse.** A body decays, can be destroyed, and
does not survive a restart, so a route that consulted one would strand
whoever came back too late — the bricking failure mode in a third costume.
Proved structurally, the same way the material slices prove their own
absence.

### What healing happens, and when

**At death, not at revival.** `divideBody` drains the body to baseline —
every vital to its species baseline, every condition cleared, the cause
nulled — and *then* captures. So the snapshot is already a clean body and
`reembody` simply materializes it. There is no healing step at the far end
because there is nothing left to heal.

That ordering is not incidental. **Revival must clear the lethal state**,
or the next reconcile re-opens the dying window and kills you again — an
infinite death loop. Coming back "in the condition you left in" can never
be literal; some healing is load-bearing, not generous.

### Diminishment, and why the floor has to hurt

A free revival at full strength would make **every** resurrection service
in the world strictly worse than doing nothing. There would be no market
for coming back, because nobody can beat free-and-perfect.

So the cost lives in the **floor route**, not in the transition:

- `reembody` hands back a clean body and takes no position on what dying
  should cost — that is the caller's business, which is exactly what lets
  a temple or a clinic offer a better return;
- the `passage` verb, being the cheap way, drains the biological reserves
  and leaves a `recovering` affliction. It clears itself as metabolism
  refills, so nobody has to build a cure for it.

Two properties the diminishment must keep, and a service's must too:

- **Unpleasant, never dangerous.** Reserves only, never a vital sign —
  anything that could carry a body back across a lethal threshold would
  re-open the death loop through a side door.
- **Legible.** A player has to feel it, or a service has nothing to
  advertise against.

  Legibility was a *claim* until the end-to-end pass, and the pass found
  it false: `assess` rendered only the band, the dying readout and trauma
  **wounds**, so a freshly re-embodied body read exactly `You look
  unhurt.` The price of dying was invisible to the person who had just
  paid it, and there was nothing for a temple to undercut. `assess` now
  carries an **affliction readout** — signs, not names, on the same
  competence rule as the dying block: an untrained looker sees `unsteady`,
  a competent medic names the condition. (`AssessController`.)

  It degrades on purpose. `observableSigns` and the real name come off the
  authored `Condition` Idea — but **no Condition Idea is live at any path
  today**: the seeds are inserted as template ROWS and nothing clones them
  into Ideas at boot, so `findByTemplatePath` answers null for every one
  of them, `starvation` as much as `recovering`. That is pre-existing and
  world-wide (`Metabolic.resolveToxinBehavior` and `MagicLogic` already
  swallow the same null), and instantiating the catalogue is its own
  build. Until then the readout falls back to the template path's leaf, so
  it works now and gets strictly better — authored signs, real names — the
  moment the Ideas exist.

The competitive axis is therefore *how little you are diminished*, and it
needs no engine work: content calls `reembody`, gets the body back, and
applies less than the floor does — or nothing, or a boon.

### The floor

`passage` — zero arguments, always available, afforded only by being
incorporeal. It exists so a player can never be stranded because the
content that would have brought them back does not exist, is unreachable,
or was deleted. Content is free to be better than it.

**There is no wake point.** You come back where you are standing: the
shade walked somewhere, and that is where it takes a body. `reembody` has
no destination argument at all, and nothing is relocated.

That is a deliberate retreat from an earlier shape that woke you at
`defaultStartLocation`. Two things were wrong with it. A global default
start is not where any *particular* player belongs — that lives in their
own player data — and more to the point, there was no reason to move
anybody: a shade that wants to return somewhere in particular can walk
there first, exactly as a living person would. Content offering a passage
on its own terms walks you to its temple, or moves the body it gets back;
the engine decides where nobody ends up.

## Inside a circle

A vessel's death is real in the holodeck and discarded with it — the point
of a holodeck, and what lets an author test a lethal trap on themselves.
The body leaves a **circle-scoped corpse** and the player is **ejected** to
the field body that was parked all along. No shade, no arc, no snapshot:
minting a real body from inside a circle is the boundary the sandbox exists
to hold.

The discriminator is the receiver's circle stamp, not `instanceof
WireBody`. `SandboxApi.respawnWireBody` was deleted — no production caller,
and its behaviour is now wrong.

If the **field** body dies while its player is parked in a circle, exit
composes with no extra plumbing: a parked avatar still reads as connected,
so the split ran normally and put a shade in the registry slot. The player
exits as a shade.

### Circle-marked rows convict nobody

A hole this build inherited rather than created, but made matter. The PM
policy table classifies `accountability_events` **PASS(mark)**: a row
written from circle context persists carrying `circleScope`, and readers
"may lens the mark". Nothing did. So anyone who could open a circle could
stage a killing and mint a real crime row against a real identity.

Fixed in the **consumer**: `deriveBlame` ignores circle-marked rows.
Derive-on-read re-legislates every row ever written without rewriting one.
The writes still ride the policy table rather than any bespoke suppression
— the table is the sandbox's contract. Chronicle keeps its marked deed,
because "you died in a holodeck" is a true thing about you.

`circleScope` is now **declared** on the event, because
`Document.fromDocument` only reads declared fields: the mark was being
written and silently dropped on the way back.

## What survives a new body

Almost everything, with no carrying mechanism at all — the durable ledgers
key on `getIdentityPath()`, not the object: `ChronicleLogic`, `TraitLogic`,
`BeliefStoreLogic`, `AdvancementLogic`, `RenownLogic`, `AccessRegistry`. So
chronicle, transcript/competence, traits, beliefs, renown, authority, bank
accounts, chattel and parcel titles all come back with you.

Lost by design — this is the stake, not a leak: **gear** (to the corpse),
**vitals and wounds** (reset — that *is* rebirth), **location,
engagements, posture, combat state**.

## Reading lifecycle state

Three predicates, and picking the wrong one has bitten twice:

- `isAlive()` — exactly `'alive'`.
- `isDead()` — exactly `'dead'`.
- **`isLivingBody()`** — what every survival driver actually wants. Neither
  `!isDead()` (misses `undead`, so a shade starves and freezes) nor
  `isAlive()` (`lifecycleState` defaults to `''`, so an unhydrated body
  would have metabolism silently switched off — this regressed 34 tests
  when tried).

Metabolism had **no** lifecycle guard at all before this build, so corpses
went on metabolizing.

## Files

```
lib/mortality/MortalArc.ts     the durable arc + decay vocabulary
lib/mortality/Postmortem.ts    the corpse clock + canEvict terminus
lib/mortality/Incorporeal.ts   the capability lever (prison reuses it)
lib/mortality/Shade.ts         the between-bodies vessel
lib/command/validators/requiresEmbodied.ts
cmd/charactergen/passage.yaml + platform/idea/cmd/charactergen/PassageController.ts
```

`ConditionApi` grew `die` / `embodyForSession` / `reembody`; `VitalsMixin`
grew the dying clock, the material fork slices and `adoptMaterialState`.

## Deferred

- **The underworld.** A `perceptualPlane` axis was built here and then
  **cut**, and the reasoning is worth keeping. It could not do the job it
  was justified by: `Location` does not compose `ConcealableMixin`, so a
  *place* could not be tagged at all — and traversal is gated by
  `Exit.canTraverse()`, not by perception, so a tagged exit was invisible
  rather than impassable. A living player who guessed the exit name walked
  into "the underworld" and stood there seeing nothing.

  What an underworld actually needs is a **traversal gate on
  incorporeality** — rooms are ordinary content, and once the door is
  gated nothing inside needs a plane. That is a smaller primitive, and it
  should be built against a real quest's requirements rather than guessed
  at.

  If the ghost-in-the-tavern beat is wanted later, **concealment already
  abstracts it**: a band only an incorporeal viewer clears rides the
  shipped per-viewer detection face. One system, one mental model for
  players, and no second invisibility axis for authors to track — which
  was the real cost of the cut feature.
- **The re-embodiment service** — decided as lore, unbuilt: contested
  metaphysics, two competing vendors (temple and clinic), coverage as the
  hook. See mortality-slate. The
  seam it needs is already open: `reembody` returns the body, so a service
  applies its own terms to it.
- **The recuperation model.** The floor ships one crude cost — drained
  reserves plus a self-clearing `recovering` affliction. What diminishment
  should actually *be* is undesigned: a temporary competence penalty, a
  wound that heals over time, a diminished vessel in the mortal-vessel
  sense, or something a patron marks you with. The `recovering` condition
  is deliberately a near-empty seed (no `signature`, no staged prose) so it
  can grow into whichever of those wins without a migration. The dial
  (`RECOVERY_RESERVE_COST`) is a placeholder, not a balance decision.
- **The in-circle death arc** — a circle death ejects, so the full arc
  cannot be rehearsed in a holodeck.
- **Corpse custody** (a titled body), **remains** after terminal decay, and
  the coroner economy.
- **Declarative reference lifetimes** —
  reference-lifetime-slate,
  surfaced by this build. `MortalArc` holds **no handle to the corpse**:
  it was redundant (a corpse is laid at the body's own container, so
  "beside your corpse" and "where you fell" are the same room) and
  unreadable when read (a corpse is runtime-only, so a handle could never
  resolve after the restart the field exists to survive). Every field in
  the arc is a durable scalar; nothing in it can be stale by the time it
  is read.
- **Forensic examination verbs** — the readability curve ships; nothing
  consumes it yet.

## History

Built 2026-07-31 in eight waves. Two decisions reversed during the build,
both recorded above: the shade **is** registered with `PlayerApi` (the
requirements said never, copied from `WireBody` without its rationale), and
the survival guards read `isLivingBody()` rather than the `!isAlive()` the
plan specified. Three latent defects were found and fixed along the way —
metabolism's missing lifecycle guard, `Avatar.shouldPersist` not chaining
to `super` (which made `markForRevert` dead), and three drivers sharing an
`applyDeath` method name so the outermost shadowed the other two.
