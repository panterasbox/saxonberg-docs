# Crafting — the three branches & the repair economy

Crafting is the **transformation** stage of the economy: where raw
inputs become valued goods — the point at which value is minted. The
first build shipped the **served path** at Dave's Bar (the bulk
cocktail branch); the crafting-branches build grew it to **three
branches over the one skeleton** — the bar's bulk drinks, **smithing**
(discrete metal stock + heat → formed gear), and **cooking** (discrete
food stock + low heat → stamped edible meals) — plus the full
**lifecycle**: crafted → used → worn → sharpened/repaired → broken →
salvaged. Crafted gear is real gear: a forged blade's material, grade,
and condition are consumed by the shipped readers (`analyze weapon` /
`analyze response`, combat's instrument-delivery scale, the covering
stack).

The substrate (`lib/craft/`, `api/crafting.ts`, `CraftingLogic`,
`RecipeCatalogue`) is **general and reusable**; Dave's Bar, the
Hearthworks smithy, and the cookhouse are authored *content* over it
(templates + recipe Documents), never venue-specific classes.

## The model: crafting is location-agnostic

The thing crafting represents is **transformation**, which has nothing
to do with place. The primitive is:

```
recipe + maker + reachable tools/inputs + (fixed) control → stamped output
```

`CraftingApi` has **no "venue" concept**. Feasibility is **emergent** —
"can I craft this here?" reduces to "are the required tools + inputs
reachable from the maker?" There is **no `CraftingVenueMixin`** (an
early draft's mistake that fused recipe-knowledge, maker,
means-of-production, and the offer onto one Location flag). Each of
those is its own concept with its own home:

- **Recipe** — the transform spec, a `Document` (see below).
- **Maker** — the agent who performs the craft and is stamped as
  provenance (a present bartender for v1). `MakerMixin` is
  **augment-gated** (`_augmentGated = true`): a bar `Crafter` composes it
  always but is a maker only while its **on-shift** Position confers it, so
  `MixinApi.isMaker` (now routed through `isActive`) and thus
  `resolveMaker` select only the on-shift bartender. See
  [employment.md](./employment.md).
- **Tools + inputs** — ordinary `Tangible` matter, wherever it
  physically sits; reachability is the only feasibility gate.
- **Offer / `Menu`** — a *curated* list of offered recipes, its own
  `Tangible` object you `read`/`order` from, decoupled from
  craftability, from where crafting happens, and from the maker.

"Dave's Bar" is therefore **emergent**: a room that contains a `Menu`,
a maker, and the matter. Remove any and you can't get a drink there —
with a diegetic reason, not a flag flip.

## Value-objects & vocabulary (`lib/craft/`)

- **`Grade`** (`Grade.ts`) — an ordinal quality band, the first quality
  axis. `GRADE_BANDS = ['poor','fair','fine','exceptional','masterful']`
  (low→high, doubles as the validation array). `Grade.of(band)`,
  `getBand`/`getOrdinal`/`renderBandWord`/`compareTo`/`min`/`max`.
  `deriveAtFixedControl(inputs, _control?)` = **weakest-link** (`min`
  across input grades), `fair` fallback for an ungraded craft; `_control`
  is the deferred skill seam (ignored in v1). A plain value-object, **not**
  a `Quantity` (grade is ordinal-categorical, not a measured scalar).
- **`ToolCapability`** (`ToolCapability.ts`) — the capability
  **contract**: an **open** vocabulary (any kebab name a recipe's
  `toolCapabilities` and a tool row's `capabilities` agree on — the
  kernel keeps no list, so a pack that ships a `still` and the recipes
  that need one edits nothing here) + `ToolCapabilities.validateEntry`
  and the `CapabilitySpec` parameterized-entry shape (`kind`, **`verbs`**
  — the command-view keys the entry confers, the row's own data —
  `rate`/`control`/`technique`, `RATE_MIN`/`RATE_MAX`). There is **no
  workbench concept** —
  "workbench" is just the word for capabilities too heavy to carry: the
  pot and whetstone are portable capital, the anvil (60 kg — a fixture
  by encumbrance, not by flag) and forge are not, so camp-stew is
  possible in the wilds and a blade is venue-bound, diegetically.

## Mixins (`lib/craft/`)

All are registered in `lib/mixin.ts` (`Mixins.Graded`/`Tool`/
`Crafted`/`Maker`/`ManualBuild`, plus the material-axis
`Durable`/`Keen`) with a `MixinApi.isX` predicate. No
`#`-private instance state (proxy trap); persisted fields are public for
the Hydrator; per-field invariants ride accessor pairs.

- **`GradedMixin`** (`Graded.ts`) — the shared grade carrier (input
  bottles **and** `CraftedMixin`). The persist-string / value-object
  split: the persisted field is the band **word** (`gradeBand: string`),
  and the `Grade` value-object rides separate contract methods
  (`getGrade`/`setGrade` ↔ `getGradeBand`/`setGradeBand`). `setGradeBand`
  validates via `Grade.isBand`.
- **`DurableMixin`** (`lib/material/Durable.ts` — the wear-state axis of a
  physical object, sibling of `Tangible`/`Constructed`; **not** a crafting
  mixin) — the durable-good substrate: a `condition` (0..1, private
  `_condition` accessor pair clamping) that `wear()`s on **use, not the
  clock** (economy Law 2), plus `isBroken()` (the capability-loss
  threshold — see the lifecycle section; `repair` is its restore).
  Composed by tools, **weapons, and armor alike** (durability is not
  "tool"); narrow via `MixinApi.isDurable`. Its fast-cycling sibling is
  **`KeenMixin`** (`lib/material/Keen.ts`) — the edge axis, on `Weapon`.
- **`ToolMixin`** (`Tooled.ts`) — the crafting **capabilities** layer:
  `capabilities: string[]` a recipe requires by kind. A `ToolItem` composes
  `ToolMixin(DurableMixin(…))` — a tool is a durable good that *also* offers
  capabilities; the durability lives on `DurableMixin`, not here.
- **`CraftedMixin`** (`Crafted.ts`, composes `GradedMixin(Base)`) — the
  per-instance **maker's mark**. Own persistent fields `{maker (templatePath),
  recipe (recipeId), craftedAt (game-seconds)}`, set once via `stamp(spec)`
  at craft-resolve. `renderVerdict()` renders the **DF-style** verdict —
  band-word headline + grade-keyed prose + maker name (resolved via a
  module-private `resolveMakerName` → `getPresentation()`) — **never a
  number**. The crafted instance's `templatePath` still points at the
  shared output template; the mark is the per-instance overlay. The future
  corpos(marks) build extends this.
- **`MakerMixin`** (`Maker.ts`) — a minimal role marker (`isMaker()`),
  used **only** to identify the present `order` fulfiller (the bartender).
  **Not** used to gate `serve`/`mix` (those are general agent verbs;
  maker = the giver).
- **`ManualBuildMixin`** (`ManualBuild.ts`, the `Builds` interface,
  `MixinApi.isBuildVessel`) — the **vessel-as-buffer** for a step-by-step
  build (the shaker / mixing glass). A runtime-only buffer of graded
  `BuildContribution`s (`addContribution`/`getContributions`/`clearBuild`)
  the `pour`/`add` verbs bank into and `strain` mints from, plus the
  recorded **command sources** (`recordCommand`/`getCommandSources`) the
  demonstration capture transcribes (see [scripting.md](./scripting.md)).
  No persistent fields — a build is transient.

## Recipe = a `Document` (the Emote pattern)

A recipe is **data** — no mutable state, no lifecycle, no per-instance
identity. So it is **not** a Stuff/`Idea` (the clone pipeline, proxy
wrap, and singleton registration would buy nothing) and **not** a
`domain` template (a template's one job is to hydrate a Stuff; a
never-cloned template is a wart — the rejected `Topic` shape). The home
is the codebase's pattern for authored reference data that isn't Stuff:
a **`Document`** managed by a catalogue singleton — the
`Emote`↔`SoulCatalogue` relationship.

- **`Recipe`** (`lib/craft/Recipe.ts` — since content-packs wave 2 a
  **value shape** over one `documents` row of `kind: 'recipe'`, unique
  per kind on `data.recipeId`; `fromDocument` validates what the retired
  seeder validated, `toData` is the inverse, `fieldMeta` stays as the
  wiki's spoiler schema) — typed accessors over `name`, `keywords`,
  `inputSlots: RecipeInputSlot[]`, `toolCapabilities: string[]`,
  `outputTemplate` (a **real** cloneable template), `outputMaterial`,
  `baseGradeBand?`, and the branch fields below. The boundary stays
  honest — `outputTemplate` is a Stuff cloned by the pipeline; only the
  recipe-as-knowledge is a Document. `Recipe ≠ template` (a form can
  have several recipes).

  **The schema grew without forking — one Document, additive fields**
  (old rows parse byte-identically; `Recipe.schema.test.ts` is the
  round-trip guard):

  - `RecipeInputSlot` is discriminated by `kind`: absent/`'bulk'` is
    the bar's slot (`measureL` litres); `kind: 'item'` is a
    discrete/glob slot (`count` units of a reachable Tangible whose
    Material carries the `category` tag — an ingot, a root vegetable;
    ungraded stock derives at `fair`).
  - `requiresHeatK?` — the **heat gate**: craft-resolve declines
    (`insufficient-heat`, diegetic) unless
    `reachableHeatK(maker) ≥ requiresHeatK` — the fire
    build's D9 seam, consumed. The forge holds 1300 K (2080 K with the
    bellows worked), the oven 500 K — so the belt-knife's 1400 K
    genuinely requires the bellows and the whole cooking ladder runs at
    a lit hearth. Heat is a *reachability* fact like tools: camp-stew
    works at any campfire.
  - `outputApplication?: 'bulk' | 'tangible' | 'edible'` (absent =
    `'bulk'`) — the branch discriminator the output/consume seams
    dispatch on.
  - `outputPortionL?` — the edible portion the dish is filled to.
  - `outputAppearance?` — a **derived** blend's prose (per-dish content
    belongs on the recipe, never on a Material row).
  - `outputMaterial?` — now an OPTIONAL authored-substance **override**;
    empty (the shipped roster) ⇒ the blend **derives** (below).
  - `difficulty?` / `discipline?` — the authored **ladder placement**
    the craft-resolve evidence records (see the knowledge ladder below);
    absent ⇒ no advancement row (every bar row).
- **`RecipeCatalogue`** (`obj/RecipeCatalogue.ts`, singleton
  `PostRegistrationMixin(Idea)` at `/platform/idea/RecipeCatalogue`) — caches
  `DocumentApi.listOfKind('recipe')`, resolves by id + keyword (`order
  martini` → one recipe), `warm()` on `postRegister`,
  `invalidateCache()` + `warm()` as the installer's go-live after a live
  `pack sync`. The `SoulCatalogue` shape. **Read methods
  are ungated** (the `TopicCatalogue` "public knowledge" precedent — a
  catalogued recipe is openly resolvable; v1 has no minting verb, so no
  `RecipeApi` and no dangling gate string).
- **The `generic-objects` pack** (`packages/content/generic-objects/
  content/recipes/<recipeId>.yaml`, one file per recipe; the basename IS
  the `recipeId`) — installed by `PackApi.install` and reconciled
  three-way; a file with an empty `inputSlots` or no `outputTemplate`
  fails the pack at `read`. The former `RecipeSeeder` +
  `mud/config/recipes.yaml` are gone; the collapse migration carried an
  existing dev DB's `recipes` rows across (`_id`s preserved).

## Craft-resolve: `CraftingApi` / `CraftingLogic`

The gated forwarding pair (the `ProvenanceApi`↔`ProvenanceLogic` shape):
`api/crafting.ts` is the thin gated shell (`SecurityApi.decorateApiClass`),
`platform/idea/api/CraftingLogic.ts` is the `@internal` logic singleton at
`/platform/idea/api/crafting`, methods gated
`@CallSecurity(FromModule('/api/crafting#CraftingApi'))`. Sub-logic is
**module-private** functions (no intra-singleton `this.x()`).

**Call shapes** (their home is `api/crafting.ts`):

- `CraftRequest { recipeRef: string; makerMode: 'self' | 'fulfilling-bartender'; brand?: string }`
  — **no principal**. (`brand` is v1's single-`with` selection; the
  per-slot `inputSelections` map is deferred.)
- `CraftOutcome = { ok: true; output: Stuff; grade: Grade } | { ok: false; reason: CraftDeclineReason; detail: string }`
  — a discriminated union; **declines** (out of gin, no shaker) are
  data, **programmatic/conservation breaches throw**.
- Statics: `craft(req)`, `lookupRecipe(ref)`, `offeredRecipes(menu)`.

**`craftImpl` algorithm:**

1. **Resolve recipe** via `RecipeCatalogue` (id or keyword) → decline
   `no-recipe`.
2. **Resolve maker by `makerMode`** (un-spoofable; wire carries only the
   mode): `'self'` → `ExecutionContextApi.getActingAuthor()` (the giver,
   for `serve`/`mix`); `'fulfilling-bartender'` → the giver is the
   *patron*, so find the present `MixinApi.isMaker(...)` agent in the
   patron's location. Never off the wire.
3. **Gather reachable matter** (`gatherMatter`) — the walk's real
   shape: the room's direct contents (surface-resting items already
   have `container = the room`), **the maker's own inventory** (held
   kit + carried stock), and **one-level descent into open room
   containers** — a Sealable-closed (or locked) container never feeds
   a craft; open-ness is the switch (the pantry-chest pull, honestly);
   a non-Sealable room container counts always-open; containers
   carried by *other* agents are never descended into. Partition into
   tools / graded bulk bottles / **item candidates** (Material-bearing
   Tangibles that are raw matter — never a tool, crafted gear, a
   container, a bottle, or anything living: the anvil never feeds the
   forge).
4. **Match input slots** — dispatched per slot on `kind`. Bulk: a
   bottle whose `Material` carries the `category` tag, `grade ≥
   minGrade`, available litres ≥ `measureL`, per-bottle claims
   tracked. Item: `count` units drawn greedily across candidates by
   the same tag rule (a glob covers many units, a discrete Tangible
   one; ungraded stock counts `fair`); per-source claimed units
   tracked. Honor `brand`, else highest grade → decline
   `insufficient-input`.
5. **Match tools** — per required capability, a present tool with
   `hasCapability` → decline `missing-tool`. (A **broken** tool offers
   no capabilities — see the lifecycle below.)
6. **The heat gate** — `requiresHeatK > 0` and
   `reachableHeatK(maker) < requiresHeatK` → decline
   `insufficient-heat` ("the forge is cold").
7. **Derive grade** — `Grade.deriveAtFixedControl(matched)` (bulk +
   item grades together), floored at `baseGrade`.
8. **Clone output** — `StuffApi.clone(outputTemplate)` → decline
   `no-output`.
9. **Apply output properties** — dispatched on `outputApplication`:
   - `'bulk'` / `'edible'` — fill the output's bulk slot at
     `Σ measureL` / `outputPortionL`. The substance is **derived by
     default**: the slot points at the ONE generic blend base
     (`/platform/idea/material/blend` / `/platform/idea/material/cooked`) and a per-instance
     **`BulkPayload`** carries the blend's identity (recipe
     name/appearance/keywords) + macros **summed from the consumed
     inputs** — macros in = macros out (a martini's 26 mg of alcohol
     IS gin 19 + vermouth 7; a stew's carbs ARE its roots'). An
     authored `outputMaterial` remains a supported override (fills
     with that substance, no payload — asserts edibility for
     `'edible'`). See [bulk.md](./bulk.md) § BulkPayload and the
     fixed-vocabulary rule in [race.md](./race.md).
   - `'tangible'` — `applyTangibleOutput`: flow the **primary matched
     item input's Material** + the summed consumed mass onto the
     cloned Tangible (an iron ingot makes an iron knife —
     mass-conserving; the `ThermalLogic` casting-stamp surface).
10. **Stamp** — assert `isCrafted(output)` (else throw — output
    template misauthored), `output.stamp({maker, grade, recipe,
    craftedAt})`.
11. **Consume inputs** — `consumeBulkInputs` (strict `BulkableApi`
    debits) + `consumeItemInputs` (a glob debited by exactly the
    matched units, destructed when fully drawn; a discrete Tangible
    destructed whole — its chattel id released by the shipped
    `onDestruct` path). Short debits throw (conservation breach).
12. **Wear tools** — each matched tool `.wear()` (Law 2).
13. **The evidence tail** — `recordCraftEvidence` (see the knowledge
    ladder): an advancement deed for the maker + the known-of claim
    for watching agents, iff the recipe authors a `discipline`.

Placement is the verb's job, not craft's (`craft` returns the output).

### The domain seams (the branch architecture)

The apply/consume steps are the **only** domain-specific steps,
isolated behind named module-private fns — the promise "other domains
arrive as new branches, not a rewrite," kept: smithing arrived as
`applyTangibleOutput` + `consumeItemInputs`, cooking as
`applyEdibleOutput` over the same item consume; `craftImpl` dispatches
on the recipe's `outputApplication` and the skeleton
(slots-by-constraint, tools-by-capability, the heat gate, weakest-link
grade, provenance, the fixed-control seam) is domain-neutral.
**Physics forms state, recipes form shape**: smelting/casting stays the
phase engine (Forge + `MeltableMixin` + `Casting` — see
[fire.md](./fire.md)/[thermal.md](./thermal.md)); recipes own the
*forming* physics can't express (ingot → blade). **Assembly** (output
properties emerge from components) is a genuinely different model,
still deferred — not faked.

## The offer: `CommerceMenu` + the venue subclasses

The venue-neutral base is **`lib/commerce/Menu.ts`** (`CommerceMenu`, a
`PricedOfferMixin(DetailedMixin(Thing))` object — a `Tangible` you
`read`/`order` from, **not** a place capability and **not** a mixin).
It owns the offer machinery: `offeredRecipes: string[]`,
`resolveOrder(keyword)` (gates on the **offer**, not craftability),
`resolveIn(context)` (the affording menu or the reachable one — the
`instanceof CommerceMenu` filter matches any venue subclass), the
inherited `prices`/`priceFor` (Law 1: worth on the offer — the same
`PricedOfferMixin` the store's `Stock` composes), and the offer-listing
`getLong()`.

A menu affords **commerce only** — the base's `commandContributions` is
`menu`/`order`, any venue, and the ONE concrete `platform/thing/Menu`
(content packs wave 4b collapsed the three empty venue subclasses —
the lounge's `Menu`, `SmithyMenu`, `KitchenMenu` — into it; what
differed was the rows' data: `offeredRecipes`, prices). The
*working* verbs are **instrument-conferred, and named exactly once** —
on the class of the thing that performs the act, as `static
commandContributions`. There is **no kernel table** (libations retired
`CAPABILITY_TABLE`: the kernel could not name a trade's view without
owning the trade) **and no row-level verb list** — `capabilities[].verbs`
and `placement` were deleted after them, because two records of one fact
is the same failure one level down (see
[command-routing.md](./command-routing.md) § *There is ONE record of verb
affordances*). A capability entry now names a **requirement and a
working**, never a verb.

⭐ **Split by what performs the act.** That is the discipline that keeps
one record honest, and the bar is the worked example — its six verbs
used to be a single list copied verbatim onto two rows:

| verb | performed by | declared on |
|---|---|---|
| platform `pour`, `stir` | the build VESSEL — what a build is banked into and worked on | `ManualBuildMixin` (shaker, mixing glass, cook pot) |
| hospitality `strain` | the STRAINER | `/trade/hospitality/thing/Strainer` |
| hospitality `mix`, `serve`, `garnish` | the STATION — whole-drink acts at the bar | `/trade/hospitality/thing/BarStation` (back-bar, well) |
| hospitality `muddle` | the MUDDLER | `/trade/hospitality/thing/Muddler` |
| cooking `cook`, `plate`, platform `heat` | the POT | `/trade/cooking/thing/CookPot` |
| platform `boil` | the FURNACE — you cannot boil without a heat source | `FurnaceMixin` (oven, kiln, forge) |
| smithing `hammer`, `quench`, `forge` + platform `repair`, `salvage` | the ANVIL | `/trade/smithing/thing/Anvil` |
| platform `repair`, `salvage` | mending capital | `/platform/thing/MendingTool` (sewing kit, sewing machine) |
| smithing `sharpen` | the STONE, **carried only** | `/trade/smithing/thing/Whetstone` |
| platform `water` | the CAN, **carried only** | `/platform/thing/WateringCan` |
| `striking` · `strainer` · `juicer` · `tap` · `bar-spoon` · `still` | — | bare kinds: recipe-side requirements, no verbs |

**Carried vs reachable is the bucket, not a second word.** A class
declaring both `environment` and `peers` is reachable (carried or on the
floor); one declaring only `environment` is personal capital — your own
whetstone anywhere, and nothing from a stone across the room. The
row-level `placement: reachable | carried` was a coarser rename of a
subset of the four buckets, and could not express `self` or `inventory`
at all.

⚠ **A class per instrument is the price, and it is the right one.** A
row can no longer vary its verbs — the sewing kit and the sewing machine
differ in `rate` and `control`, which is row data, and afford
identically, which is class data. That is the point. It also closes a
cross-pack leak for free: a pack's classes name only that pack's views,
so `Whetstone` moved out of `/platform/thing/` rather than let the
kernel name `trade/smithing/…/sharpen.yaml`.

The surface follows capital, not venue flags (the menu is for ordering,
not making; camp cooking works because reachable heat + a pot IS a
kitchen).

**Where a crafting verb lives** — with the pack whose content affords
it: `make`, `heat`, `boil`, `pour`, `stir`/`shake`, `repair`, `salvage`,
`wash` are platform (a pot and a shaker both pour and stir; a furnace
heats and boils; a basin washes; mending is any trade's); `menu`/`order` are platform
**retail** (a menu is commerce, any venue); `muddle`, `strain`,
`garnish`, `mix`, `serve` are `trade-hospitality`'s; `cook`, `plate`
are `trade-cooking`'s; `forge`, `hammer`, `quench`, `sharpen`
are `trade-smithing`'s — each a capability pack with its controllers in
`src/idea/cmd/crafting/` and the tests beside them. The kitchen bundle
and the cook-pot row are cooking's too (the bundle collects the
trade's instruments).

`FurnaceMixin` statically confers `heat` (with `ignite`/`douse`/`pump`)
— an appliance mixin, not a `Tooled` host — and `make` is innate on
`Avatar` (knowledge-driven, no instrument). Capability entries are
**parameterized**: `{ kind, rate?, control?, technique? }` (a bare
string = the defaulted spec) — `rate` is a work-rate multiplier (clamped 0.25–10 at read)
that divides the engaged duration of the steps the kind confers (the
**conferring kind paces the step**: the anvil paces `hammer`/`quench`,
the `mending` instrument the now-engaged `repair`; the `striking`
hammer is a requirement, never a pacer), and `control` is a Grade band
embedded in the capital that **floors** the outcome grade of work done
with the instrument (craft, mint, repair — floor only; the ceiling
stays the skill seam's). `analyze` reads a control-bearing tool's
band. `CocktailShaker` backs both the shaker and the mixing glass and
carries no affordance statics of its own — what they perform in common
(`pour`, `stir`) is `ManualBuildMixin`'s, and what differs between them
is the technique, which is row data. Affordance surfaces refresh on the
next containment delta, so a `setCapabilities` edit or a break/repair
doesn't re-push mid-placement — which no longer matters for the verb
SET, since that is now fixed by the class.

`MenuController`/`OrderController` import the lib base, so `order` is
venue-generic: a smithy with a menu and an on-shift maker just works.

## Verbs (the `crafting` command category)

`content/platform/cmd/crafting/*.yaml` views + `mud/platform/idea/cmd/crafting/*Controller.ts`.
A `CraftController` base centralizes decline rendering
(`declineToScene`) **and the can-make deed gate** (`requireDeed` — the
one gate `forge`/`cook`/`make` share; a non-catalogue ref passes
ungated, `order` never calls it); all controllers return `void`,
emitting via `ctx.note` + `MessageApi.scene`.

- **`menu`** — render the present Menu's offer.
- **`order <item> [with <brand>]`** — maker = **fulfilling maker**
  (`'fulfilling-bartender'` — the historical name for "present active
  maker", reused unchanged across venues); resolve the Menu,
  `resolveOrder`, craft, hand the output to the patron (the giver).
  **Never knowledge-gated.**
- **`serve <patron> a <item> [with <brand>]`** — maker = **giver**
  (`'self'`); craft and move to the patron.
- **`mix <item> [with <brand>]`** — maker = **giver**; craft and hold.
- **`forge <item> [with <metal>]`** / **`cook <dish>`** — the branch
  one-shots: maker = **giver**, **deed-gated** on the knowledge ladder
  (below); `with` steers the stock pick exactly as the bar's brand.
  ⭐ A `CraftRequest` may also carry a **`target`** — the item an act is
  performed ON, preferred for any input slot it satisfies. Without it,
  `dry` would pick a plain cut off the table instead of the one you had
  just salted, and hurdle-stacking would be unreachable by a player.
  A preference, not a gate.
- **`sharpen <blade>`** / **`repair <item>`** / **`salvage <item>`** —
  the maintenance acts (see the lifecycle below). Standalone diegetic
  verbs per the subcommand sieve; any future introspective growth
  (inspect/estimate/quote) grows as subcommands, not verb sprawl.

`serve`/`mix`/`forge`/`cook` are general agent verbs; `order` routes to
the present on-shift maker.

- **`wash <glass>`** (libations) — afforded by **`WaterFixture`** (the
  bar basin, the water tap, the dorm tap, the standpipe) in the `peers`
  bucket. ⚠ It shipped on `UnboundedReceptacle` — *inexhaustible liquid
  source* — whose other row is the demo's **coffee urn**, and in the
  `environment` bucket, which grants OUTWARD to the containers ABOVE a
  thing. A basin is the player's SIBLING and nobody carries one, so
  **`wash` was afforded to nobody, anywhere it shipped**; the controller
  tests passed throughout because they call the controller. Affordance
  is wiring, and wiring needs its own assertion
  (`lib/command/__tests__/WaterFixture.affordance.test.ts`).
  The controller stays deliberately more permissive than the affordance
  — any reachable bulk holder whose matter is water, a carried jug
  included. The fixture provides DISCOVERABILITY; water in reach is what
  makes it work. A static cannot read a holder's contents, and should
  not: that is the state-dependent affordance the `InstanceContributor`
  seam expressed, deleted for the reasons in
  [command-routing.md](./command-routing.md).
  An engaged step (~3 s) that needs a
  reachable water source: a `Bulkable` holder whose material carries the
  **`water` tag** (the basin/tap `UnboundedReceptacle` affords the view).
  ⚠ The tag, and only the tag — this once also matched the material's
  keyword and display name, which is how it worked at all, since the
  water row carried no `water` tag; see
  [antipatterns.md](../antipatterns.md) § *Keywords Where You Mean
  Identity*.

  ⚠⚠ **And it is no longer a glassware verb.** It was
  `instanceof CraftVessel`, so a **knife** could not be washed at all —
  which the food-safety build made load-bearing, because a knife that has
  been in a carcass is the one implement that most needs it and washing is
  the whole counterplay to cross-contamination. It now accepts anything
  `Serviceable` (the soil mark) or `Contaminable` (the pathogen load), and
  a target may be either or both. ⚠ It clears a **surface** and never
  contents: washing a pot of bad stew is not a cure for the stew.

  ⭐ **The effect is `vessel.wash()` on `CraftVessel`**, not an Api call.
  It tips the dregs, destroys any garnish inside, drops the ice and the
  technique stamp, and marks it clean. It was `CraftingApi.washGlass`
  (placed there because an engaged `onComplete` runs under the scheduler
  frame, so a `FromModule(WashController)` gate could never pass) — but
  that was a *gating* argument, not a placement one: every line touched
  nothing but the vessel's own state, and with the writer inside the
  class `soiled` needs no gate at all. Named for the vessel rather than
  the glass, because a syrup bottle and a juice bottle are
  `CraftVessel`s too. Bussing is `get <glass>` / `put <glass> in rack` —
  shipped verbs.
- **`cure`/`salt`**, **`dry`/`hang`**, **`smoke`** (trade-cooking) — the
  **preserving acts**, one `PreserveController` base and three six-line
  subclasses that name only a recipe id and their prose. Each recipe
  authors a `cure: { moisture?, solute? }` block applied to the output's
  water state as the **stronger** of each axis, so a weaker second
  treatment never un-cures and two separate acts stack (salt it, then dry
  it). ⚠ **Not deed-gated**, unlike `cook`: the can-make deed is earned by
  a faithful by-hand build, and the cooking branch's by-hand path banks
  into a pot — the wrong shape for a discrete transform. A gate whose key
  does not exist is a lock, so these follow `order`.
- **`butcher`/`dress`** (trade-cooking) — take a dead animal apart. Gated
  on a reachable `constructionForm: 'bladed'` implement, refused on a
  sentient corpse, and the cuts carry the carcass's AGE (see
  [spoilage.md](./spoilage.md)).

⭐⭐ **One verb, one station, and the class name predicts its surface.** The
four larder verbs are each afforded by the fixture that performs them —
`ButcherBlock` · `SaltingTrough` · `DryingRack` · `SmokeChimney` — with
`CookPot` keeping only `heat`/`cook`/`plate`.

⚠ They were all briefly conferred by **`CookPot`**, which handed anyone
standing near a saucepan the power to take a hog apart, and stopped that
class's name from predicting its own surface. It is the
`wash`-on-`UnboundedReceptacle` mistake exactly — *an urn is not a
degraded basin*, and a pot is not a degraded butcher's block. Pinned by
`src/__tests__/kitchen-affordances.test.ts`, because **affordance is
wiring and wiring needs its own assertion**.

⭐ Each keeps the fixture/instrument split `wash` documents: the station
gives DISCOVERABILITY, and something in reach gives capability — an edge
for the block, salt for the trough, a cool fire for the chimney. And all
four are `peers`-only and fixed in place, which has an honest consequence:
**you cannot field-dress a boar in the woods.** You carry it home to the
block with the clock already running, which is exactly the pressure the
cuts' age model exists to create.
- **`muddle`** — a `ManualBuild` step like `stir`; needs a reachable
  `muddler` capability; records `BuildMethod` `'muddled'`.
  `shake` already existed (`stir.yaml` is `verbs: [stir, shake]`); `mix`
  is the resolve verb — nothing was renamed.

## ⭐ The glass pool, the technique, ice, garnish (libations D9/D10)

**A drink is made IN a real glass.** For `outputApplication: bulk` (the
bar's default) `craftImpl` no longer clones the output: `claimGlass`
takes the first reachable **clean, empty** vessel of the output's
**kind** (the gather walk already descends open room containers, so a
hospitality's `GlassRack` — an open `Container(Thing)` — is in the pool
scan; the kernel knows no rack class, only that the walk descends open
containers); none → the diegetic decline **`no-glass`** ("no clean
coupe").

⭐ **The rail rule: an unnamed pour takes the CHEAPEST liquid that clears
the recipe's `minGrade`; `with <brand>` overrides it.** That is what a bar
does, and it is what makes stocking a decision — your well sets the margin
on every drink nobody specified, which is most of them, while the good
bottle is not squandered on a guest who did not ask for it. The resolve
took the *highest* grade before, which poured the premium gin into every
anonymous G&T. Item inputs order the same way (the bruised lime goes in
the daiquiri).

⭐ **The match is the vessel kind (`category`, [bulk.md](./bulk.md)), not
the template path** — `outputVesselKind` reads the output row's own
`category`, and a row that declares none falls back to path-matching, the
behaviour before kinds existed. This is what makes a washed-out vessel
and a factory-fresh one the **same input to a fill**, which is what a
real line does. Path-matching meant a drained can of cola could never be
refilled — walked past in favour of one nobody had drunk from — so an
emptied vessel was economically dead the moment it was emptied, and the
returns loop was blocked at its first step. The claimed glass is marked
`soiled` at fill; `serve`/`order` hand over the claimed glass. `tangible`
outputs still clone (smithing's transform); **`edible` outputs claim from
the same pool** since the cooking build — see *Dinnerware is a pool* below. Breakage needs no mechanism: `throw` /
`StuffApi.destruct` remove a glass and the par sheet reports the
shortfall.

`CraftVessel` is now `Crafted(Thermal(Bulkable(Container(Detailed(
Thing)))))` — a `Container` so a garnish is a thing *in* the glass and
leaves with it (`Surfaced` was rejected: a resting item's container is
the room, so a handed-over glass would leave its olive behind); `Thermal`
so the temperature is real. Fields: `soiled` (`isSoiled()`; **`soil()`** is the public
one-way act anyone who USES a vessel may call, and `wash()` the only road
back — the raw `setSoiled` stays gated to `CraftingLogic` + the two
Hydrator arms precisely because it is the only other way to set it
`false`), `category` (the glassware par key — `coupe`,
`rocks`, …; `getCategory()`), `iceKg`, `iceForm`, `technique`. Glass
rows: `class: /platform/thing/CraftVessel`, `category`, `interiorBulk:
true`, `interiorCapacity`.

**Recipe** (`lib/craft/Recipe.ts`) gains `garnish?: { category, count?
}` and `ice?: 'cubes' | 'crushed' | 'none'`. The count measure is the
shipped `count` on `kind: item` slots (no new field); a dash is
`measureL: 0.001`.

### ⭐ Technique — an OPEN vocabulary the instrument owns

**The kernel keeps no technique list and no effect table.** A technique
is any kebab word a tool row names, and its numbers are that row's data:

```yaml
capabilities:
  - { kind: shaker,
      technique: { name: shaken, chillK: 8, dilutionL: 0.02, aerated: true, priority: 2 } }
```

⭐ **The tool is what knows.** A shaker is what makes a drink shaken, and
a shaker is what knows shaking chills ~8 K and folds in ~20 mL of
meltwater. A kernel table of `shaken | stirred | built | muddled` said,
wrongly, that those are facts about *crafting* rather than facts about
*bar tools* — so a cheese pack could not have `pressed` without a kernel
MR. Same smell, same fix, as the capability vocabulary above.

Resolution (`Techniques.fromTools`):

- ⚠ **The recipe's `toolCapabilities` filter the field.** Only an
  instrument offering a capability the recipe *required* may name the
  working — otherwise the furniture in the room would decide and one
  recipe would come out differently in two bars.
- **A recipe requiring no instrument is `built`.** A gin & tonic is built
  in the glass; a mixing glass on the well must not quietly stir it.
- Among the eligible, highest **`priority`** wins — *the finishing
  instrument names the drink* (a whiskey smash is muddled, then shaken,
  so shaker and mixing-glass outrank muddler).
- The **hand path** (`stir`/`shake`/`muddle`) names the working by verb
  but still takes its numbers from the instrument in reach — you cannot
  shake without a shaker. An unauthored word finishes neutral.

Hospitality authors the bar's four: shaker (shaken), mixing-glass
(stirred), muddler (muddled), bar-spoon (built).

At the fill (`applyBulkOutput`): the technique's chill + dilution;
**ice** — `crafting.iceKg` (0.15) of `ice` bulk moved from any reachable
`Bulkable` whose material carries the `ice` tag (the hospitality
`IceBin`; `insufficient-input: ice` if wanted and absent); **garnish** —
a reachable item matched by category, moved INTO the glass (the
`GarnishController` flourish now does the same at completion);
**carbonation** — a `BulkPayload.tags` union of the inputs' tags, so a
soda input reads "fizzing". Any Bulkable holder in reach is now an
input candidate (ungraded ⇒ `fair`) — that is how the ice bin and the
water tap feed a craft; a Crafted vessel is never descended into.

**Ice melts on read.** `CraftVessel.reconcileThermal` clamps the
temperature at water's `meltingPoint` while `iceKg > 0`; heat that would
raise it instead melts `ΔJ / latentHeatOfFusion` kg into the slot as
water (dilution — a real bulk credit), and Newton cooling resumes when
the ice is gone. Applied at the fill too: a room-temperature pour over
150 g of ice melts ~55 g at once (the honest arithmetic; tests assert
conservation). Nothing is scheduled. Presentation reads the stamps and
contents: "shaken, cloudy with air", "on the rocks", "fizzing", "with an
olive".

**House-made inputs.** Two rules the menu forced: a Crafted bulk vessel
holding a real (non-blend) material — a recipe with authored
`outputMaterial`, like pressed juice or syrup — is a bulk **source**, not
a pool glass; and such a recipe **tops up** a reachable bottle of the
same template already holding that material (one juice bottle a day,
not one per lime). `press-{lime,lemon,orange,grapefruit}` (tool `juicer`,
`kind: item` produce, `outputPortionL`) and cooking's
`simple-syrup` (sugar + water, `requiresHeatK: 340`, tool `pot`) are
ordinary `bulk` recipes over this.

The kernel's two generic materials moved to the **platform pack** —
`/platform/idea/material/blend` (`GENERIC_MIXED_MATERIAL`) and
`/platform/idea/material/cooked` (`GENERIC_COOKED_MATERIAL`) — because a
kernel module may not name a trade pack's row.

## The manual build (the by-hand path)

Alongside the one-shot served path, a drink can be **built by hand**, one
command at a time — the path the scripting build added (its manual-build
verbs are also the substrate the demonstration-capture records). Each step
is an **engaged activity** (`ManualBuildStep`, the `'hands'` slot, the
activity substrate's first durative-verb consumer — see
[activity.md](./activity.md)): the effect lands **at completion**, so a
barge-in `cancel` mid-step leaves partial matter standing.

- **`pour <spirit> into <vessel>`** / **`add <spirit>`** — debit a standard
  measure off the chosen reachable graded bottle (to the discard sink —
  conservation) and bank a graded `BuildContribution` into the vessel's
  buffer. `add` (no vessel) finds the build vessel you're working in.
- **`stir`** / **`shake <vessel>`** — record the mix method on the build.
- **`strain [<vessel>] into <glass>`** — the **terminal mint**.
- **`garnish <glass> with <garnish>`** — the finishing flourish.

The vessel is a `ManualBuildMixin` build vessel (`CocktailShaker` backs
both the shaker and the mixing glass — `capabilities` decide which recipe
tool it satisfies). At `strain`, **`CraftingApi.mintFromBuild`**
reverse-matches the accumulated contributions to a recipe (`matchBuild`:
heat gate ≤ the build's latched heat, bulk slots each covered by a
distinct contribution, item slots' counts covered by the tag rule, **no
leftovers** — a faithful build is exactly the recipe) and mints the
graded, maker's-marked drink into the glass, **reusing the one quality
model** (weakest-link `Grade`, the `applyBulkOutput` fill). An off-spec
build still yields *a* thing — the generic blend base named by its own
row (`mixed drink` / `cooked fare`), its payload still honestly summed
from what went in — but matches no recipe (`recipeId === ''`), the
discriminator the knowledge ladder rides. The maker is derived from
`getActingAuthor`, never a parameter.

### The by-hand paths, per branch (the uniform ladder shape)

The bar's shape — a build buffer, 3–4 engaged step verbs, a terminal
mint that reverse-matches — is the **universal learning route**; the
buffer grew `kind: 'item'` contributions (category + tags + count +
grade) and a max-latching **`heatedToK`** (`noteHeat` — the `heat` step
records the reachable heat; a recipe's `requiresHeatK` is checked
against it at the mint; reset by `clearBuild`).

- **Smithing** — the **workpiece IS the buffer** (`Ingot` composes
  `ManualBuildMixin`): `heat [<workpiece>]` (engaged; requires
  reachable heat > 0; latches the reached K at completion — re-read, so
  a fire that dies mid-step latches nothing) → `hammer [<workpiece>]`
  (engaged; requires a reachable `striking` tool + `anvil` capability +
  a workpiece that has taken heat; **banks the workpiece's own
  item-contribution once** via the substrate's idempotent
  `Builds.bankWorkpiece()` — the once-rule lives on the mixin, not in
  the verb, so no authored verb can double-bank; the hammer wears) →
  `quench [<workpiece>]` (the terminal
  mint — `mintFromBuild` with `workpiece`: a matched tangible recipe
  clones its output and the workpiece's Material + mass flow onto it,
  the workpiece consumed; an off-spec build mints a **generic worked
  lump** — a re-meltable `/stuff/thing/Casting`, `recipeId ''`, no mark).
- **Cooking** — the pot is the vessel (`CookPot` — `ManualBuild` +
  `Tool(pot)` + `Crafted`, itself a smithing recipe output): the
  existing `pour`/`add` verb grew a **discrete-ingredient branch** (a
  Material-bearing Tangible added to a build vessel is consumed at
  completion and banked as an item-contribution) → `stir` (unchanged)
  → `heat [<pot>]` (the same HeatController) → `plate [<pot>] into
  <dish>` (the terminal mint — a matched edible recipe fills the dish
  with the authored food Material at the portion; off-spec mints the
  generic **pot-luck** fill).

`quench`/`plate` replicate `strain`'s capture tail
(`ScriptApi.captureManualBuild`) — the first faithful hand build mints
the can-make **deed** + transcribes the personal recipe-script, the
same act.

## Drink → metabolism (honest alcohol)

The output glass is `Bulkable`, holding the recipe's authored cocktail
`Material` (`edibility: true` + `toxicity: [{type:'alcohol', amount}]`).
The existing `drink`/`sip` → `BulkableApi` → `MetabolicMixin.ingest` →
`getBAC` path is unchanged, so the drinker's BAC rises by how much they
drink. Per-instance ABV-from-spirit-choice is **deferred** — v1's
cocktail Material is fixed per recipe; the instance variable is the
*volume*. Quality stays a `Grade` verdict, never a number.

## Surface presentation: resting items aren't loose

The back-bar's bottles + tools sit **on** a `Surfaced` fixture (placed
via the bar's `props: { onto }` — see [spatial.md](./spatial.md)),
so they read "on the back-bar," reachable but **not loose room
clutter**. The shared rule is `ContainmentApi.looseContents(items)`: it
filters out any item whose `getRestingOn()` is itself in the set, and is
applied by `look`/`sense` (room branch) **and** the inspection card
(`Container.contents`). Examining the surface (`look back-bar` /
`sense back-bar`) reveals what rests on it via an "── On it:" drill-in
(`Surfaced.getResting()`). This is the discovery path that keeps the
stock out of the room view.

## Dave's Bar content (where it lives)

No bar-specific classes — content composes general mixins. Classes are
homed by what they *are*:

- **Building blocks** → `lib/`: `Surface` (`lib/spatial/`, a
  `SurfacedMixin` fixture), `ToolItem` (`lib/craft/`), `Crafter`
  (`lib/character/`, `MakerMixin(NPC)`), `NPC` (`lib/character/`, the
  minimal concrete `Character` — shares its path with the npc-behavior
  lane's richer `NPC`, which the add/add merge resolves to).
- **Commons** → `platform/thing/` (content packs wave 4b — composition-only
  classes are commons, not content): `CraftVessel`
  (`Crafted(Thermal(Bulkable(Container(Detailed(Thing)))))` since
  libations, `getLong()` appends the verdict), `Menu`, `GradedReceptacle`
  (`BrandedMixin(GradedMixin(BulkableMixin(Thing)))`) and over it
  `Bottle` (the stock vessel — [retail.md](./retail.md)), `Crate`,
  `CocktailShaker` (the build vessel), `NeonSign`, `TipJar`. ⭐ A class
  named for ONE trade's fixture ships in that trade's pack, not here —
  `GlassRack` is hospitality's, `CookPot` cooking's, `SpiritBottle`
  and `Still` distilling's. See [content-packs.md](./content-packs.md) § *Where a class lives*.
- **Singleton** → `obj/`: `RecipeCatalogue`.
- **Recipes** live where the trade that introduces them lives (content
  packs wave 4a/4b): `trade-smithing` ships the smithing five at
  `/trade/smithing/recipes/<id>`, `trade-cooking` its four
  (toasted-ration, root-mash, fine-roast, hearty-stew),
  `trade-hospitality` the bar's 21 cocktails + coffee and the four
  presses, each stub trade its serving recipe (`pint`, `glass-of-{red,
  white,sparkling}`, `soft-drink`, `simple-syrup`) — the 24-line menu of the
  libations slate; `generic-objects` ships none. The
  catalogue is **path-agnostic** — it rebuilds from every `recipe`
  document whoever installed it — so a venue's `craft` never knows
  which pack its recipe came from.

**Seeds:** the station templates are the hospitality trade's
(`trade-hospitality/content/trade/hospitality/thing/` — back-bar, well,
shaker, mixing-glass, muddler, bar-spoon, strainer, juicer, tap, ice-bin,
water-tap, basin, glass-rack, house-tablet, tip-jar, the nine pool
glasses + the juice bottle; `src/thing/{IceBin,Tap}.ts` — hospitality is
a capability pack since libations); the venue's own rows sit in
`saxonberg-lounge/content/world/lounge/thing/` (bar-menu, house-tablet,
tv, remote) and `agent/` (dave, mara). **The `Bar` populates NO bottle**
— the rail is stocked by Mara's `restocks` brain buying at the
distributor ([retail.md](./retail.md), [employment.md](./employment.md)).
Every libation `Material` lives with the trade whose PROCESS makes it
(`/trade/{distilling,brewing,winemaking,bottling,produce,cooking}/idea/material/`);
base-library keeps water, air, salt-water. `trade-hospitality/content/
archetypes/hospitality.yaml` is the venue archetype its own `menu.test.ts`
materializes a bar from (`archetype.materialize()`) to order all 24
lines. Crafted drinks are transient runtime matter (persisted nowhere;
reset on restart).

## Persistence story

| Kind | Form | Lifetime |
|---|---|---|
| Room / NPC / bottles / tools / Materials | **templates** (cloned) | re-seeded; cloned fresh |
| Recipe knowledge | **`documents {kind: recipe}`** (the `generic-objects` pack) | persisted reference data, pack-installed |
| Crafted drinks | transient runtime matter | reset on restart |

## The knowledge ladder, generalized (open canon, earned shorthand)

Recipes are open information — readable in-world, wiki-equivalent by
design. What is *earned* is the shorthand (`lib/script/RecipeKnowledge`
— chronicle-derived, see [scripting.md](./scripting.md) /
[chronicle.md](./chronicle.md)):

- **Claim (known-of)** — minted by *reading* a recipe source (the
  menu), **or by watching a maker perform**: `recordCraftEvidence` (the
  craft-resolve tail, both one-shot and by-hand mint) grants every
  other present command-giving agent with a durable identity the
  known-of claim, idempotently.
- **Deed (can-make)** — only your own first faithful by-hand
  performance (the reverse-match verified it; the same act transcribes
  the personal recipe-script). The book isn't enough — the hands learn.
- **The gates**: `make`/`forge`/`cook` decline without the deed
  ("work it by hand first"); `order` is never gated. A spoiler wiki
  yields exactly what an in-world recipe card yields — information buys
  optimization, never competence (the wiki-parity test:
  `knowledge-ladder.test.ts`).

**Advancement evidence** rides the same tail: a matched recipe that
authors a `discipline` appends a Transcript deed
(`creditDeed`) at its authored `difficulty` — the seeded
`smithing` (ISCED-F 0715) / `cooking` (1013) Disciplines, the
`mixology` shape. Control stays **fixed** this build (the skill seam is
the declared next crafting wave); the evidence simply accrues honestly,
and the BKT difficulty coupling already makes trivial-recipe grinding
worthless. The seeded rosters span a deliberate difficulty ladder per
branch (trivial → hard rungs in the `generic-objects` pack — the ZPD
obligation: the recipe tiers ARE the ladder a learner climbs).

## The lifecycle: two wear axes, repair, broken, salvage

**Keenness vs condition — two axes, two cadences** (collapsing them was
the named antipattern):

- **`condition`** (`DurableMixin`, `lib/material/`) — structural; decays
  slowly (combat: a landed weapon strike wears the weapon
  `crafting.wear.weaponPerStrike`; each covering layer that attenuates
  a mechanical blow wears `crafting.wear.armorPerBlow` — see
  [combat.md](./combat.md) / [materials-response.md](./materials-response.md));
  scales the response height as shipped; restored by **`repair`**.
- **`keenness`** (`KeenMixin`, `lib/material/Keen.ts` — `Mixins.Keen`,
  `MixinApi.isKeen`; composed on `Weapon`, blades only this build) —
  the working surface; decays fast (landed edge/point strikes `dull()`)
  and modestly scales edge/point *delivery*
  (`keennessDeliveryFactor()` = lerp from
  `crafting.keenness.deliveryFloor`); fully restored by **`sharpen`** —
  a carried-whetstone engaged ritual (interruptible, hands slot,
  `crafting.keenness.sharpenDurationMs`, the rasp emitted as an
  `Audible` so the room hears it; the stone wears). Afforded by
  *carrying* the whetstone — personal capital, works anywhere. Bands
  only (`keen/serviceable/dulled/blunted`) everywhere but `analyze`.
  Sharpen never touches condition; repair never touches keenness —
  independently observable via `analyze`.

**Broken is capability loss, not a state machine**:
`DurableMixin.isBroken()` = condition ≤ `crafting.brokenThreshold`; a
broken tool's `hasCapability` goes dark (it fails recipe tool-matching
until repaired); a broken weapon's delivery is clamped to
`crafting.brokenDeliveryFloor` (combat's bounded
`instrumentDeliveryScale` — the shared
`MaterialApi.gradeConditionScale` × the keenness factor, broken-floored,
folded into strike energy; material *height* stays analyze-only, a
deliberate combat-balance asymmetry).

**`repair <item>`** (`CraftingApi.repair` — deficit-priced
reverse-craft): cost mass = `item mass × (1 − condition) ×
crafting.repair.costFactor`, **doubled broken**
(`crafting.repair.brokenFactor`); domain-gated by matter — `metal`
wants reachable heat ≥ `crafting.repair.metalHeatK` (the forge),
soft goods a reachable `mending` tool (the sewing kit — the whetstone
is *sharpening's* tool, never repair's); stock drawn by the same
gather walk (a glob debits partially; a discrete donor is consumed
whole only when its mass ≤ 2× the need). Restores to full —
**ceiling-free** (the maintenance-relationship thesis: gear never
obsoletes, it asks for care; scarcity is the material cost, not decay
ratchets).

**`salvage <item>`** (`CraftingApi.salvage` — the one generic lossy
melt-down, the entropy sink): flatten the item's Material composition;
each constituent above the dust floor yields `mass × fraction ×
crafting.salvageRate` in its natural raw form — `metal` → a
re-meltable `/stuff/thing/Casting`, anything else → an `/stuff/thing/Scrap` stack (a
`GlobbableMixin(Thing)`, material-stamped, **quantity by mass** at
0.1 kg units). Conservation asserted (Σ output ≤ input × rate, throw
on breach); provenance, grade, and the chattel id die with the form
(the shipped destruct release). Salvaging the forged knife yields less
iron than the ingot that made it — lossless would break conservation;
losing the value-add makes it self-limiting.

## The venues (content over existing patterns, wholesale)

The Hearthworks smithy grew into a working venue (anvil + smith's
hammer + workbench + ingot/hide stock + a `Menu` + Berta), and a
**cookhouse** joined the zone (a 500 K clay hearth + `CookPot` + the
open pantry chest — `/platform/thing/Chest`, `Sealable + Container + Populates`,
the honest chest-pull — + a `Menu` + Odo). Both paths at each:
order it (served) or make it yourself with their tools (the DIY floor,
unpriced — the teaching venue). The general store sells the personal
kit: whetstone, iron ingots, sewing kit.

**The Business wiring is load-bearing, not decoration** (learned in
this build): `order` resolves its maker through the augment-gated
`MakerMixin`, so a venue with no rostered on-shift position has **no
active maker** — `hearthworks/content/world/hearthworks/idea/business.yaml` rosters the
smith + cook 24/7 with `confers: [MakerMixin]` (the Dave's-Bar pattern
verbatim; see [employment.md](./employment.md)). New graded-stock
form: `/platform/thing/Provision` (`GradedMixin(DetailedMixin(Thing))`) — the
discrete sibling of the graded bottle; a *fine* prime cut is what the
fine-roast's `minGrade: fine` slot demands (the grade spread on solid
stock).

## ⭐ The cooking branch: method, dinnerware, cutlery, palate

The cooking build (2026-09-03). Four things, and the first is the one
that decides the other three.

### The MEDIUM — why boiling cannot brown, and no table says so

`Recipe.medium` is `water` | `fat` | absent (dry), and it does exactly
two things:

1. the recipe must actually HAVE an input carrying the medium's **tag**
   (`water` ships on water; a cooking fat authors `fat`) — you cannot boil
   without water. Absence declines through the ordinary
   `insufficient-input` shape, with no new reason word;
2. the effective heat becomes `min(fire, the medium's phase ceiling)` —
   `Material.boilingPoint` for water, the new `Material.smokePoint` for a
   fat.

⭐ **`boil`, `simmer`, `poach`, `fry` and `roast` are not engine words and
never become any.** A wet recipe demanding 450 K declines
`insufficient-heat` at an 800 K hearth because the WATER stops at 373; the
same demand in tallow (smoking at 478 K) is simply cooked. The three-media
root-vegetable spine — the same two roots boiled at 373, roasted dry at
430, fried in fat at 440 — is three dishes out of one ingredient with no
method table anywhere. Syrup's elevated boiling point needs no special
case: it rides its own row.

An unknown medium word **throws at `Recipe.fromData`**: a typo must not
silently cook a stew over an uncapped fire. The by-hand `matchBuild`
clamps identically (`resolveMediumCaps` pre-resolves the banked media so
the reverse-match stays synchronous), so the same meat banked with water
comes out the 373 K stew even at 800 K, and banked without it comes out a
dry roast.

⚠ **A cook reaches for a fat that will take the heat.** The medium pick is
two passes: prefer a candidate whose ceiling clears the recipe, else take
the best there is so the gate still says `insufficient-heat` rather than
the lie `insufficient-input` over a full bottle. Three readings, all
correct — a fat that works, a fat that cannot, no fat at all.

⭐ **The working heat is not the room's heat.** The gate asks what the
setup can SUPPLY; what the food was held at is the recipe's own demand.
That second number is what the spoilage kill and the heat-labile toxin
kill read (see [spoilage.md](./spoilage.md)).

⚠⚠ A working that reaches the kill temperature starts the dish sterile
AND deposits the dose its inputs' spoilage had already earned as a formed
toxin. Cooking rotten meat gives you a dish that keeps and poisons you —
never a free lunch.

### Dinnerware is a POOL, not a mint

`Dish extends CraftVessel`. The two were parallel implementations of one
thing — claimed, filled, marked used, washed, claimed again — except that
Dish had none of it, so every plated meal CLONED a plate into the world
and nobody ever washed up. Bussing was real work at the bar and free in
the kitchen, for no reason but that the classes were written a month
apart. Dish's whole remaining delta is `NutritionLabelMixin`.

Three commons dinnerware kinds — `plate`, `bowl`, `platter` — and a
recipe's output row names which one its dish comes out in, so the method
shows up on the table.

⭐ **`CookPot` is a `CraftVessel` too**, which is what makes
*pot-as-last-resort* a claim rather than a special case: with no clean
crockery in reach, dinner lands in the pot it was cooked in and you eat
standing over the fire. Every strand is one a pot wants — Bulkable holds
the stew, Container holds what you dropped in, Thermal gives it a
temperature, soiled/wash because you wash a pot.

⚠ **The bar's `no-glass` stays HARD.** No clean coupe, no martini. Dinner
is not cancelled for want of a bowl; a round of drinks is. The asymmetry
is deliberate.

⚠ A pot full of dinner must not become stock. The generic cooked base is
excluded from the gather's intermediate test alongside the mixed one, so
a served bowl (and a full pot) is never a bulk SOURCE. Dish-as-ingredient
is out of scope for v1.

### Cutlery — it reads, it never gates

`lib/bulk/Utensil.ts` is a small enumerated kernel vocabulary (`spoon` ·
`fork` · `table-knife`, in preference order). `eat` claims the first clean
one in reach — held kit before the table — dirties it, and says so; with
nothing to hand you eat with your fingers. ⚠ **Same act, same nutrition,
same grade; only the sentence differs.** Food you can only eat with the
right implement is a lock dressed up as flavour, and this must never
become one.

A utensil is an ordinary `CraftVessel` with its interior slot never
filled — serviceware without contents — so it is claimed, soiled, washed
at the basin and counted on the par like any other vessel. The kernel has
to know the list because `eat` cannot ask "is this a spoon?" without one;
enumerating it makes adding a kind a diff a reviewer sees.

### The palate — taste is DERIVED, and it reads you

`taste <dish>` renders on the taste channel only, and what it says depends
on the taster's competence **in the discipline that made the thing**
(`BulkPayload.discipline`, recorded from the recipe) — so a cocktail reads
through the bartender's craft and a stew through the cook's, and the
kernel never knows a discipline word. A blend no recipe made records none
and reads at the floor:

| band | what you get |
|---|---|
| untrained / novice | the dominant basic tastes |
| competent | …and the ingredients, by name |
| proficient / expert | …and the grade of the making |

⭐ **No dish anywhere authors what it tastes like.** `BulkPayload` carries
`parts` (the ingredients, by their Materials' names) and `tastes` (the
union of their basic tastes), both derived at the craft; change what goes
in and the reading changes with nothing else edited. `Material.tastes` is
a CLOSED five-word set — sweet · salty · sour · bitter · umami, the
physiology's own list, not a flavour-note bank. A per-dish flavour string
is the retired per-dish-material anti-pattern wearing a different hat.

The competence read is the SYNC digest cache, and a cold cache reads as
the floor band — honest rather than defective: an unexercised palate IS a
novice palate. ⚠ Never a gate: every band tastes the food.

⚠⚠ **It lives on `PalatableMixin` (`lib/metabolism/Palatable.ts`),
composed on `ServingVessel`** — a vessel a made portion reaches a person
in. It took two wrong hosts to get there, and both were defended with a
sentence that was true:

1. **`BulkableMixin`** — which put a taste-palate augmenter on `Floor`
   (puddles), `GardenBed`, `PlantPot`, `AirTank` and `WateringCan`, made
   it re-derive "…but only a food vessel with contents" in four guard
   lines, and dragged `lib/advancement` into the bulk substrate.
   **Firing on hosts you then have to guard your way back out of is the
   tell that a mixin is on the wrong host.**
2. **`CraftVessel`** — "everything somebody made something in". Also
   true, and still too wide: that class is the wort bucket, the must
   bucket, the tallow crock, the **wash bucket** and the cutlery too, so
   a table knife and a bucket of dirty water read as things you taste.
   ⭐ **The second tell is quieter and worth learning:** the doc block
   listed its own hosts as "dishes, platters, the cook pot, the bar's
   glasses, the syrup and oil bottles" — a list already narrower than
   where the mixin composed, and still wrong at both ends. **When a
   comment has to enumerate a subset of its own hosts, the subset is the
   class.**

`ServingVessel extends CraftVessel`: the shared half is claim · soil ·
wash · par, which a wort bucket and a coupe both do. The palate is the
half only one of them has.

### ✅ The serviceware tier

```
CraftVessel        + VesselKind + Serviceable   a trade's working vessel
  │                                             wort/must/wash bucket,
  │                                             tallow crock, oil bottle
  │
  └─ ServingVessel + Palatable        a made portion reaches a person IN it
       │                              the 9 glasses, syrup + juice bottles
       ├─ Dish     + NutritionLabel
       └─ CookPot  (trade-cooking)

Cutlery            + Serviceable + CutleryMixin   what you eat WITH
                                                  spoon · fork · table-knife
```

⭐ **`ServiceableMixin` (`lib/craft/Serviceable.ts`) is the half a coupe
and a horn spoon genuinely share** — claimed clean, dirtied by use,
washed at the basin, counted on the house par. `CraftVessel` overrides
`wash()` and `isClaimable()` to add the dregs, the ice and the slot, so
**neither needs a guard**.

⚠⚠ **What it fixed.** The cutlery was a `CraftVessel` — its row argued
for it, correctly, on the claim/soil/wash/par behaviour — and paid three
prices: an interior bulk slot it never fills, an ice charge, and a
`wash()` that opened with *"⚠ … `getBulk` THROWS on a host that has no
such slot"*. **A method that throws on part of its own host set is the
host set being wrong.**

⭐⭐ **And the root cause was one field in the wrong place.** The utensil
kind was stored in `category`, which lived on `BulkableMixin` — so *"this
is a spoon"* required *"this is a bulk vessel"*, and `eat` found one by
asking `MixinApi.isBulkable` and then duck-typing `isClaimable`/`soil` off
a `Partial<{…}>`. `category` was right where it was: it is the VESSEL
kind, shared with vats and kegs, and a vat has one without being
serviceware. The cutlery was borrowing a vocabulary that was never about
it. `utensilKind` on `CutleryMixin` is the one that is, and `eat` now
narrows on `isCutlery` + `isServiceable`.


## Deferred (non-goals)

**Skill-as-control** — the declared **next crafting wave**
(`deriveAtFixedControl`'s `_control`: tightness / reachable envelope /
defect rate; every Transcript row from this build already counts when
it lands); **assembly** recipes; the **tailoring branch** (the jerkin
recipe + `mending` are its attach points; waits on a fiber source);
recipe-spread vectors beyond watching (taught curricula, discovery,
tradeable recipe-items); runtime affordance recompute on
`setCapabilities`/break-repair (surfaces refresh on the next
containment delta);
powered tool variants / the electric machine's supply gate (the
forge's `requiresHeatK` shape is the socket — the electricity consumer
build); per-capability wear differentiation + machine-vs-hand
advancement-evidence asymmetry; workshop lockers; batching
(`forge nails --count 5`); DIY stock-pricing; skill-scaled salvage
yield; working-surface maintenance beyond edges (seasoning/tuning —
`KeenMixin` is deliberately edge-only); environmental decay (rust/rot
at rest); glassmaking/brewing/alchemy (later branch consumers of the
same seams); corpos/brands at corporate scale; viewer-relative
appraisal + congener→hangover.


> ⭐ **A recipe's requirements and outputs carry a reveal level.** Inputs, tools, heat, difficulty, grade band and every output field. `name`/`keywords`/`discipline` stay open — a search index nobody can read indexes nothing. `spoiler: 1` with
> `spoilerName: 0`, so a reader sees which properties exist with the
> values collapsed — **one click, not a lock**, and a reader who set
> `wiki.spoilerAppetite` higher never sees a collapse at all. The rule
> the sweep settled on: *collapse what the WORLD measures, never what
> the PLAYER operates.* Levels are declared on the field and therefore
> apply wherever the value surfaces. See
> [wiki.md](./wiki.md) § the reveal model.

---

## Cross-references

- **Substrate consumed:** [templates](./templates.md),
  [persistence](./persistence.md), [race](./race.md) (Material),
  [bulk](./bulk.md), [metabolism](./metabolism.md),
  [spatial](./spatial.md) (Surfaced + `looseContents`),
  [provenance](./provenance.md) (the authorship ledger this instance-mark
  sits beside), [command-routing](./command-routing.md) /
  [command-spec](./command-spec.md), [mixins](./mixins.md),
  [call-security](./call-security.md), [watershed](./watershed.md) (`boil`
  — an **act**, not an outcome: what boiling *does* is decided by what you
  boiled, so a build latches `noteHeat` + the method `boiled` while a
  vessel whose contents declare `purifiedByBoiling` gets the swap. The
  method vocabulary is open, so a cooking pack gets boiling by authoring
  `{ requiresHeatK: 373, method: boiled }` and changing no kernel code).
- **Seeding slates:** `docs/slates/tails/crafting-slate.md`,
  `docs/slates/builds/daves-bar-slate.md`; parent
  `docs/slates/builds/economy-slate.md`.
- **Future builds grafting on:** corpos(marks) extends `CraftedMixin`;
  deconstruction reverses craft-resolve; the recipe-learning loop builds
  on the script-shaped verbs; npc-behavior wraps the serve-on-order
  reflex.
