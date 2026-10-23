# Cleanliness — dirt by kind, the wash, the clean tiers, kept places

> *A thing gets dirty only through an act, and comes clean only through
> one.* There is no clock on dirt anywhere in this subsystem — with one
> declared exception (the byre's stalls, § Places).

Built by the cleanliness build (2026-10-17 → 10-19, MR !380; the plan —
decisions D1–D31 and build decisions B1–B58 — retired at the sweep, see
§ History). Seeding slates: laundry, room-condition, rendering § 9,
bathroom.

## The model

**`SoilableMixin`** (`lib/cleanliness/Soilable.ts`) is the one carrier.
State, all persistent runtime state, scalar-decomposed (⛔ never a `Map`):

| field | what |
|---|---|
| `deposits` | `site\|kind` → a dimensionless load. `site` `''` is the whole thing; a body keys by body part. |
| `setDeposits` | which of those have SET (blood washed hot) |
| `stains` | blood only: the bleeder's type, system, species and identity — ⚠ spoiler level 2, the forensic read is `analyze` |
| `washes` | per site: the last wash (cleanser, action, water heat, hardness) |
| `heat` | the hottest hold since the last deposit (`peakK`, `heldS`) |

**`soil(kind, amount, by?, site?)` is the ONE writer.** It clears the heat
stamp (a sterile thing touched is not sterile) and, on a Contaminable host,
seeds what the DirtKind row says the dirt carries (manure → *e. coli*).
Bands are words: `clean · marked · soiled · filthy` (dials
`cleanliness.band.*`); the look line is `soilPhrase` — *soiled, bloodied
and greasy* — never a number.

Hosts: `Garment`, the body (per part, through `HygieneMixin`), the `Floor`
twin, `Window`, `Chair` (seats, beds, the `Tub`), `Lamp`, `Bandage`,
`CraftVessel`, trade-cooking's `KitchenTool`, trade-medicine's
`SurgicalKit` and `SutureKit`, `Rug`. ⛔ Not `Thing`, `Good`, `Tool` or
`Receptacle`.

### Two row families (no code to add one)

- **`DirtKind`** (`/platform/idea/DirtKind`, rows at
  `/platform/idea/DirtKind/<key>`, a closed set): the adjective, the `form` (`film` washed,
  `debris` swept, `stain` a film that can set), what cleanser `answersTo`,
  `setsAboveK`/`setAnswersTo` (blood: 318 K, then only a bleach),
  `waterLifts` (what plain water takes — mud 0.7, grease 0.05),
  `contaminates`.
- **`Cleanser`** (`/platform/idea/Cleanser`): the material `agent` tag
  the wash finds it by, one `action` from a CLOSED ten-word mechanism
  vocabulary (`surfactant · alkali-mild · alkali-strong · acid-mild ·
  acid-strong · solvent · absorbent · abrasive · oxidiser-slow ·
  oxidiser`), non-corrosive `damages`, `hardWaterPenalty`, `softens`,
  `consumesPerWash`, `skinHarshness`, `ambient` (sunlight, sulfur fume —
  read from the room). ⭐ Burning and rotting are DERIVED from the agent
  material's `corrosiveTo` ∩ the fibre's tags, never authored twice.

⭐ **No catalogues** (B50 — the scan sweep's rule: load by an authored
reference, never by searching). A dirt kind is a closed platform
vocabulary at `/platform/idea/DirtKind/<key>`, loaded by path the first
time something is soiled with it (`Soilable.dirtKind`). A cleanser is named
by its MATERIAL (`cleanser:`, `identity:constituent`, loaded with it —
`Material.getCleanser()`). Each row reads itself (`descriptor()`).

## The producers — every act names its deposit

`Recipe.soils` and `WorkResult.soils` carry a
`SoilSpecs` value (`lib/cleanliness/SoilSpec.ts`): `{kind, amount,
where}` with `where` ∈ `hands · apron · feet · floor`. The act lands it
through `Hygiene.soilFrom`; the wearer decides which layer it reaches
(`outermostAt`). ⭐ A TAP's dirt is derived, never authored (B49): an
animal's take puts a share of the manure on the floor she stands in on the
taker's hands (a fouled byre, or a mucked-out one) and a liquid drawn by
hand splashes the apron with what it is (`Producing.soilTheTaker`); a
plant's tap dirties nothing. Butchery bloodies the apron and the hands; ⭐ an external
bleed (a laceration, a puncture, an avulsion, an embedded object — not a
rupture, which bleeds inward) stains the INNERMOST layer over the wound
with the bleeder's own blood, the bare skin when nothing covers it, and a
heavy bleed drips on the floor (`HARM_DEFAULTS.BLOOD_*`, B44); a glove takes
a hand's deposit once per layer; eating drops `crumbs` on the floor (the
one debris producer, B6); a fire's incomplete exhaust soots every Soilable
fixture in its room; walking carries a floor's soil onto the feet
(`receiveTraffic`).

**Hands** (`HygieneMixin`, `lib/cleanliness/Hygiene.ts`, B42 — split out of
`AttiredMixin`, which keeps the covering stack and the body-site reads;
Hygiene composes outside both it and `Soilable` on `Creature`, so clothing
knows nothing of dirt): `handCleanliness()` (the
derived number the treat path reads — the old six-hour hands clock is
deleted), `soilHands`, `washHands`. `remove gloves` quickly puts half a
dirty glove's load on the bare hand; `remove gloves carefully` puts none.

## The wash

`Soilable.washSite(spec, site)` is the chemistry (B8 — not `wash()`, which
is the vessel service wash). A `WashSpec` is the water's heat and
hardness, zero or more cleansers, and the effort (a scrub is 3). Per kind:
what a cleanser's action answers to comes off, scaled by heat, effort and
hardness (soap scums in hard water; soda softens it); blood washed hot
SETS — in a thing that `holdsSetStains()`; skin does not (B37). A host adds what washing does to IT by override — a `Garment`
felts (protein fibre at a boil), shrinks (≥ 333 K), rots or burns
(`corrosiveTo`), fades and bleeds an unfast dye onto everything in the
same water. Cleansers are spent by use (B12).

⭐ **A wash is water** (B56): `Soilable.washSite` leaves what it washed at
least `washedShare(site)` wet — the whole of a shirt, a rug or a floor (a
mopped floor is slick until it dries, `Floor` ≥ 0.3), a hand's surface
share of a body (`Creature.washedShare` over `Attired.surfaceShareOf`:
washing your hands leaves you dry; a bath soaks you). A thing boiled in a
pot (`holdAtHeat(…, {wet})`) comes out wet. A WADE wets the body by the
share under water, each layer under it whole (`Attired.immerse`). And a
**towel** (`platform/thing/Cloth`, held, confers `dry`) takes it off:
`dry me with towel` → `Wet.soakUp`, the water moved by mass into the
cloth, which then dries on the rack like anything else. The bathhouse
stocks three, each bathroom one, the general store sells them.

Hardness rides the water (`WaterHardness.ts`'s payload merge): a pail keeps its
source's hardness; a tap reads its main's; unknown water washes as 8 °dH.
`Soilable.washRefusal()` lets a host refuse in words — an earth floor is
strewn and swept, not scrubbed (B19).

**Airing** settles at `look` (`reconcileAiring`, B17): a garment lying in
a room by daylight bleaches (linen) or yellows (wool); sulfur fume whitens
wool back.

## The clean tiers — derived, never stored

`cleanTier(site)`, lowest first:

| tier | when |
|---|---|
| `dirty` | soiled or worse, or marked with a film on it |
| `presentable` | marked, no film — or clean to look at but never washed or heated since it was touched (B22: *raw*) |
| `clean` | washed |
| `sanitised` | washed hot (≥ 333 K) with a surfactant or alkali |
| `disinfected` | a boil held (≥ 373 K, 300 s), and no pathogen survives |
| `sterile` | a dry heat or a flame ≥ 433 K for 60 s |

Writers of the heat stamp: `boil <cloth> in <pot>` and `heat <instrument>`
(held in the flame), each also running the shipped kill law
(`Contamination.killOver`; spores survive a boil). Readers:

- **the clinic** — a dressing's quality IS its tier (raw 0.3, washed 0.6,
  sanitised 0.8, boiled 1); `applyTreatment` seeds sepsis from the WORST of
  the hands, the dressing short of disinfected and the instrument short of
  sterile (surgery) or disinfected (a suture needle); surgery seeds
  whatever the wound's type (B23). A competent nurse boils a raw bandage
  first.
- **the kitchen** — a board, knife or hand short of sanitised inoculates
  the food it touches with `staph-aureus` BEFORE the working's kill (B24):
  a stew boiled long enough is safe, a salad cut on a dirty board is not.
  Perishable inputs only.

## Knowing — how a player tells it needs washing (B57)

- **`look`** — the band and its kinds (*soiled, muddy and greasy*); a body
  by region on the bare skin; the wet band.
- **`smell`** — dirt with a nose has an `odour` on its row (sweat → *stale
  sweat*, manure → *dung*, milk → *sour milk*, blood → *old blood*, grease
  → *rancid fat*; mud, ash, soot, flour and crumbs none). `soilOdour()`
  derives the emission from the loads (a body adds what it wears), the
  smell walk scents the room with it, and `smell <thing>` reads *it smells
  of …* (*reeks* at ≥ 100 ppm) once past `cleanliness.odourNoticedPpm`.
- **`analyze cleanliness <thing>`** — the reading
  ([instrumentation.md](./instrumentation.md)): where, what has set, what
  lifts it, what was done to it, how long ago — by `domestic-services`
  band, never the microbes.

## Places

- **The floor** (`FloorMixin` + the `Floor` twin): `takesSoil` (a water
  surface or ice sheet takes none), `cleaningMethod()` exhaustive over the
  ground kinds (`strew and sweep` for earth, `scrub` for rock, `scrub or
  sand` for boards…), `strew`/`takeStrewn` (rushes, sawdust — a strewn
  absorbent drinks blood, grease and milk), `sweepUp` (on a strewn floor
  everything; on a scrubbed one the debris), `sweepsInto` (the heap row:
  sweepings, the byre's muck). Verbs: `sweep`/`muck`, `strew`, `dispose`
  (into a midden), `brush` (alias `beat`: the loose dirt out of any
  thing — a coat, a cushion, a rug — dry, by each kind's `brushLifts`, B46)
  — ⭐ each conferred by what PERFORMS it
  (B43): `SweepingTool` (broom, flue brush, clothes brush, carpet beater),
  `Litter` (rushes), `Heap`
  (sweepings, muck) and the `Midden` itself, never by the place. ⭐ The acts
  are the objects' (B45): `Floor.sweepInto` / `strewWith`,
  `Flued.sweepChimney`, `Burner.rakeGrate`, `Midden.takeRefuse`. `survey` reads *the floor is soiled; to
  scrub it would see to it*.
- **The stalls** — the ONE calendar accrual: a byre floor row names its
  herd book (`stalls: {herdbook}`) and fouls by head × days, reconciled on
  read, following the authored reference.
- **Rot away** (B47) — rotten food left loose in a room three game days is
  gone, as crumbs on the floor; a midden takes rotten food as refuse.
  Where waste goes, and how: waste-slate.
- **Pests** — derived, never a field: debris over
  `cleanliness.pests.debrisAt` and open food in the room. The floor says
  *something has been at the crumbs*; loose food spoils faster by
  `cleanliness.pests.ratePerHour` (B21). Sweep up and they go.
- **The flue** (`FluedMixin`, on `Hearth` and `Oven`; a `Brazier` has
  none): soot by kg burnt × incompleteness; a foul flue under a roaring fire
  CATCHES (heat and smoke into the room). `sweep hearth with flue brush`
  clears it; `sweep hearth` rakes the grate's ash.
- **The grate** — a wood fire leaves `ashFraction` of what it burns; past
  `fire.ash.chokeStartShare` of the burner's capacity the ash **chokes the
  draught** (down to `fire.ash.chokedDraught`), and the fire says *the grate
  is choked with ash*. Raked out as `/trade/fuel/thing/wood-ash`, **made
  from the hearth** — the hearth owner's by the fresh-title rule, so raking
  a stranger's grate is the taking case. ⭐ Since the potash build the ash
  has a STRENGTH (the burnt material's `alkaliShare`), comes out in heaps of
  `fire.ash.heapKg`, and any fire's grate is raked — not only a flued one.
  The hopper derives lye strength from the heaps, leaves spent ash, and the
  soap kettle slakes lime for a caustic lye: [potash.md](potash.md). Big ash
  (a burn's ground) is potash-slate.
- **The window** — grime is a factor inside `Window.transmissivity()`
  (a filthy pane passes 40 %). ⭐ A sky-lit room's named opening may be a
  one-sided pane IN the room (`/stuff/thing/fixture/window`): the vision
  walk multiplies the sky leg by `AmbientLit.openingTransmissivity()`, so
  grime and shutters darken it (B18). The cookhouse, the Hinkley kitchen
  and the cooking kitchen carry one.
- **Rest** — `Postured.restQualityAsKept()`: a filthy bed in a filthy room
  rests worse, never to nothing (the archetype still calls it a bedroom).
- **Dust** — the mill and the scutch put flour/flax dust in the room's
  air; a material's `flammableLimit` flashes on a flame; damped and swept
  it does not. The Hearts' Delight stones moved indoors to the mill house,
  a 4 m room (`extent: 4`) — a dust share is per volume, so the room's
  size is the dial (B36).

## People — somebody is paid for it

- **`order wash <garment>`** — the hand (`CraftingApi.fulfillingHand`,
  restricted to the tariff's own house and the row's
  `serviceDisciplines`) reads the garment at HER band
  (`Garment.describeForLaundress`), names a price (`Tariff.quoteFor`),
  checks you can pay, asks; then washes it HER way: `WashPolicy` — the
  band chooses the METHOD (untrained boils and reaches for lye, novice
  boils with soap, competent reads the fibre, a master never shrinks it)
  and the cloth decides what that does. Ruined comes back with the house's
  posted `cover`, paid by `BankingApi.indemnify` — never to one of the
  house's own seats (B26).
- **`order bath`** / **`bathe`** — `Hygiene.bathe(spec, s)`: every part of
  the body washed, then `immerse` (bathe dressed and your clothes soak),
  then `ThermalRegulation.exchangeWithWater` (B29). A `Tub` holds water; a
  `Copper` is a boiler over its own fire (B30); a shore answers through its
  one draw (frozen refuses).
- **A bath at home** (B41) — the bathroom archetype's tub has a real brass
  tap beside it (`bath-tap`, a cold `WaterFixture` on the house's water):
  `fill tub from bath tap`, `bathe`. Hot water is a copper's, and whether a
  house has one is the house's row — the Hinkley Hills houses' bathroom
  (`lots/bathroom`) carries one: `fill tub from copper`. A boiler on the
  main is a later epoch answering the same `fill`.
- **The doorman** — a `KeptDoor` exit asks a keeper on either side with a
  door-judging brain (`doorman`); HE decides he is on duty (on shift for the
  door's `keptBy` house — a seat, never presence) and judges by the dress
  code on his row; the refusal is his words (B31). `remarks` is the same
  shape on arrival: a fisher notices a banker's coat.
- **The clean contract** — `job post clean <place>` / `clean <thing> at
  <place>`; `Condition.cleanHolds` reads the floor's band, a flue's soot,
  a garment's band (B20).

Venues: the lavoir off the wharfside bank (`lavoir`), the bathhouse north
of the market square, the Goodkin parlor arch. A second of any is rows.

## Persistence

`PersistentLocale` keeps a place's floor, fire and fixtures
across a restart through Adornable's fixture slice (`captureHostOf` walks
fixture → host; ⚠ keyless, so a kept singleton never becomes a keyed holding —
B38). Kept places: the cookhouse, the chandlery, the glazier's
hut, the shop floor, the lavoir, the bathhouse rooms.

## Fidelity limits, stated

- Particulates (flour, flax dust, soot smoke) ride the atmosphere as
  gases.
- Airing samples daylight in two-hour steps at the read.
- Light-strike and pests are sampled once per freshness reconcile.
- The house's soap is not spent by a paid wash (B28) — a paid BATH spends
  it, the bath's own act (B45); the copper's fire is the house's
  abstraction (B30).

## Where the logic lives (review, B45–B48)

⭐ A verb is input and output: it reads the model, calls the act, and words
the outcome. Every act is a public METHOD on the thing that performs or
undergoes it — `Soilable.washSite` / `brushOff` / `holdAtHeat`,
`Hygiene.washHands` / `bathe` (each spends its own soap, `CleanserUse`),
`Metabolic.eat`, `Floor.sweepInto` / `strewWith`, `Flued.sweepChimney`,
`Burner.rakeGrate`, `Midden.takeRefuse`, `Garment.launderBy`,
`Tub.drawFrom`, `Wet.dryingHours`, `Vitals.loseBlood`, and the house's
own on its `Tariff` (`houseVessel`, `houseWaterHardness`, `houseCleansers`,
`quoteLaundry`, `payCover`). Only what NO one object owns is an Api static:
`CraftingApi.wash({items, spec})` — several things in one water, beside `repair` (the upkeep twin: dirt where repair is wear); no Api of its own.
No `lib/` statics, no module functions for an act's internals. The
tree-wide rule is controllers-are-io-slate.

## Open — the deferred seams (graduated from the plan at the sweep)

Each names where it goes; a slate that does not exist yet is the line that opens it.

- **The Washer at the Ford and the dying index** → `docs/slates/tails/washer-at-the-ford-slate.md`
  (cut by the user, 2026-10-17; the slate exists). The attach point
  is the dialogue-effect registry (`BankCounter.onCreate`'s shape) and a
  maintained index at `beginDying`/`stabilize`/`expireDying`.
- **Grime on a construction glass face** (rubber's hothouse roof) →
  `glass-slate`: the attach point is rubber's `Face.transmittance()` /
  `Exposure.faceTransmittance(face)` (asked of rubber — today its two
  readers each read the cached figure); a face then multiplies by a
  per-place soot term the way a `Window` does; no per-place field until
  then. The stove house (a sooty stove under glass; a grimed roof → less
  solar gain → a colder house → the latex tap closes) is the first
  consumer. ⭐ Beside it: repair W7 weathers a *glazing layer* (putty)
  while this build grimes a *Window object* — different objects until a
  Window is a face of `Exposure`; note both in glass-slate.
- **The house mends it** — a laundress-rotted garment is `wear`-damaged,
  so the tailor's mend (repair's band table) is the second remedy beside
  the cash cover → `retail`/`contract` slates with the appraisal line.
- **Wind on the drying line, pests in a store, sawdust strewn on ice for
  footing** — `windEnters()` is consumed once rubber lands (D11); storage's
  `StoreConditions` reads `Floor.hasPests()` when the granary arrives
  (storage-slate names it); strewn sand on a sheet clears at thaw (storage's
  `clearSurface` hook) — all later.
- **A flue on the forge and the kiln; a building's shared stack; air
  mixing between rooms** → `structures-slate`.
- **Dirt from the calendar** (road dust, damp, mildew) →
  `stewardship-doctrine.md § 3a`'s legible-local-process rule; the
  byre's stalls are the one instance and the shape to copy.
- **Stale urine's collection** (who fills the tub) → `sanitation-slate`.
- **An appraisal of a garment's value** (the laundress pays a cover,
  not a value) → `retail`/`contract` slates (an `appraise` reading).
- **Silk** → ⭐ a raw-goods chain nobody has designed (sericulture: mulberry,
  the worm, reeling, throwing) — the user's call, 2026-10-17; no slate yet.
  **Cotton** → `textiles` content. The laundress's ruin case uses wool.
- **Seznick's and the dorm's windows as objects** → content, one row
  each over `Window`, when those homes want the drive's light step.
- **Washing soda as a trade's product** → the kelp burn (underwater build);
  the shipped boiled-lye "soda" row was potash and is retired — potash is
  [potash.md](potash.md)'s. Glycerin's products → `cosmetics-slate`.
- **Clean-home-is-immunity** → `disease-slate`; **eyes as anatomy** →
  `physiology-slate`; **the sterile field** → `surgery-specialty-slate`;
  **chlorine, detergent, the machine** → `laundry-slate` / `inquiry-slate`.
- **Hiring a cleaner onto your own premises** → `standing-instructions-slate`
  (player-hires-player ships through the `clean` contract).
- **The `props`-only half of `Staged`** (repair's finding) — unchanged
  here; no NPC in this build is born holding kit.

### Found by the browser walk (2026-10-19), not this build's

- `smell` / `feel` / `listen` on a thing or a person print its untagged
  VISION prose (the targeted sense keeps untagged text) → `senses-slate`.
- The transfusion order collects its fee in the completion, outside the
  command frame — probably free, like the bath was → the blood build's
  `OrderController` arm.
- The Terminus arrival hall is pitch dark at night; a fresh character
  shivers in the lounge → content (a lamp; the lounge's warmth).
- The client's command-slot popup swallows Enter; a teleport typed under a
  `terminal>` slot echoed without its destination → `client-shell`.
- `bank open` on an existing account says it opened one (it re-links the
  implant) → `banking`.

## History — the cleanliness build (MR !380, 2026-10-17 → 10-19)

Built W0–W7 (`reqs/cleanliness`), then a long review that reshaped it; the
decision log is B1–B58 (the plan, retired at this sweep — `git log --all --
docs/plans/cleanliness-plan.md` recovers it). The review's load-bearing
moves: logic out of controllers onto the objects (B45–B48: `Metabolic.eat`,
`Floor.sweepInto`, `Burner.rakeGrate`, `Midden.takeRefuse`, `CleanserUse`,
`Tub.drawFrom`, `Garment.launderBy`; the one-water wash is
`CraftingApi.wash`, no `CleanlinessApi`); hygiene off `Attired` onto
`HygieneMixin`; no catalogues (a material names its cleanser, a dirt kind
loads by path, B50); a tap's dirt derived (B49); `beat` → `brush` (B46);
the place taxonomy — `Place` · `Locale`/`Location` · `Persistent…`, every
place on a grid but the void, a room's size its zone's cell (B51–B54);
`RoomWarren`/`CirculationWarren` (B55); water and drying by one law, the
towel (B56); `analyze cleanliness`, `domestic-services`, dirt that smells
(B57); the browser walk's fixes — the lit bathhouse, the bath paid at the
desk, the honest unlinked-implant refusal (B58).
