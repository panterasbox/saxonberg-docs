# Ground — what you are standing on

**The build:** `build/ground`, MR !283, 2026-09-24. Its slate, requirements and
plan retired into this doc at the pre-merge sweep.

Before this, three different things were called *ground* and none of them
answered the question a player asks by standing somewhere:

| | owned by | answered |
|---|---|---|
| the **column** — strata, the orebody | `trade-mining`'s `Deposit` | only where a mine was cut |
| the **surface character** — texture, drainage, pH | `trade-farming`'s `GroundCharacter` | only where a field was ploughed |
| the **floor** — the thing you sit on | a `Floor` row 27 rooms authored | almost nowhere |

⭐ And the third was not merely sparse: `sit`, `lie` and `kneel` were
**refused in the room a brand-new character opens their eyes in**.

---

## Three claims, and the rest follows

**1. Every Location has a floor, by construction.** `Location.ensureFloor()`
runs at `postRegister` — the one lifecycle every clone *and* every `create`
passes through, which is why a warren-minted Lounge room gets its floor from
the same line an authored room does. A room opts out with
`noDefaultFloor: true`; the void is the only shipped user.

⚠ That is why `PostRegistrationMixin` sits in `Location`'s **base** stack.
The mixin's default hook is a **non-chaining no-op**, so a second
composition anywhere above the base *swallows* the base's — which is what
the ten Location classes that used to compose it individually would each
have done. Every override in the family now chains `super.postRegister()`,
and `lint:ground` clause (e) is what keeps it that way.

**2. A floor knows what it is made of, and never has to be told.** Five
rungs, authored first, resolved once at attach and stamped — because every
consumer of a floor's material is synchronous and rung 3 is not:

1. the floor row's own `_materialPath`;
2. the Location's `floor: { material }` spec;
3. **if on grade** — the ground beneath, through a `GroundSource`;
4. `Location.floorDefaults(onGrade)` — the room-kind default;
5. the plain default (a plain floor is a board floor).

**3. The KIND is derived, never authored.** `f(materialClass, onGrade,
worked, standingWater)` over `GROUND_KIND_FOLD`, closed at ten words:
`rock · set-paving · slab · earth · mire · beaten-floor · loose · boards ·
plate · contrived`.

⭐ The point of deriving it is that **two rooms paved in the same stone read
the same without anybody having chosen it** — there is no `kind:` field for
two authors to disagree in. The market square and the goods yards are the
shipped proof, and they reach the same answer from *different rungs*.

⭐ And `contrived` is what lets the list stay closed: a material the fold
does not recognise gets an honest answer and everything downstream keeps
working, so an author can invent matter without a kernel MR.

---

## The two questions that are NOT the same question

1. **Is there a floor here at all** — is this place *standable*?
   `noDefaultFloor`. Defaults to yes.
2. **Does the ground continue beneath it** — `onGrade`? Tri-state, `null`
   meaning *derive*: on grade iff the room is **sky-exposed** or sits
   **below datum**.

⚠⚠ Conflating them is the trap. A flying-only room **is sky-exposed**, so
deriving existence from `onGrade` would hand it an *earth* floor and you
could sit down on the sky; a mid-column water band **sits below datum**, so
it would floor open water when only the **bed** has ground under it. Neither
exists in the game yet — the declaration and its test ship so the builder
who arrives finds the seam. See
underwater-slate: *a band is not
standable; its bed is.*

⚠ `onGrade`'s sky-exposure limb is only as live as the biome roster —
`BiomeApi.isSkyExposed` answers `false` when no biome resolves, and
`getBiome()` is a **registry read**. `BiomeCatalogue` is what warms it; see
[biome.md](./biome.md).

---

## `/system/ground` — the pack

`Deposit` (the column) and `GroundCharacter` (the surface) live here, not in
the two trades that used to own them. ⭐ The `/system/` test is *a system is
true whether or not anyone is participating in it*: geology is there with
nobody mining and dirt is there with nobody farming. While they were a
trade's, a wood could not read its own ground, a quarry would have had to
depend on a mine, and the kernel's floor had nothing to ask.

⚠ **A system's classes are the pack's; its instances are the realm's.** The
Ferrow deposit stays in `rejection` and names `/system/ground/idea/Deposit`.
Same split as `Locality` and `Government`.

Both models compose **`GroundSourceMixin`** — the capability the *kernel*
declares and the *pack* implements, because the kernel cannot import a pack
and a pack cannot add a field to a kernel class (the failure already
recorded in `SpatialZone.ts`, where the two citations sit as strings the
kernel interprets nowhere). The **address** crosses the seam, never a seed:
each model derives its own.

⭐ They do not overlap. A character answers within its `topsoilM`; a column
answers from the collar down. That boundary is what lets a field read loam
while the gallery beneath it reads its host rock — **one ladder, two
sources, nothing arbitrating.**

`StrataMixin` (the pack's `lib/`) holds the five position reads lifted out
of `WorkingMixin`, which composes over it: knowing where you are in the
column is the ground's business, cutting it is the trade's.

---

## Reading a floor

⚠⚠ **`floor` and `ground` are keywords on the CLASS, not on the row.** The
MQL scope walk pools a thing's own `getKeywords()`, and `pushDetails` gives
a detail the pool `[<its id>]` and **never** its authored `keywords:`. So a
floor row's `details.floor.keywords: [ground]` is dead text — and attaching
a floor row to all 139 Locations would have fixed `look floor` while leaving
bare `sit` broken. `FloorMixin.getKeywords()` unions both words;
`lint:ground` clause (d) makes every row say them out loud as well.

⚠ For the same reason a floor row must **not** author a detail named `floor`
or `ground`: the detail wins the resolve, and a detail renders without the
host's `markupAugmenters`, so the derived sentence vanishes. Clause (f).
Details naming a real sub-feature — the crossing's worn track, the goods
yards' gutter — are the opposite case and are what details are for.

**One read for the room's floor: `Adornable.getFloor()`**, returning
`FloorThing`. Three resolvers used to answer it three times by scanning
fixtures-then-contents for any Bulkable with a surface slot; they all ask
this now and keep their own `hasSurfaceBulk()` check, because a dry posture
floor is still a floor and a puddle still needs the slot.

⚠ A floor is a **fixture** — in `Adornable.fixtureSlots`, not in contents.
Two consequences that cost real defects: presence cannot keep it resident
(hence `AdornmentMixin.canEvict`), and it has no container (hence
`VisionModality` resolving a fixture's light to its **host's**, without
which every fixture in the game rendered as *"something"*).

---

## What it costs

Measured at the build (400 clones each way, in-memory store):

```
a room clone   bare 0.519 ms   →   floored 1.282 ms   (+147 %, +0.76 ms)
```

⭐ **Cloning a room costs one more object, and that roughly doubles it** — but
the absolute number is what matters: a cold world boot is ~93 s, and 139
Locations × 0.76 ms is **0.11 s**, an upper bound assuming every room clones
at once, which nothing does. Rooms are cloned lazily and culled by residency,
so **an unvisited room costs nothing** and entering one costs 0.76 ms more.

⚠ The rejected alternative is worth recording: minting the floor lazily on
the first `getFloor()` would need `await StuffApi.clone` inside a read that is
**synchronous for all four of its callers**, including the binder's scope
walk. Paying 0.76 ms is cheaper than making the floor's existence
asynchronous — and lens 3 agrees: a floor that appears when somebody looks at
it is the gauge-shaped answer to a sim question.

---

## The census

```bash
pnpm -C packages/server lint:ground            # the gate, clauses (a)–(f)
pnpm -C packages/server lint:ground --report   # every Location and its ground
pnpm -C packages/server lint:ground --seed     # the claim heuristic, for curation
```

List 1 is derived. List 2 is hand-curated — rooms whose prose names a floor
material the room does not author — and its length is a **ratchet**: clause
(c) fails a listed row that has since been answered, so paying the debt and
recording the credit are the same commit. See
lint-family.md.

---


## ⭐ The floor and the envelope

The envelope build (2026-09-24) gave a `Location` a second material
citation beside its floor's: `fabric: { material, thicknessM }` — what
the room is **built of**, as against what it stands **on**.

They are deliberately the same shape, because they are the same kind of
claim. `FloorSpec`'s docstring is the one `FabricSpec` was written
from: *"a room that just wants cobbles, laid writes three words here."*
A room that is a timber shed writes two.

⚠ And the same doctrine holds for both: **a Location represents space,
not matter.** Naming a material confers no `Tangible`, no mass and no
`getMaterial()` — the room *names* what it is made of exactly as it
names its floor's, and the physics reads the Material row.

⭐ The floor itself is **not** part of the envelope's loss term. `U` is
computed over four walls and a roof (`5 · side²`); the floor is the
ground, and the ground does not leak to the sky. Coverings — a rug, snow
over paving — remain the named open seam they were, and if they ever
land they are a floor concern rather than an envelope one.

## Seams left open

- **`dig` reads the floor** — `getGroundKind()`, `isOnGrade()`, and
  `resolveUnderfoot()` again after a strip → § The worked-act protocol above (the
  extraction slate retired absorbed, 2026-09-24).
- **Coverings** — a rug, snow, mud over paving; a layer *above* the floor,
  and the reason rugs and carpets are excluded from the census →
  field-substrate-slate.
- **Consequences of the read** — traction, footstep sound, fire across
  boards, a body landing on flagstone rather than mire
  ([materials-response.md](./materials-response.md) — a floor is exactly a
  material plus a construction). ⚠ Two Larian-style surfaces already ship:
  a spill pools in the floor's surface slot, and electricity conducts
  through that puddle. What this build added is the **material** the rest
  of them need.
- **Combat** — `prone` is a session flag, and the bum's rush calls
  `setPosture` rather than `transferPosture`, so a rushed body reads *lie*
  while occupying no posture-bearing slot. That passed only because there
  was no floor to be on. → [combat.md](./combat.md).
- **Two rooms cite no biome at all** (`rejection`'s pithead yard and adit),
  so they take the interior default outdoors. Content debt; the census
  counts it.

---

## ⭐⭐ The worked-act protocol — `dig` and `split` (extraction, 2026-09-24)

`lib/ground/Workable.ts` declares what it means for ground to be **worked**,
as shapes rather than a class hierarchy:

```ts
planWork(by, tool, what): Promise<WorkPlan | WorkRefusal>   // no side effects
completeWork(by, tool, token): Promise<WorkResult>          // once, at completion
```

`Diggable` and `Splittable` are markers over it, discriminated on a `kind`
literal. **Two phases, not one**, because a swing is an engagement and a
barge-in must leave the ground exactly as it was — a single
`dig(by, tool, what)` could not have both planned and committed.

⭐⭐ **The controller knows nothing about stone, peat, clay or worms.**
`platform/idea/cmd/ground/WorkedActController` speaks this protocol and
nothing else: the thing being worked prices its own pace, refuses **in its own
words**, mints its own product, and names the Discipline it wants credited
(`WorkResult.credit` — the one field that lets a platform verb earn a *trade's*
competence). Take every content pack away and both verbs still work: they
refuse, naming the tool.

⚠ **The god-verb guard, and it lives in that file because that is where it
would break:** *if a new case needs the controller to branch on what kind of
digging it is, it is not `dig`.* `dig` shipped once before, in the fishing
build, and was **withdrawn in review for hard-coding its yield to a worm**.
The shape that survives that review is one where the controller cannot name a
yield even if it wanted to.

- It extends `GroundWorkController`, so this is the **zeroth** new copy of
  `engageAct` rather than the fourth (`MiningActController` and
  `FieldWorkController` are the first two, and forestry's doc already named the
  third copy as the trigger to promote it).
- `dig`'s resolution ladder is bound target → the room → **the room's floor**.
  The third rung is what makes a foraging `Soil` host a drop-in rather than a
  controller edit.
- ⚠⚠ **`dig`'s tool default is `me:i:[mixin.ToolMixin]`, not
  `[capability.digging]`** — the one instrument arg in the tree that departs
  from the capability atom, and the reason is load-bearing: a spade offers
  `digging`, a pick offers `winning`, and **both are right for some ground**. A
  capability default would silently fail to bind the pick and the player would
  be told *there is no face* rather than *that is the wrong tool*. Pinned by a
  binder test, because nothing else would notice it being "tidied" back.

### ⚠⚠ Bare-handed and wrong-tool are DIFFERENT refusals

A live browser walk caught the ground answering *"A pick will not shift drift
— take a spade to it."* to somebody carrying **nothing at all**. One branch
served both cases with prose written for the second.

⭐ The wire checkpoint passed throughout, because it asserts the refusal
`reason` and the sentence does go on to mention a spade; the unit test was
`expect(prose).toMatch(/take a spade to it/i)`, which is true of **both**
limbs. **A refusal that names a tool the player never had is the progression UI
lying**, so the split is not cosmetic — it is what acceptance criterion 2 of
the extraction requirements actually asked for.

## Ground improvement is the kernel's (extraction W2)

`lib/ground/Improvable.ts` holds the mixin, the jobs (`clearing` · `draining` ·
`liming` — a closed, ordinal vocabulary), the percept bands (`rough` ·
`broken` · `worked` · `in-heart`), the reversion rates and `ImprovementCost`.
It left `trade-farming` on the documented test: **its composers have no common
pack ancestor** (a field, a turbary, and whatever improves next).

⭐ **Three host hooks, not one**, and the shape is worth copying:

```ts
improvementBill(): Promise<ImprovementCost | null>;   // null ⇒ the acts refuse in words
improvementPace(job): number;                          // default 1
improvementSpoils(job): Promise<readonly Stuff[]>;     // default []
```

The bill returns **`null`** rather than a zero cost, so a host that composes
the mixin and answers nothing is *visibly* broken instead of silently free.
`grub`'s spoils are farming's **rows** and its pace reads farming's **sample**;
a kernel controller may know neither, which is what the other two hooks are
for.

⚠ **`plough` nearly went silent in the move.** It was listed in
`ImprovableMixin`'s contributions, and ploughing is not improvement, so it
could not travel with them. `Field` now declares its own
`commandContributions` naming `plough.yaml` — mixins **union** with a class's
own static, so it sits beside the mixin's three platform views. **Every
controller test would still have passed.**
