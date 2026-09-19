# Materials response

The materials-response substrate is the **`response = f(mechanism,
material, construction)`** function — the physics that decides what a force
actually does when it meets a made thing. It exists because "how do you
model armor?" hits the wall that **chainmail and plate are the same steel**:
threat mitigation cannot live on the `Material` alone. This build ships the
standalone physics core and wires its first consumer, the harm driver
([harm.md](./harm.md)) — so a knife, a club, and a knife *through a boot*
produce honestly different wounds, **demonstrable with no combat loop**.

Seeds from `docs/slates/deferred-rpg/materials-response-slate.md` (Settled
1–11). Reuses [vitals.md](./vitals.md) (`BodyPart`, tissue Materials,
`SlotSpec.covers`), [crafting.md](./crafting.md) (`GradedMixin`, the
`DurableMixin` wear gauge split out of `ToolMixin`),
[slot.md](./slot.md)/[embodiment.md](./embodiment.md) (`WearableMixin`,
`WieldableMixin`,
coverage), [quantities.md](./quantities.md), and
[app-settings.md](./app-settings.md).

## The three axes

A blow is decomposed into three orthogonal axes. Picture the response as
a curve with the channel across the bottom: the **channel** selects the
*point* on it, the **construction** picks its *shape*, and the
**material** (with quality) scales it *up or down*:

- **Channel** (`lib/material/Channel.ts`) — the closed, additively-growable
  mechanism vocabulary — **`edge` / `point` / `blunt`** (mechanical),
  plus **`shock`** (resolves by circuit) and **`heat`** (resolves by
  insulation). The single shared
  interface a weapon's *delivery*, an armor's *resistance*, and a tissue's
  *failure* all transact over. Not a "damage type" — the *shape of the
  force*. (`crush`/`heat`/`cold`/`corrosion` join as columns when their
  consumers land.) The `Grade`/`ToolCapability` value-object precedent:
  vocabulary tuple + type + a thin `Channels` static holder.
- **Material mechanical properties** — `Material` gained grounded
  `hardness` (`Quantity<'MPa'>`, indentation hardness) and `toughness`
  (`Quantity<'MJ/m³'>`, energy absorbed before fracture), siblings of the
  existing `density`/`specificHeat` (strict-unit accessors, marshaller
  binding, `fieldMeta`'s persistent entries). How far a material scales the
  response. Materials stay content (packs supply the roster + values); this is
  their mechanism-response face. **Not** the old 0–1 `resistance.<type>`
  scalars (that convention is dead).
- **Construction** (`lib/material/Construction.ts`) — a per-domain
  value-object (the `Grade`/`WeatherType` data-table precedent, **not** a
  mixin) carrying a per-channel profile. Two v1 vocabularies share one
  shape: **armor forms** (`plate`/`mail`/`padded`/`hide`) with a *resist*
  profile, and **weapon-delivery forms** (`bladed`/`pointed`/`hafted`, plus
  the guardless `flail` (blunt) and `whip` (a cutting `edge` lash) added by
  the weapon-playstyle build, and `blunted` — a sword-shaped implement
  presenting no edge, the waster/practice-sword form: blunt-primary
  delivery like `hafted` but a blade's `good` self-guard in
  `WeaponProfile`, so a trainer with a real sword's mass/length derives a
  real sword's playstyle while its blows resolve as contusions.
  Harmlessness lives in the delivery *shape* — the live fold never reads a
  weapon's material, so a "wooden sword" authored `bladed` still cuts. The
  guardless/reach distinctions are a *playstyle* concern keyed on the form
  in `WeaponProfile`, see [combat.md](./combat.md), not a delivery-shape
  one) with a *deliver* profile. Persisted by hosts as the form word,
  reconstructed via `Construction.of`.

### The taxonomy grid (shape, in code)

`Construction` ships the **qualitative** per-channel grid — the *shape* of
the curve, transcribed verbatim from the slate:

| covering form | edge | point | blunt |
|---|---|---|---|
| plate | deflect | resist | transmit |
| mail | resist | fail | transmit |
| quilted | poor | poor | absorb |
| padded | poor | poor | absorb |
| hide | moderate | poor | moderate |
| *any fabric* | poor | poor | poor |

| delivery form | primary | secondary |
|---|---|---|
| bladed | edge | point |
| pointed | point | edge |
| hafted | blunt | — |
| flail | blunt | — |
| whip | edge | blunt |
| blunted | blunt | — |

Covering forms also carry a canonical outside-in **`LAYER_DEPTH`**
(padded 0 · quilted 1 · hide 2 · mail 3 · plate 4) so a covering stack
orders itself with no authored number.

### ⭐⭐ The covering domain has TWO sources

A padded gambeson *is* quilted cloth, so a shirt is not armor but it
**is** a covering — which is why the domain word is `covering`, not
`armor` (`ARMOR_FORMS` → `COVERING_FORMS`, `ARMOR_PROFILES` →
`COVERING_PROFILES`, `isArmor()` → `isCovering()`, and
`responseFor()`'s domain guard now says so).

The two halves of that vocabulary answer to different people:

| | where | may a pack add one? |
|---|---|---|
| **resist-bearing** — `plate` `mail` `padded` `quilted` `hide` | a closed kernel `as const` | **no** — a resist profile is combat mitigation |
| **non-resisting textile** — `woven` `knit` `felted`, and lace or netting later | template rows at `/stuff/idea/fabric/`, class `/platform/idea/material/Fabric` | **yes** — *a pack must never need a kernel list edit* |

⭐ **Content never authors a resist profile.** One kernel constant
answers for every textile form at once — `TEXTILE_RESIST_PROFILE`,
`poor` on all three mechanical channels. *That is the split made
literal:* content chooses drape, loft and weave density; **the kernel
decides that cloth resists poorly.** It is *"a linen shirt is armor that
does not work"* as one line of kernel data, and a pack adding `lace`
changes nothing whatever about combat. `poor` is deliberately not an
inert token, so `doesNothing()` stays false for every fabric — a shirt
attenuates a little, which is the honest answer.

⚠ **Do NOT add a `kevlar` form.** `MaterialLogic` scales the resist
*magnitude* by the material while `responseFor()` supplies only the
*shape*, so a `woven` fabric of aramid gets the `poor` shape scaled by
aramid's toughness and comes out genuinely protective. That is
`response = f(mechanism, material, construction)` doing its job.

**Four constraints hold simultaneously**, and each is load-bearing:

1. `getLayerDepth()` stays **total** — it is called unconditionally in
   three hot paths (`MaterialLogic.heatAttenuationFraction`, the
   struck-site covering stack, the trauma covering walk), so a fabric
   row's `layerBand` is **required and range-validated at
   registration**, loudly, rather than throwing when somebody swings.
2. `Construction.ts` stays **import-pure** (only `./Channel`) — two
   build-time lints instantiate it outside the runtime. The bridge is a
   module-private `Map` filled by `Construction.registerFabric`.
3. `check-does-nothing` covers every present and future fabric in **one
   assertion** over `TEXTILE_RESIST_PROFILE`, because they all share it
   — the whole reason that profile is a kernel constant.
4. `FabricCatalogue` (`platform/idea/FabricCatalogue.ts`) warms the
   roster at `postRegister`, **never an operator `boot()`**, and rides
   the platform pack's `boot:` manifest. ⚠ This is the
   *reference-Ideas-inert-at-boot* rule, and here the failure would not
   even be silent: a garment row authoring `constructionForm: woven`
   **throws at hydration** against a cold roster. The manifest is the
   install-order union DFS-sorted and the platform pack installs first,
   so the warm precedes any locality's `props:` clone.

## The response function (the Api home)

The function lands on the **existing `MaterialApi`** (`api/material.ts`) +
`MaterialLogic` (`platform/idea/api/MaterialLogic.ts`, gated
`FromModule('/api/material#MaterialApi')`), filling the surface
`MaterialLogic` had reserved for `damageResistance`. No new Api. The same
function is read from both sides of a blow (Settled-6 "one function, two
consumers"):

- **`attenuate(channel, energy, material, construction, grade, condition)`**
  — one armor layer. `token = construction.responseFor(channel)` (SHAPE) →
  an AppSettings base fraction (MAGNITUDE) × `materialScale` × grade×condition
  → clamped attenuation; residual energy passes inward.
- **`resolveTrauma(channel, energy, tissueMaterial, partHasBone)`** — the
  residual meeting tissue: `edge → laceration`, `point → puncture`, `blunt →
  fracture` (a boned part at/above the threshold) else `contusion`; `null`
  when the residual is below the no-wound threshold (deflected). Both
  severity AND type derive here.
- **`previewBand` / `severityToBand`** — the legibility chokepoint (below).
- **`deliverableChannels(construction)` / `primaryChannel(construction)`** —
  the weapon-delivery derivation (a dagger delivers edge, a mace blunt).

`materialScale` is a small documented pure function: `edge` is
hardness-driven, `blunt` toughness-driven, `point` mixes both; each
normalized against a reference (steel) magnitude and lifted by a structural
**floor** so a construction's token sets the protection ceiling and material
modulates within `[floor, 1]` (a hide boot still turns a shallow cut, a
soft absorber still absorbs). `grade × condition` scales the response and
**never its shape** (Settled-4), tuned so a masterwork at ~50% condition
≈ a common piece pristine. The scalar is exposed as **`MaterialApi.gradeConditionScale`**
so combat's instrument-delivery fold reads the SAME formula (the
crafting-branches coupling — see [combat.md](./combat.md)).

⭐⭐ **Both ends of a blow read `materialScale` now** (metallurgy,
2026-09-15). It always priced what a blow lands ON; combat's delivery
deliberately folded quality only, leaving the material's scale analyze-only
— which meant `analyze response` told a player a bronze blade was worse
than a steel one and the exchange disagreed. That asymmetry was
defensible while every shipped weapon was steel and material was
decoration, and it is not defensible in a game whose metal chain exists
so that **which metal you made** is a decision. `MaterialApi.materialScale`
is exposed and `instrumentDeliveryScale` multiplies by it, so the
preview and the fight agree by construction rather than by anybody
keeping two formulas in step. ⚠ Steel is the reference, so a steel
weapon lends exactly 1.0 and nothing shipped moved (the gym was run
either side of the fold, unchanged); iron lends ~0.83 on an edge, bronze
~0.77, copper ~0.70. ⚠⚠ A `null` material lends **zero**, which is right
on the covering side (no covering protects nothing) and wrong on the
delivery side (the weapon is present; only our knowledge of it is
missing) — so `instrumentDeliveryScale` GUARDS the null and stays
neutral. Conflating the two cost a regression: two fixture blades with
no material stopped drawing blood the moment the fold landed.

**Wear-on-use joins the fold** (the repair economy): inside
`ConditionLogic`'s outside-in walk, each covering layer that attenuates a
**mechanical** blow wears `crafting.wear.armorPerBlow` — armor degrades by
taking hits, never by the clock; `repair` restores it (see
[crafting.md](./crafting.md) § the lifecycle).

### The shape-vs-magnitude split

The clean seam that satisfies both "construction owns shape" and "constants
→ AppSettings": `Construction` ships the qualitative tokens (code);
`AppSettings` (`response.*`) ships the coefficient each token resolves to,
plus the reference magnitudes, the grade/condition bounds, the tissue-tail
thresholds, and the outcome-band cutoffs. **No magic balance number ships
as a code invariant** — every magnitude is a dial read with a seeded-literal
fallback (safe pre-warm/test).

### The boot roster warm (`MaterialApi.boot`)

Every material read the engine performs (`Tangible.getMaterial`, a bulk
slot's material, `Combustible`'s autoignition, composition expansion)
is a **sync** `findByTemplatePath` against the live index — and nothing
else ever stood Materials up in a running server (tests hand-construct
theirs), so every live read was null and nothing could ignite or melt.
`MaterialApi.boot()` (called from `AppBootstrap` after the seeders)
stands the whole `/stuff/idea/material/**` roster up as live singletons — the
`SpeciesApi.preloadAnatomy` tolerant-ensure made total over a small,
hot, reference-data roster. Folder rows (`FolderZone`s) are the zone
substrate's and are skipped. `Material.canEvict` vetoes the residency
sweep: a culled material would be a null read until the next process.

## Armor mitigation — emergent, layered, outside-in

There is **no `ArmorMixin`** (Settled-4). A piece of armor is an *emergent
composition*: a `Wearable` `Thing` carrying a `Material` (via `Tangible`), a
`Construction` (its resist form, via `ConstructedMixin`), a `Grade`,
and a wear-on-use `condition` (`DurableMixin` — the durable-good half split
out of `ToolMixin`, so armor wears out without being a "tool").

⭐⭐ **And there is no `Armor` CLASS either, since the textiles build.**
`Armor` was a sibling of `Garment` composing exactly those four extra
mixins and adding no behavior; once `Garment` composes them the two
stacks are byte-identical, and the surviving class was asserting
something the model does not believe. **Armor-ness is material +
construction form.** A steel breastplate is a
`platform/thing/equipment/Garment` whose material is steel and whose
form is `plate`; a linen shirt is one whose material is linen and whose
form is `woven`. The covering walk asks the material and the form and
**never asks what class they are**, so both resolve through one path.

⚠ The `armor/` **content directory keeps its name** — it is a content
namespace, not a class, and renaming it would churn `lint:census` for
nothing. `platform/thing/equipment/Weapon` remains the wielded sibling;
its "weapon-ness" is likewise purely the composition.

`ConstructedMixin` (`lib/material/Constructed.ts`) is the **form axis** — the
sibling of `TangibleMixin`'s material axis: a durable `constructionForm`
word + a resolve-on-read `getConstruction()` value-object. Registered in the
`Mixins` roster (`MixinApi.isConstructed`).

The resolution spine, in `ConditionApi.inflict` for a `Channel` insult:

```
inflict(target, { mechanism: Channel, site, energy })
  ├─ resolveCoveringStack(target, site):
  │     getSlotsCovering(site) → occupants that are Constructed armor + Wearable
  │     sorted outside-in by construction LAYER_DEPTH (plate outer … padded inner)
  ├─ fold MaterialApi.attenuate through the layers (residual passes inward)
  └─ residual meets the site's tissue:
        MaterialApi.resolveTrauma(channel, residual, tissueMaterial, partHasBone)
        → { type, severity } | null (deflected)   → build Trauma → afflict
```

A covered site turns what an uncovered gap takes — coverage is **degree, not
presence**. The binary `ConditionApi.isSiteCovered` is retired.

**Wielded shields** join the same stack (the weapon-playstyle build): a
`Wieldable` carrying an *armor* `Construction` (a shield — armor you hold, not
wear) is folded in as a **directional** front cover, gated by an optional
`InflictSpec.shieldFacing` hint — `true`/absent (a faced or non-combat blow) →
the shield attenuates; `false` (combat routing a flanking blow under
focus-fire) → it is bypassed. Keyed off the `Wieldable`-carries-armor
composition (not a slot-name), so it's a general front cover over any struck
part, not tied to a body-plan `covers` slot. See [combat.md](./combat.md) §
weapon playstyle.

## Weapon delivery (delivery-forms only)

An implement *derives* which channel(s) it presents from its
weapon-delivery-form (`deliverableChannels`). A `Weapon` **is `Wieldable`**
— it claims a body-plan hand slot via `slotClaims`, so you can hold it — and
composes `DurableMixin` for its wear-on-use `condition` gauge (a *durable
good*, not a crafting tool — durability was split out of `ToolMixin` so
weapons/armor wear out without carrying an inert capability list). What v1
defers is the combat **loadout + playstyle**
(reach / poise / guard→parry / afforded gambits), orthogonal to holding the
thing. The channel driving a given `inflict` is **explicit** at the call
site (Settled: no auto-pick).

⭐⭐ **What a weapon is MADE OF now reaches the blow** (metallurgy,
2026-09). `CombatLogic.instrumentDeliveryScale` multiplies its
`grade × condition × keenness` scalar by
`MaterialApi.materialScale(weapon.getMaterial(), channel)` — the same
function the covering stack reads from the other side, so the preview and
the exchange agree by construction rather than by two formulas being kept
in step. Steel is the reference (1.0 everywhere), iron ~0.83 on an edge,
bronze ~0.77, copper ~0.70.

⚠⚠ **A weapon with no material at all is NEUTRAL, not zero**, and the
asymmetry with the attenuation side is deliberate: there a null material
is *no covering* and zero is right; here the weapon is present and only
our knowledge of it is missing. Scaling by zero let one unauthored
content field silently delete combat — two fixture blades stopped drawing
blood the moment the fold landed.

## The legibility surface (mandatory — Settled 11)

Ships with the model or it doesn't ship. All three are **server-computed
derived projections** over the single `MaterialApi` chokepoint; the client
renders MML it is given and holds zero response semantics.

- **Preview** — `analyze response <target>` (a subcommand of the existing
  `analyze` verb + `AnalyzeResponseController`): point at a made thing → the
  per-channel outcome band (a weapon's delivery, an armor's protection). The
  previewed band **matches** the resolved `inflict` outcome for identical
  inputs (both read `MaterialApi.previewBand`/`resolveTrauma` at the same
  reference energy — a test asserts equality).
- **Per-item pips** — a `markupAugmenters` contribution on `ConstructedMixin`
  (the `BrandedMixin` "a product of X" precedent) renders the derived
  per-channel profile as pips (`edge ●●●○ · point ●●○○ · blunt ●○○○`) on the
  long description, for author *and* player.
- **Does-nothing lint** — `scripts/check-does-nothing.ts` (the
  `check-gate-strings` standalone-WARN/`EXIT_ON_FINDINGS` precedent, wired
  into CI + `lint:does-nothing`): flags any construction that mitigates /
  delivers nothing on every channel. The check itself is the pure,
  fixture-tested `Construction.doesNothing` predicate.

Authors author **concepts** (a *steel breastplate, fine*), never numbers;
the tuning constants are operator-only.

## The tearing passthrough

`InsultKind = Channel | 'tearing'`. A `Channel` runs the full
stack→tissue response; **`'tearing'` is the one remaining
magnitude-only passthrough** (direct → `avulsion`, byte-preserving
harm's shipped math), the documented seam that folds into a tearing
channel when one lands.

⭐ **`'thermal'` used to sit beside it and no longer does** — and that
transition is the worked example of how this seam is meant to close. It
did not become "a channel that skips the fold"; it became `heat`, a real
channel with **its own physics** (a covering attenuates by its material's
`thermalConductivity` inverted × layer depth) resolving into a tissue
`burn`. Leather turns a burn; plate conducts it, so a steel gauntlet is
*worse* than a bare hand. **A passthrough retires by acquiring a
mechanism, never by being promoted.**

## ⭐⭐ The seven channels, and the THREE folds

The vocabulary is `edge · point · blunt · shock · heat · cold ·
corrosion`, and what matters is that they do not all resolve the same
way. There are **three distinct architectures**, and each is the honest
physics of its own thing rather than one damage formula with different
labels:

| fold | channels | how a covering answers |
|---|---|---|
| **mechanical** | `edge` `point` `blunt` | a shape token × material property height (hardness/toughness) × grade/condition |
| **thermal** | `heat` `cold` | ⭐ the layer's **real `clo`** (`Wearable.getClo()` — thickness / k_eff, loft and wetness included), as a pulse: `1 − exp(−clo / ref)` |
| **corrosion** | `corrosion` | ⭐ **material MATCH.** No hardness, no thickness. |
| *(no fold)* | `shock` | resolves by **circuit** upstream — see below |

**`Channels.FOLDED`** is everything but `shock` — the set that walks the
covering stack, and the set the legibility surfaces iterate. ⚠ They used
to iterate `MECHANICAL_CHANNELS`, so a player examining a gambeson was
told how it answers a sword and never that it is the best thing in the
game against a burn: a column that exists in the model and not in the
readout is the same class of defect as one that does not exist.

### ⭐ `cold` — the heat fold run the other way

One fold, two wounds. The insulation arithmetic is **identical** for heat
and cold, because what a garment does is resist a temperature
*difference*; the direction is read exactly once, at the end, to name the
wound (`burn` vs `frostbite`). So leather turns a freeze for the same
reason it turns a burn, and steel betrays you either way.

⚠ Cold has **one dial of its own** (`response.cold.severityPerResidual`)
and shares `response.heat.*`, because those describe the **covering**.
Giving cold its own copy would be two numbers for one fact, and they
would drift.

### ⭐⭐ One insulation number — the thermal fold reads the garment's `clo`

The thermal fold used to score a layer by a heuristic of its own —
`refCond / (refCond + thermalConductivity)` × an ordinal "layer depth" —
while thermoregulation read the garment's derived **`clo`**. Two
insulation models for one physical fact, from different inputs, and
nothing asserted they agreed: a wool glove and a wool greatcoat were the
same number to a firebolt, and a soaked cloak insulated exactly like a
dry one.

Now the fold reads **`Wearable.getClo()`** — `thickness / k_eff` over the
garment's actual mass, density, covered area, loft and wetness — the same
number that widens its wearer's comfort band and damps how fast they shed
work-heat. **Three readers, one derivation.** So:

- a **thicker** coat of the same cloth stops more of a blow, by exactly
  the physics (`thickness` is in the number);
- a **soaked** coat stops *less* — water at 23× air's conductivity floods
  the loft and the clo collapses. No special case; it fell out of reading
  the real number;
- a **worn-through** coat stops less — grade and condition scale the
  block as they scale the mechanical fold.

⭐ **The reconciliation is the input, not the formula.** Each reader keeps
its own physics: shedding is *steady-state loss*, a resistance in series
with the body's own (`THERMAL_DEFAULTS.SHED_BODY_CLO`); a blow is a
**pulse**, and a thin layer stops a flash disproportionately — which is
why firefighters wear layers — so the fold is `1 − exp(−clo /
response.heat.referenceClo)`. At the seeded reference (0.1 clo, a
t-shirt) a hide jerkin stops ~90 %, a wool coat everything, plate almost
nothing, a soaked wool coat about half.

⚠ A layer with **no derived clo** — a held shield (Wieldable, not
Wearable), a preview from material alone, or a Wearable a term was
missing from (no mass, no density → `getClo()` is honestly `0`) — is
scored as a **slab** of its material at `response.heat.referenceThicknessM`
(5 mm), `R = t / k`. "We do not know how thick; assume a typical slab" —
and it keeps a wooden shield opaque to fire and a steel one transparent.
The old heuristic's three dials are retired.

### ⭐⭐ `corrosion` — the channel where thickness is irrelevant

Three outcomes, decided by two reads of the layer and no hardness
anywhere:

- **consumed** — the layer's tags intersect the agent's
  `Material.corrosiveTo`. It is being eaten, so it stops nothing: the
  full contact passes, and the layer **wears** for having taken it.
  (⚠ The inverse of the mechanical wear rule: a mechanical layer wears
  because it *stopped* something.)
- **wicks** — not attacked, but absorbent past
  `response.corrosion.shedAbsorptionMax`. It carries the agent through to
  the skin. A linen shirt is worse than nothing.
- **sheds** — not attacked, not absorbent. It runs off.

The claim in one pair: **a steel breastplate sheds lye and is eaten by an
acid whose row says `corrosiveTo: [metal]`**, and no amount of steel
changes either answer. For corrosion the armour question is not *how much
is there* but *what is it made of* — which is why it needed its own
`InflictSpec` variant: the agent's chemistry cannot ride an energy
scalar.

⚠ A preview has to assume an agent, because *"how does this answer
corrosion"* is not a well-formed question without saying against what.
`response.corrosion.previewCorrosiveTo` names the reference (seeded
`organic`).

#### ⭐⭐ How corrosion reaches a body — the substance-contact seam

Corrosion is the one channel a weapon blow and a spell's `inject-channel`
cannot deliver, for the same reason: the insult carries the agent's
chemistry (`corrosiveTo`), and neither a mechanical energy scalar nor a
bare channel token can source it. So corrosion arrives by a caustic
**substance in contact with a body**, not by an energy channel. Producers:

- **the lime-seep hazard** — `HazardDelivery.toInflictSpec` on traversal
  (the original, and long the only one);
- **`Material.corrodeOnContact(victim, {energy, site})`** — a caustic
  material's own capability: it reads *its own* `corrosiveTo` and routes a
  `CorrosionInflictSpec` through `ConditionApi.inflict`. No-op (false) for
  any non-caustic material. This is the general seam, and it makes three
  events deliver corrosion with no second code path — a **thrown flask**
  that shatters on a body (the `throw` splash), a **conjured caustic** the
  acid-splash spell puts on the mark (`execConjure`), and a **spilled
  vial**. Honest across all three: the substance's chemistry does the
  work; magic only *collects* the caustic, it does not mint "acid damage".

## The `shock` channel — resolves by circuit, not the fold

The [electricity](./electricity.md) build added **`shock`** to the `Channel`
vocabulary — but it does **not** resolve through the energy-attenuate covering
fold. The three mechanical channels (edge/point/blunt) are now their own
closed subtype **`MECHANICAL_CHANNELS`**: `Construction`'s shape tables, the
`materialScale`/`attenuate`/`resolveTrauma` fold, and the `analyze response` /
pip loops all key on *that*, so `Construction` stays honestly mechanical
(`responseFor('shock')` throws — a shock is not a construction-shape axis).
`shock` resolves by **circuit** (`I = V/R`, reading conductivity not hardness)
on the same `MaterialApi`/`MaterialLogic` home, and `ConditionApi.inflict`
intercepts it into a **third path** that skips the fold entirely (the
mechanical + passthrough paths are byte-identical). The armor inversion falls
out — metal conducts (betrays), rubber insulates. See
[electricity.md](./electricity.md).

## Deferred (named seams)

- **Combat playstyle + loop** — reach/guard/gambits, shield-as-armor,
  unarmed/grapple (combat-slate.md).
- **Ranged / thrown** — the **thrown** half **shipped** (ranged Wave 1 →
  [ranged.md](./ranged.md)): a `DeliveryProfile` derives `channel` +
  `energyJ` from mass × speed and hands the grid an ordinary
  `InflictSpec`, so an arrow and a bullet with the same profile cut the
  same wound and nothing here needed a projectile concept. Still
  deferred: **armor's** point→blunt conversion for stopped strikes,
  `penetration` as a profile term, and the launcher families
  (ranged-slate.md W2/W3/W4).
- **The economic lifecycle tail** — repair (reverse-craft), scrap/reforge,
  the `Recipe` craft-stamp that carries `{material, construction, grade}`
  onto made things (the armorer economy). Armor/implements carry their axes
  as *authored* data for now.
- **Other channels** — `crush` (structures/destructibility), `heat`/`cold`
  (thermal `clo`/`burn` unification), `corrosion` — each pulls its
  channel/construction slice when its consumer lands.
- **Tissue as a construction axis** — tissue stays material-only v1; the
  tissue-vulnerability severity term at `resolveTrauma` is a named seam.
- **Avulsion sever / part-promotion** — at `AVULSION_BEHAVIOR.onset` (harm).
