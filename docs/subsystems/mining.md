# Mining — the ground, the workings, and the metal chain

The substrate behind `trade-mining`, `trade-fuel` and `trade-smelting`,
and the venue pattern `rejection` is the reference implementation of.

Four packs:

| Pack | Ships `src/`? | Root | What it is |
|---|---|---|---|
| `trade-mining` | yes | `/trade/mining` | how a mine WORKS |
| `trade-fuel` | yes | `/trade/fuel` | the coppice, the burn, the charcoal |
| `trade-smelting` | yes | `/trade/smelting` | ground into metal |
| `rejection` | **no** | `/world/rejection` | what THIS mine is like |

---

## ⭐⭐ The governing split: the warren creates rooms, it does not interpret them

> **Reads go to the space. Mutation goes to the warren.**

Every read a mine needs lives on **`WorkingMixin`**
(`trade-mining/src/location/Working.ts`) and derives from the room and
its zone:

| Read | Derived from |
|---|---|
| `facesOf()` | the neighbour cells' geology — a zone lookup by coordinate |
| `stabilityAt()` | span (which neighbours are open rooms — **a zone lookup, not a ledger scan**) · ground (the deposit's host material) · support (the timber sets in THIS room) · water |
| `airAt()` | a walk over the exit graph to a room that breathes |
| `groundTelegraph()` | the same number `stabilityAt` returns |

**`MineWarren`** owns only what a warren genuinely owns and a static mine
genuinely cannot do: **carve · abandon · the tier ledger · seal-and-reap.**

Why it matters: **a bespoke, hand-authored mine must work.** The
exemplar claim is that a second mining town needs zero pack code, and a
static hand-built mine is the most likely second mine anybody authors. A
room composing `WorkingMixin` at real coordinates in a deposit-bearing
zone behaves identically to a carved one, because nothing in the reads
consults how it came to exist. What a static mine cannot do — carve,
tier, seal-and-reap — is the elastic half; **a static mine is a mine that
does not grow**, which is a coherent thing to be rather than a degraded
one.

`getTier()` is the one read that consults a warren, and it has an honest
static answer: a room with no warren is `spine`, because authored ground
does not need admitting.

---

## Three-tier room identity (residences D17: no rowless mints)

- **Spine** — the pithead rooms, the adit, the authored galleries: real
  rows, one instance each, `SingletonCartesianLocation`. Never members.
- **Workings** — every carved cell is a **keyed member**: scope is one of
  the venue's four type rows (`MineRoom`, the permissive
  `CartesianLocation` face), key is `<claimExtent>/<cell>`.
  `PersistableApi.restoreOrSeed(room, key)` is the whole mechanism.
- **Geology** — no identity at all.

⭐ The key is not a bare coordinate. `<claimExtent>/<cell>` matches
`HoldingWarren`'s shipped `<extent>/<leaf>` convention, so the Stage-B
graduation to a kernel holding is a **base swap**, and it puts claim
scoping IN the key so part of `claimFor` is derivable.

**The warren extends `InnerWarren`** — its members are rooms, and
`Warren.occupantsOf` is abstract so the tier cannot be skipped.
⚠ Not `HoldingWarren`, and for one reason only: that is
`packages/content/residence/` content and a trade pack must not depend on
a residence pack. Conceptually the mine IS a holding — *a parcel being
put to a use*, with `landUse: industrial` naming the purpose.

---

## The geology field

Three layers (field-substrate-slate):

- **the model** — `Deposit`, a pure-data `Idea` in `trade-mining`;
- **the instantiation** — the zone naming a row of it, with the seed
  **derived from the covering Locality's address** (the
  `WeatherLogic.localitySeed` rule). No seed is stored anywhere: rename
  the mine and its ore moves.
- **the values** — computed and stored nowhere.

⭐ **One resolved read, and no raw branch.** `sampleAt(at, seed)` folds
authored **pin** over authored **lean** over the **procedural** value and
returns one shape, so an authored pocket and a computed cell are
indistinguishable to every consumer. Nothing may reach past it.

⚠ **The deposit speaks METRES**, not grid cells — rock does not know what
cell size somebody chose for the workings cut through it.
`Working.metresOf` is the single conversion and reads the zone's own
`cellSize`.

⭐ The `deposit:` field goes on the **shared parent zone**, not the mine
zone: `lookupField` walks outward, so the surface pithead resolves the
same body and the outcrop, the float and the three-point problem are all
played above ground.

**`strike` and `dip` ARE θ and φ.** A plane in the ground is two angles
and an offset, which is why the survey can solve it from three points.

Nothing rolls. FNV-1a over the cell string, XOR the derived seed:
process-independent, and the player's uncertainty is **epistemic** — the
ground was always there.

### Surveying — three layers, not one

| | Verb | What it is |
|---|---|---|
| **the mirror** | `survey` — shipped, free, ungated | *what is this place* |
| **the measurement** | `measure strike` / `measure dip` | instrumented, banded, load-bearing |
| **the interpretation** | `analyze ground` | route-gated synthesis with error bands |

⭐⭐ `survey` is a MIRROR and the mine's read is a MEASUREMENT. The
geological read is instrument-mediated, competence-banded and the thing a
player pays for and acts on — the opposite of a read nothing is gated on.
`MineWarren` answers the duck-typed `WarrenMember` shape `SurveyController`
reads, so `survey` reports honestly in a working for free.

⭐ **Competence buys resolution and the availability of an inference,
never outcome.** The error bar IS the competence; the underlying figure is
identical for both readers, and the suite asserts that identity rather
than a range.

⚠ **Dip is unobtainable at the surface by geometry.** A lode that reaches
daylight gives you a stain — the trace in plan — and reading fall needs
the vein in *section*, a face cut across it. The missing parameter IS the
reason to cut.

Survey records are **per-viewer beliefs** in the DISCOVERY realm, keyed on
`(deposit, point, channel)`, riding `knownAs` on a durable feature
handle. Two characters on one outcrop hold different books, and a survey
record is a tradeable asset. The reading is stored and the error band is
not: a prospector who improves re-reads their own old notes at their new
resolution.

---

## The face model, and support

`facesOf()` reports ten directions: what is behind each (`seam` /
`carve-face`), its grade, its hardness, ore remaining, and whether it is
blocked.

⭐ Per-face depletion is state about the **room**, so it rides the room's
own record — which is also the only way a static mine has it.

**Stability is `f(span, ground, support, water)` and a THRESHOLD, never a
roll.** A ground FLOOR term says *rock is rock* (an untouched drift holds
itself up whatever it is cut in); the hardness term above it says *granite
is better than slate*. Support is a **sum over the timber sets actually
standing in the room**, condition-weighted — so ground support rides the
shipped repair economy rather than a flag, and a mine that runs out of
timber has a supply problem.

⭐ **`shore` is the mine's provisioning act.** It promotes a cell from
Provisional to Held and writes the record in the same breath, so *"a
working persists iff it is shored"* needs no separate bookkeeping.

⚠⚠ **A fall blocks a FACE, never a room.** Every exit stays open, nothing
cascades, nobody is buried. The cost is a shift barring the loose down
and a bruise through the shipped harm channel. The last cut off an
undercut face in unquiet ground is the one that runs — so the free
telegraph (which rides the **same number** as the refusal) has already
fired, and an attentive player is never hurt.

---

## Air — the one lethal hazard

**Air is the shortest distance, along the carved graph, to a room that
breathes** (one with a way out of the workings' own zone — the adit).
A dead end degrades as you drive it and **recovers the moment you hole it
through**, so planning a connection is a real decision.

The working memos its derived air onto its own `_atmosphere` override —
**derive-on-read for truth, write-through for consequence** — so a spent
heading simply *reports a different atmosphere* (`blackdamp`) and every
shipped consequence follows: the respiration crisis, mortality's
rescuable dying clock, recovery on walking out. **Nothing in respiration
learns what a mine is.** The write-through fires after a carve (the
warren knows when the shape changed) and at `postRegister` (so a static
mine holds foul air too).

⭐ Air is the RIGHT lethal hazard for a build with no population: a free
continuous warning, an obvious unilateral escape, and **no rescue
required**. Ground cannot kill; air can.

⭐⭐ **The canary is not redundant with a nose.** `stinkdamp` reeks and
smell catches it; **`blackdamp` is odourless**, and the only free reading
of it is a bird that stops singing. Nothing makes the canary die
specially — it breathes what everybody breathes and succumbs first
because its species row puts `spo2` survivable at 82 against a person's
50.

---

## Ore, grade and the smelt

`Ore` is `Globbable` with **one new field: `grade`** — a fraction, and
explicitly *not* `GradedMixin` (that is the `poor…masterful` quality
band). Two lumps of one row pool regardless of grade, which is what a
cart does; `onMerged` mass-weights the average and `onSplit` carries it
onto a sample.

⚠ The merge order is load-bearing: `canMergeWith` → `setQuantity(total)` →
`destruct(absorbed)` → `onMerged(absorbed)`, and **a destructed stack's
accessors return `undefined`**. The two figures are captured in the
`canMergeWith` pre-hook, keyed on the candidate's `stuffId`.

⭐ **The lie moves from physics to declaration.** Ore that pools cannot be
audited lump by lump, so high-grading is a theft that works *because* ore
pools. Stage A ships the pooling and the honest assay.

⭐⭐ **The smelt's yield is chemistry, never a recipe constant**:

    metal out = Σ (lot mass × lot grade × the mineral's metal fraction)

Every term is a fact something else already knows. Nobody authors how
much copper comes out, which is what makes grade load-bearing end to end.

---

## Title

A claim is a `ParcelRecord`, written **only** by the gated `ParcelApi` and
never declared in content — which is exactly why ⚠ **a content edit
cannot forge a title.**

⭐ **A claim is STAKED, not bought.** `title buy` is buying from a
catalogue; staking is a first-come registration, so `trade-mining` ships
its own `stake` verb calling `ParcelApi.subdivide` + `transfer`. It mints
**a title and no room**.

The cell↔claim mapping is **declared on the warren** (`claimBlocks`),
never derived from geometry: parcels are titled over PATHS and a claim is
a region of coordinate space. ⭐ A static mine needs none of it — authored
rooms have real paths and longest-prefix resolution answers directly.

**The split estate falls out of the path split**: the surface pithead and
the mine are different zones, therefore different paths, and parcel
ownership resolves by longest prefix. No parcel at the mine's path ⇒ the
surface holder owns what is beneath (an unsevered estate); severance is
`subdivide`.

---

## ⭐⭐ The trade/locality line, and how to check it

> **A second mining town must need ZERO pack code.** If making a mine
> look, read or feel different requires touching `trade-mining`, the
> wrong thing is in the trade.
>
> **Code is shared; content is copied.**

What the trade ships: the acts, `WorkingMixin`, `MineWarren`, the
`Deposit` **class**, `Ore`, the instruments, the tool recipes, the
`geology` Discipline, the archetype, and the two **functional** species
(pit pony, canary — haulage and air-reading are needs any mine has).

What a venue ships: the deposit **row**, the four procedural room type
rows and their prose banks, its own ecology and light, its businesses and
cast, its zones and spine rooms.

The check a reviewer can run:

```bash
find packages/content/rejection -name '*.ts' | wc -l   # 0
```

⭐ **The demonstration is the archetype's `light` slot, which ships with
NO DEFAULT.** The trade says *you need light underground* and names
nothing; Rejection answers with glowcap, another mine answers with oil
lamps. `bulkSource: lamp-oil` would bind the oil and refuse the fungus;
`tool: lamp` would make a bed of mushrooms a tool.

⚠⚠ **The glowcap is a light, not a crop, and the word *cultivated* has
been removed everywhere it appeared.** The fixture and the jar are
`PortableLight`s: they do not grow, do not die and do not compose
`GrowingMixin`, and the fungus-farmer that earlier prose implied does not
exist. The `light` slot is genuinely satisfied — 40 lux clears the
archetype's 20 — so the divergence demonstration holds; what was fiction
was the *money sink*.

⭐ It is the forcing case for the next build, and the question is bigger
than mushrooms: the Ferrow zone's `stocks:` table distributes three fauna
procedurally, while the glowcap is a hand-placed fixture with no table
entry at all, and **nothing says how those two layers relate when an
author bypasses the procedural one on purpose.** See
authored-vs-procedural-slate.

### The kernel footprint, in full

Three files, and all three are **vocabulary widenings**:

| File | Added | Why not a pack's business |
|---|---|---|
| `lib/archetype/Archetype.ts` | `lightLux` | the need vocabulary is closed and evaluated by the kernel |
| `platform/idea/api/BiomeLogic.ts` | `blackdamp`, `stinkdamp` | the atmosphere tables are the respiration driver's; the fire build widened them the same way for `smoke` |
| `lib/mixin.ts` | two `MixinRefusals` phrases | refusal prose for kernel mixins; one (`AdornmentMixin`) was already missing |

Plus `packages/types` + `packages/client` for the `survey` card, which is
the whole of the client work.

### ⚠⚠ Three things a pack author has to know, all learned the hard way

**1. A pack cannot add a field to a kernel class.** `fieldMeta` is what
the hydrator reflects through, so an undeclared key in `data:` is
*silently discarded*. `deposit:` on a plain `CartesianZone` came up
missing and `hew` refused in a room with a seam visibly in the face. The
first fix was a `MineZone` subclass shipped by the trade.

⚠ **That fix was wrong, and the reason is worth more than the rule.** It
made the zone covering a whole TOWN a `MineZone` — asserting *Rejection
is a mine*, when Rejection is the town, the Ferrow is the orebody and the
diggings are the workings cut on it. `deposit` is a fact about the
**ground**, exactly as elevation is, so it now lives on `SpatialZone`
beside the other region fields and the subclass is deleted. The kernel
carries the citation string and never interprets it — the same contract
`Locality._reach` uses for a watercourse.

⭐ So the rule has two halves: *a pack cannot add a field to a kernel
class*, and **if the field is a fact about something the kernel already
models, the answer is a kernel field, not a subclass.** Ship the subclass
only when the CLASS is genuinely yours.

⚠⚠ The failure is now caught at test time for zones —
`SpatialZone.authoredFields.test.ts` reads every shipped zone row and
checks each key against the field meta of the class the row itself names.
It found two on its first run: `deposit`, and `address` — which had been
`AddressLogic`'s documented step-2 fallthrough with **no zone class ever
declaring it**, so `source: 'zone'` was an unreachable branch of a
shipped enum.

**2. A reference `Idea` needs warming, and nothing does it for you.**
`Material` and `Biome` have boot rosters; `Deposit` does not, so
`findByTemplatePath` reads `null` forever on a fresh process. Resolve
with `StuffApi.singleton` (get-or-create) at the point of use rather
than adding to a boot list somebody has to remember.

**3. A command affordance is a STATIC ON A CLASS.** A row's
`commandContributions:` is dead silently. `WorkingMixin` carries the
five acts on `self` + `inventory`, so they belong to the working you are
standing in.

### ⚠⚠ And one about engaged acts

**A controller is EPHEMERAL and an engagement is not.** One clone per
execution, destructed the moment `execute` returns — so a completion
that calls `this.<method>` runs on a destroyed Stuff and the proxy
answers with a silent no-op, and one that narrates to a logged-out actor
throws an unhandled rejection that takes the process down.

Completion bodies are module functions over captured locals, and each
begins by checking whether the actor is still there. ⭐ A subclass that
has to be consulted AFTER the act is a subclass that cannot be consulted
at all — which is why the vertical pair passes a VALUE (`edgeMedium()`,
read at dispatch) rather than overriding a hook.

---

## What Stage A does NOT cover, and what the next stage inherits

Salvaged from the build plan at its retirement, because these are the two
places Stage A deliberately shaped itself for a later one:

- ⭐ **`MineWarren` is shaped for a BASE SWAP, not a redesign.** When
  `HoldingWarren` graduates to the kernel — with mining as its second
  consumer, which is what the two-consumers rule asks for — adopting it
  is `extends InnerWarren` → `extends HoldingWarren`. The only genuinely
  residential residue to leave behind or make optional is the shell
  condition and its weathering clock; the keyed-member convention, the
  tenure/promotion split and the reconcile are already the shared shape.
- ⚠ **High-grading ships as pooling and an honest assay; the OFFENCE
  does not.** A miner can already pool a rich lump with lean ones and the
  scale reads the pooled figure — so the lie has moved from physics to
  declaration, which is where an offence can be defined. What is missing
  is an adjudicator, and that is not a mining problem.

⏳ And the boundary of the whole stage is a fact about the ground rather
than a project line: **an adit drains by gravity and the oxide cap sits
above the water table**, so there is no shaft, hoist, pump or drainage
commons, no sulfides and therefore no roasting. Iron wants the bellows.
Tin has chemistry (`/stuff/idea/material/element/tin`) but no ore, which
is exactly what keeps bronze a later stage rather than a missing row.

⚠ **One acceptance criterion is only partly met**, recorded here rather
than quietly dropped: the requirements ask for *"no net money created,
asserted by a test that totals the ledger across the loop"* (ore →
smelter → smith → miner). What ships is the structural half — the kernel
chokepoint's own conservation test (`lib/banking/__tests__/money-integrity`),
and a content assertion that none of the four businesses carries an
endowment, float or credit line. The dynamic four-business loop test is
not written.

## Cross-references

field-substrate ·
mining ·
metal-chain ·
rejection ·
[persistence](./persistence.md) · [parcel](./parcel.md) ·
[chattel](./chattel.md) · [crafting](./crafting.md) · [fire](./fire.md) ·
[holding](./holding.md) · [residence](./residence.md) ·
[advancement](./advancement.md) · [belief](./belief.md) ·
[respiration](./respiration.md) · [content-packs](./content-packs.md) ·
uncertainty
