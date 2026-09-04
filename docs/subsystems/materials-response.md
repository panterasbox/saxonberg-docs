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

A blow is decomposed into three orthogonal axes — the channel selects the
*point* on the curve, the material scales its *height*, the construction
picks its *shape*:

- **Channel** (`lib/material/Channel.ts`) — the closed, additively-growable
  mechanism vocabulary, v1 **`edge` / `point` / `blunt`**. The single shared
  interface a weapon's *delivery*, an armor's *resistance*, and a tissue's
  *failure* all transact over. Not a "damage type" — the *shape of the
  force*. (`crush`/`heat`/`cold`/`corrosion` join as columns when their
  consumers land.) The `Grade`/`ToolCapability` value-object precedent:
  vocabulary tuple + type + a thin `Channels` static holder.
- **Material mechanical properties** — `Material` gained grounded
  `hardness` (`Quantity<'MPa'>`, indentation hardness) and `toughness`
  (`Quantity<'MJ/m³'>`, energy absorbed before fracture), siblings of the
  existing `density`/`specificHeat` (strict-unit accessors, marshaller
  binding, `fieldMeta`'s persistent entries). The *height* a material lends the response
  curve. Materials stay content (packs supply the roster + values); this is
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

| armor form | edge | point | blunt |
|---|---|---|---|
| plate | deflect | resist | transmit |
| mail | resist | fail | transmit |
| padded | poor | poor | absorb |
| hide | moderate | poor | moderate |

| delivery form | primary | secondary |
|---|---|---|
| bladed | edge | point |
| pointed | point | edge |
| hafted | blunt | — |
| flail | blunt | — |
| whip | edge | blunt |
| blunted | blunt | — |

Armor forms also carry a canonical outside-in **`LAYER_DEPTH`** (padded 0 …
plate 3) so a covering stack orders itself with no authored number.

## The response function (the Api home)

The function lands on the **existing `MaterialApi`** (`api/material.ts`) +
`MaterialLogic` (`platform/idea/api/MaterialLogic.ts`, gated
`FromModule('/api/material#MaterialApi')`), filling the surface
`MaterialLogic` had reserved for `damageResistance`. No new Api. The same
function is read from both sides of a blow (Settled-6 "one function, two
consumers"):

- **`attenuate(channel, energy, material, construction, grade, condition)`**
  — one armor layer. `token = construction.responseFor(channel)` (SHAPE) →
  an AppSettings base fraction (MAGNITUDE) × `materialHeight` × grade×condition
  → clamped attenuation; residual energy passes inward.
- **`resolveTrauma(channel, energy, tissueMaterial, partHasBone)`** — the
  residual meeting tissue: `edge → laceration`, `point → puncture`, `blunt →
  fracture` (a boned part at/above the threshold) else `contusion`; `null`
  when the residual is below the no-wound threshold (deflected). Both
  severity AND type derive here.
- **`previewBand` / `severityToBand`** — the legibility chokepoint (below).
- **`deliverableChannels(construction)` / `primaryChannel(construction)`** —
  the weapon-delivery derivation (a dagger delivers edge, a mace blunt).

`materialHeight` is a small documented pure function: `edge` is
hardness-driven, `blunt` toughness-driven, `point` mixes both; each
normalized against a reference (steel) magnitude and lifted by a structural
**floor** so a construction's token sets the protection ceiling and material
modulates within `[floor, 1]` (a hide boot still turns a shallow cut, a
soft absorber still absorbs). `grade × condition` scales *height only*
(Settled-4), tuned so a masterwork at ~50% condition ≈ a common piece
pristine. The scalar is exposed as **`MaterialApi.gradeConditionScale`**
so combat's instrument-delivery fold reads the SAME formula (the
crafting-branches coupling — see [combat.md](./combat.md)); combat's
delivery deliberately folds quality only, leaving material *height*
analyze-only (the documented asymmetry).

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
`Construction` (its resist form, via the new `ConstructedMixin`), a `Grade`,
and a wear-on-use `condition` (`DurableMixin` — the durable-good half split
out of `ToolMixin`, so armor wears out without being a "tool").
`lib/equipment/Armor`
and `lib/equipment/Weapon` are thin compose-and-name classes (siblings of
`Garment`); their "armor-ness"/"weapon-ness" is purely the composition,
nothing narrows on an `isArmor`.

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

## The thermal / tearing passthrough

The mechanism vocab is unified into the channel set, but harm ships
`thermal → burn` and `tearing → avulsion`, whose channels (`heat`, a tearing
channel) are explicit non-goals. `InsultKind = Channel | 'thermal' |
'tearing'`: a `Channel` runs the full stack→tissue response; `'thermal'` /
`'tearing'` take the **legacy magnitude-only passthrough** (direct → burn /
avulsion, byte-preserving harm's shipped math), the documented seam that
folds into a `heat` / tearing channel when those land.

## The `shock` channel — resolves by circuit, not the fold

The [electricity](./electricity.md) build added **`shock`** to the `Channel`
vocabulary — but it does **not** resolve through the energy-attenuate covering
fold. The three mechanical channels (edge/point/blunt) are now their own
closed subtype **`MECHANICAL_CHANNELS`**: `Construction`'s shape tables, the
`materialHeight`/`attenuate`/`resolveTrauma` fold, and the `analyze response` /
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
