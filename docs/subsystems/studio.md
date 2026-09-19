# Studio — the mixin-aware composition surface

The **Studio** is the model-aware half of the CMS: where [cms.md](./cms.md)
is the dumb file/tree/save surface (three backends, one node-ref API), the
Studio *understands the content model* — it reads a backing class's
**effective mixin set** and renders a schema-driven form of exactly the
authorable fields, browses/names a catalogue of composition **blueprints**,
and scaffolds new backing classes. It is the first build of
cms-slate.md § *Composition & the blueprint
catalog* + authoring-intelligence-slate.md's
two catalogs (mixin particles + named blueprints).

The governing property: **the form is a projection over the same
`template.data` the raw-JSON editor already writes** (same save, same
go-live, same gates), and **authoring is free while only *publish* is
gated** — so composition is open to everyone without opening a
code-execution surface.


> ⚠⚠ **`blueprints` is a CACHE that persists, and is a deletion candidate**
> (collection audit, 2026-09-03). `BlueprintCatalogue.postRegister` calls
> `rebuild()` — the only `rebuild()` among 39 singletons, where every other
> catalogue `warm()`s — and `Blueprint.find()` appears **twice in the whole
> non-test tree, both inside that rebuild.** 147 rows survive a restart and
> are then discarded and re-derived by the only code that reads them.
> ⭐ It is **not** a duplicate of the `blueprint` document kind: the two
> hold disjoint populations by design (machine-minted `bp-<class-path>`
> skeletons vs hand-authored curated overlays; **zero id overlap** in both
> live worlds). ⚠ The seam to prove first is `rebuild()`'s read of the
> previous generation for *drift-safe on id* + the signature dedup.
> See [document-store.md § When a collection should be a document
> instead](./document-store.md).

## Field-schema derivation — the `@authorable` classification

A form needs, per field, its *type shape* and whether it is author-facing.
The engine had neither as machine-readable data (a mixin declares fields as
a bare name list). The Studio derives
both without a hand-authored parallel schema:

- **Classification** (`@authorable` vs `@runtimeState`) is an **inline TSDoc
  block tag** on every persistent/instruction field. Every mixin in the
  `Mixins` registry is audited — each field carries **exactly one** of the
  two tags (the `authorable-coverage.test.ts` guard is the tripwire; a
  new-but-unclassified field fails it). `@authorable` = authored initial
  configuration (names, descriptions, capacities, geometry, materials,
  behavior specs, initial states); `@runtimeState` = engine-written
  live/derived/session state (clock-stamps, caches, live vitals, per-player
  accumulators, the craft maker's-mark). A ref field carries
  `@authorable ref:<Type>` (the reference-picker signal).

- **The classification is read by a source-scan, not TypeDoc.** TypeDoc does
  **not** reflect a mixin factory's instance-field declarations — a mixin is
  an anonymous class-expression returned from a factory; only its companion
  *interface*'s methods reflect. So the emitted `authorable-fields.json` is
  *sparse*, and the authoritative classification is a fast regex scan of the
  `@authorable`/`@runtimeState`/`ref:` tags over the mud source tree
  (`StudioLogic.scanClassification`, cached on the singleton; the source is
  present in the deployed server). Instruction-only fields (no instance
  declaration — `container`, `exits`, `adornments`, `props`, `warren`)
  are classified on their **`applyX` applier** (the scan maps
  `applyWarren → warren`).

- **Type shape** comes from the TypeDoc `authorable-fields.json` artifact
  *where present* (the projector renders property field types, or the
  `applyX` **parameter** type for instruction fields — the data-payload
  shape, not the runtime field type; plus `enumValues` for string unions and
  `refShape`/`refType` for `ref:` tags). Because the artifact is sparse, the
  effective fallback is **runtime sample-value inference** (`typeof`,
  `instanceof Quantity`, array), then the raw-JSON widget. The projector
  (`scripts/project-author-surface.ts`, the `@hook` precedent) and its
  artifact are gitignored/regenerated — the Studio degrades gracefully when
  absent.

## `StudioApi` / `StudioLogic`

The gated Api + logic-singleton pair at `/platform/idea/api/studio` (the
`CmsApi`/`CmsLogic` twin): a thin `StudioApi` forwarding shell
(`SecurityApi.decorateApiClass`), every `StudioLogic` public method
`@CallSecurity(FromModule('/api/studio#StudioApi'))`. The acting principal
is always resolved from `ExecutionContextApi.getActingAuthor` — **never a
caller-supplied argument** (the anti-spoof rule); a null actor fails the
gate closed.

| Op | Gate | Purpose |
|---|---|---|
| `describeClass(classPath, contextPath?)` | none (a read; reads are ungated since wave 3) | effective mixin set + authorable field list joined to type shape + effective value/source |
| `describeMixin(name)` | none | one mixin for the composer's inspector card — its FULL multi-paragraph concept comment + the authorable fields it contributes (name + type shape) + its runtime-state field names (all from the source scan) + optional HelpApi enrichment (typed relations + conferred method names, degrading to empty when the help artifact is absent) |
| `createTemplate(input)` | the write gate at the target path (`CmsApi.write`'s rules; + the code-field gate on the `class` set) | **act #1** — save a NEW content template at a fresh path (CREATE-only; updates go through `CmsApi.write`) |
| `listBlueprints()` / `getBlueprint(id)` | none | browse the catalogue (ungated reads over the singleton) |
| `publishBlueprint(input)` | the document gate on the `/blueprints` mint branch (platform-held — a `SecurityError` from `DocumentApi.save` is the denial) | **act #2** — name/publish a composition of approved classes |
| `listMixins()` | none | the palette vocabulary — `{ mixins, bases }`: the flat pickable list (each mixin carrying a one-line `summary` — the first sentence of its concept doc comment, from the source scan; always available, degrades to absent when undocumented) + each base class with its implied (`_mixinName`) mixin set for composition pre-seeding |
| `scaffoldClass(input)` | none | **inert source string** composing mixins over a base (open to all authors) |
| `commitClass(input)` | `isWizard` + `can('write', zone)` | **act #3** — write the new class source + reload |

`createTemplate` is the server side of act #1 (the CMS content-write path only
*updates* an existing template — "creating templates is out of scope"). It is
author-tier to call; the wizard-lockdown code-field gate inside
`TemplateApi.saveTemplate` still applies to the `class` value being set and, on
refusal, is surfaced as a graceful `denied` disposition (never a 500). An
existing path is likewise a `denied`, not an overwrite. REST:
`POST /api/studio/template` (CSRF, `requireAuth`).

`describeClass` reads effective values through a live representative
instance (`findByTemplatePath(contextPath)` else the first
`findAllByTemplatePath(classPath)`), exercising the engine's own
`Zone.lookupField`→biome resolution (never reimplemented); absent an
instance it reads class defaults off a guarded throwaway
`StuffApi.create`. `valueSource` records `instance` / `resolution-chain` /
`class-default`. Field ownership uses **own statics only**
(`hasOwnProperty`) so an inherited an `instruction` entry in `fieldMeta` isn't
mis-attributed to a subclass mixin.

REST: `GET /api/studio/describe|blueprints|blueprint|mixins`,
`POST /api/studio/blueprint|scaffold|commit|template` — `requireAuth`, the CMS
`X-CMS-CSRF` double-submit on POSTs, each through
`CmsSession.runAsSessionPlayer` (the same attribution bridge as the CMS).

## The blueprint catalogue

A **blueprint** is a named, canonical *kind* you instantiate from — the
reusable molecule over the mixin particles. `Blueprint` is a reference-data
`Document` (`blueprints` collection, the `Recipe` precedent — never cloned),
keyed on two durable ids: a **`blueprintId`** (the record key, stable across
rename) and a **`signature`** (`<baseClass>|<sorted mixin _mixinNames>` —
the structural dedup key). `name` is a mutable display label, **never a
key**. `kind` is `composition` (a bare recomposable mixin stack) or
`concrete` (a logic-bearing class pointing at its `classPath`).

- **`BlueprintCatalogue`** (`/platform/idea/BlueprintCatalogue`) is the boot-warmed
  singleton (the `RecipeCatalogue` shape): `cache` by id + `bySignature`
  index, `canEvict`/`canDestruct` refusals. Since content-packs wave 2 it
  owns both layers itself (the former `BlueprintSeeder` is gone):
  **`rebuild()`** regenerates the **derived skeleton** at every boot
  (every distinct `class` — `TemplateApi.distinctClasses()` → `signatureOf`
  → a `concrete` row in `blueprints`, deduped on signature, drift-safe
  on id, orphans reaped with the `deleteMany` line logged once — *the
  `blueprints` collection is a cache*), and **`warm()`** indexes those
  rows plus the **curated overlay**: `documents {kind: 'blueprint'}`
  (the `platform` pack's `content/blueprints/<blueprintId>.yaml` — named,
  `blessed` compositions and hierarchy). A curated document whose
  signature matches a derived row **blesses it in place** (a cache
  write that fires once, on an un-blessed or drifted row); a
  pure-composition one is held in memory — the document is its source
  of truth, never a `blueprints` row. `invalidateCache()` re-warms (the
  installer's go-live).
  Both layers are idempotent **across mixin-signature drift**: a class whose
  *signature* changes (e.g. a new mixin composed onto a widely-used base)
  derives the same `blueprintId` but a new `signature`, which the
  signature-dedup passes yet the unique `blueprintId` index would collide on
  — so the seeder guards on the id too (the derived pass skips a drifted id;
  `#curatedOverlay` **reconciles the existing row in place** — re-points its
  signature/composition — rather than inserting a duplicate). Without this,
  adding `ConcealableMixin` to `Thing` (which shifted every Thing-based
  blueprint's signature) would throw `E11000` on a re-seed against a
  populated DB and take boot down.

`publishBlueprint` dedups on signature (reusing the existing `blueprintId`
on collision) and records an `AuthoringEvent` against a synthetic
`/platform/idea/BlueprintCatalogue/<id>` path — the naming act is attributed to its
author while the record stays commons-owned (the catalogue is a global
commons with no per-owner namespace).

## The trust model — three creation acts, save-as-disposition

Authoring (composing, scaffolding, editing a class in Monaco) is **inert
client text and open to everyone**; authorization gates **publish**. There
are three creation acts, and only one needs a wizard:

1. **Instantiate** a template (`class:` → an approved kind, own `data`) —
   author-tier, no wizard. The common "my own X". (Ordinary CMS template
   authoring; the Studio form is its friendly projection.)
2. **Publish a composition of approved classes** to the catalogue
   (`publishBlueprint`) — author-tier (a pointer to trusted code is not
   untrusted code).
3. **A new code module / novel composition** (`scaffoldClass` →
   `commitClass`) — **wizard-gated source write**, because a new backing
   class *is* a new `class` value and naming a class confers code-trust (the
   shipped wizard-lockdown, [access.md](./access.md)). This build is the
   sanctioned bridge across that content→code boundary, not a relaxation of
   it.

The composer's **base can be any approved class**, not only the 8 fundamental
roots: "Author a new kind from this →" on a concrete blueprint makes that
class the *superclass* (`class New extends <added>(Coin) {}`), with the
selected class's mixins shown as a read-only inherited segment and the added
stack starting empty (the scaffolder resolves the base by NAME — the last
segment of its `classPath` — via the export-source scan, which already
captures default-exported concrete classes like `Coin`/`PaymentCard`). The
structural signature still keys on the fundamental *root*, so a bare subclass
over `Coin` is correctly flagged as an exact match to `Coin` ("use it?").

**Save returns a disposition, not a boolean.** `commitClass` /
`publishBlueprint` return `committed` / `denied` (with `proposed` **reserved**
for the future review workflow). A non-wizard `commitClass` is a graceful
`denied` (the client's `auth.isWizard` **warning banner** warned before
save; the server gate is the authority). A wizard write that fails to
compile is `committed` + `reloaded: false` (persisted-but-not-live, the
shipped CMS behavior — never a 500). **Class-then-template ordering** is
client-enforced: the follow-on template step is disabled until a commit
returns `reloaded: true`.

There is **no runtime dynamic compose** — a backing class is always a static
TS module; `scaffoldClass` emits a source string (imports resolved by the
same source-scan that finds `_mixinName`/`export class`), and `commitClass`
writes it via `SourceTreeApi` + `HotReloadApi.reload` (the `_writeSource`
shape).

## Client surface

The **Studio** is the "Kinds" mode of the CMS surface (`?surface=cms`),
REST-only. The client IA is **catalogue-first**, split by audience — the two
tiers that authoring a class (wizard/code-tier, artifact = TS source) and
instantiating a template (everyone/data-tier, artifact = content data) are NOT
one form. `StudioPanel` is a **view router** (a breadcrumb strip reflecting the
active view; each view carries its own back/round-trip controls, owns ZERO
authorization semantics — the server re-gates every write and returns a
`disposition` surfaced inline) over four sequential surfaces:

- **`CatalogueView`** — the content author's home and the headliner. A local
  search box filters the blueprint catalogue by name / mixin; compact cards
  (name + composition/concrete kind badge + ★) are grouped by hierarchy
  (`parent`), blessed-first, with a chevron expanding the mixin lineage in
  place. Selecting a card routes to the detail view; a prominent "＋ Author a
  new kind" button routes to the composer.
- **`BlueprintDetailView`** — one blueprint's detail (name / description / kind
  / base class / expanded mixin list / classPath). The load-bearing round-trip
  is the PRIMARY action **"New template from this →"** (act #1 → the template
  form); a secondary **"Author a new kind from this →"** routes to the composer
  pre-seeded with this blueprint's base + mixins (act #3, the any-approved-class
  superclass path).
- **`TemplateForm`** — the ONLY data form (act #1). A header names the blueprint
  being instantiated + a target-path input; the body is a **Form/YAML toggle
  over the SAME merged data object**. The Form renders `describeClass` fields
  through the shared `widgets/` registry (lookup order `refShape → enumValues →
  typeShape prefix/exact → raw-JSON`: text / number / boolean / enum /
  `Quantity<…>` / the reference-picker, which browses in-scope content templates
  via `cmsClient.listTree`, degrading to a text input for catalogue-data ref
  types). Save calls `createTemplate` and surfaces the DISPOSITION inline
  (committed → offer "open in Files"; denied → the message, e.g. the
  wizard-lockdown refusal).
- **`ComposerView`** — the wizard, code-tier surface (act #3). A left column
  composes a base class (a `<select>` over the palette bases, or an
  any-approved-class base carried in from a blueprint) + an ordered,
  drag-reorderable mixin set (the base's implied mixins shown as a read-only
  "inherited from &lt;Base&gt;" segment); a live one-line preview mirrors the
  prototype chain. The right column is a **live-scaffolded** Monaco source
  editor (debounced ~300ms `scaffoldClass` on every composition change — never a
  data form) with the commit toolbar + inline disposition, docked above a
  **resizable mixin inspector card** (the `describeMixin` consumer: sticky,
  lazy-fetched-and-cached, renders the last-hovered/focused mixin's full concept
  comment + contributed fields + relations/methods, degrading gracefully when
  the help artifact is absent). A live **matching-blueprints** panel dedups the
  current composition against the catalogue: an EXACT match raises "⚠ this is
  already &lt;Name&gt; — use it?"; SUPERSET matches list "kinds that already
  include this", each routing back to its detail. The follow-on "create a
  template" is blocked until a commit returns `reloaded: true`.

The **data-integrity overlay** (the load-bearing correctness rule, shared by
`TemplateForm`'s Form and YAML views): the slice holds the full parsed
`template.data` as `baseData` plus an `edits` overlay and a `cleared` set.
`serializeStudioData` = `{...baseData, ...edits}` minus cleared,
`JSON.stringify(…, null, 2)` — **byte-identical to the raw editor for an
unedited round-trip**, and non-authorable / unsurfaced keys ride through
verbatim (never dropped, inherited defaults never materialized). Save reuses the
existing content-write/go-live path. The `WarningBanner` (from `auth.isWizard`)
warns a non-wizard before a commit the server will decline.

## Deferred seams (do not assume built)

- **Non-wizard draft persistence** — v1 keeps a scaffolded draft client-only;
  `scaffoldClass` reserves the `/home/<self>/drafts/<Name>.ts` path and a
  documented no-op `_persistDraft`. The future review workflow's `proposed`
  disposition writes into that seam ([document-store.md](./document-store.md)).
- **The git layer, then the non-wizard-proposes → wizard-rubber-stamps →
  catalogue review workflow** (the slate's law==code gate) — the `proposed`
  disposition, the stable scaffold path, and catalog-write-separate-from-
  source-commit are the accommodations; neither is built.
- **The alias auto-approve** (a non-wizard self-committing a trust-empty
  empty-subclass via a static no-new-code check) — deferred; template-clone
  covers the common owned-instance case.
- **Composition-rule metadata** (`@requires`/`@conflicts` per mixin) — not
  authored; v1 leans on the TS-compile/reload gate as the backstop.
- **Per-type content editors** (the bespoke room editor + its custom
  widgets) and the **zone editor / map canvas** →
  map-slate.md + cms-slate Wave 3. This
  build ships the *generic* composer + the reusable reference-picker only.
- **Engine-typed IntelliSense / LSP / host isolation** →
  authoring-intelligence-slate.md.

## See also

- [cms.md](./cms.md) — the file/tree/save surface the Studio extends (the
  CmsApi/CmsLogic node-ref contract, the attribution bridge, save go-live).
- [templates.md](./templates.md) — the Hydrator two-phase `setX`/`applyX`
  dispatch (property vs instruction fields).
- [mixins.md](./mixins.md) — the `Mixins` registry, `_mixinName`, `MixinApi`.
- [access.md](./access.md) — the wizard-lockdown (`class`/`hydratorClass`/
  `behaviors[].brain` code-naming gate), `isWizard`, `canAtPath`.
- [provenance.md](./provenance.md) — the authoring ledger, `getActingAuthor`,
  the `recordAuthoring` gate (broadened to accept the studio transport).

## History

Three design→implementation shifts are worth recording:

- **The composer IA was reworked at the end of the cycle** from a single
  "Compose" tab (one form doing both jobs) into the **catalogue-first,
  audience-split** router documented above: `CatalogueView` → `BlueprintDetailView`
  → (`TemplateForm` | `ComposerView`). The driver: authoring a backing class is a
  wizard/code-tier act whose artifact is TS source, while authoring a template is
  an everyone/data-tier act whose artifact is content data — they are not the same
  form, so the earlier `ClassPicker`/`MixinPalette`/single-`StudioForm` shape gave
  way to distinct surfaces, plus the **live-scaffolded** source output (the source
  regenerates as you compose rather than on an explicit "scaffold" click) and the
  dedicated **resizable mixin inspector** card.
- **The composer base is any approved class**, not only the 8 fundamental roots —
  "Author a new kind from this →" makes a concrete class (e.g. `Coin`) the
  *superclass*, resolved by name via the export-source scan. The structural
  signature still keys on the fundamental root so a bare subclass is flagged as an
  exact match to its base.
- **Classification is a source-scan, not TypeDoc.** TypeDoc cannot reflect a mixin
  factory's instance-field declarations (an anonymous class-expression from a
  factory), so the emitted `authorable-fields.json` is sparse and the authoritative
  `@authorable`/`@runtimeState`/`ref:` read is a fast regex scan over the mud source
  tree (present in the deployed server), with runtime sample-value inference as the
  type-shape fallback.
- **`BlueprintSeeder` (now `BlueprintCatalogue.rebuild()`/`warm()`) was made idempotent across signature drift** (the
  concealment build). Composing `ConcealableMixin` onto `Thing` shifted the
  structural `signature` of every Thing-based blueprint; the seeder's
  signature-dedup passed but the unique `blueprintId` index collided, so a
  re-seed against a populated DB threw `E11000` and killed boot. Fixed by
  guarding the derived pass on the `blueprintId` set and reconciling a
  drifted curated row's signature/composition **in place** in
  the curated overlay — the reconcile-by-`blueprintId` fix. A base-class mixin
  addition is now a safe, boot-surviving re-seed. `publishBlueprint` now
  writes the curated document at `/blueprints/<id>` through
  `DocumentApi.save` (one provenance row, keyed on the document path —
  the synthetic per-id path is gone) and `upsert`s the cache.

## ⭐ The Studio is a CARD

`studio` is a verb (the platform pack's `content/platform/cmd/author/studio.yaml`, `requiresWizard`)
opening a `client`-source card. The catalogue, the template form and the
class composer speak the Studio REST routes; the server owns the card's
existence, identity, lifetime and pinned-ness.

⚠ Pinned by default and `noProse`, for the same two reasons the content
editor is — see [cms.md](./cms.md).

⚠ **`studio` collides with the sandbox's `/studio/<collective>` scope**,
and that is two concepts sharing a word rather than one concept in two
places. The sandbox build's zero-new-verbs guard listed `studio.yaml`
as forbidden; it is exempted **by path** (`author/studio.yaml`), because
the guard says *the sandbox build ships no verbs*, not *the word is
reserved forever* — and the composer is the one this subsystem doc is
named after.

See [card-surface.md](./card-surface.md).
