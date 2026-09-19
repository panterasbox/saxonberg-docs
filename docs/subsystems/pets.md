# Pets — an animal kept for itself

The kept-animal substrate: a stray on a lane that a player can come to care
about, and the moment it stops being *a* cat and becomes *your* cat. This
doc is the source of truth for `lib/creature/KeptAnimal.ts`,
`lib/husbandry/{Bonded,Feeder,OfferEngagement}.ts`, the three kernel
brains `lib/behavior/{follows,feeds,homes}.ts`, and the five verbs. Built
2026-09-15 → 09-17 (MR !257); the plan is retired, its decisions live here.

**Read first:** [ranching.md](./ranching.md) (the animal economy this
sits beside — three roles, not three classes), [belief.md](./belief.md)
(regard; per-viewer memory), [residency.md](./residency.md) (the pin —
what loads a pet), [furnishing.md](./furnishing.md) (the estate — where
its record lives). ⭐ The lens pass that shaped the last round:
design-lenses.md — *owning a pet is about its needs,
not the owner's*, and that is a choice the sim must force, not a sentence.

---

## The shape of it

**A `KeptAnimal` is a rung, not a kind.** One kernel class the cat, the
collie (`WorkingAnimal = HandledMixin(KeptAnimal)` in trade-ranching) and
the canary all clone from. What separates it from the `Creature` it
extends is not biology but that somebody could come to care about *this
particular one*, and the world is willing to remember which one it was.
Kernel, not a pack: its three composers have no common pack ancestor.

The stack, outer → inner (and ⚠ the order is load-bearing — see
*PostRegistration is innermost*):

```
Persistable   it can be promoted to something the world remembers
  Behaved     it decides things: follows · feeds · homes
    Bonded    the bond, the home, the verbs           ⟵ the point
      Status  legible attention
      PostRegistration  ⚠ innermost: its postRegister is a terminal no-op
      BeliefStore  its opinion of you — the bond's first factor
      Handling     how tractable it is — the second       ⟵ SIBLINGS
        Engaged · Mobile · Sensor · Named · Creature
```

`BeliefStore` and `Handling` are composed side by side, never nested:
`BondedMixin` narrows to both with `MixinApi` rather than demanding them,
so a host that forgets one degrades to "no bond".

**A second companion species is two rows and no code**: an agent row
naming `/platform/agent/KeptAnimal` with `handling:` and three brains,
and a species row carrying the dials.

## The bond — not a new number

`bond = regard/100 × handling`, and only affection counts (regard ≤ 0 is
*no* bond, never a negative one). Two things that already shipped,
multiplied — and the **asymmetry** is the design: regard never decays,
handling decays to a species floor (`Handling.ts`, ~a game month per
band). An animal you abandoned and came back to has not forgotten you and
is *harder to work with*. **It becomes difficult, not feral.**

The two factors are earned differently, and that is the whole feeding
design (below): proximity and routine buy tractability; only a hand buys
affection. *The floor stays delegable and the bond does not* — a friend
who keeps the bowl filled while you are away keeps your cat alive and
approachable and wins none of it.

Constants (`Bonded.ts`): `TOUCH_BAND = 'wary'` · `FOLLOW_BOND = 0.5` ·
`NAME_BOND = 0.6` · `PET_REGARD = 6` · `HAND_FEED_REGARD = 10` ·
`ILL_FROM_HAND_REGARD = −25` · `HOME_DAYS = 3` · `SURPLUS_SATIATION = 70`
· `HUNGRY_SATIATION = 40` · `TRUSTED_REGARD = 10` · `APPROACH_MS = 6000`
· `BORN_HUNGRY_SATIATION = 30` · `TRAIL_LENGTH = 16`.

## Species carries the dials

On `Species` (species-and-names): `handlingRange {floor, ceiling}`
(what handling can reach — a cat's ceiling is 0.9, a canary's 0.8),
`biddability` (0–1, **nullable** — a species that declares none is *not
askable at all*, not "average"; a cat's is 0.1, a collie's 0.9), and
`feedingStyle` over `hand · ground · graze · bowl · trough · hopper`
(`setFeedingStyle` throws on an unknown rung). `FeederKind` on a vessel
is `bowl | trough | hopper`; `FeederMixin` carries `feederKind`,
`offerings()` and `lastFilledBy`, and the shipped vessels are the saucer,
the trough and the hopper.

`lint:kept-animals` gates the triangle in three directions: a dial the
kernel cannot read; a bonded species with no biddability; a vessel rung
no row provides. It follows a pack's `const Base = Mixin(X)` and
`@saxonberg/server/mud/…` specifiers.

⚠ **The species must be warm.** `getSpecies()` is a live-only lookup and
`requiresAnimate` warms the *actor's* species, never the animal's. A
`Bonded` animal warms its own at `postRegister`; before it did, every
dial read as absent live and every refusal-shaped test passed anyway.

## Feeding is a ladder, and the rungs are a species fact

| rung | what happens | credits |
|---|---|---|
| **ground / bowl** | food left where it can reach it after you step back; `feeds` eats it on cadence | `handle(FEED_HANDLING_QUALITY = 0.25)`, **no regard** |
| **hand** | `offer` — it takes it from your hand | `handle(0.5)` + regard, **paced by need** |

**Need paces the bond.** A hand-feed's regard is `HAND_FEED_REGARD ×
deficit`, deficit read *before* the meal lands: a starving animal gives
the whole of it, one nearly full gives almost nothing, one at surplus
refuses. Affection cannot be farmed faster than the animal gets hungry —
the relationship runs at *its* pace. Being made ill (a contaminated meal
from your hand, `ILL_FROM_HAND_REGARD`) is not hunger-weighted.

**`wouldEat(food)` — four refusals, one sentence.** `not-edible` ·
`not-hungry` (satiation ≥ 70) · `turned` (freshness band) · `sensed-bad`
(pathogen load over the species' nose threshold — keen 0.1 · normal 0.35 ·
dull never). ⚠⚠ Every refusal renders identically: *"sniffs at it and
turns away."* An animal that told you why would be a contamination
detector you can read off. The reason is for the code, never the prose.

## ⭐⭐ The offer is a moment — the animal decides

`offer <food> to <animal>` is *the give that can be refused*. After the
food itself is judged, the animal answers on one of three rungs —
`Bonded.offerRung(person)`, deterministic from its handling band and its
regard for **this** person (each can stand in for the other one rung):

| rung | when | what happens |
|---|---|---|
| `hand` | steady+ · or wary and it knows you (`TRUSTED_REGARD`) | takes it now |
| `approach` | wary and a stranger · or flighty and it knows you | ⭐ **the taming scene**: `OfferEngagement` holds your `hands` for `APPROACH_MS`; keep still and it comes and takes it (a hand-feed); step out, give the food away or `cancel` and *"it does not come"* |
| `after-you-go` | wild · or a species with no `hand` rung · or somebody who made it ill | the food is set down; the brain decides later |

⚠ A wild animal never comes close enough to sniff, so the set-down rung
runs *before* the food refusal. Lazy revalidation, like every v1 activity:
the beat checks same room · food still in hand · still willing at
completion, and every way of it not working out reads the same.

No reptilian brain: the appraisal is the same shape as `wouldComply` — a
threshold over measured state, closed outcomes, and a player learns the
animal by watching it decide.

## The animal acts — three kernel brains

All three are `cadence:` / `departure` brains in `lib/behavior/`,
authored per row (`behaviors:`), and ⚠ a brain reads the animal's state
through `Bonded`, never a number the player could see.

- **`feeds`** (cadence 60s). Eats from a feeder of its rung or, for a
  ground-feeder, loose food. **It steps back**: below `steady` it eats
  only when every person in the room is one it knows
  (`feelsSafeToEatAmong`) — *"it waits until you step back"* is a
  mechanism, and the loop it promises (set it down, walk away, come back,
  it is gone) exists. **It asks**: hungry (`isHungry`), nothing it can get
  at, somebody here → it goes to whoever is present — the one it likes
  best if several, a stranger if that is who there is — and says so once
  on the transition; `look` reads *"It is at your feet, looking up at
  you."* It never begs while food it is only waiting on lies there.
  ⚠⚠ **The guard:** with nothing to eat and nobody owning it, the beat
  returns *before touching metabolism* — a hunger read reconciles, and an
  unstamped stray would starve in an afternoon of uptime. So only a
  stamped animal asks; the lane's stray does not beg because, by the same
  guard, it does not get hungry. Food from a **bowl** also credits
  `creditHomeCandidate` — home moves where it is fed, over `HOME_DAYS`
  distinct game days.
- **`follows`** (`departure`). Goes after a person it is bonded to
  (`FOLLOW_BOND`) when they leave; balks at an armed `Hazard` with a
  cause-free line; `waiting` holds it. Following someone home once is the
  naming gate.
- **`homes`** (cadence 90s). Steps toward the earliest reachable of
  `[home, ...trail]`. ⭐ Memory, not navigation: `rememberPlace` keeps a
  trail of `TRAIL_LENGTH` places, rewinds on revisit, clears at home. An
  animal carried somewhere it has never been genuinely **cannot get
  back, and nothing announces it** — the BFS this replaced made *lost*
  impossible. Pathfinding for every consumer is
  pathfinding-slate.

**Born hungry.** `reserves` is engine-written, so a row cannot say a
stray is thin because it is hungry; `Bonded.postRegister` seeds an
**unkept** animal at `BORN_HUNGRY_SATIATION`. A kept one restores its own
reserves from its record over this.

## The verbs — afforded by the animal

`pet · call · stay · name · offer` (`social/` and `inventory/`), conferred
by **`BondedMixin.commandContributions.peers`** — a static on the mixin,
so `KeptAnimal` and `WorkingAnimal` both afford them and neither shadows
the other. ⚠ A row's `commandContributions:` is dead silently.

- **`pet`** — refuses below `TOUCH_BAND` (*it moves off*), names no
  threshold and no number; `PET_REGARD` when it allows it.
- **`call [name]`** — ⭐ an **acoustic emission** (`Scene.toAudible` +
  `AudienceGather.gather`, `CALL_DB = 85`): no target; whatever hears you
  decides. `answersTo()` matches name or keyword; `wouldComply` is a
  threshold on `bond` against `biddability`, so a cat never comes however
  devoted and a collie comes once well bonded; a busy animal *"is X, and
  does not come"*. The refusal never reads as a shortfall.
- **`stay`** — a **gesture**: it needs the animal to see *you*
  (`PerceptionApi.perceives(animal, actor)`), so in the dark you can know
  where your dog is and have no way to signal it. Freed from `combat`'s
  `intervene` alias.
- **`name <animal> <name>`** — ⭐⭐ **naming is the promotion**, four
  things in one act: `setName` (public to everybody — an animal's name is
  not a person's), `stampChattel` (title, chain of title, and the moment
  its clock starts integrating your absences — `integratesLongAbsence() =
  isChattel && isStamped`), an explicit persistence key **before** the
  place write (so the residency pin lands on the row), and its first
  capture. Two gates, one sentence: bond ≥ `NAME_BOND` and *it has
  followed you home at least once* — *"It has not chosen you."* Witnessed
  by everything that can perceive it (the `introduce` shape), which is the
  second route home. A connected player's name may not be taken.
- **`offer <food> to <animal>`** — above.

**An unnamed animal is free.** Real, present, interactable, and the world
keeps no record whatsoever; its regard for you lives until the restart.
That is the feature.

## Persistence — the containable remembers, the room does nothing

A named animal is a **keyed `Persistable`** (scope = its template path,
key = a uuid) writing its own record; it is **owner-persisted chattel**,
so its owner's estate carries it as a *reference* (`EstateEntry.key`,
`state: {}`) rather than a copy. Where it stands is the `chattel` row's
`place` plus its own `HostPlacement` (anchor + `via`;
[persistence.md](./persistence.md)). ⚠ Four location classes compose
`Persistable` and this build added none: persisting a container preserves
its whole tree and is the rare, expensive model.

**What loads it** is [residency.md § the load half](./residency.md):
`KeptAnimal.pinsResidency()` answers true, the pin is stamped on its
`chattel` row, and `ResidencyWarden` stands every pinned good up once at
boot; the owner's login is the other ask. Pinning, not swap — no room asks
for what stands in it. Who may honour a pin is
eager-residency-slate.

`[mine]` (MQL) is the explicit stamp, or title over an extent — never
author, never group — and ⭐ it says **what** is yours, never **where**.
`find world:mine` naming a place would make a companion a tracking device
and a thief a bloodhound; if that assertion ever fails, the fix is never
the test.

## Belief, recognition, age

- **Recognition withholds names from persons only** — an animal's name is
  public (`RecognitionLogic`, narrowed `isOrganism → isPersona`), and ⭐ an
  animal keeps its authored keywords as targeting handles after naming
  (`look cat` still reaches Mouse); a person is targetable only by what
  you perceive them as, which is the whole of disguise.
- **A role-filler holds no personal regard** (`keepsPersonalRegard()`
  false on `platform/agent/Extra`); a `Cast` NPC's regard survives a
  restart under a **durable unique** `viewerKey` — see
  [belief.md](./belief.md).
- **Age.** `senescent` is a life stage that reads as old; death of age
  (`reconcileSenescence`, `age ≥ lifespanMax × 365`) bites for
  **non-sentient** animals only ([race.md](./race.md)).

## What the drives found, and where the doctrine is written

Eight review rounds, four defects that shipped green, two design
reversals, and a lens pass. The durable lessons:

- **A refusal is only honest if something lifts it** — the cat shipped
  untameable while every suite was green. The wire files assert the lift.
- **Prose asserting behaviour the model does not back** is the immersion
  lens's failing pattern, and it shipped twice ("it waits until you step
  back", "it moves off as you look at it" over nothing).
- **`PostRegistrationMixin` innermost** — its `postRegister` never calls
  `super`; composed above a layer with its own, that layer is silently
  dead ([antipatterns.md](../antipatterns.md)).
- **The room never pulls.** A per-room "who stands in me" query at
  `postRegister` was tried and reversed the next day.

The drives are `packages/wire/tests/pets.wire.test.ts` (clean) and
`pets-offer.dirty.wire.test.ts` (buys rations with founder coin; the four
beats). The restart half — a stranger sees the named cat with her owner
offline — is a two-boot run and stays manual.

## Open — the tail

`pets-slate` Wave 2: the accept/refuse hook for verbs other than `offer`
(the appraisal feeding `pet`/`call`/`stay`), the off-screen resolution +
digest, home range, pet combat staging, breeding. Findings offered:
nothing on Hinkley Lane yields food a stray would take (the producer gap,
recorded green in the wire file); a plain room's `cast:` re-mints its
stray on every load once the first is named.
