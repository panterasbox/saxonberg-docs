# Soil

**What ground is, what has been done to it, and what it costs to bring
in.** Three axes that must all be satisfied and none of which substitutes
for another:

| axis | who decides | where it lives |
|---|---|---|
| **permission** — may anything be grown here? | the polity | `LandUse`, on the parcel |
| **capability** — what is this ground made of? | the world | `GroundCharacter`, seeded |
| **improvement** — what has been DONE to it? | the holder | `ImprovableMixin` |

Source of truth for the field, the soil ledger, ground character and the
survey ladder. The plant that grows in it is
[husbandry.md](./husbandry.md); the animals that eat it are
[ranching.md](./ranching.md).

---

## The split: `SoilMixin` is kernel, `GroundCharacter` is the pack's

> ⭐⭐ **Moved 2026-09-23 (the ground build):** `GroundCharacter` is no
> longer `trade-farming`'s — it lives in the **`/system/ground`** pack, at
> `/system/ground/idea/GroundCharacter`, beside `Deposit` (which came out
> of `trade-mining`) and `StrataMixin`. Nothing about the model changed;
> what changed is who owns it. It failed the `/system/` test where it was:
> *a system is true whether or not anyone is participating in it*, and dirt
> is there with nobody farming it. While it was a trade's, a wood could not
> read its own ground, a quarry would have had to depend on a mine, and the
> kernel's `Floor` had nothing to ask. It now composes `GroundSourceMixin`
> and answers `groundMaterialAt(spot, zM, address)` **within its topsoil
> depth**, which is how a floor on grade says what it is made of. See
> [posture.md](./posture.md) for the floor's five-rung ladder.

Cultivation before the farmstead build was `PlantPot` and `GardenBed`: a
*Thing with a bulk interior of soil in litres*. A field-room is not that
shape and must not be made that shape, so the soil half was lifted whole
out of `CultivableMixin` into **`lib/husbandry/Soil.ts`**, whose host
constraint is `Stuff & Reserved` and nothing more — which a Location can
satisfy.

⭐ **The two halves of soil have different consumer sets, and that is the
whole placement decision:**

| | consumers | home |
|---|---|---|
| **derived** — moisture, nitrogen, organic matter, structure; the reconcile; the sky edge | `GardenBed`, `PlantPot`, `Field` **and** the forestry `Wood` (a location that is ground: reserves installed from its area, the stand as its drinker — [forestry.md](./forestry.md)) | **kernel** |
| **seeded** — texture, drainage, aspect, depth, stoniness, native pH | **only `Field`** | **`trade-farming`** |

A bed's soil is *imported* (D65), so a bed has no ground character; a pot
certainly has none. Nothing in the kernel needs the seeded half.

⚠ **Not `/system/soil`, yet.** The axis test passes — soil is true
whether or not anyone farms it — but the field-substrate slate's own rule
governs: *two instances is where a pattern is NAMED, not factored*.
Promote when a third, non-farming consumer appears; a move is a path
rename and the standing rule is *no migrations, a rename is a DB drop*.

### The two host hooks, and one protected seam

The lift needed exactly two seams, which is the measure of how
self-contained the soil already was:

- `soilWaterDemandPerGameDay()` — *who is drinking?* A bed answers with
  its occupants' summed demand; a field with its sward's transpiration.
- `soilCatchmentAreaM2()` — *how much sky?* A pot answers `0`, so a pot
  catches nothing and is watered by hand, which is correct: a houseplant
  is furniture, not production.
- `watershedScope()` (protected) — carried ground asks its **container**;
  a field, being a place, answers itself. ⚠ The default returns `null`
  rather than falling back to `this`, because unplaced ground has no sky
  and resolving it to "nothing" would swallow the rain it catches once it
  IS put down.
- `soilLeachRate()` — the host answers, because only the host knows its
  texture.

### ⚠⚠ Two checkpoints, and the tri-state

Soil keeps **two** stamps: the reconcile window and the sky edge.
Resolving which locality covers a piece of ground is asynchronous and the
reconcile is synchronous, so there is a window — sometimes across a
restart — in which the ground does not know where it is.

> **Unresolved must never read as zero.** While `_rainResolved` is false
> the rain stamp does **not** advance, so the first successful resolve
> integrates the whole backlog. *Not yet resolved* and *resolved to
> nothing* are different values and only the second credits zero rain.

The same tri-state is repeated by `Field`'s season edge (`_ambientK`,
`_daylightFraction`) and by `GrowingMixin`'s `_lastAmbientK`, for the
same reason: this codebase has been bitten three times by a cache nothing
warms reading a default forever while hand-built tests stayed green.

---

## The four reserves

| reserve | in | out | teaches |
|---|---|---|---|
| **moisture** | rain, watering | ET, uptake | evapotranspiration |
| **nitrogen** | manure, legumes | uptake, leaching | the cycle |
| **organicMatter** | manure, residues, roots | slow mineralization | the long game |
| **structure** | rest, roots | compaction, poaching | why you don't graze wet clay |

⚠ **Unmodelled is not zero.** Ground authoring no structure reads `null`
and is not structureless, exactly as a pot authoring no nitrogen is not
starved.

### ⭐⭐ Crude protein is nitrogen × 6.25

The engine had both halves and had never connected them: soil carried a
`nitrogen` reserve, and `Material` carried `nutrientAmounts` where
`stew-meat` authors `protein: 26000`. `PROTEIN_PER_NITROGEN` is the join,
and it closes the faucet **mechanically** — harvested matter carries away
the nitrogen it took — and **pedagogically**, because the player watches
fertility become feed value become growth become fertility.

⭐ It is also *why* understocking hurts: stemmy grass is low-protein
grass, so a sward that got ahead of the herd is both worse feed and a
smaller export. One fact, two consequences, no second rule.

### The two honest openings

**In: legume fixation** — clover takes its nitrogen out of the
atmosphere, which is a genuine faucet in reality and the one that makes
the rotation derivable rather than a bonus. **Out: leaching** — rain
carries nitrate past the roots, scaled by texture.

⚠ Where the nitrate **goes** is a civics build's (D18). The loss leg
ships; the commons dilemma does not.

⭐ **Only RAIN waters the ground** — and since the climate build, what is
rain is decided by the temperature at the soil's own place (the
integral's phase at the watershed scope's climate site, warmed in the
same resolve as its Locality). What fell as SNOW lies on the floor
(ground.md) and melts into the watershed's integral, not the soil's: a
winter's snow does not water a bed until a melt model routes it there
(field-substrate-slate's coverings seam).

---

## Ground character — the third seeded field

`trade-farming`'s `GroundCharacter`, after weather and mine geology, and
it answers the field-substrate slate's own open question: **seeded and
derived compose, by multiplication.** Character sets the curve; state is
the position on it.

⭐ **Two things it does that `Deposit` does not, both because dirt is not
rock:**

- **The model is OPTIONAL.** `resolve()` is a static taking a nullable
  model, so ground has a character whether or not anybody authored one —
  *a total function under a sparse graph*. An orebody is a claim somebody
  makes about a place; dirt is just there.
- **The properties are CORRELATED.** Slope drains and sheds its topsoil;
  fine texture holds water. Six independent draws would produce
  free-draining clay on a flat bottom, which no field has ever been.
  Texture is a **smooth** field over a 5-cell lattice for the same
  reason: ground has a light end and a heavy end, not a chequerboard.

⚠ The thirty lines of hash-and-mix are **re-implemented, not imported**
(D6) — the fourth instance, same ruling: shared shape, not shared code.

### The citation — how a zone says what its ground is

A zone carries `groundCharacter`, the path of a `GroundCharacter` row,
and it is inherited by the ordinary `lookupField` ancestor walk — so a
farm's own zone declares its ground, or a region declares the ground
under a whole valley and every field cut in it reads the same clay.
`GroundCharacter.forZone(zone)` is the **one** place the citation is
followed; `Field.onCreate`, the three field-work verbs and the survey
channel all go through it, so they cannot disagree about the dirt they
are standing on.

The shipped instance is the university's, and it teaches **infield and
outfield**: a *pin* on the home field's own spot (the ground nearest the
yard, which is where two centuries of muck went), a *lean* over the far
end (out of cart range, thin and sour), and a wet corner along the spring
line. None of that is a fact about the land — it is a record of what
people did to it, which is precisely the part a procedural field cannot
know.

> ⚠⚠ **`groundCharacter` is declared on the KERNEL's `SpatialZone`, and
> it has to be.** A pack cannot add a field to a kernel class: the
> hydrator discards what no `fieldMeta` declares, silently, and the walk
> then answers `null` forever. Farming shipped three readers against a
> field no zone class declared, so the entire authored layer above was an
> unreachable branch — and every unit test passed, because tests hand the
> model straight in. **A cold boot found it**, the same way it found
> `deposit` for mining and `address` for the resolver.
>
> ⭐ The general form is now a gate:
> `SpatialZone.authoredFields.test.ts` reads the problem from **both**
> ends — every key a shipped zone row authors must be declared, and
> **every name any code looks up must be declared somewhere**. The second
> half was added here and immediately found two more:
> `celestialProfile` and the region-scale `suppressesMagic`, both
> documented, both shipped, both unreachable since the day they were
> written.

### ⭐⭐ Character prices IMPROVEMENT, never yield (D55)

`improvementCost` is the whole payload and there is deliberately **no
yield term** anywhere in the class. Stony ground costs stone-picking, wet
ground ditching, steep ground terracing, sour ground lime — *that* is why
some land was farmed for a thousand years and some never was, and the
player pays the difference in **labour** rather than reading it off a
modifier.

---

## Water let onto ground — inlets and irrigators (water supply)

Besides the rain, a ground takes water two ways. **Inlets** (`inlets:` on
the row, a gate and a method) integrate the gate's opening × capacity ×
the method's share. **Irrigators** (`IrrigatingMixin`) are things standing
on the ground or beside it — a sprinkler, rain in a circle — which settle
on their own clocks when the ground asks. ⭐ The sync reconcile kicks a
background `settleInlets`, so a crop read after a turnout's turn or a
sprinkler's run sees the water; before it, inlets integrated only when
somebody surveyed. `settleInlets` claims its window before it awaits, so a
background settle and a survey's never count it twice. What the soil
cannot take ponds; ponds drain and leach salt. See
[water-supply.md](./water-supply.md).

## The survey ladder (D4) — four rungs, no new verb

| rung | cost | reading |
|---|---|---|
| `look` | free | `lookPhrase` — a percept, and it does NOT leak pH |
| `measure texture` | a spade | the **ribbon test**, performed as the real procedure |
| `measure acidity` | a soil kit | pH ± the band your eye earns |
| `analyze soil` | many samples | the holding, and what it costs to bring in |

⭐⭐ **A procedure is a verb; a reading is a channel.** The ribbon test is
a physical act, so it sits on `measure`; what your samples ADD UP TO is
an interpretation, so it sits on `analyze`.

⚠ **The channel is `analyze soil`, not a second reading inside `analyze
ground`.** `AnalyzeGroundController` is the mining pack's; teaching it
soil would put farming code in that pack, and the shape-dispatch
alternative would make farmland unreadable in any install with no mine in
it. They are genuinely different channels — a plane in the rock solved
from strike and dip against texture, sourness and an improvement bill —
sharing the verb and nothing else.

### ⭐⭐ D5 — the survey is per-viewer

*The map is a record of your sampling, not of the world.* Readings are
DISCOVERY beliefs keyed `(locality, spot, channel)`, so two characters
standing on one field hold different surveys — which makes buying land a
real risk and an honest surveyor worth paying for.

⚠ The **truth** is stored and the error band is not: the band is
recomputed at read time, so an agronomist who improves re-reads their own
old field book at their new resolution.

---

## `Field` — a piece of ground you can stand in

`PersistableMixin(WarrenMemberMixin(ImprovableMixin(SwardMixin(
SoilMixin(ReservedMixin(Location))))))` — `MineRoom`'s stack
with soil where the working is, which is not a coincidence: a mine and a
farm are the same shape of problem, and the residences build already made
that one substrate.

⚠ Over the **permissive** `Location`, never the singleton one,
because a field is a KIND of place minted many times — safe only because
every instance is keyed `<holding extent>/<leaf>`.

### ⭐⭐ Tracts and fields (tracts build, 2026-10-31)

The ground a farm is made of comes in two sizes, and each has its own
word. A **tract** is walkable open ground — the bench above a farmyard,
the meadow below it, a cottage garden — and it is a *place*
(`trade-farming/src/location/Tract.ts`, the class this doc called
`Field` until then). A **field** is what you plant: a named part of a
tract with soil, grass and a history of its own (`src/thing/Field.ts`,
a fixture of the tract — the "patch" of the field-population build,
promoted). *Patch* is retired in every player-facing word; the kernel's
word for a part of a soil stays *part* (`soilParts`, `groundPart`,
`part-unnamed`, `part-sown`).

- **No verb makes a tract.** Every tract comes from a holding's LAYOUT
  (holding.md § Tracts in a layout): a farm sold off a plat, a house
  lot's garden, or — for the authored venue farms (Quist's, the dairy,
  the campus farm, Waponi's pieces, the goat pen) — its own row. `plot`
  is retired.
- **A holder marks fields** on a tract they hold — `mark <name> [as
  <share> of it]`, `unmark <field>` — and the TRACT decides and words
  every refusal: not yours (the sowing gate: the holder, or a hand on
  shift for the business that works it); zoned below `field` (*"… is
  zoned residential — dwelling, companions and small cultivation — and
  takes beds, not fields."*); never measured; more than is left; a line
  through a standing crop. Unparcelled ground is not policed.
- **The tract keeps soil of its own for its unmarked ground** — the
  REMAINDER — and the kernel counts it as one more area-weighted part
  (`Soil.soilRemainderAreaM2`), so `analyze soil` reads the whole
  ground. A new field takes the remainder's soil fractions with it;
  unmarking folds a field back in by area.
- **The grass, the seedbed and the herds are a field's.** The field
  composes `SwardMixin`; `plow`, `mow` and `plant` name the field on a
  tract of several (`plow at the gate end`); a herd grazes a field
  (ranching.md); a trough on the tract feeds any field's herd. A tract
  with no field marked reads *"Nobody has marked a field on it. It is
  rough ground."*, and nothing is sown on it.
- **A field keeps its own record**, keyed `<tract key>/<field key>`
  (`PersistableMixin` on the fixture); the tract's capture writes its
  fields (`Persistable.dependentPersistables`) and an unmarking deletes
  the record.
- **A tract is read from where it is reached** — `look upper bench`
  from the farmyard lists its fields (`Place.isSeenAcrossExit`); working
  a field still means standing in its tract.

### ⭐⭐ The ground key is not the coordinates

A holding's member is on nobody's grid, so every copy of a layout would
sample the same dirt. A tract that is one copy of a layout takes a spot
on the soil field seeded from its key (`<extent>/<leaf>`) when its
holding wakes it, so the valley's four farmsteads are one layout on four
grounds. An authored tract authors its spot.

### ⭐ An AUTHORED tract stands itself up

`reserves` is engine state, so a row cannot declare soil. `onCreate`
resolves the ground and installs what its character calls for, which is
what makes authoring a tract possible with **zero pack code** (AC 62).
⚠ It skips a tract with no area: a holding's copy is sized only when its
holding wakes it (`onWarrenWake`), and installing at registration would
seed a zero-capacity moisture reserve the real sizing would then find
present and skip.

### ⭐⭐ A soil made of parts — the field (field population, 2026-10-30; the "patch" until the tracts build)

A tract may author **fields** (or have them marked at runtime): named
parts of its ground, each soil of its own. The bench is *the top end ·
the middle · the lane end*; the campus home field *the gate end · the
middle · the far side*; every one-piece venue tract carries one
pre-marked field (`sward`, or `planting` on the cane pieces).

```yaml
fields:
  - key: top-end            # stable — crop rows root here by key
    name: the top end       # a definite phrase; its "the" is the register
    share: 0.25             # of the tract; the last absorbs rounding when the shares tile it
    lean: { depthScale: 0.35, stoninessScale: 1.5, nitrogenPct: 12 }
    description: The top of the strip, where the soil thins to a skin.
```

**The parts protocol is the KERNEL's, on `SoilMixin`** — so every verb
reaches a field without importing the farming pack:

| hook / read | what it answers |
|---|---|
| `soilParts()` | the ground's parts (`[]` = one piece) — a `Tract` returns its fields |
| `soilPartKey()` · `soilDepthM()` | a part's key, and its topsoil depth (the roots read) |
| `soilPart(key)` · `soilPartNoun()` | find a part; the host's word for one (*field*) |
| `soilRemainderAreaM2()` | the host's own soil between its parts — counted as one more term when it has area and a reserve |
| `resolvePartFor(named)` | the GROUND decides: the named part, the only part, a CHOICE of its parts when none was named (`PartChoices`, `part-unnamed` — controller-liquidity D8: the controller words *"Which field? Name one: …"* with `CommandController.partLine`, or asks with `chooseAmong`), or a refusal with facts (`not-a-part-here`) — never a sentence |
| `rootedAreaCrop(part)` | the area crop rooted in that part (asked of a part, it asks its ground) |
| `soilMoistureHeadroomL()` · `soilNutrientHeadroom()` | litres (the SUM of the parts), points (the area-weighted MEAN) |

⚠ **A divided field's own reads AGGREGATE its parts, and its writes act
on them.** Litres are extensive: `waterSoil` spreads by area share and
returns the sum. Per-cent reserves are intensive: `feedSoil`,
`drawNutrient`, `fixNitrogen`, `addOrganicMatter` and `poach` apply the
SAME amount to every part and return the area-weighted mean — spreading
ten points 60/40 would have halved a divided field's legume nitrogen.
The sward, `turnUnder` and the grazing integral keep reading the field
and get the aggregate. A divided field has NO soil of its own.

**`Field`** (`trade-farming/src/thing/Field.ts`, a "patch" until the
tracts build renamed it; the walkable ground it is marked on is a
**`Tract`**, `src/location/Tract.ts`) is a fixture, minted get-or-create
in slot `field:<key>` (the `ensureFloor` shape) by `Tract.ensureFields`: `CharacteredSoilMixin(SoilMixin(ReservedMixin(
AdornmentMixin(VisibleMixin(Thing)))))`. Its ground is the field's
sample with the spec's **lean** applied LAST
(`GroundCharacter.resolve(model, at, seed, nudge)`), and its moisture
capacity scales by `clamp(field.topsoilM / tract.topsoilM, 0.2, 2)` — a
thin top end holds a third of the water and dries first. Its watershed is
its field's; its thirst is the crops rooted in it plus its area share of
the sward; `ensureCropTitle` and `sowingRefusal` forward to the field.
Its soil stamps are `runtimeState`; since the tracts build a field keeps its own record (keyed under its tract), and before that its state rode its tract's
fixtures slice across a restart (the R4 test advances the clock across
the "restart", or it passes vacuously).

⭐ **A fixture's jurisdiction is its host's** (`AdornmentMixin.
jurisdictionHost`, 2026-10-30) — a field is pathed by its kind row and
held in `getFixtures()`, not contents, so a governed `eval --parcel
<extent>` was refused on every field (and every floor) inside its own
extent until the hook said so.

**The look.** The field's own look carries one line per SOWN field, read
at the VIEWER's band (*"At the top end it is in ear and is failing. It is
short of something."*). `look at the top end` is the field's own look —
the ground's phrase, the crop rooted there read at your band, and any
labeled plant (*"Trial stands here, labeled."*). The stand answers to its
field's words (`look top end wheat`) — derived on `Plant.getKeywords()`
from the part it is rooted in, only for an area crop, so a renamed field
renames the words. A stand is not listed among loose contents (D17): the
field's line is where it reads.

⚠ **Content rule: keys are stable, names may change.** A crop row roots
by `groundPart: <key>`; a key renamed in the field orphans nothing but
silently re-roots the crop in the field's aggregate soil.

The plant side — `groundPart`, the specimen, the labeled plant's root
zone, the banded read, the campus hand — is
[husbandry.md](husbandry.md) § *A plant's part*.

---

## The plot lifecycle (D54–D60)

> **ground → claim → clear → treat → establish → maintain → *revert*.**

Claiming is buying the holding (`plot` retired with the tracts build).
Three acts, all real words for real jobs: **`grub`**
(thorn, root and stone), **`ditch`**, **`lime`**. Terracing
is priced and has no verb: steep ground simply costs more to grub and
never comes fully into heart, which is honest — most steep ground never
was terraced either.

- ⭐ **Newly plotted ground is NOT plantable.** Clearing is the gate, and
  the only one: you can sow sour, wet ground and get a bad crop, which is
  a lesson. You cannot sow a thicket.
- ⭐⭐ **The cleared stone IS the wall** (D56), and limy ground gives up
  **marl** (D66) — the pH lever that needs no kiln, no fuel and no other
  trade. ⚠ Burnt lime is a **stated seam**: `lime` asks for the `liming`
  TAG, never for a row, so a kiln's output works the day somebody ships
  one and nothing today references a row that does not exist.
> ### ⚠⚠ D61 (`forage`) is CUT
>
> Rough ground was to be forageable, with the forage declining as you
> cleared it — *the neolithic transition as a cashflow decision*. It was
> a late requirements addendum that borrowed its design wholesale from
> discovery-slate, and what
> shipped was one verb reachable **only on ground you had already
> plotted**, drawing on a table no row authored, yielding one item
> nothing consumed. The decision it was supposed to dramatise — gather or
> clear — barely existed, because there was no unclaimed wilderness to
> gather from.
>
> **A system half-built is worse than a system absent: it reads as
> designed.** So the verb, its view, its controller, `wild-greens` and
> `Improvable.wildness()` are gone, and foraging is a follow-on that owns
> the whole thing — wilderness included. Clearing is pure cost again,
> which is what clearing a thicket is.

- ⭐ **It goes back** (D58), and the ordering is the lesson: scrub
  returns fastest, drains silt up over years, lime leaches slowest. A
  derelict farm reads as *wild again* long before *sour again*. ⚠ No
  far-past guard — land reverts over the whole absence, or a derelict
  holding could never come to exist.

---

## The sward, and the land uses nobody declares (D7)

**There is no `use` field on a `Field`**, and its absence is the
deliverable. Land use in reality is a description of what you did, not a
declaration; the four uses fall out of two facts — what is standing on
the field, and whether the mouth is standing there too.

| | standing on it | mouth | nutrients |
|---|---|---|---|
| **crop** | a sown crop | elsewhere | **exported** |
| **hay** | sward | elsewhere | **exported** |
| **graze** | sward | **on the field** | **cycled in place** |
| **orchard** | trees | elsewhere | mixed, multi-year |

⭐⭐ Grazing and `mow` are the **same draw on the same reserve**. The only
difference is whether the mouth was here, and that difference is where
the nitrogen goes — so **fertility follows the mouths** is a sentence a
player derives rather than a rule they are told.

⭐ **The *crop* row has its producer** (cover build, 2026-10-28). A field
holds ONE area crop — a single `Plant` loose in its contents with a sown
area, rooted in the field's soil by containment (`Growing.getBed()`, see
[husbandry.md](husbandry.md#ground-is-a-bed-by-containment-and-the-area-crop-cover-build-2026-10-28)).
The crop's ground leaves the sward (`swardAreaM2 = area − crop`), and
what stands drinks the soil per square metre. `plow` (once the clearing
is done) turns the sward under — the grass into organic matter — and
stamps a **seedbed good for a season**; `plant` asks the field
(`sowingRefusal`) and the field answers from that stamp; a second plowing
that season, and a plowing under a standing crop, are refused. `grub` ends
a standing crop. A crop leaving (cut down, grubbed out) leaves
**stubble**, which reads until the grass comes back over it. The crop is
**titled to the business that works the field** (`ensureCropTitle`, a
`Soil` hook the harvest awaits before it mints) — so whoever cuts it, the
sheaf is the holder's; *the holder* is realized as the operator because a
group cannot hold goods. `look` reads all of it in words.

⭐ **The *graze* row has its mouth** (cover build). The field keeps a
**grazing ledger** by herd id (`setGrazer` / `clearGrazer` /
`grazerLedger` / `foldGrazer`): each line's demand
(`tally × Species.dryMatterIntakeKgPerDay()`, `0.1 × adultMass^0.75`), the
feeder kinds it eats from, and what it was given against what it wanted
since it came on. The sward's demand is the sum of the lines; what the
grass does not give, **the troughs on the field give** — asked once per
step of the integral (`swardShortfallFed`), drawn by mass, and a trough
whose `suppliedBy` names a rick draws on the rick when it is empty. So
*fertility follows the mouths* is true of the game now:
`cycleGrazedNitrogen` runs on what the herd actually ate. A herd is put
on a field by ranching's `move` ([ranching.md](ranching.md)); a herd on
no field eats nothing.

⚠⚠ The key is **`sward`, not `forage`**. What a *person* gathers off
rough ground is forage; the standing grass a *cow* eats is the sward.
Forage is **not in this build** — see below — and the distinction is
recorded because the follow-on will land beside this key.

### ⭐ Residual and recovery (D9)

Below half its ceiling a sward has spent its root reserves and the
regrowth **RATE** is penalised until it rebuilds. Not a second stock,
never a dead field. And **understocking is a mistake too**: growth falls
away as the sward closes over, so grass ahead of the herd stops paying.
Two faults pointing opposite ways, and the bands say which is which.

The numbers are real: 0.3 kg DM/m² at grazing height (3 000 kg/ha), a
residual at half of that, 0.008 kg/m²/game-day peak growth.

---

## Winter (D10–D12)

**Winter is not a mode; it is cold and short days at a place.**
`CelestialApi.daylightAt` is arithmetic on two shipped calls — the
declination was already real orbital geometry — and `GrowingMixin` gained
a fifth limiting factor, `cold`.

Three consequences, all the point: the dorm houseplant does not die every
real month; ⭐ **the greenhouse falls out for free** (somewhere warm and
lit in February is an economic decision against the fuel chain, not an
architectural unlock, and nothing knows the word); and the mechanism is
identical indoors, outdoors, under glass and underground.

⭐ **Backwards compatibility is a property, not a promise:** `coldStopK`
is optional and a profile declaring neither cold field is never
cold-limited.

⚠ **D12 holds: winter stops the FIELD and does not touch the MOUTHS.** A
grazed paddock in January still goes down, which is why the feed budget
exists in the design (⚠ none does yet — no head has an intake path; see
the graze note above) and why husbandry does not stop when farming does.

⭐ **Winter stays HARD, and the reason is not difficulty tuning** (the
farming slate's decision, graduated here). Without a season in which
outdoor production stops, everyone eats fresh forever and salt, smoke,
drying and the cellar have no reason to exist — softening winter would
quietly gut the preservation chain ([spoilage.md](./spoilage.md)) and the
greenhouse's only argument. And because the season is global (one
`CAMPUS_LATITUDE` — [weather.md](./weather.md)), winter is a **shared
world event**: production halts everywhere at once, stored goods spike,
and that is demand arriving on a schedule — once every real month (a
season is 7.5 real days under the 12× clock) — independent of how many
players are online. The counterweight is the one the mechanism already
gives: somewhere warm and lit in February is a fuel bill, never an unlock.

---

## Bands

Every band vocabulary in this subsystem is an **exhaustive `Record`**, so
a new band cannot be added without writing its phrase — the coverage half
of the contract, enforced by the compiler.

⚠ What the compiler cannot check is whether adjacent bands are
**distinguishable in prose**, which is the half that matters: two bands
that read alike collapse the whole opacity ladder silently. The
reviewer's test is *can a reader who does not know the number tell this
band from the one on either side of it?*

⭐⭐ And a band is a **percept, not a number in words**: *"the crop is
pale and thin in the low corner"*, never *"nitrogen: moderate"*. The
state band says what it looks like; a separate cause line says why.
