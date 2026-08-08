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
- **`ToolCapability`** (`ToolCapability.ts`) — the capability vocabulary
  (`shaker`/`strainer`/`muddler`/`mixing-glass` + the branches'
  `striking`/`anvil`/`whetstone`/`mending`/`pot`) +
  `ToolCapabilities.isCapability`/`definitionOf`/`validateEntry` and the
  **capability table** (each kind's conferred verb family + placement —
  see § The offer) with the `CapabilitySpec` parameterized-entry shape
  (`rate`/`control`/`placement`, `RATE_MIN`/`RATE_MAX`). There is **no
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

- **`Recipe`** (`lib/craft/Recipe.ts`, `recipes` collection, unique
  index on `recipeId`) — typed accessors over `name`, `keywords`,
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
    `ThermalApi.reachableHeatFor(maker) ≥ requiresHeatK` — the fire
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
  `PostRegistrationMixin(Idea)` at `/obj/RecipeCatalogue`) — caches the
  collection, resolves by id + keyword (`order martini` → one recipe),
  `warm()` on `postRegister`. The `SoulCatalogue` shape. **Read methods
  are ungated** (the `TopicCatalogue` "public knowledge" precedent — a
  catalogued recipe is openly resolvable; v1 has no minting verb, so no
  `RecipeApi` and no dangling gate string).
- **`RecipeSeeder`** (`backend/RecipeSeeder.ts`) + `mud/config/recipes.yaml`
  — the `EmoteSeeder` precedent. Reads `recipes.yaml` (NOT under `seeds/`
  — recipes are Documents, not templates), idempotent by `recipeId`, runs
  in the seeder block before `BootstrapManager`.

## Craft-resolve: `CraftingApi` / `CraftingLogic`

The gated forwarding pair (the `ProvenanceApi`↔`ProvenanceLogic` shape):
`api/crafting.ts` is the thin gated shell (`SecurityApi.decorateApiClass`),
`obj/api/CraftingLogic.ts` is the `@internal` logic singleton at
`/obj/api/crafting`, methods gated
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
   `ThermalApi.reachableHeatFor(maker) < requiresHeatK` → decline
   `insufficient-heat` ("the forge is cold").
7. **Derive grade** — `Grade.deriveAtFixedControl(matched)` (bulk +
   item grades together), floored at `baseGrade`.
8. **Clone output** — `StuffApi.clone(outputTemplate)` → decline
   `no-output`.
9. **Apply output properties** — dispatched on `outputApplication`:
   - `'bulk'` / `'edible'` — fill the output's bulk slot at
     `Σ measureL` / `outputPortionL`. The substance is **derived by
     default**: the slot points at the ONE generic blend base
     (`cocktail/mixed` / `food/cooked`) and a per-instance
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
`menu`/`order`, any venue, and the venue subclasses
(`domain/lounge/Menu`, `SmithyMenu`, `KitchenMenu`) are empty
template-path anchors (seed `class:` refs + bar parity untouched). The
*working* verbs are **instrument-conferred through the capability
table** (`lib/craft/ToolCapability.ts`): each kind's definition names
the verb family it confers and its placement, and `ToolMixin` derives
its per-instance contributions from the table over the instance's
authored `capabilities` (the `InstanceContributor` seam — see
[command-routing.md](./command-routing.md)). The surface follows
capital, not venue flags (the menu is for ordering, not making; camp
cooking works because reachable heat + a pot IS a kitchen), and **a
tool variant is pure seed data** — the sewing machine is
`class: /lib/craft/ToolItem` + one spec entry, no code:

| kind | confers | placement |
|---|---|---|
| `shaker` / `mixing-glass` | pour, stir, strain, garnish, serve, mix | reachable |
| `pot` | pour, stir, heat, plate, cook | reachable |
| `anvil` | hammer, quench, forge, repair, salvage | reachable |
| `mending` | repair, salvage | reachable |
| `whetstone` | sharpen | **carried** (personal capital) |
| `striking` / `strainer` / `muddler` | — (recipe-side kinds) | — |

Placement is a kind default with a **per-entry override** (a 40 kg
grinding wheel authors `placement: reachable` on its `whetstone`
entry). `FurnaceMixin` still statically confers `heat` (with
`ignite`/`douse`/`pump`) — an appliance mixin, not a `Tooled` host —
and `make` is innate on `Avatar` (knowledge-driven, no instrument).
Capability entries are **parameterized**:
`{ kind, rate?, control?, placement? }` (a bare string = the defaulted
spec) — `rate` is a work-rate multiplier (clamped 0.25–10 at read)
that divides the engaged duration of the steps the kind confers (the
**conferring kind paces the step**: the anvil paces `hammer`/`quench`,
the `mending` instrument the now-engaged `repair`; the `striking`
hammer is a requirement, never a pacer), and `control` is a Grade band
embedded in the capital that **floors** the outcome grade of work done
with the instrument (craft, mint, repair — floor only; the ceiling
stays the skill seam's). `analyze` reads a control-bearing tool's
band. Class-carried behavior is orthogonal: `CookPot`/`CocktailShaker`
keep their classes for the build-vessel buffer, `Whetstone` for its
Audible rasp — none carry affordance statics. Affordance surfaces
refresh on the next containment delta (the documented
`InstanceContributor` limitation — a `setCapabilities` edit or a
break/repair doesn't re-push mid-placement).

`MenuController`/`OrderController` import the lib base, so `order` is
venue-generic: a smithy with a menu and an on-shift maker just works.

## Verbs (the `crafting` command category)

`mud/cmd/crafting/*.yaml` views + `mud/obj/command/crafting/*Controller.ts`.
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
- **`sharpen <blade>`** / **`repair <item>`** / **`salvage <item>`** —
  the maintenance acts (see the lifecycle below). Standalone diegetic
  verbs per the subcommand sieve; any future introspective growth
  (inspect/estimate/quote) grows as subcommands, not verb sprawl.

`serve`/`mix`/`forge`/`cook` are general agent verbs; `order` routes to
the present on-shift maker.

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
  lump** — a re-meltable `/obj/Casting`, `recipeId ''`, no mark).
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
via the bar's `populates: { onto }` — see [spatial.md](./spatial.md)),
so they read "on the back-bar," reachable but **not loose room
clutter**. The shared rule is `ContainmentApi.looseContents(items)`: it
filters out any item whose `getRestingOn()` is itself in the set, and is
applied by `look`/`sense` (room branch) **and** the inspection pane
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
- **Bar content** → `domain/lounge/`: `CraftedDrink`
  (`CraftedMixin(BulkableMixin(DetailedMixin(Thing)))`, `getLong()`
  appends the verdict), `Menu`, `GradedReceptacle`
  (`GradedMixin(BulkableMixin(Thing))`, the stock bottle).
- **Singleton** → `obj/`: `RecipeCatalogue`.

**Seeds:** instance seeds in `seeds/domain/lounge/` (back-bar, the four
bottles, shaker + mixing-glass, cocktail-glass, bar-menu, dave); the
`Bar` self-stocks via `populates:` (bottles/tools `onto` the back-bar,
then dave + menu). Cocktail/spirit `Material`s in `seeds/lib/material/`;
recipe knowledge in `config/recipes.yaml`. Crafted drinks are transient
runtime matter (persisted nowhere; reset on restart).

## Persistence story

| Kind | Form | Lifetime |
|---|---|---|
| Room / NPC / bottles / tools / Materials | **templates** (cloned) | re-seeded; cloned fresh |
| Recipe knowledge | **`Document`** (`recipes` collection) | persisted reference data |
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
(`AdvancementApi.recordDeed`) at its authored `difficulty` — the seeded
`smithing` (ISCED-F 0715) / `cooking` (1013) Disciplines, the
`mixology` shape. Control stays **fixed** this build (the skill seam is
the declared next crafting wave); the evidence simply accrues honestly,
and the BKT difficulty coupling already makes trivial-recipe grinding
worthless. The seeded rosters span a deliberate difficulty ladder per
branch (trivial → hard rungs in `config/recipes.yaml` — the ZPD
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
re-meltable `/obj/Casting`, anything else → an `/obj/Scrap` stack (a
`GlobbableMixin(Thing)`, material-stamped, **quantity by mass** at
0.1 kg units). Conservation asserted (Σ output ≤ input × rate, throw
on breach); provenance, grade, and the chattel id die with the form
(the shipped destruct release). Salvaging the forged knife yields less
iron than the ingot that made it — lossless would break conservation;
losing the value-add makes it self-limiting.

## The venues (content over existing patterns, wholesale)

The Hearthworks smithy grew into a working venue (anvil + smith's
hammer + workbench + ingot/hide stock + `SmithyMenu` + Berta), and a
**cookhouse** joined the zone (a 500 K clay hearth + `CookPot` + the
open pantry chest — `/obj/Chest`, `Sealable + Container + Populates`,
the honest chest-pull — + `KitchenMenu` + Odo). Both paths at each:
order it (served) or make it yourself with their tools (the DIY floor,
unpriced — the teaching venue). The general store sells the personal
kit: whetstone, iron ingots, sewing kit.

**The Business wiring is load-bearing, not decoration** (learned in
this build): `order` resolves its maker through the augment-gated
`MakerMixin`, so a venue with no rostered on-shift position has **no
active maker** — `seeds/domain/hearthworks/business.yaml` rosters the
smith + cook 24/7 with `confers: [MakerMixin]` (the Dave's-Bar pattern
verbatim; see [employment.md](./employment.md)). New graded-stock
form: `/obj/Provision` (`GradedMixin(DetailedMixin(Thing))`) — the
discrete sibling of the graded bottle; a *fine* prime cut is what the
fine-roast's `minGrade: fine` slot demands (the grade spread on solid
stock).

## Deferred (non-goals)

**Skill-as-control** — the declared **next crafting wave**
(`deriveAtFixedControl`'s `_control`: tightness / reachable envelope /
defect rate; every Transcript row from this build already counts when
it lands); **assembly** recipes; the **tailoring branch** (the jerkin
recipe + `mending` are its attach points; waits on a fiber source);
recipe-spread vectors beyond watching (taught curricula, discovery,
tradeable recipe-items); runtime affordance recompute on
`setCapabilities`/break-repair (surfaces refresh on the next
containment delta — the documented `InstanceContributor` limitation);
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
  [call-security](./call-security.md).
- **Seeding slates:** `docs/slates/tails/crafting-slate.md`,
  `docs/slates/builds/daves-bar-slate.md`; parent
  `docs/slates/builds/economy-slate.md`.
- **Future builds grafting on:** corpos(marks) extends `CraftedMixin`;
  deconstruction reverses craft-resolve; the recipe-learning loop builds
  on the script-shaped verbs; npc-behavior wraps the serve-on-order
  reflex.
