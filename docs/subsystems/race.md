# Race subsystem (v1)

The race subsystem describes who/what an in-world entity *is*,
biologically. It splits cleanly into:

- **Material** — what the bulk of an entity is *made of* (iron, oak,
  flesh).
- **Clade** — the taxonomic scope an organism *belongs to*
  (Animalia, Plantae, Fungi, Constructa).
- **BodyPlan** — the *anatomical layout* (unified `slots: SlotSpec[]`, locomotion modes,
  sensory ports) shared across many species.
- **Species** — the *biological capability* attached to membership
  in a species (binomial, lifespan, vision profile, sex-determination
  system, reproductive mode).
- **OrganismMixin** — runtime composition that says "this Stuff is a
  member of a Species."
- **SexedMixin** — biological sex (orthogonal to `GenderedMixin`).
- **SpeciesApi** — kingdom resolution and the `isAnimate` predicate that
  gates command dispatch (plain lifecycle-state reads live on
  `OrganismMixin`).

Three things are *deferred* — the design context lives here for the
follow-on builds:

- Death/resurrection flow — **shipped 2026-07-31**, see
  [mortality.md](./mortality.md). Race still owns the state machine +
  `isAnimate` gating; the transition is now a single call
  (`ConditionApi.die`) reached from every lethal driver, and crossing a
  fatal floor opens a rescuable **dying window** rather than killing
  outright.

  ⭐⭐ **The `death ≠ destruction` rule, as of the carcass-chain build —
  ONE BODY, two choreographies.** The rule means what it always promised:
  a death leaves a persistent, examinable, Creature-tier body in the
  world. It never meant that the *object that died* is the body. What
  varies between the two paths is only whether an **identity has to
  leave**:

  - **NPCs, creatures, beasts** — the body is **replaced by its corpse**:
    `ConditionApi.die` mints a `Corpse` from the dead thing and then
    destructs the dead thing. Nothing is waiting to re-enter it.
  - **PCs** — the body *divides*, over the same mint. A corpse takes the
    material half (the wound map, the cause stamp, the loadout) and
    persists on its own decay clock; the identity walks away as a shade,
    and the drained shell is destructed so the shade can hold the
    identity path.

  ⚠ **It used to be two mechanisms, and the cheap one was wrong.** A
  beast or an NPC was flipped to `dead` *in place* and kept every mixin
  it had — so a dead ewe was a ewe that could no longer be milked,
  sheared, handled, herded or driven, and a dead clerk was a clerk with a
  job. The realm shows a thing's capabilities, so the two deaths read as
  two kinds of object with nothing to explain the asymmetry, and the
  asymmetry was not a design: it was what fell out of the branch that
  needed no new machinery. A corpse's lifecycle is genuinely a different
  lifecycle — it cools, it decays, it is evidence, it is taken apart, and
  it is **never alive again** — so it is a different object, not a living
  body minus some verbs.

  ⭐ **The mint is lossy on purpose.** A herd place, a tap's standing, a
  bond, a job, a brain: none of it survives, because none of it is true
  of a body. What the corpse carries is the species, the cause, the time,
  the mass the body actually had, its birthday (so its age is the age it
  died at), the condition it died in, its own keywords, the material
  slices and any silent contamination load. See
  [mortality.md](./mortality.md) § One body, two choreographies.

  Both paths end with a persistent Creature-tier body in the world,
  which is the invariant that actually mattered.

  `undead` also becomes load-bearing for the first time: it is the
  shade's lifecycle state — animate without being alive. Anything
  branching on lifecycle should read `isLivingBody()`, which is
  deliberately neither `!isDead()` nor `isAlive()` (see
  [mortality.md](./mortality.md) § *Reading lifecycle state*).
- Diet — `Material`'s edibility data now has a consumer:
  [metabolism](./metabolism.md). `edibility`/`nutrients` drive intake
  routing; `nutrients` gained a parallel `nutrientAmounts`
  (tag → mg/serving) for the inspectable profile, and `toxicity` was
  reshaped from `string[]` to **`ToxinTag[]`** (`{type, amount}` — the
  per-consumable dose; a toxin's rate params live on its `Condition` seed,
  not the food). The label render is `NutritionLabelMixin`, not a
  `DietApi`.
- Tissue authoring (named Details with their own descriptions and
  materials) — **partly earned by Vitals**: `BodyPlan.bodyParts`
  carries typed `BodyPart` descriptors with per-part tissue
  composition (muscle / bone / flesh masses), the model layer for
  anatomy and the deferred strength substrate. See
  [vitals.md § Anatomy](./vitals.md). Genetics, per-individual feature
  mixins, polymorph, sleep/circadian, aging, character-creation UI
  remain deferred.

`OrganismMixin` (and Vitals / Reserved / anatomy) now compose at the
**`Creature`** layer — the body class between `Agent` and `Character`
(see [docs/architecture.md](../architecture.md)).

---

## Material substrate

> ⭐⭐ **Decomposed 2026-11 (the material decomposition build).** A material has exactly the
> properties it has. The base class carries only what every material has; everything else is a
> **capability** a **kind** composes. History and the census: § History below;
> material-decomposition-slate §§ 7¾–7⅞.

`Material` (`lib/material/Material.ts`) is a singleton-by-templatePath `Idea` holding only what
EVERY material has: `name`, `appearance`, `tags`, `density` (`kg/m³`), `thermalConductivity`
(`W/(m·K)`), `specificHeat` (`J/(kg·K)`), `electricalConductivity` (`S/m`), the derived
`getEffusivity()`, and `cleanser` (what it washes as — a role soap, lye, sand, spirit and vinegar
all play, so not a capability). Nothing instances it directly: **no row names the base class.**

### Capabilities — named for the physics

Each group of fields a material may or may not have is a mixin in `lib/material/`, named for the
property as a **noun**, with a `MixinApi.has<X>` predicate:

| capability | fields |
|---|---|
| `Combustion` | heatOfCombustion · autoignitionTemperature · flammableLimit · combustionImparts · combustionProducts · ashFraction · alkaliShare · smokePoint · burning |
| `Fusion` | meltingPoint (= the freezing point, read both ways) · latentHeatOfFusion · castTemplate · chillsTo · ruinedByFreezing (+ castShrinkage) |
| `Volatility` | boilingPoint · latentHeatOfVaporization · vapourLPerLPerHour · vapourMaterial · vaporizesAboveK |
| `Solidity` | hardness · toughness · elasticity · cleavability · thermalExpansion · brokenTemplate |
| `Porosity` | waterAbsorptionCapacity · wetStrength · soddenTemplate (+ the wet law) |
| `Edibility` | tastes · nutrients · nutrientAmounts · servingKg — composing it IS "edible" |
| `Toxicity` | toxicity · leaches (what a face gives its contents) |
| `Perishability` | spoilActivationEnergy · waterActivity |
| `Acidity` | ph · corrosiveTo (+ acidAttack, corrodeOnContact) |
| `Solution` | solubility · crystallizesTo · filteredTo · solidsKgPerL · boilingPointElevation · concentratesTo · purifiedByBoiling |
| `Kinetic` | kinetics · formTemplates · drainSetting |
| `Shrinkage` | seasoningDays · greenShrinkage |
| `Coating` | laysCoat · coatWord · weatheringMPerGameYear · pigment · adhesive |
| `Optics` | lightAbsorption |
| `Tissue` | biologicalSource |
| `Odor` | smells |
| `Radioactivity` · `Muscle` | halfLife · decayMode · decay product · work |

**Chemical identity** is three exclusive capabilities — `Element` (symbol · atomicNumber ·
molarMass), `Compound` (formula · molarMass), `Mixture` (composition, and the recursive elemental
composition `MaterialApi` derives) — and no kind composes two. Iron is an element and bronze a
mixture; both are metals.

⛔ **Consumers depend on capabilities, never on kinds.** Ask `MixinApi.hasFusion(m)` before reading
a melting point; never whether the material is a `Metal`, and never sort materials by folder.
`lint:material-kinds` arm (a) holds the count at zero; the wiki's `<materials>` listing is the one
reader of kinds, by class name.

### Kinds — what the substance is

A material's **class is its kind** (`platform/idea/material/<Kind>.ts`, over the abstract
`Mineral` / `Organic` / `Plant` / `Animal` in `lib/material/`). Organic or mineral is the first
split; `PigmentPaint` and `Glue` sit outside it (ochre and lead white are mineral). Each kind
composes what its members **physically share or most use**; an exception is a one-step child
(`EdibleSalt`, `ToxicAlloy`, `FuelGas`); a capability most members turn out to need moves up a level.
Every kind file's header names what it composes — that is the table. The rules the tree keeps
(owner, 2026-11-07): capabilities describe the substance, not the game · the tree is chemistry, not
culture (uses and lore live in rows) · the leaves are the fragile, cheap part.

**Absent is absent.** A material without a capability has no such field: there is no
`edibility: false`, no `meltingPoint: 0` meaning "does not melt". `lint:material-kinds` refuses an
authored "not applicable" (arm c) and a key the kind does not declare (arm b — the applier drops
undeclared keys silently, which is how `chemistry:` sat dead on 43 rows). A capability a kind
composes but a row has not authored yet is a **data gap**, counted per pack by arm (d) and
ratcheted down by the trades that own the rows.

**The test for an author** — does a new material need code? No, if an existing kind fits (one
row); one class in the author's own pack if it needs a capability its kind lacks (arcana's
`PotionMaterial extends …(Drink)` is the precedent); a kernel change only for a capability that
does not exist yet.

**Substance identity (Bulk build).** `Material` also composes `PerceptibleMixin` (a keyword pool)
and carries an `appearance` prose phrase. The keywords let a bulk holder be addressed by what it
holds (`drink coffee`), and `appearance` is the rendered phrase a holder speaks / shows. `Material`
deliberately does **not** compose `Visible` / `Named` — substance identity stays out of the
perception-target machinery. See [bulk.md](./bulk.md).

Phase-of-matter (`solid/liquid/gas`) is not a capability — it is state, not identity.

### The fixed-vocabulary rule

The material library is a **curated, closed vocabulary an author selects from** — never an open
content catalog. Two rows exist only when they differ in what the machinery reads (macros, density,
hardness, spoilage) — eggs and steak are different rows; ribeye and sirloin are one `meat`. Growing
the library is a **vocabulary decision, not a content decision**.

**Per-dish / per-drink substances are never material rows.** A mixture
(a plated stew, a mixed cocktail) is a *derived blend*: its bulk slot
points at ONE generic `Food` row (`/platform/idea/material/cooked`,
`cocktail/mixed` — physics + routing homes) and the blend's identity +
macros ride the holder's per-instance **`BulkPayload`** (see
[bulk.md](./bulk.md)), computed from the consumed inputs — macros in =
macros out ([crafting.md](./crafting.md) § the mixture strategies).
The early per-cocktail / per-dish rows (`cocktail/martini`,
`food/hearty-stew`, …) were retired for exactly this drift.

### `TangibleMixin` and per-Detail materials

`TangibleMixin` is composed onto every physical Stuff base —
`Thing`, `Location`, `Vessel`, `Agent`. State:

- `_materialPath: string | null` — bulk default Material's
  templatePath.
- `_detailMaterialPaths: Record<string, string>` — per-Detail
  Material overrides.
- `mass: Quantity<'kg'>` — the Stuff's mass. Round-trips through
  `QuantityMarshaller.pathFor('kg')`; setter is strict on
  `Quantity<'kg'>`. The kg marshaller absorbs YAML authoring shapes
  (numeric, tag string, alt-unit literal, JSON `{value,unit}`) at
  the persistence boundary.

`getMaterial()` resolves on each call against live Stuff — HMR-safe —
and is **loud when cold**: a non-empty path whose row nothing loaded throws
`DocumentNotResidentError('material', path)`; `null` means only an empty
path (load-on-use round two).

### ⭐⭐ A material loads when something reaches it, never at boot

`MaterialCatalogue` (`platform/idea/`, reached as `MaterialApi.materials()`
by `singletonSync` — no `onCreate`, no row, no boot line) is the one door
every material read goes through: `need(path)` (sync, throws cold),
`peek(path)` (a read that may honestly miss), `ensure(path)` (async). A
material becomes resident by one of four doors:

1. **its holder** — a thing's own material (`Tangible._materialPath`,
   `_detailMaterialPaths`, a bulk slot's) is `identity:constituent`, so the
   template applier loads it before the setter runs, on mint, go-live and
   restore;
2. **a relation on a row** — a melt product (`Kinetic.kinetics`,
   `drainSetting`, `Fusion.chillsTo`), a mixture's parts
   (`Mixture.composition`), `Solution`'s products, `Steepable.steepsInto`,
   `Composition.makeup`, `Compounding.productMaterial`,
   `Comminuting.productMaterial/residueMaterial`, a body plan's tissues,
   a deposit's lode/zones/gas/fluids, a place's `enclosure`/`roof` — is
   declared `identity:constituent`, and the applier's path walk reads
   nested values, so a list of records loads too. A maturation profile's
   or fraction schedule's materials load in its catalogue's `afterRead`.
   A material-to-material cycle (water ↔ ice) is safe: an in-flight load
   is returned, not awaited;
3. **an act** — a runtime-chosen path (a smelt's product, a draw of
   water, a conjured matter) is `ensure`d by the act that names it; a
   runtime FIELD that carries material paths (a bulk payload's makeup and
   `madeFrom`, a face's coats, a joint's glue, a fuel bed, a strewn floor,
   a kinetic record's origin) is reloaded by its restoring setter, which
   returns `ensureCited(paths)` / `ensureCitedIn(value)` — `undefined`
   when nothing is cold, never a rejection;
4. **the roster** — the few reads that ask for every material (the wiki's
   `<materials>` listing, a pattern's metal match) call async
   `MaterialApi.all()` / `findByTag(tag)`, which load the roster
   themselves (`ensureRoster()` → `roster()` / `byTag`); no caller
   readies it, and it is NOT among `MaterialApi`'s boot `needs`. The roster is every
   root's `/idea/material/` subtree filtered to rows whose class extends
   `Material` — never an allowlist of roots.

⭐ **A material a class used to hard-code is content now** (owner,
2026-11-11): `PipeRun.carries`, the sugar copper's `boilsTo`/`skimsTo`,
`SugarLoaf.drainsTo`, `Churn.churnsTo`, `SoapKettle.slakesTo`, a cellar's
`wallMaterial`, a bottle's or potion's glass — each an authored row field
(each `identity:constituent`; the glass is the row's own `_materialPath`), so an author can make a copper
that boils something else. The universe's default wall is the root biome's
`_enclosureMaterial` ([biome.md](./biome.md)).

⚠ **Left on `peek`, deliberately:** fields that mix material and thing rows
— `Species.production[].yieldRow`, `Comminuting.products`,
`Deposit.stratigraphy`. Declaring them constituent would stand thing rows
up as singletons; their material halves want a split field (a follow-up).

`stuff.getMaterial(detailKey?)` is the direct read for "what is this
made of?" — callers narrow with `MixinApi.isTangible(stuff)` first
(a non-Tangible has no material).

`getMaterial(detailKey)` walks **longest dotted prefix first** down
to the bulk default — so a sub-detail without its own override
inherits whichever ancestor path most recently set one. Mirrors the
parent-then-child path convention `DetailedMixin` uses (`'blade.edge'`
is the edge of the blade).

```ts
axe.setMaterial(oak);              // bulk = oak
axe.setMaterial(iron, 'head');     // override on 'head'
axe.setMaterial(steel, 'head.edge'); // sub-override on the edge

axe.getMaterial();                 // → oak  (bulk)
axe.getMaterial('haft');           // → oak  (no override → bulk)
axe.getMaterial('head');           // → iron (exact)
axe.getMaterial('head.spine');     // → iron (no exact, inherits 'head')
axe.getMaterial('head.edge');      // → steel (exact)
axe.getMaterial('head.edge.tip');  // → steel (no exact, inherits 'head.edge')
```

`getMass()` returns the typed `Quantity<'kg'>`. The `weigh <target>`
verb (Balance instrument) reads it directly.

### v1 roster

Materials are organized under `/stuff/idea/material/<category>/...`.
Categories track everyday "what kind of stuff is this" rather than
a single science's classification — chemistry, biology, geology
all overlay via tags. Path depth varies by branch: shallow where one
level reads naturally; deeper when content earns it.

- `/stuff/idea/material/element/iron, copper, carbon` — pure elements
- `/stuff/idea/material/element/uranium` — `RadioactiveMetal` (the
  capability-mixin demo)
- `/stuff/idea/material/alloy/steel` — Fe + C composition
- `/stuff/idea/material/rock/granite` — igneous; mineral composition unmodeled
  in v1
- `/stuff/idea/material/wood/oak` — once-living plant tissue
- `/stuff/idea/material/tissue/flesh, plant-tissue, fruit-flesh` — biological

These are leaf templates; Material isn't a folder class.

### `PropertiedMixin` and damage resistance

`Material`, `Species`, `BodyPlan`, and `Clade` all compose
`PropertiedMixin` (`SingletonMixin(PropertiedMixin(Idea))`); `Material`
additionally composes `PerceptibleMixin` (Bulk build — keyword pool +
`appearance`, see above), and `Species`
additionally composes `VisibleMixin`, so it speaks the standard
`shortDescription`/`longDescription` surface (the old
`defaultDescription` was subsumed into `longDescription`). Most
fields stay first-class — they're part of the engine's vocabulary,
all instances have them, the schema is stable. But where the keys
are *content-defined* and the engine just stores/queries by name,
the property bag is the right home. The deferred combat /
mechanism-of-injury system's per-material damage resistance is the
motivating example — keys like `resistance.slash` are content-defined
(the engine doesn't enumerate damage types), stored and queried by name:

```ts
// Illustrative — the convenience accessors were removed until combat
// lands; the PropertiedMixin pattern below is what's permanent.
material.setProp(Property.of<number>('resistance.slash'), 0.7);
material.getProp(Property.of<number>('resistance.slash'));

// Equipment / buff / curse can mask the effective value:
shield.maskProp(
  Property.of<number>('resistance.fire'),
  (_p, v) => v + 0.2,
);
```

Free as a side effect: `maskProp` lets equipment / buffs / curses
modify the effective value without touching the base, with
ownership-keyed unmask for clean removal.

The hydration story: PropertiedMixin reads `savedProps` from the
template's `data` block. `getProp` lazily auto-initializes prop
options on first read for hydrated entries, so seed YAML can
populate `savedProps: { "resistance.slash": 0.7 }` directly without
an explicit `initProp` call from the host.

### `MaterialApi` query surface

- `materialOf(stuff, detailKey?)` — the bulk + per-Detail lookup.
- `compositionOf(material)` — recursive weight-fraction expansion to
  leaf elements. A pure element returns `{ Fe: 1 }`; steel returns
  `{ Fe: 0.998, C: 0.002 }`; granite (no composition refs in v1)
  returns `{}`.
- `containsElement(material, symbol)` — "does this material's
  recursive composition contain element `symbol`?" Walks
  `compositionOf` and matches against leaf-element symbols.
- `all()` / `findByTag(tag)` — the roster reads, **async**: each awaits
  `MaterialApi.materials().ensureRoster()` itself, then reads `roster()` /
  `byTag(tag)` (load-on-use W10; `findByElement` had no caller and was
  retired in round two).

Future surfaces land as their consumers do. Thermal heat-flow reads
`thermalConductivity` when the Thermal capability arrives; per-material
**damage resistance** returns when combat lands (likely
`MaterialApi.damageResistance(stuff, type, detailKey?)` over the
`resistance.<type>` prop convention above). Fire/electricity surfaces
(`flammabilityOf`, `canConduct`) wait for those channels — and re-add
the underlying material properties as real Quantities at that point,
not as the fake 0–1 scalars that were removed.

---

## Clade — taxonomic scope

`Clade extends Zone` (the *bare* `Zone`, not `SpatialZone`).
`ZoneApi.isFolderClass(/platform/idea/species/Clade)` returns true (it extends
Zone), so taxonomic templates can hold descendants under the
folder/leaf invariant. `ZoneApi.isSpatialZoneClass` returns false
(Clade does NOT extend SpatialZone), so a species member's
`Stuff.zone` reads `null` rather than pointing at its kingdom —
kingdoms aren't spatial.

A Clade has a `name`, a `rank` (`'kingdom'` … `'species'`), and a
runtime-only `Set<Species>` of members. Members are populated as
Species singletons load.

**Sub-clade hierarchy is encoded in the template path itself**
(`/stuff/idea/species/animalia/chordata/mammalia/.../sapiens`). Each path
segment between the kingdom and the species leaf is a candidate
sub-clade; v1 ships only the four kingdom-rank Clades because that's
all `SpeciesApi.getKingdom` actually consults today, but any of the
intermediate path segments can hold a `/platform/idea/species/Clade` template
the moment a content reason arrives — Phylum, Class, Order, Family,
Genus all plug in without a schema change. `SpeciesApi`'s walk
already iterates every ancestor segment looking for Clade
singletons; populating intermediate Clades is purely additive.

The four v1 kingdoms:

- `/stuff/idea/species/animalia` — Animalia
- `/stuff/idea/species/plantae` — Plantae
- `/stuff/idea/species/fungi` — Fungi (no v1 species)
- `/stuff/idea/species/constructa` — Constructa

Per-Clade defaults (e.g. "all Hominidae default to body plan X") are
deferred until a sub-clade lands and earns the inheritance machinery.

---

## BodyPlan — anatomy

`BodyPlan` is a singleton `Idea` declaring the physical anatomy:

- `bodyParts: BodyPart[]` — typed anatomical part descriptors (the
  model layer), declared once on the shared flyweight. Each part:
  `{ key, parent, tissues, governsVital?, severable?,
  innervatedBy?, suppliedBy? }`, with stable dotted `body.*` keys. The
  slot↔part relations live on the slot side (`SlotSpec.bodyPart` /
  `covers`), not on the part — see [vitals.md](./vitals.md).
  Added by the Vitals build — the anatomy *site* + tissue/strength
  substrate. Instances carry only deltas; the resolver lives on
  `VitalsMixin`. See [vitals.md § Anatomy](./vitals.md).
- `slots: SlotSpec[]` — the unified slot universe. Each spec carries
  the canonical slot name (`hand:left`, `back:1`), the mixin an
  occupant must compose (`'WearableMixin'`, `'WieldableMixin'`,
  `'SlottableMixin'`), optional capacity, optional posture
  decoration, and optional user-facing detail keyword. Replaces the
  older `wornSlots: string[]` + `heldSlots: string[]` split (deleted
  outright in the embodiment MR — no shims). See
  [slot.md](./slot.md) for the SlotSpec shape.
- `locomotionModes` — short names of `LocomotionMode` singletons this
  body plan supports: `walk`, `climb`, `swim`, `fly`, … Drives the
  body-plan gate in `LocomotionApi.canTraverseExit`. v1 ships biped
  = `[walk, climb, swim]`, quadruped = `[walk, swim]`, sessile = `[]`.
  See [locomotion.md](./locomotion.md).
- `defaultLocomotionMode` — short name of the mode an organism of
  this body plan defaults to when `movement.defaultMode` isn't set
  (NPCs without `EnvironmentMixin`, fresh avatars). Layer 2 of the
  three-tier chain `LocomotionApi.defaultModeFor(actor)` consults
  (layer 1 is the explicit setting, layer 3 is universe `'walk'`).
  `null` for sessile body plans.
- `sensoryPorts` — anatomy only: `{ modality, count, position }`.
  Indexed against the perception substrate's
  `PerceptionApi.modalityByOrganKey` — `sensoryPort.modality` is the
  organ key (`'vision'`, `'hearing'`, `'smell'`, `'touch'`,
  `'taste'`).

  ⚠⚠ **A port nobody declares is a verb nobody can use, and it fails
  SILENTLY.** `biped` and `quadruped` shipped with vision, hearing and
  smell only — no touch, no taste. The `feel` and `taste` verbs ship,
  their `requiresTouch` / `requiresTaste` validators ship, the modality
  singletons ship, and every character in the world got *"You can't feel
  anything."* forever, because the last link was a line in this list. It
  surfaced only when somebody typed the verb during a live drive. **A
  new modality is not live until some body plan grants it**; both plans
  now declare touch (`circumferential` — skin is all around) and taste
  (`forward`). ESP modalities (`'verbal-esp'`, `'emotive-esp'`) are
  reserved values the substrate accepts but no v1 species declares —
  ESP capability arrives via augment-conferral instead (see
  [augmentation.md](./augmentation.md)). A future telepath species
  (magical empath, alien biological aether receiver) MAY declare an
  ESP modality on its BodyPlan; the substrate handles it.
- `thermalStrategy: 'endotherm' | 'ectotherm'` — added by the thermal
  build (default `'endotherm'`). Read by `ThermalRegulationMixin` (an
  endotherm defends its setpoint; an ectotherm floats to ambient) and
  by metabolism's Q10 basal scaling. See [thermal.md](./thermal.md).
- `breathableMedia` — the atmosphere tags an organism of this body plan
  exchanges gas in (the [respiration](./respiration.md) medium
  determinant). Default `['air']`; a species **inverts** by authoring
  `['water']` (a fish drowns in air, breathes water). An empty array is
  the breathe-nothing trap, not an opt-out.
- `respires` — the respiration opt-out (boolean noun-form: `isRespiring()`
  / `setRespires()`). Default `true`; `false` (a construct, undead) makes
  the respiration engine never engage a drain — no medium threatens it.

The biped and quadruped body plans also declare a `cranial` slot
(`accepts: SlottableMixin`) — the v1 implant slot the baseline comm
implant occupies via `Avatar.enter`'s bootstrap. Sessile body plans
deliberately omit the cranial slot (plants don't get implants).
See [augmentation.md](./augmentation.md) for the full implant
substrate.

The unified slot universe lets all body-side affordances flow from
one declaration. A quadruped's `back:1` slot
(`accepts: 'SlottableMixin'`) is the slot a saddle's Wearable claim
targets and the slot a rider occupies bareback — same slot, two
consumers. `BodyPlanSlotsMixin` (Pattern B in slot.md) is the
sibling mixin Avatars / NPCs compose to expose the body-plan's
slots through their `Slotted` surface.

**Anatomy only.** Capability — vision range, hearing acuity, scent
acuity — does NOT live here. Humans, dwarves, elves, and orcs all
share the canonical `biped` body plan; their vision profiles differ
on the Species template.

v1 ships three body plans: `biped`, `quadruped`, `sessile`. The
sessile plan is the stand-in for organisms with no agency anatomy
(plants, corals) so `species.getBodyPlan()` never null-checks.

---

## Species — capability

`Species` is a singleton `Idea`. Carries:

- `binomial`, `commonNames`
- `shortDescription`, `longDescription` (from `VisibleMixin`) — the
  species' generic appearance, themed per species; subsumed the former
  `defaultDescription`
- `nameBanks` — the paths of `name-bank` documents that feed the char-gen name suggester
  (`suggestName`/`rerollName`). See [char-gen.md](./char-gen.md).
- `_bodyPlanPath`, `_parentCladePath`, `_defaultMaterialPath` —
  cross-references to BodyPlan, Clade, Material
- **`baseMass` (kg) + `stature` (m)** — the size pair, each `0` =
  *inherit the plan*. See § *Size* below.
- `lifecycleStates` — the species' valid set
  (e.g. `['alive', 'dead', 'undead']`)
- `sexDeterminationSystem`, `reproductiveMode`
- `lifespanMin`, `lifespanMax`
- ⭐⭐ `adultMass` — **what a grown one weighs, in kg**, and the number
  the body plan cannot give. `BodyPlan.baseMass` is a body-SHAPE default:
  every quadruped shares it, so it cannot tell a cow from a collie.
  `Species.massAt(ageDays)` walks the adult mass back down the
  `ageCurve` — linear from 7 % of adult at birth to the whole of it at
  `matureAt`, flat after — and `Creature.getMass()` seeds from that,
  falling back to `baseMass`. `0` means *this species does not say*.

  ⚠⚠ **Until 2026-09-06 only `biped` authored a `baseMass`, so every
  quadruped and every bird in the realm massed ZERO** — and four
  subsystems read that number (encumbrance for carry capacity,
  metabolism for the Kleiber basal drain, thermal for thermal mass,
  ranching for what comes off a carcass). Nothing threw: `butcher` on a
  healthy cow answered *"there was less on it than you hoped"*, a true
  report about a body that weighed nothing. `Species.authoredMass.test.ts`
  now reads it from the reader's end — every shipped animal species must
  arrive at a real mass by some route.
- `circadianBand`
- `diet` (DietApi-deferred)
- `visionProfile` — flat 3-scalar record consumed by `VisionModality`
- **the combat vocabulary** (`@authorable`, the combat-hooks build):
  `naturalAttacks` — a `NaturalAttackSpec[]` list (bite/claw/tail; per-
  attack delivery channel + optional mass/length/reach profile hints,
  rotated deterministically by beat; hint-less profiles derive from
  `Species.getBaseMass()` (own, else the plan's), exactly neutral below
  `combat.natural.largeBodyMassKg` so seeded bodies are byte-preserved,
  "an ogre punches at ogre reach" above it) — and `affordedGambits` —
  existing gambit keys a species affords bodily (a tail affords `sweep`;
  bogus keys inert). See
  [combat-hooks.md](./combat-hooks.md) § the species vocabulary. (The
  legacy single `CombatantMixin.naturalAttackChannel` remains as the
  byte-preserving fallback. A `shock`-innate species' creature class
  composes `EnergizedMixin` **directly** — `innateMixins` conferral is
  activation-gated and not a reliable carrier for the narrowing.)

The roster (`/stuff/idea/species/...`). The char-gen Wave 1 build expanded the
`homo` genus to seven playable humanoid species; the species-expansion
pass added **three more playable** (gnome, half-elf, orc) and **five
NPC-first** casts (troll, ghoul, ogre, kobold, satyr). The species are
now installed from the `@saxonberg/content-species-and-names` content
pack (`content/lib/species/**`), not the kernel seed tree — see
[content-packs.md](./content-packs.md).

| Path | Body plan | Kingdom | Notes |
|---|---|---|---|
| `animalia/.../homo/sapiens` | biped | Animalia | Human reference. |
| `animalia/.../homo/khazadicus` | biped | Animalia | Dwarf — scotopic-shifted vision, 400-yr lifespan. |
| `animalia/.../homo/draconicus` | biped | Animalia | Dragonborn (char-gen). |
| `animalia/.../homo/eldarinus` | biped | Animalia | Elf (char-gen). |
| `animalia/.../homo/infernalis` | biped | Animalia | Tiefling (char-gen). |
| `animalia/.../homo/periannath` | biped | Animalia | Halfling (char-gen). |
| `animalia/.../homo/semiorcus` | biped | Animalia | Half-orc (char-gen). |
| `animalia/.../homo/gnomus` | biped | Animalia | Gnome (char-gen) — small/wiry tinker-folk, low-light, ~350-yr; `gnomish` names. |
| `animalia/.../homo/semieldarinus` | biped | Animalia | Half-elf (char-gen) — mild low-light, ~180-yr; `elvish`+`common` names. |
| `animalia/.../homo/orcus` | biped | Animalia | Orc (char-gen) — broad, keen-nosed, short-lived ~60-yr; `orcish` names. |
| `animalia/.../homo/trollius` | biped | Animalia | Troll (NPC-first) — big/rooted, sun-shy low-light, ~300-yr; the "brute" framing is slander, not biology. Katie's cast. |
| `animalia/.../homo/ghulius` | biped | Animalia | Ghoul (NPC-first) — living people, runs cool, keen nose, `lifecycleStates: alive/dead` (the undead framing is slander). Dr. Vance's cast. |
| `animalia/.../homo/ogrus` | biped | Animalia | Ogre (NPC-first) — enormous, slow, gentle heavy; the "dumb brute" framing is persona, not law. |
| `animalia/.../homo/koboldus` | biped | Animalia | Kobold (NPC-first) — small, scaled, draconic-kin, nocturnal, runs cool. |
| `animalia/.../homo/satyrus` | biped | Animalia | Satyr (NPC-first) — pastoral hedonist/reveler (bar texture), ~200-yr; `sylvan` names. |
| `animalia/.../lithobates/catesbeianus` | quadruped | Animalia | American bullfrog — non-mammal entry. |
| `plantae/.../spathiphyllum/wallisii` | sessile | Plantae | Peace lily; monoecious. |
| `constructa/metallica/tutor-bot/mk-iv` | biped | Constructa | Robot — `lifecycleStates: powered/unpowered/destroyed`, `sexDeterminationSystem: 'none'`. |

### Why the roster is cast by persona

A species is **casting, not biology.** Pick the species whose
pop-culture persona pre-loads the role the way a character actor
pre-loads a part — say *troll* and the gruff, immovable threshold-guardian
is felt before a line is written; the archetype does the negative-space
carving an NPC author would otherwise chisel by hand. The test for a
cast is that the persona rhymes with the role's *function* (Katie is a
troll because a troll IS a bridge-keeper who decides who crosses).
The gate is **recognizability, audience-relative**: a reference the
player does not hold loads nothing, so casts come from the library this
audience owns — the D&D / Tolkien / NetHack canon, plus the creatures
that crossed into it (naga, kitsune, oni, djinn, ghoul); obscure
folklore is spice, never load-bearing; personas from living traditions
are evoked, not caricatured. Beneath casting sits the *why*: species are
defamiliarized vehicles for group allegory, and the engine fits it
because **prejudice is the belief gap** (how a person is seen vs. who
they are) — a bias is modeled **viewer-side (regard), never as a species
stat**. The craft that keeps it commentary rather than reproduction: the
stereotype must be **false as a law** in the fiction (the essentialism
trap — *orcs really are evil* is a biological alibi); abstract the
dynamic, never reskin one real people 1:1; leave *is it true?*
underdetermined; and no *good one who escaped her kind*. That is why
every NPC-first row above says the "brute" framing is slander, not
biology (the ghoul is the exemplar: the stigmatized death-caste, the
contempt is the cover). The aesthetic that rides alongside: lean into
genre convention hard, up to but not including cliché — ride it, then
swerve before it goes predictable. Source: `species-expansion-slate`.

### What a kind of body smells of, and how it tastes

`Species.smells` is what a living body of this kind gives off ([senses.md](./senses.md#one-smell-source-air-nose)):
OU of one kilogram, null on plants. `Species.palate` is optional: `spread` multiplies how far
apart its people are born, and `curves` gives any word its own hedonic curve, which is how a
realm authors a species with different tastes ([metabolism.md](./metabolism.md)).

### Why species differ without ranking

The rule every species difference is judged by — the one the size pair
(*paid for*), the no-lifespan-clock decision and the *worse-than-a-real-
weapon* natural attack already obey: **difference that COSTS is
character; difference that RANKS is essentialism.** Correlation was
never the hazard; hierarchy is. The defence against essentialism is
therefore not sameness but **incomparability** — *can work underwater*
and *can work in the cold* cannot be ranked against each other, so the
design leans harder into difference, not softer, provided the
differences share no scale. The quality criterion: **a good species
difference changes WHERE YOU CAN GO and WHO YOU NEED — never how hard
you hit** (*possible-not-optimal*, applied to species); the best ones
create interdependence — a reason to need someone not like you — and
some make you need your own kind, and a world with both has real
communities *and* real cosmopolitanism, which is what stops either
being the message. Prejudice lives in the beholder: regard is
per-viewer, so bigotry is authored into specific NPCs and localities
and is a fact about the bigot; **a species with a low-reputation stat
would be the game asserting the prejudice is warranted.** Refuse, for
the same reason: stat modifiers (one scale, therefore a ranking);
culture as biology (*naturally greedy / warlike* — the genuinely racist
note); ⭐ a species being BEST at a job (*if a species is optimal for a
role, species becomes class* — the moment a guild wants a particular
species, the thing argued against has been built; a SYNERGY is a
tendency, a CLASS is a prerequisite — water-breathers over-represented
among divers is fine, a job a non-ideal species *cannot* do is not);
and long lifespans as a player mechanic (§ *DECIDED — curves without
lifespans*). Source: `species-slate` § The governing rules · § What to
refuse.

---

### ⭐⭐ Size — `baseMass` + `stature`, and why it is one accessor

`Species` carries **`baseMass` (kg)** and **`stature` (m)**, both
defaulting to `0` = *inherit the body plan*, and both read through
**`Species.getBaseMass()` / `getStature()`** which resolve *own → plan →
0*. `BodyPlan` carries the plan-level defaults (`baseMass`,
`baseStature`); `biped.yaml` authors 70 kg and 1.75 m.

⚠ **Before this, no species overrode the plan**, so every playable
species massed 70 kg — a halfling and a dragonborn carried the same
weight, punched with the same energy, ate the same and cooled at the
same rate. Fit cannot key on a constant, which is what forced the fix.

⭐ **`stature` is deliberately a SCALAR**, not two axes. Lineage's body
budget already owns *build* via the fat/muscle/bone split at one mass, so
a second build axis here would duplicate it. What stature buys is the
linear half of a fit measurement: with mass it yields a ponderal index,
`girth = √(mass / stature)`, and those two numbers are what a garment is
cut to.

⚠⚠ **And it came back.** `Creature.seedMassFromBodyPlan` read
`species.getBodyPlan().getBaseMass()` — reaching *past* the species
accessor to the plan — so every playable species was 70 kg again, and
stayed that way through the whole textiles/combat/metallurgy run. It went
unseen because the husbandry build's `Species.massAt(ageDays)` answers
first and is correct for **animals** (every ranched species authors
`adultMass`) while no playable hominid does, so `massAt` returns `0` for
exactly the rows whose `baseMass` the next line then failed to read. The
only instrument that saw it is `test:gym`, which is outside `pnpm test`.
⭐ **The rule the relapse teaches: read `Species.getBaseMass()`, never
`getBodyPlan().getBaseMass()`** — the species accessor IS the own→plan→0
resolution, and going around it is going around the species.

| species | fiction | stature (m) | baseMass (kg) | girth |
|---|---|---|---|---|
| gnome | *small, wiry* | 1.00 | 30 | **5.48** |
| halfling | *small… fond of a good meal* | 1.05 | 38 | 6.02 |
| dwarf | *stone-sturdy and stout* | 1.35 | 68 | **7.10** |
| elf | *measured, watchful grace* | 1.80 | 64 | 5.96 |
| half-elf | *elven poise and human drive* | 1.78 | 67 | 6.14 |
| human | *adaptable and ambitious* | 1.75 | 70 | 6.32 |
| tiefling | *horned and ember-eyed* | 1.78 | 73 | 6.40 |
| half-orc | *broad and powerful* | 1.88 | 92 | 6.99 |
| orc | *broad and fierce, short-lived* | 1.92 | 100 | 7.22 |
| dragonborn | *the bearing of an elder wyrm* | 2.00 | 125 | **7.91** |

⭐ **The fiction falls out of the arithmetic rather than being
asserted.** The gnome is *wiry* — the lowest girth of anyone. The
halfling eats well, so it carries near-human girth at 60% of the height,
which is exactly what separates it from the gnome. The dwarf is
third-shortest and fifth-heaviest, so *stone-sturdy and stout* is a
number rather than an adjective. The half-elf sits between elf and human
on both axes, at home in neither.

⚠ **And the size is PAID FOR.** A dragonborn carries and eats ~1.8× a
human and cools more slowly — a real cost for a real benefit, which is
the incomparability the lineage slate wants. **Do not "balance" it
away.**

⚠ Every playable species stays **under** `combat.natural.largeBodyMassKg`
(150), so none silently gains ogre reach; the dragonborn is closest at
125. `scripts/__tests__/species-mass.bench.test.ts` asserts that, prints
the whole derived table (mass · stature · girth · carry capacity · basal
factor · thermal τ · fist scale) and snapshots it — because the other
two gym benches drive **synthetic** body plans and are blind to a
shipped-row change by construction.

⭐ **The lineage seam is this one accessor.** Individual variance arrives
later through `Creature.getMass()` alone, so neither `Species` nor
`BodyPlan` re-opens when it lands — and neither does textiles: garment
fit reads `getMass()` for its girth index, so a lineage-varied body
starts fitting its clothes differently the moment lineage ships, with no
code in either subsystem touched. `Wearable.fit.test.ts` asserts it by
changing only `getMass()`.

### ⭐ Habitat — what a species needs of a water (fishing A2/B8, 2026-09)

`Species.habitat` (`Habitat`): `tolerances` over the closed
`WATER_PARAMETERS` vocabulary (each a `Tolerance {min?, max?, margin?}`),
`seasons?`, `role` (`bait · forage · predator · apex`), `abundance`
(individuals per km of reach at a perfect fit), `fightRating`, and
`feedsAt?` (`surface · mid · bottom` — where a hook finds it; absent =
anywhere). `fitIn(state, season?) → {fit, limiting}` is **Liebig's
minimum**: each authored tolerance is a factor in `0..1`, the season `0/1`,
the fit their minimum, and the limiter is NAMED — which is the whole
pedagogy of the water read. Unauthored = factor 1 (soil's rule); a null
habitat = 0. ⚠ `feedsAt` is a habitat fact, deliberately not the pet
`feedingStyle` rung (how an animal takes food from a PERSON); the carp
authors both. `'surface'` was added to `FEEDING_STYLES` for the same carp.
The fishery record that derives a reach's population from this is the
water pack's ([fishing.md](./fishing.md)).

## OrganismMixin — runtime biology

Composing `OrganismMixin` declares: "this Stuff is a member of a
Species, with biological state." The mixin carries:

- `_speciesPath: string | null` — cross-reference to Species
- `bornAt: number` — the birthday, in game-seconds
- `diedAt: number` — `0` while it lives
- `lifecycleState: string` — initial value from the leaf template's
  `data`

### ⭐⭐ The species loads WITH the body (the scan sweep, 2026-10-14)

`_speciesPath` is declared `ref: 'identity:constituent'`, and so are a
species' own `_bodyPlanPath`, `_parentCladePath` and `_defaultMaterialPath`:
the `TemplateApplier` loads each before the field is set — on mint,
go-live and restore — so `getSpecies()` (a synchronous registry read)
never meets a cold row and no species is booted. ⚠ The belt:
`SpeciesApi.isSentient` **fails closed** — a body that cites a species
whose row is not resident answers SENTIENT (consent, blame, no cull). It
answered false until the sweep, which let a collie be butchered on a cold
world. It reads the CITED path through `getSpeciesPath()` — the row the
body names whether or not it is resident — beside `getSpecies()`, which
answers only for a resident one.

### ⭐⭐ Age is a DATE, and what it confers is the whole design

`getAgeDays()` is `(diedAt ?? now) − bornAt`, derived on read and stored
nowhere. There is no counter and no reconcile, and that is not a
micro-optimisation:

**Why lifespan describes the world and never the player** (graduated
from the lineage slate, 2026-09): if aging were real *and* terminal,
lifespan would be the most rankable stat in the game — an elf gets 750
years of play, a human 120, single scale, strictly ordered, and no
countervailing cost can exist, because "more playtime" cannot be made
incomparable with "less." It would be the cleanest violation of the
species doctrine anywhere in the design. So lifespan is for NPC
generations, family trees, and *elves remember the founding*; never a
clock on a player character.

- ⚠ **There is nothing to farm.** An accumulating counter rewards leaving
  a character logged in, or logged out, or simply existing — which is why
  the first cut needed an absence guard and an argument about whose clock
  stops. Arithmetic on a fixed date has no such question.
- ⭐ **A long absence is a non-event.** Nothing to integrate, nothing to
  drop, no stepped walk to bound.
- ⭐ It is the same primitive the herdbook uses for a head born into the
  record (`HeadOverlay.bornAt`), so a lamb and a person are old in one
  way.
- ⚠ **The dead stop.** `setLifecycleState('dead')` stamps `diedAt`, so a
  corpse's age is the age it died at — which is what a forensic read of
  one is asking, and a fact the old counter got by skipping.

**What differs between a player and an NPC is not the clock. It is what
reads it.**

| | the fact | what reads it | what it confers |
|---|---|---|---|
| **player** | `bornAt` | the birthday, milestones, seniority | **nothing, ever** |
| **NPC** | `bornAt` + the species' `ageCurve` | life stage, and in time capability | ability — but ⚠ **never death**, see below |
| **livestock** | `bornAt` (or the herd's founding) + curve | maturity, breeding, cull, the generation interval | the management game |

⚠⚠ **A played human and an innkeeper are the same species row**, so the
authored curve cannot be what tells them apart. `getLifeStage()` returns
`null` for a body somebody plays — always — because **we do not model a
player character's biological arc**. Their age is seniority: a number to
say out loud, never an input to a capability. That is what makes parking
a character worth exactly nothing, and it is why the number can be honest
wall-clock time rather than something defended against being farmed.

⭐ The number stays readable for everybody. It is the *consequence* that
stops, never the telling — a birthday is worth having.

### ⚠⚠ What an NPC author has to know before writing a curve

Two facts, and the second is the one that bites.

**1. The world clock STOPS when the server does.** `WorldClockRegistry.
restore()` re-anchors to `now` and does not add the downtime —
`lastShutdownRealMs` is persisted and passed around but never read to
advance anything. So an outage costs zero game time, for age exactly as
for weather, seasons, soil reversion and herd metabolism. **Nobody ages
through a shutdown.** (This is worth stating because the opposite is the
natural assumption, and designing around a phantom is expensive.)

**2. ⭐⭐ The scale factor is the whole effect.** `DEFAULT_SCALE = 12`, so:

> **one real year of uptime is TWELVE game years.**

That is the arithmetic to design against, and it is unforgiving:

| species lifespan | visibly old after |
|---|---|
| 15 years (a dog) | ~15 months of real operation |
| 40 years | ~3⅓ real years |
| 120 years (a human) | ~a real decade |

So author a curve **only on a species whose ageing you actually want to
watch**. A long-lived, persistent, named NPC either wants no curve at all
or wants numbers chosen against that table — not against intuition about
how long a person lives.

⚠ **And age is the one clock-driven quantity with no recovery path.** A
field can be re-cleared, a herd re-fed, a crop replanted; every other
thing the clock drives can be restored by playing. **An NPC cannot be
un-aged.** That asymmetry, not the shutdown question, is what earns age
more caution than soil.

### ⭐ DECIDED — curves without lifespans, for PERSONS

`ageCurve` is live: it gives life stages, and a life stage may confer
ability. **`lifespanMin` / `lifespanMax` do not bite for anything
sentient.** No person dies of old age.

That is a decision, not an omission, and it should not be quietly
"finished" by a later build wiring `lifespanMax` into mortality for
people. The reason is the table above: under 12× compression, lifespans
that bite mean a named NPC reliably dies inside the game's ordinary
operational life, and *"does the innkeeper die, and who replaces her?"*
is a succession problem nobody has solved. **Ability from age, yes; death
from age, not for persons.**

⚠⚠ **Amended by the pets build (2026-09-16, MR !257): it bites for
NON-SENTIENT animals.** `OrganismMixin.reconcileSenescence()` calls
`ConditionApi.die(host, 'old age')` when three things hold — the body
reads the new `senescent` life stage, its species is **not** sentient,
and it is past `lifespanMax`.

⭐ The reservation above is untouched by that, and the sentience test is
exactly why. What this section reserved is the **succession problem**:
offices, titles, contracts, a chronicle, and who answers for them when
the holder dies. *None of that is a question about a cat.* So a companion
dying of old age is the requirement this section's caution was never
about, and the boundary is drawn by a structural fact rather than by
anyone remembering to check.

⚠ Two further guards, both asserted: a body with an `Interactive` reads
`getLifeStage() === null`, so a **played** body is excluded before the
sentience test is reached; and nothing drives the reconcile on a
schedule — it rides a beat the host already runs (the `feeds` brain), so
an animal nothing is paying attention to does not age out.

⭐ `senescent` also closes a loop this file left open: `AgeCurveSpec.
senescentAt` shipped with the curve, five species rows author it, and
**nothing read it** until now. An old animal reads old — *grey about the
muzzle and slow to rise* — well before the span ends, so the warning
arrives long before the consequence.

⭐ The lifespan numbers are not inert, though — they are *informational*.
`SpeciesLogic` prints `~120 years` on the char-gen species dossier: a
fact the world knows about a kind of creature, told to a player choosing
one. The world knows how long its people live; it just does not kill them
with it.

⚠ When it is time, it belongs with [mortality.md](./mortality.md) and
needs the succession answer first. The mechanism is ready — `ageCurve` +
`lifeStageAt` is general and extending it is authoring, not engineering.

### ⚠ Two more authored-but-unconsumed `Species` fields

Found by the consequence build's `lint:unconsumed-seams` census (MR!254,
2026-09-10), alongside `lifespanMin`/`Max` above: `_parentCladePath` and
`lifecycleStates` (plural) are both authored on every species row and
both have live getters, but nothing outside `Species.ts` itself calls
either. `SpeciesApi.getKingdom` derives an organism's kingdom by walking
the *template path* instead of reading `getParentClade()`, which makes
the field a fully redundant cross-reference today; `lifecycleStates`
declares the valid-state set per species, but nothing validates
`OrganismMixin.setLifecycleState()` against it — any string is accepted
regardless of what a species' `lifecycleStates` says is legal. Neither
is broken; both are declared and inert, same as the lifespan pair. Leave
them — they are a seam for a later species-differentiation build to wire
(validating transitions, or retiring the field if the path-walk stays
the only source of truth), not something to quietly finish now.

`OrganismMixin` is composed:

- Via `Creature` — every Avatar is an Organism. ⚠ It used to sit between
  `NamedMixin` and `GenderedMixin`; `NamedMixin` left the creature base
  on 2026-09-10 (a body is not a somebody — it composes on `CastMixin`
  and on `Avatar` now), and `PerceptibleMixin` joined it. See
  [presentation.md](./presentation.md).
- Via concrete plant/NPC subclasses — a houseplant Thing composes
  Organism on its own class.

Detached tissue is **NOT** an Organism. The apple-on-the-ground case
is `Tangible` (made of fruit-flesh) but not Organism — its parent
tree is the organism, the apple is bulk material.

`getSex()` returns `null` by default; when the host also composes
`SexedMixin`, that mixin's override shadows the default through the
standard mixin chain.

---

## SexedMixin — biological sex

`SexedMixin` is biology, not gender. The two compose orthogonally:

- A frog NPC composes `OrganismMixin + SexedMixin` (no gender
  surface — frogs have no pronouns).
- A human Avatar composes `OrganismMixin + SexedMixin +
  GenderedMixin` (sex from biology, pronouns from social
  presentation).

The valid sex set is read from the host organism's species'
`sexDeterminationSystem`. The lookup table:

| System | Valid set |
|---|---|
| `xy`, `zw` | `male`, `female`, `intersex` |
| `environmental`, `haplodiploid`, `dioecious` | `male`, `female` |
| `hermaphroditic-simultaneous` | `hermaphrodite` |
| `hermaphroditic-sequential` | `male`, `female`, `hermaphrodite` |
| `monoecious` | `male-and-female` |
| `none` | (empty — every set call rejects) |

`setSex(value)` rejects values outside the set. `setSex(null)` always
clears the value.

---

## SpeciesApi — dispatch surface

The single entry point for "what is this Organism, biologically?":

- `getKingdom(o)` — walks the species' templatePath ancestors to
  find the rank-`'kingdom'` Clade.
- `isAnimate(o)` — the load-bearing predicate. Composes kingdom +
  lifecycle state per the slate's table:

(The plain lifecycle-state reads — `isAlive()` / `isDead()` /
`isUndead()` / `isPowered()` / `isLivingBody()` — are the Organism's own answer and live on
`OrganismMixin` as no-arg instance methods; callers narrow with
`MixinApi.isOrganism` and call `organism.isAlive()` directly. The
2026-07 antipattern sweep removed the thin `SpeciesApi.isAlive(o)`-style
forwarders — a lifecycle-state read belongs to the one object, not an Api
hop. `isInKingdom` went the same way; use `getKingdom(o)` and compare.)

| Kingdom | Actor when |
|---|---|
| Animalia | `lifecycleState ∈ {alive, undead}` |
| Constructa | `lifecycleState === 'powered'` |
| Plantae / Fungi | never (no Agent surface in v1) |

Non-Organism Stuff is never animate. `isAnimate` is what the
verb-level `requires-animate` validator reads.

---

## Animacy gating at the command layer

YAML commands carry an optional top-level `validators: []` array.
Verb-level validators fire **before** field validators with
`context.commandGiver` populated, and short-circuit on the first
failure. The shape:

```yaml
verbs: [say, "'"]
controller: SayController
description: "Say something to everyone in your location"
validators:
  - /lib/command/validators/requiresAnimate
args:
  - name: message
    type: string
    required: true
    greedy: true
```

A `CommandValidator` is a sync `(context: CommandContext) => string |
undefined` — distinct from the field-level
`(value, field, context) => …`.

`requiresAnimate` also declares an async `preload` hook that runs
before the sync validator phase. The preload reads the giver's
cited path (`getSpeciesPath()`, not `getSpecies()`, since that uses
`findByTemplatePath` which would return null for the very
singleton we're about to ensure) and calls `StuffApi.singleton(...)`
for the species template AND every ancestor in the kingdom walk.
Ancestor segments without a seeded template (e.g.
`/stuff/idea/species/animalia/chordata/mammalia` is a folder rather than
a Clade leaf) are tolerated — the singleton-not-found error is
swallowed, and `SpeciesApi.getKingdom`'s null-tolerant ancestor
walk takes it from there.

This is what lets Clade singletons NOT be bootstrapped: the verb
dispatch that actually needs them ensures them on-demand, and
subsequent dispatches reuse the cached clones. See
[command-routing.md § Validator preload phase](./command-routing.md#validator-preload-phase).

v1 tags `requiresAnimate` on self-action verbs:

- `say`, `dm`, `go`, `get`, `drop`, `open`, `close`, `inventory`

It does **not** tag passive/meta verbs:

- `look`, `help`, `ping`, `alias`, `var`, `settings`, `player`,
  `focus`

Future verb-level validators (`requires-mobile`, `requires-vocal`,
`requires-hands`) plug into the same machinery.

---


> ⭐ **`Material`'s measured properties and `Species`' combat profile carry a reveal level.** Density, hardness, the conductivities, autoignition and the heats; a species' `vitalProfile`, `facultyProfile`, `innateMixins`, `naturalAttacks` and `affordedGambits`. Identity and natural history stay open — name, tags, the chemistry anyone could look up, diet, lifespan, vision, reproduction. `spoiler: 1` with
> `spoilerName: 0`, so a reader sees which properties exist with the
> values collapsed — **one click, not a lock**, and a reader who set
> `wiki.spoilerAppetite` higher never sees a collapse at all. The rule
> the sweep settled on: *collapse what the WORLD measures, never what
> the PLAYER operates.* Levels are declared on the field and therefore
> apply wherever the value surfaces. See
> [wiki.md](./wiki.md) § the reveal model.

---

## Cross-references

Race-build cross-references (`OrganismMixin._speciesPath`,
`Species._bodyPlanPath`, `Species._parentCladePath`,
`Species._defaultMaterialPath`, `TangibleMixin._materialPath`) all
follow the same locked shape:

- The persistent field is the **path string**, not a serialized
  instance.
- The getter resolves on each call against live Stuff — sync,
  HMR-safe, no instance cache (a material through
  `MaterialApi.materials().need`, loud when cold).
- The setter accepts the resolved Stuff (or `null`) and stores
  `value?.getTemplatePath() ?? null`.
- No marshaller required — the persistent value IS a string.

This rules out the older marshaller-with-cached-instance pattern
(e.g. `Containable.environment`) — that pattern would break HMR
replacement of singletons.

---

## Future seams

- **Per-species breathing.** A future build will gate movement /
  damage-over-time on a species' compatibility with the surrounding
  atmosphere medium. The biome substrate already resolves the
  current atmosphere via `BiomeApi.resolveAtmosphereFor(scope)`
  ([biome.md](./biome.md)); the species side will add either a
  per-Species `breathableAtmospheres: Set<string>` field or a
  per-atmosphere `breathers` map, alongside the consumer that
  enforces it. Out of scope for v1 — the seam is documented here
  so authoring decisions about atmosphere tags can already account
  for it.

## See also

- [templates.md](./templates.md) — folder/leaf invariant, the
  `ZoneTemplate` / `LeafTemplate` split, `ZoneApi.isFolderClass`.
- [spatial.md](./spatial.md) — `Zone` / `SpatialZone` layering;
  `ZoneApi.resolveZoneForPath`; null-environment behavior matrix.
- [biome.md](./biome.md) — atmospheric substrate + outward-walking
  chain resolver. Material's prefix-walk shape is the direct prior
  art that `AtmosphericMixin`'s detail walk mirrors.
- [perception.md](./perception.md) — viewer-aware queries; species
  visionProfile feeds `VisionModality`.
- [mixins.md](./mixins.md) — composition mechanics for the new
  mixins.
- [quantities.md](./quantities.md) — `Quantity<U>` substrate;
  `Material.density` (`Quantity<'kg/m³'>`), `Material.molarMass`
  (`Quantity<'g/mol'>`), `TangibleMixin.mass` (`Quantity<'kg'>`).
- [roadmap.md](../roadmap.md) — v1-deferred work (death/resurrection
  flow, DietApi, tissue authoring at the Detail level, sleep,
  polymorph, genetics, character-creation UI). Each will land with
  its own fresh slate when the build starts.

## ⭐ `Muscle` — the Material subclass that is meat

`MuscleMixin` + `platform/idea/material/Muscle` (the capability-kind
pattern) carry one field, `work` — *how hard this muscle worked in life*,
`0..1` — from which a cut's texture derives, and which nothing authors on
the cut. Rows live at `/stuff/idea/material/tissue/muscles/<name>`, a
sibling folder because `tissue/muscle` is an existing leaf.

⚠ Not a field on `Material`: granite did not work, and every reader would
have had to guard on a tag. ⚠ Not a reuse of `toughness`, which is the
blunt channel's MJ·m⁻³ — reusing it would make a shank harder to *bruise*
than a loin in a fight. See [butchery.md](./butchery.md).


## History

- **2026-11 — the material decomposition.** `Material` had grown to ~80 fields over every build that
  needed a property (a granite row carried heat of combustion, edibility and seasoning days, set to
  "not applicable"), and each capability pulled out before (`Radioactive`, `Odorous`, `Muscle`)
  needed a combination class the moment a second met it (`OdorousConsumableMaterial`). The build
  split the class into the capabilities and the kinds above, re-homed all 389 rows, retired the
  `edibility` flag (composing `Edibility` is the fact), the platform `Material` twin and the four
  combination classes, and fixed the defects the census found (lye and saltpeter "edible", two
  `iron`s, metals holding water, `chemistry:` never applied). MR !555; the census and the tree's
  first draft are material-decomposition-slate
  §§ 7¾–7⅞, and where the build departed from that draft is below.

### The decisions the build made (!555)

- **D12 (W0) `cleanser` stays on the BASE; no Detergency.** The census found sand, chalk,
  vitriol, neutral spirit, naphtha, buttermilk and vinegar all naming a cleanser — cleaning is a
  role many substances play, not one physics. Lens 1: a capability is a property a student can look
  up; "is a cleanser" is not one.
- **D13 (W0) `leaches` → Toxicity** (a toxin release through a face), **`drainSetting` → Kinetic**
  (a timed set). Neither is Acidity's / Fusion's physics.
- **D14 (W1 folded into W2b)** No separate narrowing wave: while the base composed everything, a
  predicate was always true, so W1 would have been the same edit twice. The cut's compile errors
  were the census (286 production sites in 83 files), each narrowed behavior-identically — the
  branch the absent field's old default took.
- **D15 (W2a/W2b one commit)** The deleted platform `Material` twin had ~60 test users that
  needed the FINAL compositions to pick their kind, so the placeholder state was never landable.
- **D16 the tree, re-measured.** The planned tree left ~100 (kind, capability) gaps on the
  current 389 rows. Resolved by rule 4 — a capability most members use, or every member physically
  has, moves up; a real minority is a named child: Stone gains Fusion + Kinetic (every mineral melts
  and changes in a fire — so Ceramic carries Kinetic too, contrary to "kinetics on the clay, not the
  brick"); Food gains Perishability, Toxicity, Acidity, Tissue (Toxicity on Food replaces a
  per-drink Brew kind: metabolism reads a toxin list on whatever is eaten); Drink gains Solution +
  Volatility (D11's recorded move-up); Lye, Gas, Water, Salt, Soap, FatOilWax, Polymer, SolidFuel,
  PigmentPaint gain what most members author. New children: `NonMetal` (carbon, sulfur), `Crystal`
  (quartz), `Soil`, `Plaster`, `Acid` (vitriol), `FuelGas`, `ToxicAlloy` (pewter), `Pulp`,
  `EdibleHerb`, `Hide`, `Horn`, `Blood`, `SolidFood`, `Syrup`, `Fat`, `CoatingOil`, `Resin`.
  `RadioactiveMetal` extends `ToxicMetal`. `Ore` is Stone + Mixture (its composition is what a smelt
  reads; the mineral formula was dropped — exclusive identity). `Soap` is Organic (a salt of a fat)
  and keeps `Shrinkage`: **the "soap borrows wood's seasoning" defect was not one** — `SoapBar`
  cures by drying over the water model, which is what seasoning is. `PigmentPaint`/`Glue` sit
  outside the organic/mineral split (ochre and lead white are mineral). The final table is
  `platform/idea/material/*.ts`, each class's header naming what it composes.
- **D17 minority values stripped (recorded):** metals' `waterAbsorptionCapacity` (the planned
  defect); `nutrients` on non-foods (wood "cellulose", ores "copper", potash "potassium" — no reader
  but metabolism); `tastes` on lye, glycerin, saltpeter, skimmings→(moved to Syrup); blood
  hardness; slip's boiling point; leaf-fat and chalk water absorption; plaster-slurry's smell; the
  ores' formulas. Latex's water share moved from `nutrientAmounts` to `composition` (the mould
  drain reads either) and Polymer composes Mixture. Zero-amount toxin entries are sentinels (every
  reader skips them).
- **D18 `chemistry:` was dead on every row** — not a persistent field, so the applier dropped it;
  the 4 rows with real values were flattened to `formula`/`molarMass`. A defect fixed in passing:
  linen, alum, copperas now report their chemistry.
- **D19 one iron:** the dyeing pack's `iron` is copperas (FeSO₄) — renamed `copperas`, kind
  `Salt`; the dyer's word stays "iron" and `MordantController` maps it.
- **D20 identity** — Element and Compound each carry their own `molarMass` + `getChemistry()`
  (no shared helper; exclusive by construction). Readers ask `hasElement || hasCompound`.
- **D21 sentinel refusal in setters is NOT built.** Gate arm (c) refuses an authored sentinel at
  lint time; a runtime throw would also refuse a test or a child override clearing a parent's
  value. One place, the gate.
- **D22** `description:` on vitriol and quicklime was an orphan key (materials read by
  `appearance`) — kept as a YAML comment.
- **D23 gate (d)** measured 6,057 composed-but-unauthored fields at the cut; ceiling pinned there.

- **2026-11-11 — load-on-use round two (MR !575).** The material roster
  left the boot: `MaterialCatalogue` lost its `onCreate`, its row and its
  boot line and became the `singletonSync` door `MaterialApi.materials()`
  (`ensure`/`need`/`peek`/`ensureCited`/`ensureRoster`). Every material
  loads by its holder, a constituent relation, the act that names it, or
  the roster (§ A material loads when something reaches it); class
  constants became authored row fields; `findByElement` was retired. In
  its review (W10) `all()` / `findByTag()` became async and load the
  roster themselves; the roster is deliberately not a boot `needs`.
