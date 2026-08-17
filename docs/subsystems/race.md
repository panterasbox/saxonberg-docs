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

  **The `death ≠ destruction` rule is amended — one rule, two
  mechanisms.** It still holds that a body persists as a corpse and is
  never routed through `StuffApi.destruct` as part of dying. What varies
  is whether an **identity has to leave**:

  - **NPCs, creatures, beasts** — unchanged: the same Stuff becomes the
    corpse. Nothing is waiting to re-enter it.
  - **PCs** — the body *divides*. A corpse takes the material half (the
    wound map, the cause stamp, the loadout) and persists on its own
    decay clock; the identity walks away as a shade, and the drained
    shell is destructed so the shade can hold the identity path.

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

`Material` is a singleton-by-templatePath `Idea` carrying physical
properties — `density` (`Quantity<'kg/m³'>`), `thermalConductivity`
(`Quantity<'W/(m·K)'>`), and `specificHeat` (`Quantity<'J/(kg·K)'>`,
added by the thermal build — the C = m·c half of the Thermal
capability's τ = R·C, seeded with real values across the material tree
plus new glass / ceramic / wool / down leaves; see
[thermal.md](./thermal.md)), the real measured quantities — plus three
orthogonal layers of classification. (The old fake 0–1 `hardness` /
`flammability` / `opacity` / electrical / magnetic fields and the
`resistance.<type>` damage seam were removed: normalized scales with
zero consumers; they return as real Quantities when a consumer — fire,
electricity, combat — actually lands.) Later builds add more measured
properties as their consumers land: `hardness`/`toughness` (materials-
response), `electricalConductivity` (electricity), and
**`waterAbsorptionCapacity`** (`Quantity<'%'>`; weather Wave 2 — the real
ASTM-D570 figure for the water a material holds at saturation, as a percent
of dry mass: wool ≈ 33 %, wood ≈ 28 %, flesh ≈ 25 %, metals / glass ≈ 0.
The `WetMixin` gauge reads it to derive the dry rate from evaporation
physics; see [weather.md](./weather.md)). The **fire build** adds six more:
`autoignitionTemperature` (`Quantity<'K'>`) + `heatOfCombustion`
(`Quantity<'MJ/kg'>`) drive the combustion driver, and `meltingPoint` /
`latentHeatOfFusion` (`J/kg`) + `boilingPoint` / `latentHeatOfVaporization`
(`J/kg`) drive the phase-change layer — real figures (wood 570 K / 16 MJ/kg,
iron mp 1811 K, water mp 273 / bp 373), `0`-until-authored (an unauthored
material never ignites, never melts); see [fire.md](./fire.md).

- **Tags** (`tags: string[]`) — free-form classification strings
  (`'metal'`, `'alloy'`, `'igneous'`, `'organic'`, `'fantasy'`).
  Vocabulary is not centrally registered; content devs introduce new
  tags as they need them. Used for educational filtering and any
  orthogonal grouping.
- **Composition** (`composition: { materialPath, fraction }[]`) — for
  mixtures and alloys, weight-fraction references to constituent
  Materials (steel → 0.998 iron + 0.002 carbon). `MaterialApi`
  recursively expands these to leaf-element symbol fractions, so
  "does this contain iron?" works regardless of how nested the
  alloy/mixture chain is.
- **Chemistry** — atomic-/molecular-level science data, decomposed
  into four flat scalar persistent fields per the scalar-default
  rule:
  - `symbol: string | null` (element-only)
  - `atomicNumber: number | null` (element-only)
  - `formula: string | null` (compound-only)
  - `molarMass: Quantity<'g/mol'> | null` (element-or-compound;
    round-trips through `QuantityMarshaller.pathFor('g/mol')`)

  Elements populate `symbol/atomicNumber/molarMass`; compounds
  populate `formula/molarMass`; mixtures leave all four unset and
  rely on the composition layer.

  `getChemistry` / `setChemistry` is a convenience aggregate over
  the four fields — `getChemistry` returns null when none are
  populated, otherwise returns an `ElementChemistry` view; the
  setter REPLACES the whole chemistry block (use the per-field
  setters for partial updates). See
  [quantities.md](./quantities.md) for the `Quantity<'g/mol'>`
  substrate.

Plus `biologicalSource: { speciesPath, tissueType } | null` for
organic materials linking back to a Species template (bidirectional
with `Species._defaultMaterialPath`).

**Substance identity (Bulk build).** `Material` also composes
`PerceptibleMixin` (a keyword pool) and carries an `appearance` prose
phrase. The keywords let a bulk holder be addressed by what it holds
(`drink coffee`), and `appearance` is the rendered phrase a holder
speaks / shows. `Material` deliberately does **not** compose
`Visible` / `Named` — substance identity stays out of the
perception-target machinery, so material keywords never leak into room
scope. See [bulk.md](./bulk.md).

### Capability mixins

Most classification is metadata — flat strings or refs. Some
classifications carry behavior. **`RadioactiveMixin`**
(`/obj/material/Radioactive.ts`) is the v1 demonstration of the
capability-mixin pattern:

```ts
export class RadioactiveMaterial extends RadioactiveMixin(Material) {}
```

The mixin adds `getHalfLife()`, `getDecayMode()` (alpha / beta-minus
/ beta-plus / gamma / spontaneous-fission / electron-capture), and
`getDecayProduct()` — a lazy cross-reference to the daughter
Material. Templates that need it use `class:
/obj/material/RadioactiveMaterial`; everything else stays on
`class: /obj/material/Material`. `MixinApi.isRadioactive(m)` narrows
the typed surface.

Why a mixin and not a tag: radioactivity carries non-trivial
behavioral surface (half-life arithmetic, decay chains, decay-mode
taxonomy) and is sparse — most materials aren't radioactive, and the
fields are meaningless on those that aren't. Future capability
mixins (`SuperconductorMixin`, `PiezoelectricMixin`, …) layer the
same way: one subclass per composed combination.

Phase-of-matter (`solid/liquid/gas`) is not a capability mixin —
it's state, not identity. Iron is solid at room temp, liquid at
1538 °C; the material's identity is invariant to state.

### The fixed-vocabulary rule (and `ConsumableMaterial`)

The material library is a **curated, closed vocabulary an author
selects from** — never an open content catalog. The granularity test
is *substrate-read properties*: two rows exist only when they differ
in what the machinery reads (macros, density, hardness, spoilage) —
eggs and steak are different kinds; ribeye and sirloin are one `meat`
(that spread is `Grade` + prose on the instance). Growing the library
is a **vocabulary decision, not a content decision**.

**`ConsumableMaterial`** (`/obj/material/ConsumableMaterial`, the
capability-subclass pattern with no extra fields — taxonomy) marks the
ingestible half of the vocabulary: the curated food/drink *kinds*
(meat, root-vegetable, trail-ration, gin…) whose worth is what a body
does with them, as distinct from the structural matter the
response/repair/salvage machinery organizes around.

**Per-dish / per-drink substances are never material rows.** A mixture
(a plated stew, a mixed cocktail) is a *derived blend*: its bulk slot
points at ONE generic ConsumableMaterial (`food/cooked`,
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

`getMaterial()` resolves on each call via
`StuffApi.findByTemplatePath` — HMR-safe.

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

Materials are organized under `/obj/material/<category>/...`.
Categories track everyday "what kind of stuff is this" rather than
a single science's classification — chemistry, biology, geology
all overlay via tags. Path depth varies by branch: shallow where one
level reads naturally; deeper when content earns it.

- `/obj/material/element/iron, copper, carbon` — pure elements
- `/obj/material/element/uranium` — `RadioactiveMaterial` (the
  capability-mixin demo)
- `/obj/material/alloy/steel` — Fe + C composition
- `/obj/material/rock/granite` — igneous; mineral composition unmodeled
  in v1
- `/obj/material/wood/oak` — once-living plant tissue
- `/obj/material/tissue/flesh, plant-tissue, fruit-flesh` — biological

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
- `findByTag(tag)` — every registered Material carrying the tag.
- `findByElement(symbol)` — every registered Material whose
  composition contains the element. Combines the above.

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
`ZoneApi.isFolderClass(/obj/species/Clade)` returns true (it extends
Zone), so taxonomic templates can hold descendants under the
folder/leaf invariant. `ZoneApi.isSpatialZoneClass` returns false
(Clade does NOT extend SpatialZone), so a species member's
`Stuff.zone` reads `null` rather than pointing at its kingdom —
kingdoms aren't spatial.

A Clade has a `name`, a `rank` (`'kingdom'` … `'species'`), and a
runtime-only `Set<Species>` of members. Members are populated as
Species singletons load.

**Sub-clade hierarchy is encoded in the template path itself**
(`/obj/species/animalia/chordata/mammalia/.../sapiens`). Each path
segment between the kingdom and the species leaf is a candidate
sub-clade; v1 ships only the four kingdom-rank Clades because that's
all `SpeciesApi.getKingdom` actually consults today, but any of the
intermediate path segments can hold a `/obj/species/Clade` template
the moment a content reason arrives — Phylum, Class, Order, Family,
Genus all plug in without a schema change. `SpeciesApi`'s walk
already iterates every ancestor segment looking for Clade
singletons; populating intermediate Clades is purely additive.

The four v1 kingdoms:

- `/obj/species/animalia` — Animalia
- `/obj/species/plantae` — Plantae
- `/obj/species/fungi` — Fungi (no v1 species)
- `/obj/species/constructa` — Constructa

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
  `'taste'`). ESP modalities (`'verbal-esp'`, `'emotive-esp'`) are
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
- `nameBankKeys` — references to `NameBank` Documents (in the
  `name_banks` collection) that feed the char-gen name suggester
  (`suggestName`/`rerollName`). See [char-gen.md](./char-gen.md).
- `_bodyPlanPath`, `_parentCladePath`, `_defaultMaterialPath` —
  cross-references to BodyPlan, Clade, Material
- `lifecycleStates` — the species' valid set
  (e.g. `['alive', 'dead', 'undead']`)
- `sexDeterminationSystem`, `reproductiveMode`
- `lifespanMin`, `lifespanMax`
- `circadianBand`
- `diet` (DietApi-deferred)
- `visionProfile` — flat 3-scalar record consumed by `VisionModality`
- **the combat vocabulary** (`@authorable`, the combat-hooks build):
  `naturalAttacks` — a `NaturalAttackSpec[]` list (bite/claw/tail; per-
  attack delivery channel + optional mass/length/reach profile hints,
  rotated deterministically by beat; hint-less profiles derive from
  `BodyPlan.baseMass`, exactly neutral below
  `combat.natural.largeBodyMassKg` so seeded bodies are byte-preserved,
  "an ogre punches at ogre reach" above it) — and `affordedGambits` —
  existing gambit keys a species affords bodily (a tail affords `sweep`;
  bogus keys inert). See
  [combat-hooks.md](./combat-hooks.md) § the species vocabulary. (The
  legacy single `CombatantMixin.naturalAttackChannel` remains as the
  byte-preserving fallback. A `shock`-innate species' creature class
  composes `EnergizedMixin` **directly** — `innateMixins` conferral is
  activation-gated and not a reliable carrier for the narrowing.)

The roster (`/obj/species/...`). The char-gen Wave 1 build expanded the
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

---

## OrganismMixin — runtime biology

Composing `OrganismMixin` declares: "this Stuff is a member of a
Species, with biological state." The mixin carries:

- `_speciesPath: string | null` — cross-reference to Species
- `age: number`
- `lifecycleState: string` — initial value from the leaf template's
  `data`

`OrganismMixin` is composed:

- Via `Character` — every Avatar is an Organism. Inserted between
  `NamedMixin` and `GenderedMixin` in the composition chain.
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

| Kingdom | Animate when |
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
raw `_speciesPath` field (not `getSpecies()`, since that uses
`findByTemplatePath` which would return null for the very
singleton we're about to ensure) and calls `StuffApi.singleton(...)`
for the species template AND every ancestor in the kingdom walk.
Ancestor segments without a seeded template (e.g.
`/obj/species/animalia/chordata/mammalia` is a folder rather than
a Clade leaf) are tolerated — the singleton-not-found error is
swallowed, and `SpeciesApi.getKingdom`'s null-tolerant ancestor
walk takes it from there.

This is what lets Clade singletons NOT be bootstrapped: the verb
dispatch that actually needs them ensures them on-demand, and
subsequent dispatches reuse the cached clones. See
[command-routing.md § Validator preload phase](./command-routing.md#validator-preload-phase).

v1 tags `requiresAnimate` on self-action verbs:

- `say`, `tell`, `go`, `get`, `drop`, `open`, `close`, `inventory`

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
- The getter resolves on each call via `StuffApi.findByTemplatePath`
  — sync, HMR-safe, no instance cache.
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
