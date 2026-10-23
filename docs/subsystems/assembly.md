# Assembly — things made of parts

A **thing made of parts that keep their identity.** Crafting is
transformation by default: a recipe consumes its inputs and the output is
one material and one mass, with no memory of what went in. That is right
for a knife from a bar and wrong for a pick, whose haft splits while its
head is still good — and for a cask, which is thirty staves, two heads
and six hoops, and is mended one stave at a time.

So an assembly carries a record of what went in, the **joints** that hold
it are rows with state of their own, and the three lifecycle verbs —
`fit` (new), `repair`, `salvage` — read that record. ⭐ **The joint is the
epoch axis**: a wedge or a friction fit needs nothing but hands, a hoop
needs the cooper's driver, solder needs a forge, and how a thing comes
apart is decided by how it was joined, not by what it is.

First consumers: the miner's **pick** (the shipped `pick` recipe,
unedited — its `head` and `haft` slots are already two parts), then the
**wood column** (`trade-forestry` rives, the new `trade-carpentry` saws
and carves, the new `trade-coopering` raises casks) and an exemplar span
chosen for the joint vocabulary: armchair and bed, lantern, plough, bow,
spear/mace/warhammer/flail, shield, boots, the surveyor's compass, the
drop spindle, the hive frame, the smoker.

Kernel: `lib/craft/Assembled.ts` (`AssembledMixin`),
`platform/idea/{Joint,JointCatalogue}.ts`, the assembly arm and the three
verbs' rungs in `platform/idea/api/CraftingLogic.ts`, the family host
classes listed below.

## The model — a bill on the kind, a record on the instance

Two records, on two different things (`lib/craft/Assembled.ts`):

| | lives on | what it says | fieldMeta |
|---|---|---|---|
| **the BILL** | the KIND — the row's `bill:` | what parts a thing of this kind is made of, in what roles, held by which joints | `{persistent, authorable}` |
| **the PARTS record** | the INSTANCE — `parts` + `joints` | what actually went in: material, form, grade, wear, failures, makers; every joint's tension and who made it | `{persistent, runtimeState}` |

```yaml
bill:
  parts:
    - { part: stave, template: /trade/coopering/thing/stave, count: 30, role: structural, plural: staves }
    - { part: hoop,  template: …/hoop, count: 6, role: fastener }
    - { part: head,  template: /trade/coopering/thing/cask-head, count: 2, role: structural }
  joints:
    - { key: hooping, method: hooped, members: [stave, head], fastener: hoop }
```

- **Roles** are a closed four (`PART_ROLES`): `structural · fastener ·
  wear · facing`. A bill part may author `material:` — what a FOUND
  thing's part is (a spawned pick's haft is ash, not the head's iron);
  absent, a found part is the whole's material.
- **A joint spans a SET**, not a pair: `members` names parts, `fastener`
  names a `role: fastener` part.
- ⭐ **Nesting is by row reference.** A part whose own row has a bill is a
  sub-assembly; a fitted sub-assembly carries its record inside the line
  (`PartLine.parts` / `.joints`). So the depth of an assembly is bounded
  by what a player DECIDES about, never by how many pieces it has — the
  compass's card is a leaf with fifteen parts in prose (the leaf rule).
- ⭐ **No child objects.** The record is plain arrays of plain records (the
  `imparts` precedent — no Map, no class instance, no marshaller), so it
  persists with its host's capture and nobody walks thirty staves: they
  are one line with `count: 30`. A part therefore cannot be inspected in
  place — there is nothing there to look at; `look` at the whole reads it.

### ⭐ Wear and failure are two things

A `PartLine` has `condition` (0..1, the set's gradual wear) and `failed`
(a count of discrete member failures). *The haft is split* is `failed 1`
on a count-1 line; *one stave is sprung* is `failed 1` on a count-30 line
with condition untouched; *the hoops are slack* is a joint whose
`tension` fell below `crafting.joint.slackThreshold` (0.5) with every
line sound. Replacing members takes failed ones first and re-averages the
set's condition.

Other line fields: `form` (a stock or covering form), `green`/`greenAt`,
`warped`, `cupped`, `makers[]` (identity paths). The host also keeps
`hands` — the name each maker went by while they were live, so *"It is
the work of …"* survives the maker being offline — and a `gauge` (the
gauger's record, below).

### Found versus crafted

- **A crafted instance** gets its record at the mint (below).
- **A found instance** (spawned from a row, never crafted) reads every
  derived property off the bill at defaults — grade `fair`, condition 1,
  no makers — and **materializes** `parts` on its first mutating act
  (`ensureParts()`: the first routed wear, `fit`, `salvage`). A spawned
  cask yields thirty generic oak staves to a salvager. Crafted and found
  casks are the same kind: same class, same bill.

### The mint — `applyAssembledOutput`

The tangible arm still flattens (one material, one summed mass — still
the whole's). The assembly arm, a new branch beside it in
`CraftingLogic.ts`, keeps what the flatten loses: a line per recipe item
slot carrying that input's material, grade (weakest across the members),
mean condition, form, greenness, cupping, makers, and the input's own
record when it was itself an assembly. Joints come from the row's bill at
tension 1 under the assembler's identity path. It runs when the output
composes `AssembledMixin` **and** has a bill or two-plus distinct item
slots — ⭐ the pick needs no bill for its parts; its two slots are two
parts. A one-slot craft whose input was an assembly carries the record
across. Bill parts no slot supplied (a wear part the recipe does not
consume) are recorded at the bill's defaults.

⚠ **The gather had to learn parts.** The craft gather refuses a `Tool` or
a Crafted thing as an item input (it protects your hammer from being
"metal"), and `pick-head`/`pick-haft` are Tool rows — so the shipped pick
recipe could never have matched. The gather keeps a pool of MADE things,
and an item slot whose name is a part on the output's bill may draw from
it — but only a thing that IS that part (its row, or its keyword), so a
wrought bar is never taken as a froe's blade.

## Joints — rows, loaded on first use

`platform/idea/Joint.ts` — a leaf Idea, the `Operation` shape: authored
under any root at `<root>/idea/Joint/<key>.yaml`, read straight from
`template.data`, never cloned. A bill names a joint by `key`.

| field | meaning |
|---|---|
| `portability` | `hand · tool · premises` — the ladder a trade climbs |
| `instrument` | a capability kind the maker must have in reach; `''` by hand |
| `competence` | `{discipline, band}` to MAKE (and re-make) it, or `null` — legal only on a `hand` row: ⭐ the bootstrap root as data (anyone can wedge a haft, which lets the hand rung make the tool rung's tools) |
| `reversible` | `freely · effort · never` — how it comes apart |
| `strength` | 0..1, the share of a jar the joint absorbs before it slackens |
| `structuralRecovery` / `fastenerRecovery` | what share of members / fasteners a take-apart gets back |
| `tightenable` | whether `repair`'s tighten rung applies |
| `needsFlat` | holds only between flat faces (glue) — a cupped member seats it short |
| `failure` | `slack · sprung · parted · split` — the word |

The shipped rows, and ⭐ **who ships a joint is decided by its competence
Discipline, which must exist when the row ships** (`lint:bills` arm 5):

| pack | rows |
|---|---|
| `platform` | `friction`, `wedged`, `seated`, `glued` (hand, no competence) · `riveted` (tool `striking`, smithing, `never`) · ⭐ `peened` (tool `striking`, smithing novice, **`effort`** — a sword's tang hammered over its pommel: the one joint a smith EXPECTS to undo, so a blade outlives its hilts; the pommel is lost; repair D17) · `soldered` (premises, no instrument — there is no `heat` capability — smithing) |
| `trade-tailoring` | `stitched` (tool `mending`, tailoring) |
| `trade-carpentry` | `pegged` (tool `driving`, carpentry novice) |
| `trade-coopering` | `slack-hooped` novice · `hooped` competent · `tight-hooped` proficient (tool `driving`) |

⭐ `pegged` and the three hoopings being shipped by the trades that
practise them is the proof that a new way of joining is a row with no
engine change.

**`JointCatalogue`** (`platform/idea/JointCatalogue.ts`) warms by CLASS
(`Template.findByClass`), so any root's rows are found. ⭐ It loads on
first use — no `onCreate`, **no boot line** (the scan sweep's rule:
`StuffApi.singleton(TemplatePaths.jointCatalogue)` stands it up). Two
reads on purpose: `warmed(key)`/`allWarmed()` are async and warm on a
miss — every DECIDING read (a joint made, fitted, repaired, taken apart)
goes through `CraftingLogic`, which awaits them; `peek(key)` is sync and
never blocks — the wear route and the look use it, because `wear()` runs
inside a blow. A cold peek answers `null` (wear uses a default strength
0.5) and kicks the warm. A malformed row is skipped and named, never
thrown.

## Host placement — the families; a monolith is one piece

`AssembledMixin` requires `DurableMixin` beneath it
(`__validateComposition__` throws otherwise — an assembly's wear has to
have somewhere to go). ⭐⭐ **It composes on the FAMILIES we write recipes
for, and a thing with no bill is ONE PIECE** (repair D4, owner's ruling
2026-10-16): a pick and a whetstone are both `Tool`s, a cask and a carboy
both `Vat`s. `isOnePiece()` (no bill, no record) is the mixin's own
answer, and every method is total on it — `getParts()` is `[]`, wear and
edge and name and mends are the host's own fields, `fit x to it` refuses
*"It is one piece."*, `look` says so, `salvage` finds no joints and melts.

| class | composition | parts or one piece |
|---|---|---|
| `platform/thing/Tool` | `AssembledMixin(NamedMixin(KeenMixin(CraftedMixin(ToolMixin(DurableMixin(ConstructedMixin(Good)))))))` | pick, felling axe, froe, drawknife, croze, inshave, plough · every other tool one piece |
| `equipment/Weapon` (→ `Launcher`) | `Wieldable(Slottable(Keen(Assembled(Named(Crafted(Durable(Constructed(Good))))))))` — Keen OUTSIDE (it reads outward to edged lines) | spear/mace/warhammer/flail, hunting bow · dagger, sword-as-shipped one piece |
| `equipment/Garment` | `Wearable(Slottable(Assembled(Crafted(Durable(Constructed(Dyed(Good)))))))` | leather boots · every other garment |
| `equipment/Shield` | composes it directly | every shield row is a face over boards |
| `platform/thing/Vat` | `Assembled(Durable(VesselKind(Maturing(Crafted(Sealable(Thermal(Bulkable(Good))))))))` | cask, dry-cask and its `extends:` · carboy, culture jar, crock |
| `platform/thing/Lamp` | `Assembled(Crafted(Durable(Burner(LightSource(Bulkable(Thermal(Good)))))))` | lantern · torch |
| `platform/thing/Chair` | `Assembled(Crafted(Durable(Postured(Slotted(Good)))))` | armchair, bed · stool, tub |
| `platform/thing/Receptacle` | `Assembled(Durable(Thermal(Bulkable(Good))))` — ⚠ NOT Crafted: on a bulk holder Crafted is the BATCH's identity (`wash`/`rinse`/`cool` skip a crafted holder; a pour carries maker and recipe holder to holder) | every receptacle one piece today |
| `platform/thing/Assembly` | `AssembledMixin(CraftedMixin(DurableMixin(Good)))` | the made-of-parts thing with no other capability: cask head, frame, cushion, mattress, bellows. A noun — the twin of `Good` on this axis |
| `platform/thing/Barrel` | `AssembledMixin(DurableMixin(CraftedMixin(SealableMixin(ContainerMixin(Good)))))` | `slack-barrel` — a container of staves, not a vessel |
| pack: `CookPot` (cooking) — a recipe output that wears | `Assembled(ManualBuild(ToolMixin(Durable(ServingVessel))))` | one piece |
| pack `Tool` subclasses (`SurveyInstrument`, `SpinningTool`, `Whetstone`, …), apiculture `Frame`, `Smoker` | inherit it | compass, drop spindle · the rest one piece |

`Timber` composes `Durable` (its condition is stack identity — a worn haft
never merges with a sound one) but NOT the mixin: a stave is stock, it
goes INTO a whole. ⛔ Not on `Good`, `Thing`, `Fitting`, `Chest`,
`WaterFixture`, `Bottle`, `Provision`, `CraftVessel` (a recipe FILLS a
glass, it does not make one), the moulds (no recipe makes one yet —
the moulding build's one line), and the hive rows stay billless on
purpose (a hive is a box you put frames IN; `lint:bills` refuses a bill on
it).

⭐ **The gate that keeps the guard from coming back is "every method
total", not a narrowing.** `lint:bills` arm 7 holds what is checkable: a
bill on a class without the mixin (7a), a two-part `tangible` recipe output
with no bill (7b — a census of seven, `UNDECLARED_ASSEMBLIES`), and the
owner's rule (7d — a recipe output that wears composes the mixin; stock is
exempt). Arm 8 holds that every bill part is MADE by some recipe, or is
named on `UNMADE_PARTS` with why — the trade that mends a part must be
derivable.

⚠ **Base-class statics shadow; mixin statics union.** A class's own
`commandContributions` hides its parent's, but the mixin's verbs
(`fit`/`repair`/`salvage`) are collected from the mixin chain regardless —
which is why, with the mixin deep inside the family, no subclass restates
anything. (The narrow twins had to: `AssembledLauncher` restated `shoot`.)

### ⭐ A name — `Named`'s, living in the part that bears it

A proper name is `NamedMixin`'s and nothing else's (repair D18, folded
onto `Named` in review). `Tool` and `Weapon` compose it BENEATH the
mixin — a sword may be *Hollin*, a grandfather's axe may have a name; a
pail bears none. What an assembly adds is WHERE it lives: a fitted part is
not an object of its own, so its name rides its line (`PartLine.name`),
and `Assembled.getName()` reads the named part before the host's own
field. So a drawn blade carries the name onto the loose blade's `Named`
field, a re-hilted sword answers to it, and because the name IS the
presentation, a typed `Hollin` binds by the scope walk's name score — no
keyword is added. A found thing names its part from the bill
(`BillPart.name`); a crafted one is nameless.

⭐ **Narsil's rule is history, not a grant.** A named part re-made by a
reforge loses its name, and the reforge's `MendRecord` says which
(`nameLost`). `canBeNamedBy(hand)` is DERIVED from that record: the thing
bears no name now, and the latest mend that took one was this hand's. The
name a reforger gives goes on the part that lost the old one. Once it
bears a name the right is spent — there is no field to clear.

## Wear — routed to the part that answers it

`AssembledMixin.wear(amount?, channel?)` overrides Durable's
(`Durable.wear` was widened to take a channel):

- **With a channel** — the line that gives FIRST on it takes the wear: the
  weakest resister among structural and wear lines, scored by
  `MaterialApi.resistanceTo(channel, candidates)` (each line's material ×
  its stock form's `integrityFor`). Every joint that line sits in
  slackens by `amount × (1 − strength)`. The haft takes the sledge's jar,
  never the head. Shock comes from mining's `hew` (0.02 a blow — nothing
  wore a pick before, so a haft could never split in play), every combat
  strike, and a `striking` craft tool.
- **Without one** — the declared `wear` line takes it (the share, the bow
  string, the sole, the wick), else it spreads across structural lines.
- **The covering fold** (`ConditionLogic`): an Assembled layer asks
  `partAnswering(channel)` — the BEST resister, the layer that stops the
  blow — for its material and form, and the wear lands with `wearLine`.
  No synthetic construction, no composite arithmetic.
- **A line fails** when it wears through the broken dial: one member
  fails and the rest of the set reads sound again (a count-1 line has no
  rest). Green lines wear faster (`crafting.greenWearFactor`, 2).

The derived reads: `getCondition()` = the host's own gauge min every
line's condition, and 0 for any failed structural line; `isBroken()` = a
failed structural line, a joint at or below the broken dial, or the
condition test; `leaks()` = any structural line failed or any joint
slack. ⭐ Everything downstream reads those and nothing else:

| host | what a part's state moves |
|---|---|
| `Vat` | `getClosure()` is `'open'` while `leaks()` — the authored closure is the ceiling, the joints the floor (a vat with no joints never leaks this way) |
| `Lamp` | `getEmittedFlux()` × `lamp.crackedGlassFlux` (0.35) per failed `facing` pane — the room is darker; the ONE funnel for what a pane does to the light |
| `Chair` | `getRestQuality()` scales by structural condition; a failed cushion leaves you on the bare frame (`chair.bareFrameRest`, 0.5) |
| an instrument | `Reading.ceilingOf` already reads `getCondition()` — a damaged compass reads coarser for free |
| the plough | the bout wears the share; ploughing refuses naming a failed part |

### ⭐ Green work warps; flat-sawn work cups

- **Warp** (`reconcileWarp`, on read): a line fitted green records
  `greenAt`; once it has had a tenth of its species' `seasoningDays` in
  place, one member fails as `warped` and its joints slacken by
  `greenShrinkage × 6`. That is the stated reason green work fails —
  `look` says *warped*, and `repair` refuses naming the part.
- **Cupping** (D16): a seasoned, through-sawn board with a WIDE face
  (`wideFace`, authored on the board row — a dowel or haft carved from it
  does not carry it) is cupped. A line records `cupped`, and a joint whose
  row says `needsFlat` (only `glued`) seats short of it by the species'
  shrinkage, re-checked whenever a joint is made or re-made — so `repair`
  cannot close it. Pegged and wedged joints do not care. The look says
  why: *"The facing has never held: the shield-board has cupped as it
  dried."* The remedy is a quartered board fitted in its place.

## The verbs

### `fit` — the one new verb (`platform/cmd/crafting/fit.yaml`, `FitController`)

- **Raise** — `fit cask`: a word that names a recipe whose output row has
  a bill. Every joint the bill names is gated (instrument in reach, band
  reached) before anything is spent, then it is an ordinary craft whose
  mint keeps the parts. A recipe whose output has no bill is refused —
  ⭐ `fit` raises things of parts; a transformation is `forge`'s or
  `carve`'s. No can-make deed gate: fitting by hand IS the hand build that
  gate asks for, and a fit records the deed.
- **Replace** — `fit haft to pick`: ⭐ a part fits a line **by row
  lineage** — the same row, either row `extends:` the other (the
  quartered board extends the board), or the part answers to the line's
  name. Never a list of substitutes. The line most in need takes it
  (failed first, then most worn); the line's joints are gated, the part
  consumed, the members replaced, every joint they sit in re-made at
  tension 1 under the fitter's identity, and the whole's grade becomes its
  worst line. A SOUND count-1 member swapped out comes back to hand; a
  failed one is not minted. Ends with `captureHostOf(whole)`.
- The controller prefers the raise when the bound part has no whole and
  the word names a recipe that is not the part's own word (`fit pick`
  beside a head and a haft meant the pick). ⚠ Part names are words
  nothing else answers to: four colliding keywords were removed (`whorl`,
  `needle`, `bellows`, `wick`, and `pick`/`handle` off the haft).

### `repair` — the ladder (`mendAssembly`, one pass)

(Rewritten by repair W2, 2026-10-17 — a mend is per part and can be done
badly; the numbers are `crafting.md § The lifecycle`.) One pass, every
rung, and the whole mends every line it can and reports what it could
not (`RepairSuccess.mended` / `.skipped`); it is a refusal only when
nothing was mended:

1. **A failed member** — metal is RE-MADE at its recipe's heat and tools
   (`requiresHeatK`, `toolCapabilities`; a named part loses its name, and
   the mend records it — `nameLost`, from which the reforger's right to
   name it again is read); anything else is skipped as
   `part-failed`, naming the part (below the joint's band, *something
   about the staves*). The cure is `fit`.
2. **Slack joints** are tightened, consuming nothing — never round a
   member that is still failed.
3. **A spent cask** is re-fired (unchanged).
4. **Every worn line** is mended by its own material's instrument under
   its own trade at the hand's band (`Assembled.mendLine` — the ONE write:
   condition up to the ceiling, grade down, joints slackened by an
   untrained hand, the `MendRecord` on the line, the hand among its
   makers).

The whole's grade is then its worst line's. ⭐ `CraftingApi.diagnose` is
this ladder run without committing — what the paid mend quotes from, and
what `repair` reads up front so a thing with nothing mendable is refused
before the spell of work, not after it.

### `salvage` — by the joints, then the melt

`salvageByJoints` runs first on an assembly: per line, sound members ×
the joint's structural (or fastener) recovery × the salvager's band
(`RECOVERY_BY_BAND`: untrained 0.5 · novice 0.7 · competent 0.85 ·
proficient 0.95 · expert 1.05), rounded; a `never` joint gives nothing
back. Members knocked out by a hand short of the joint's craft come back
one grade worse. What does not come back — failed members, a glued
joint's everything, lost fasteners — melts down the shipped irreversible
branch **per part, in its own material**, weighted by each part row's
`mass`. Conservation holds: recovered mass ≤ input mass.

### Affordance — and the `self` lesson

`AssembledMixin` affords `fit`, `repair` and `salvage` in **`environment`
+ `peers`**: a held assembly grants the verbs to its holder, one on the
floor to the room. ⭐ `fit` is **also the person's** — on `Avatar`'s
`self`, beside `make`: the raise arm IS `make` for a thing of parts, its
hand rung has no instrument, and a pick head is a plain `Tool` that can
afford nothing. `Timber` adds `fit` to `peers` (staves at your feet make
`fit cask` sayable).

⚠⚠ **`self` is a command GIVER's own verbs** — read off a person, a
shadow or a hosted aether app, never off an object. The mixin, `Blank`,
`Bole` and `Billet` all shipped verbs in `self`; every drive checkpoint
had the thing on the floor (which is `peers`), so a pick in your hands
could not be repaired until a browser walk found it.
`lint:self-contributions` now gates dead `self` lists at **zero**.

### The wood column's conversion acts

Trade verbs for rung-1/2 transformations, not assembly's budget:

- **`rive`** (`trade-forestry`, `RiveController`) — a bole (taking one
  length) or a billet, with a `riving` tool; output `riven` stock. ⭐ Only
  a straight grain rives (D17): `Material.cleavability` (oak 0.9 … elm
  0.1); below `RIVE_FLOOR` (0.3) refused naming the interlock, below
  `CLEAN_CLEAVE` (0.8) proportionally fewer billets. ⚠ Latent: no wood
  stands elm yet. Afforded by `Bole` (`peers`) and `Billet`.
- **`saw`** (`trade-carpentry`, `SawController`) —
  `saw <log> [--quarter] [at <saw>]`. Through-sawn gives the most boards;
  `--quarter` gives two-thirds as many `quartered-board`s (its own row,
  `quarterSawn: true`) that never cup. Duration = mass / rate. Afforded
  by the `Sawmill` class. ⭐ **The kerf falls as sawdust** (storage-and-
  ice, 2026-10-17): one kerf a board (two quartered) at the saw row's
  `kerfM` (the sawmill 3 mm, the pit saw 5 mm, default 4) against a 25 mm board, minted
  as whole kilograms of `/trade/carpentry/thing/sawdust` (a `Poolable`
  stack, one heap per wetness band) and the boards lightened by exactly
  what was minted — mass conserved. It is the icehouse's packing (see
  [storage.md](./storage.md)).
- **`carve`** (`trade-carpentry`, `CarveController`) — runs any installed
  one-slot wood recipe that asks for `cutting` and no heat (haft, handle,
  peg, dowel, leg, rail, mallet…); no list in the controller. It sets
  `CraftRequest.subtractive`: the output keeps its row's size and the rest
  is shavings (a dowel weighed 6 kg before). Afforded by `Blank` and
  `Board`. ⚠ No knife offers `cutting` (the knives are Weapons), so the
  by-hand root starts at the billhook — the scarcest instrument in the
  column. See carpentry-slate.

## The grain — stock forms, seasoning, the weakest link

**Stock forms** — a third closed domain in `lib/material/Construction.ts`,
`STOCK_FORMS = riven · sawn · hewn`, with `isCovering()`/`isWeapon()`
false so every existing guard holds. `integrityFor(channel)`: riven is
the reference (1.0); sawn `blunt 0.7, shock 0.7, point 0.85` (the saw cut
across the fibres); hewn `blunt 0.85, shock 0.85`.

⭐ **The tangible arm carries both wood axes, weakest link** (D16): the
output's stock form is the weakest (by shock integrity) of its own row's
and every input's — a haft carved from a sawn board is sawn, and no act
raises integrity, so no list of acts is needed. The output's seasoned
fraction is the greenest input's. `look` names a part's form (*a sawn oak
shield-board*) and a stock piece reads its end grain in words.

**Seasoning** — `lib/maturation/Seasoning.ts` (`SeasoningMixin`), the
second host of the maturation clock. `MaturingMixin` could not host it
(its composition guard demands an interior bulk slot — the batch IS the
slot), so the host-agnostic curves were lifted, byte-identical, into
`lib/maturation/MaturationClock.ts` (a pure value object; the gap loop
stayed in `Maturing`). Seasoning owns one fraction (0 green → 1 seasoned),
reconciled on read over `BiomeApi.airSegmentsFor`:

- rate = the species' `Material.seasoningDays` × how hard this air dries
  (`Evaporation`, wood's equilibrium 85 % RH — a damp shed seasons
  nothing) × the clock curve (stall below 273 K, happy at 290 K, damage
  above 330 K);
- rain on an exposed stack sets it back; dried too hot, the grade caps at
  the worst stretch (the clock's band rule);
- `isSeasoned()` ≥ `wood.seasonedThreshold` (0.8); vacuous on a material
  with `seasoningDays 0`;
- ⭐ **a merge takes the greener** — and so `Timber` is
  `SeasoningMixin(StackableMixin(…))`, Seasoning OUTSIDE, because the
  outer class's `onMerged` is the one that runs.

**`Timber`** (`platform/thing/Timber.ts`, kernel — its composers are
three packs with no common ancestor) =
`SeasoningMixin(StackableMixin(ConstructedMixin(CraftedMixin(Good))))`:
billets, boards, staves, blanks and the carved wooden parts (hafts,
handles, legs, rails — `Timber` so a green haft keeps its greenness and
warps). `constructionForm` and `quarterSawn` are stack identity: riven
and sawn never merge. ⚠ The shipped `/trade/forestry/thing/timber` row is
a plain `Good` and keeps its class; the name collision is recorded. Pack
subclasses exist only to afford a pack's verb: `Billet` (forestry,
`rive`), `Blank` and `Board` (carpentry, `carve`) — each spreads
`Timber`'s `peers` back in.

The Material fields: `seasoningDays`, `greenShrinkage`, `cleavability`
(`lib/material/Material.ts`), authored on the ten wood rows.

## Coopering and the cask

`trade-coopering` (`/trade/coopering`): staves (riven from a billet),
the cask head (boards + dowels, pegged — an `Assembly`), the tools (croze,
driver, inshave, gauging rod — forged blades on carved handles), and the
products. ⭐⭐ **Three products by competence, not three grades of one** —
the three hooping joints are the ladder, because what the trade's
competence IS is how tight the hoops are driven:

| hand | joint | product (commons, `/stuff/thing/vessel/`) | holds |
|---|---|---|---|
| novice | `slack-hooped` | `slack-barrel` — a `Barrel` | things (nails, apples); you cannot pour into it — it is not a vessel |
| competent | `hooped` | `dry-cask` — a `Vat` with a bill, `closure: dryTight` | granular bulk (`requiredClosureFor` asks `dryTight` of a `granular` material; the flours carry the tag) |
| proficient | `tight-hooped` | `cask` — a `Vat` with a bill, `liquidTight` | liquor |

`fit cask` by a hand short of proficient is refused naming the band.
`dryTight` is a new `ClosureLevel` between `open` and `liquidTight`. The
commons `cask` row is what brewing's `cask` and distilling's `cask` and
`charred-cask` now `extends:` (brewing states `imparts: []` so its ale is
unchanged; the charred cask is the plain cask with `charLevel: 1`). ⚠ The
fuel pack's oil casks are `Bottle`s and the water-butt is a source; both
are untouched.

- **Dry-out** (`Vat.reconcileDrying`, on read — including `getJoints()`,
  so `look` sees it): the first read that finds it empty starts the
  clock; after `cask.dryOutDays` (7) empty every joint slackens once by
  0.6 with every stave sound — the joint failing with no part damaged,
  and `repair` drives the hoops back down consuming nothing.
- **Character is spent and re-fired** (on `MaturingMixin`, true of every
  vat): `fillCount`, `fillsSinceRefire`, `impartsSpent` (each fill after
  the first adds `maturing.impartsSpentPerFill`, ~a third), `charLevel`,
  and a second authorable profile `charredImparts`. Effective imparts =
  `(charLevel ? charredImparts : imparts) × (1 − impartsSpent)`.
  `repair`'s rung 3 re-fires: spent reset, charred — re-firing a plain
  cask MAKES a charred one.
- **The gauger** — `measure capacity <cask>` (`CapacityReading`, a
  coopering `Reading` row + class). ⭐⭐⭐ It answers **anyone**
  (competence resolves detail, never access); a holder of the polity's
  `gauger` seat also writes the gauge onto the cask (`setGauge`), and
  everyone else is told the figure is theirs and not of record. The
  standard is the polity's `Government.standards[]` entry (Terminus:
  `cask 25 L`), else the row's realm default; more than 5 % off reads
  *off standard*, to anyone. Terminus's seat is a row on
  `terminus-city.yaml` and a position on the registry chart, rostered to
  the clerk — a second polity needs rows only. See
  [civics.md](./civics.md), [instrumentation.md](./instrumentation.md).

## The compartment — `Chamber` and the drying loft

`platform/thing/Chamber.ts` = `AtmosphericMixin(PlacingMixin(Thing))`: a
space inside a thing with its own air. ⭐ **Occupancy by PLACEMENT, not
containment** (D15, reconciling the fridge-design-pack with the
chambered-vessels slate): `put board in loft` places it `in` the chamber,
its container stays the room, `getEnclosingScope()` answers the chamber,
and every *what air reaches me* read (thermal ambient, the seasoning
clock, the biome walk) starts there. No change to scope-walk or
`canReach`. The one override is `occupants()` → `getPlaced('in')`.

`lib/spatial/Chambered.ts` (`ChamberedMixin`) is the host half: a row
declares `chambers: [{key, template}]`, an INSTRUCTION field applied at
template phase 2, idempotent by key. A chamber stands BESIDE its host in
the host's container, moves with it, and is an owned ref (the destruct
cascade takes it); it is re-minted from the row, not persisted itself.

First composer: `trade-carpentry`'s **`LoftedSawmill`** =
`ChamberedMixin(Sawmill)` (the `sawmill` row; `chambers: [{key: loft,
template: drying-loft}]`). ⚠ Not on `Sawmill` — the pit saw is a blade
and would have had an empty compartment. The loft is an indoor, warm,
dry, draughty `Chamber` row: rain never reaches it. ⛔ **No shipped verb
places a PERSON in a compartment** (`put` is inventory-only; nothing
`enter`s), and none was invented — the goods half shipped; the person
half is the slate's.

### The sawmill

`trade-carpentry/src/thing/Sawmill.ts` = `ToolMixin(Good)`, **one class,
two rungs, both rows**: `pit-saw` (30 kg/h, holds your hands for the cut)
and `sawmill` (20 kg/h per kW off the race, ≤ 300; holds nothing of yours
— the boards are on the floor when you come back). The power read is the
grist mill's verbatim — a room sibling answering `generationW` +
`getReachRef`, the drainage catalogue by path, never an import — so a dry
August cuts slower and a mill with no race cuts at zero. Sited at Heart's
Delight's `sawmill` location, downstream of the millsite on the Delight,
with its own race, both saws and the `timber-yard` counter; the sawyer
is cast there. A second sawmill is rows.

## Gates

- **`lint:bills`** (`scripts/check-bills.ts`) — (1) every bill part's
  `template` resolves; (2) every joint `method` names a Joint row,
  members/fastener name parts, a fastener is `role: fastener`;
  (3) ⭐ every recipe whose output has a bill covers each structural and
  fastener part by an item slot of the same name and count — *one
  declaration*; (4) no bill on `constructionForm: mail`, nor on the
  refused hive rows; (5) every Joint's competence Discipline exists, and
  `competence: null` only on a `hand` row; (6) a bill's parts weigh no
  more than its whole (salvage conserves mass — a 2.5 kg cask head of
  three 6 kg boards refused every take-apart); (7) ⭐⭐ bill ⇔
  `AssembledMixin`, both ways.
- **`lint:self-contributions`** — a non-empty `self` list on a class (or a
  mixin composed by none) that is not a command giver, shadow or hosted
  update is dead. Ceiling 0.
- **`lint:shopfronts`** — a counter that names its house must be one of
  that house's `operatingLocations`. Found when the timber yard could not
  sell a stave; six shipped counters were in the same state.
- **`lint:reachability`**'s mint arm — every shipped recipe parses
  through `Recipe.fromData` and every `outputTemplate`/`outputResidue`
  names an authored row (`spirit-bottle` was a dead output).
  `RecipeCatalogue.warm()` still only warns on a malformed row: a recipe
  is also a document a player authors, and one bad one must not take
  crafting down.
- **The drive** — `packages/wire/tests/assembly.dirty.wire.test.ts`, 23
  checkpoints including a world restart (the mended pick, carried, still
  names both hands).

## Traps the drive and the browser walks found

- ⚠⚠ **Held is not floor.** Every wire checkpoint had the assembly on the
  floor or an anvil in the room, so the `self`-bucket defect was
  invisible for 29 runs. The live browser walk found it in minutes. A
  drive that wants to prove an affordance must hold the thing.
- ⚠ **The clone gate reads a representative.** A carved peg lying in the
  world blocked a later `clone …/peg` — the gate reads the first live
  instance's zone — so the checkpoint carves a dowel instead.
- ⚠⚠ **The drive raced the async reset sweep.** A 40-day `advance` once
  queued ~960 async passes per hourly schedule (`WorldClockApi.every` now
  takes `coalesce`); the single-flight reset/spawn sweeps then DROPPED a
  firing that landed mid-sweep (now a trailing run, never overlapping,
  never lost), and the drive bought the instant after a clock jump, before
  the shelf refilled — the checkpoint now waits, bounded, for par.
- ⚠ **Darkness looks like a broken mechanism.** A reset world boots at
  midnight, and a drive that sets no daylight fails as *"It is pitch
  dark"* everywhere. The Hearthworks smithy also read pitch dark to a
  glowcap-holder at noon in the browser (not this build's; reported).
- ⚠ **A drive world was ~3 GB**: sibling worktrees' servers starved
  several runs into timeouts that looked like defects. Since 2026-10-21 an
  idle world is ~600 MB: the dev compile watcher that held ~2.4 GB is
  gone ([diagnostics.md](./diagnostics.md)), so contention is far rarer.
- ⚠ **Commerce found under a drive**: `buy` once handed over a whole
  heap for one price, a craft could gather unpaid shelf goods (a craft
  never gathers from a shop counter now), and a stocked heap restocked
  without bound.

## Deferred seams

- The table and wardrobe (`Fitting`, `Chest` as hosts), casks that are not
  Vats (the oil casks, the water-butt), the threaded fastener, the
  remaining makerless assemblies, humidity in seasoning, the sawmill's
  water right, a compartment's admission cost and a person in one →
  assembly-slate.
- Carving as an engaged act with exertion, `carve` and `make`'s can-make
  gate, competence in sawing, the lathe, a store that sells an edge, and
  the clasp knife that *"whittles a peg"* and cannot →
  carpentry-slate.
- Two power takes on one reach double-count (the sawmill and the grist
  mill read the same flow) — a coarseness of the water seam.

## History

Built in MR !361 (2026-10-09 → 10-15), plan D1–D18, 33 drive runs and two
browser walks. The review made three passes that changed the shape:
(1) **movement is two layers** — `ContainmentApi.move` and a thing moving
itself — so the plan's `ContainmentApi.land`/`place` and
`lint:mint-custody` were deleted and custody is recorded by `move`;
(2) **ownership follows the inputs** (`StuffApi.clone(…, {madeFrom})`:
the log's owner owns the boards), replacing a rule that carried the
acting person through timers and gave the sawyer your boards;
(3) **narrow assembled subclasses** in place of the mixin on whole
families. Then D16 (the grain is real: stock form and cupping ride the
transform), D17 (only a straight grain rives) and D18 (what the browser
walk found: held affordances, subtractive carving, `wideFace`). The
master scan-sweep merge took `JointCatalogue` off the boot list and
retired `EmploymentApi.ensureOrganization` (an organization resolves at
the act through `StuffApi.singleton`).

**Repair — assemblies part two** (2026-10-17, plan `repair-plan.md`)
reversed (3): the owner ruled that two classes for one kind of thing was
the wrong answer to *is this made of parts*, so the mixin composes on the
families and a monolith is one piece; the nine twins (`AssembledTool`,
`AssembledChair`, `Cask`, `Lantern`, `AssembledWeapon`,
`AssembledLauncher`, `AssembledGarment`, mining's
`AssembledSurveyInstrument`, textiles' `AssembledSpinningTool`) retired
and their 23 rows re-point to the family. What made the reversal safe is
that every method had become total on an empty record.
