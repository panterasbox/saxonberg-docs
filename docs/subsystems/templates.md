# Templates

Game-world objects in Saxonberg are not constructed directly. They are
**cloned** from templates stored in the MongoDB `domain` collection. The
clone pipeline owns construction, hydration, registration, zone resolution,
proxy wrapping, and a post-registration hook — the same plumbing every
game-world object goes through.

This is one of two persistence tracks. Auth/meta records (User,
GoogleProfile) extend `Document` and live as plain MongoDB documents;
those are covered in [persistence.md](./persistence.md). Templates cover
everything in the game world.

## The Template Class

Templates are modelled as a `Document` subclass — like `User` and
`GoogleProfile`, a Template is a record (plain persisted data, **not** a
Stuff), not a game-world entity. It is the data a game-world Stuff is
*cloned from*, never a live entity itself. `Template` itself is
**abstract**; concrete subclasses (`ZoneTemplate`, `LeafTemplate`) are
returned by `Template.findByPath` based on
`ZoneApi.isFolderClass(doc.class)` — a structural check
(`prototype instanceof Zone`) rather than a central allow-list. The
base lives at `lib/stuff/Template.ts`:

```typescript
abstract class Template extends Document {
  static collectionName = 'domain';
  static fieldMeta: FieldMeta = {
    path: { persistent: true },
    extends: { persistent: true },
    class: { persistent: true },
    data: { persistent: true },
  };

  path: string = '';
  extends?: string;                       // the parent's path, or absent

  readonly class: string = '';            // EFFECTIVE — the chain resolved
  readonly data: Record<string, unknown> = {};  // EFFECTIVE (merged)

  own: TemplateOwn;                       // RAW — what this row states
  chain: readonly string[];               // parent paths, nearest first

  public setOwn(spec: TemplateSpec): void;  // the author-side writer

  static findByPath(path: string): Promise<Template | null>;
  static findByPaths(paths: readonly string[]): Promise<Template[]>;
  static findDescendants(basePath: string): Promise<Template[]>;
  static loadById(id: string): Promise<Template | null>;
  static ancestorPaths(path: string): string[];
}

class ZoneTemplate extends Template {} // type-level marker; folders.
class LeafTemplate extends Template {} // type-level marker; leaves.
```

`findByPath`, `findDescendants`, and `loadById` all dispatch into the
right subclass — callers that hold a `Template` get back the correct
shape without needing to sniff `class`.

⭐ **`findByPath` does not go to the database.** The `content`
collection is held resident by `PersistenceManager` and a by-path read
is answered from memory, hit or miss — which is what makes the clone
pipeline's per-clone template load, and the zone walk's read per
ancestor path, free rather than a ~30 ms round trip each. The cache is
kept current at the same chokepoint every write passes through; see
[persistence.md § The resident `content` cache](./persistence.md). The
other three still query. The split is the primary
expression of the folder/leaf invariant; the `DomainHook` that fires
on save/delete is defense-in-depth at the persistence chokepoint.

`Document.findById<T>` is still available on the concrete subclasses
(`LeafTemplate.findById(id)` works); on the abstract base it's a
compile-time error because `new Template()` isn't legal — that's by
design and is the reason `loadById` is a separate method on
`Template`. `_materialize` constructs the chosen subclass with a plain
`new` (a Template is a `Document`, not a registered Stuff).

CRUD goes through the inherited `Document` surface
(`save`/`delete`/`findById`/`find`) plus the helpers above. See
[persistence.md](./persistence.md) for the `Document` contract.

- `path` is the canonical identifier — clones happen by path. Folder/leaf
  invariants (below) constrain paths.
- `class` names the runtime backing class to instantiate. Resolved by
  dynamic import; validated against an allow-list (below).
- `data` is what an author wrote, and **a row with `data` has it
  applied** — there is nothing to opt into. ⛔ A row used to name its own
  `hydratorClass`, and the field **retired 2026-10-01**: it had ONE value
  across 1,528 rows for the project's life, zero rows used the opt-out,
  and forgetting it silently discarded the row's entire `data` block. It
  expressed a choice nobody had ever made while making a whole failure
  class possible. See § The TemplateApplier.
- `data` never carries class paths itself.
- `extends` names a PARENT ROW. See the section below.

## ⭐⭐ Inheritance — `extends:` and the raw/effective split

A row may name one parent. The child states only what differs; every
value it does not state comes from the chain. The sentence the whole
mechanism exists for is *"like that one, but different"* — a can of
cola is a can, a dressed student is dressed like that student.

**The parent is an ORDINARY ROW.** There is no abstract-row concept, no
`kind: fragment`, no parent namespace. A parent is a row somebody could
clone, and mostly is one (an empty can, an empty crate).

⚠ **One parent is person-shaped and is nobody** —
`/stuff/agent/costume/student`, the mannequin the 45 dressed NPCs hang
off. It carries the three base garments, each with an `as`, and each
dressed row states only its variation: a field jacket, a blazer, a
tweed jacket, a white coat — or, for the two who are armoured, boots
`as: shoes`, which REPLACES the canvas ones rather than adding a second
pair. Six outfits from one authored bundle.

⚠ Clone the parent itself and a nameless, bodiless `Extra` stands in
the room.
That is a known and accepted rough edge, left open deliberately: the
honest fix is either an abstract-row concept or a narrower base class
to hang it on, and **the base-class build decides that across the whole
tree** rather than having it pre-decided here by one cohort.

### Resolved at READ time, never flattened

`Template._materialize` walks the chain and writes the **effective**
`class` and `data` onto the instance. So all sixty-odd
existing readers — the clone pipeline, every catalogue, zone
resolution, the designation gates — are correct unchanged, because what
they want is *what this row clones into*.

The raw row survives beside them as `own` + `extends` + `chain`, and
⭐⭐ **`toDocument` writes the raw row**, so a save can never flatten a
child into a copy of its parent. That holds for every writer that goes
through `Document.save()`: `saveTemplate`, `cp`, `mv`, the CMS,
`pack --export`. The effective fields are typed `readonly` so a stray
writer is a compile error rather than a silently forked row.

**Five surfaces read `own` rather than the effective fields**, and they
are exactly the authoring ones: `saveTemplate` + its code-field gate
(the delta baseline — a protowizard editing a class-less child must not
be refused for "changing" a class the row never stated), `cp`, `mv`,
the CMS read/write round-trip, and `cat` (which prints what the author
wrote, with inherited values annotated).

⚠ **A child that inherits its class is invisible to a raw Mongo query.**
`findByClass` and `findWhereDataHas` therefore union their raw hits
with the rows carrying `extends`, materialized and filtered on the
effective row. Without that union a catalogue warms quietly short —
the inert-roster failure, arriving from a new direction.

### The merge algebra is declared per FIELD

`FieldMetaEntry.inherit` (`lib/mixin.ts`), default `replace`:

| rule | meaning | declared on |
|---|---|---|
| `replace` | the child's value wins whole; absent → the parent's | everything not below |
| `by-key` | an object merged key by key, child wins per key | `Detailed.details` |
| `by-entry` | a list merged by ENTRY IDENTITY | `props`, `cast`, `costume`, `adornments` |
| `never` | the parent's value is not copied at all | `exits`, tpa `routes`, every `Biome` field |

The field's **owner** declares the rule, so a pack's field and biome's
fields say their own with no edit to `Template`, which holds no list of
field names.

⭐⭐ **Entry identity (`as`) is the PRECONDITION for `by-entry`, not a
nicety.** Append and replace are each right about half the time, and
the wrong one fails silently — a doubled jacket, a missing pair of
shoes. `as` makes the merge rule definable at all: an entry's key is
`as` when stated, else `template`, else the bare string. The merged
list is the parent's entries **in order**, one the child names
substituted **in place** (further parent duplicates of that key
dropped), then the child's new keys appended.

⚠ Parent duplicates the child does *not* name are **preserved** — six
same-path stool lines stay six until somebody writes `count`.

### A parent's class need not be the child's

Cross-class parenting is legal and used: every dressed `Cast` row
extends a row whose class is `Extra`, which is an honest authoring
gesture (*these people are dressed like that one* says nothing about
what kind of person they are). Requiring class-compatibility would
forbid the build's best exemplar and is unenforceable at the edge,
since a parent may state no class at all.

⚠ **What actually goes wrong is narrower**: the applier discards a data
key the effective class does not declare, and under
inheritance one junk key reaches every descendant instead of one row.
That is what `check-instanceable-placement` invariant 12 censuses and
ratchets — see lint-family.md.

⚠ **The tree's first ABSTRACT parent is a known rough edge.**
`/stuff/agent/costume/student` is a costume bundle, not a thing: clone
it and a nameless, bodiless `Extra` stands in the room wearing a
student outfit. Inheritance was designed on the rule *a parent is an
ordinary row*, from two exemplars that were objects a player can hold
(an empty can, an empty crate); a bundle is not one. Accepted
deliberately, because the honest fix is either an abstract-row concept
or a narrower base class to hang it on, and that is decided across the
whole tree by
base-class-narrowing-slate,
not pre-decided by one cohort.

### The chain fails loudly

A missing parent, a cycle, or a chain deeper than 32 throws naming the
child and the parent — at the authoring door (`saveTemplate`), at pack
reconcile, and again at materialize. A silent parentless materialize
would hand the clone pipeline a class-less row and blame the wrong
thing.

**A row that is extended cannot be deleted.** The refusal fires in
`validateFolderLeafDelete` at the persistence chokepoint, so it holds
for `rm`, `mv`, the CMS and `pack sync` alike; the pack installer
additionally plans a `deleted-vs-extended` conflict rather than
blocking the boot.

### Zone field lookup is a different mechanism

Both answer "where does this value come from", and they are not the
same question. A **zone field** (`Zone.lookupField`) answers *what is
true everywhere inside here*, at READ time, walking the path tree. A
**parent** answers *what this thing is like*, at CLONE time, walking
the `extends` chain. When both would supply a value the clone-time one
is already on the instance and the zone walk is never consulted. See
[zone.md](./zone.md).


## Class Path Validation

`StuffApi.#validateClassPath(classPath)` gates every dynamic import:

- Must start with `/`
- Must NOT contain `..` (no directory traversal)
- Must start with one of the allowed prefixes: `/platform/… + /stuff/` or `/lib/`

Class names are the last path segment (e.g., `/platform/agent/Avatar` →
`Avatar`); the import succeeds only if a named export with that name
exists in the resolved module. Anything else throws.

This validation is **format-only** (shape of the path). The orthogonal
**trust** question — *may this author name this code at all?* — is
enforced separately at the `saveTemplate` chokepoint: a non-wizard
(protowizard) author cannot introduce or change the executable
code-naming fields (`class` / `behaviors[].brain`). ⭐ `hydratorClass`
was the third until 2026-10-01, and it left the gate WITH the field
rather than by exemption: a row cannot name an applier, so there is
nothing to refuse and nothing carved out.
See [access.md § The code-trust lockdown](./access.md).

## The Clone Pipeline

`StuffApi.clone<T>(templatePath, context?): Promise<T>` runs:

1. **Load template** by path from `Collections.Domain`. Throws if missing.
2. **Validate `class` path**, dynamic-import the module, fish out the
   constructor by name.
3. **Resolve zone** via `ZoneApi.resolveZoneForPath(templatePath)` — walks
   ancestor paths nearest-first, returns the Zone clone of the first
   ancestor whose template names a Zone class. Returns `null` when the
   template is itself a Zone, or when no ancestor is a Zone. Stamped onto
   the instance before hydrate so anything that reads `this.zone` during
   hydrate sees the right value.
4. **Resolve the applier** — iff there is anything to apply. The row no
   longer names one: the test is the MERGED data (the row's, with any
   `dataOverlay` over it), because five production callers pass an
   overlay onto rows whose own `data` is empty. `TemplateApplier` is
   itself a templated `Idea`, stateless by contract, so `clone` resolves
   it via `StuffApi.singleton` — one cached instance reused across every
   backing. ⭐ Recursion terminates STRUCTURALLY: the applier's own row
   carries `data: {}`, so cloning the applier plans no applier.
   `clone`'s in-flight-path guard stays as the backstop and still
   surfaces `circular template dependency`. See
   [hot-reload.md § The applier](./hot-reload.md#hydrators).
5. **Construct** the backing under the construction sentinel:
   ```typescript
   Stuff._beginConstruction();
   try { obj = new ClassConstructor(); }
   finally { Stuff._endConstruction(); }
   ```
   The sentinel is a runtime gate — every Stuff constructor throws unless
   the sentinel is set. This is what enforces "only StuffApi may
   construct Stuff." See [lifecycle.md](./lifecycle.md) for details.
6. **Stamp `templatePath`** onto the instance so identity-keyed security
   policies (`FromTemplate`) can match against it.
7. **Wrap in a Proxy** via `ProxyApi.wrap(raw)`. Every consumer that
   resolves the object by `stuffId` thereafter sees the proxy, not the
   raw instance — the security gate is then in the call path for those
   callers.
8. **Register** the proxy in `StuffApi`'s `objectsById` map.
9. **Apply the row**: when there is data, run
   `await applier.apply(proxy, data, { mode: 'mint' })` inside a
   synthetic constructor frame (`ExecutionContextApi.run` with
   `FrameKind.Constructor`).
10. **Hydrate from every declared source**: for each
    `PersistenceContributor` of the host's class that declares a
    `hydrationSource`, run it — **whether or not the host has a
    record**. ⭐ There is no deferred option: hydration is an
    initialization step with a terminus, and fetching the same data on a
    live object is a different mandate with its own lifecycle. See
    [persistence.md § Per-mixin composition](./persistence.md).
11. **`onCreate`**: `await proxy.onCreate(context)`, same synthetic
    frame, **unconditionally** — the hook is a terminal no-op on `Stuff`
    (it was `onCreate` on an opt-in `onCreate` until
    2026-10-01). See [lifecycle.md](./lifecycle.md).

If any of those throws, the object is unregistered before the error
propagates. Half-initialised objects never linger in the registry.

Order is load-bearing, in two ways. **Register fires before the content
step** so that anything resolving the in-flight object by `stuffId`
during it (a self-referencing exit row) finds it. ⭐ And **every eager
hydration source completes before `onCreate` begins**, so a hook may
rely on remembered state being present instead of reading a collection
itself.

## ⭐⭐ `unreachable:` — the fourth top-level key, read by a LINT and nothing else

A sibling of `class:` / `extends:` / `data:` on a row file, and the one
key here that the runtime never sees.

It answers one question: **why can no player ever meet this row?**
`lint:reachability`'s arm R requires every `thing`-branch row to be
reached by one of five mechanisms or to carry this key. Closed
vocabulary:

| value | means |
|---|---|
| `exemplar` | a substrate exemplar a subsystem doc cites and tests build; not meant to stand in the world (the coffee urn's infinite source, the bag of holding, the braille slate) |
| `parent` | a base row that exists to be `extends:`-ed and which nothing extends YET. ⚠ A parent WITH a reachable child is detected automatically (arm R rule 4), so this value is only for the not-yet case |
| `awaiting:<slate-basename>` | held against a named slate, whose file must exist under `docs/slates/**`. A **stranded** reason — one naming no slate — is a finding |

⭐ **The installer never sees it.** `PackLogic` reads `class`, `extends`
and `data` off a row file and nothing else, and hashes
`{class, extends, data}` — so the key does not reach Mongo, does not
change a row's content hash, and cannot affect hydration.

⚠ A `data:`-level carrier was considered and rejected on a stronger
reason than *it would be persisted*: the `TemplateApplier` iterates the
host's DECLARED fields and lights a `reportUnapplied` diagnostic for
anything left over, so `data.unreachable` would raise an author-visible
diagnostic at the `errors` verb **on every clone of the row.** Same
conclusion, better reason — it is a top-level key.

⚠⚠ The honest objection — *a key only a script reads* — is accepted and
is the point. It is an authoring declaration the same way a `# comment`
was, except greppable, vocabulary-checked and gated. A comment saying
*"nothing places this on purpose"* was indistinguishable from an
oversight, and that is how fifteen command views and forty-five rows came
to be unreachable without anybody noticing.

⚠ On a **command view** the same key is a declared property of
`command.schema.json` instead, because a view's schema is closed
(`additionalProperties: false`) and an undeclared key throws at load. See
[command-spec.md](./command-spec.md).

## ⭐ The entry shape: `as`, `count`, `onto`

A `props:` entry is a bare path or an object:

```yaml
props:
  - /trade/hospitality/thing/back-bar
  - { template: …/house-tablet, as: tablet, onto: …/back-bar }
  - { template: /trade/farming/thing/lime, count: 12 }
```

- **`as`** — the entry's IDENTITY, on all four designation lists
  (`props`, `cast`, `costume`, `adornments`). It is what a child row
  replaces; see the merge rules above. Two entries sharing one `as`
  throw at hydrate: an identity with two claimants has no answer.
- **`count: N`** — **props only**; mints N clones from one line. Twelve
  identical lime lines and six identical stool lines were one authoring
  gesture written longhand. `count` on `cast:` throws (twelve of a
  person is twelve people, each of whom needs a name), as does `count`
  on a `Singleton` class, and any count that is not a whole number ≥ 1.
- **a placement key** — ⭐ the key IS the member's name (`on`, `from`,
  and whatever a pack ships). Its value names a `Placing` entry EARLIER
  in the list, by its `as`
  or by its path. Under `count`, the last clone is what a later `onto`
  finds.

## ⭐ Instruction appliers run ONCE — `props`/`cast` are initial furnishing

⚠ **Added 2026-08-30** (the libations review). `TemplateApi.restoreFromTemplate`
— the **CMS save go-live** (`CmsLogic`) and the **pack reconcile go-live**
(`PackLogic`) — re-runs the *full* `hydrate`, which re-dispatches every
Phase-2 instruction applier. For a *value* field that is harmless: the
field is re-assigned to the authored value, which is the point. For an
applier that **mints objects** it is a faucet: before this,
publishing an edit to a `props:` row minted a fresh set into every
live instance — every crate in the world gaining six more grapefruits,
every non-singleton fixture in a plain room duplicated.

`StagedMixin` therefore records `_propsStaged`/`_castStaged`,
and `CostumedMixin` records `_costumeWorn` — **one once-flag per list**:
`props:` the write-back set dressing, `cast:` the conserved troupe,
`costume:` what a cast member wears (see persistence.md § Props and
cast). Each applier no-ops once its flag is set.
Singletons were already safe (the applier skips one already placed);
this covers the plain-clone branch. `PersistableMixin` hosts (every
`FurnishableRoom`) were never exposed — that override only *retains*
the specs and lets the establishing context seed them once via
`seedBornWith` — and the guard now makes that idempotent too.

⚠ **Deliberately not count-aware.** "Top up to the declared list" was
rejected: the same mechanism serves a room's fixtures, which are meant
to be permanent, and a crate's contents, which are meant to be
**consumed**. "Declared minus present" would resurrect goods somebody
drank, ate or sold — the faucet the libations build spent its whole
length removing from the bar. An author who edits the list and wants it
applied **re-clones**; there are no migrations.

**The general rule for a new instruction field: if the applier creates
Stuff, it must be idempotent, because go-live will call it again.**

## ⭐⭐ The TemplateApplier

`TemplateApplier` (`platform/idea/TemplateApplier.ts`, row
`/platform/idea/TemplateApplier`) is **the content step: what the ROW
says, put onto the instance.**

```typescript
apply(host: Stuff, data: Record<string, unknown>,
      opts: { mode: 'mint' | 'go-live' | 'restore' }): Promise<void>
```

⭐⭐ **It was called `PersistentHydrator` until 2026-10-01, and the
rename is a distinction rather than a tidy-up.** *Hydration* now means
filling an instance from what the world REMEMBERED about it
(`holder_snapshots`, `beliefs`) — keyed on the instance's own identity,
with a capture counterpart. This class does the opposite job: it applies
what an AUTHOR wrote, keyed on a template path, with no capture side at
all. Calling both "hydration" is what made the first cut of the hydration
build try to unify five things that were alike with one that was not.
The `Hydrator` interface is gone with it: there was one implementer, and
a row can no longer select one.

It is stateless — one instance fills many backings, introspecting each
one's `fieldMeta` rather than mirror-composing its mixin chain.

### The three phases

Each runs to completion before the next, so every property has settled
before an instruction reads one and both have settled before anything is
written to a ledger.

- **Phase 1 — property fields.** For each entry in
  `MixinApi.getAllPersistentFields(host.constructor)`, prefer
  `await target.set<PascalCase(field)>(value)` when that method exists,
  else `target[field] = value`. The async-first dispatch lets a setter
  with side effects (`CartesianLocation.setCoords` registers with the
  zone) finish before the next field. The bracket-assign fallback still
  fires an accessor pair declared on the prototype, so a shape invariant
  on a `set` accessor continues to gate the write.
- **Phase 2 — instruction fields.** `await target.apply<Field>(value)`.
  The applier is **required** — an `applyX` that does not exist is a
  configuration bug surfaced loudly, never a silent skip.
- **Phase 3 — seed fields.** `await target.seed<Field>(value)`, also
  required. An authored HISTORY being written into the ledger that owns
  it: a `Cast`'s prologue into the chronicle, `dispositions` into the
  trait log, claims into renown and the transcript. Last, and mint-only.

A seed field is usually ALSO a property field: phase 1 keeps the authored
value on the instance (so `getRenownClaims()` can read it back) and phase
3 hands the same value to the ledger. `seed: true` adds phase 3, it does
not replace phase 1.

### The three modes

| mode | caller | 1 | 2 | 3 |
|---|---|---|---|---|
| `mint` | `StuffApi.clone` | all | ✓ | ✓ |
| `go-live` | `TemplateApi.restoreFromTemplate` (a CMS save, `pack sync`) | all but `birthOnly` | ✓ | — |
| `restore` | `PersistableApi.materialize` | all, `birthOnly` included | — | — |

⚠⚠ **`go-live` skipping `birthOnly` is a money fix, not a nicety.** A CMS
save or a pack reconcile re-applies a row's authored fields to every LIVE
instance at that path. `Coin` authors `quantity: 1`, so going live on the
coin row reset every coin stack in the world to one — minting and burning
outside the conservation chokepoint, invisibly, which is the one thing
that chokepoint exists to make impossible. `Stackable.quantity` declares
`birthOnly: true`, because the hazard belongs to the FIELD wherever it is
authored: a scrip's quantity and a crate of limes' carry the same one, and
a row-level switch would have to be remembered on every row that authors a
stack.

⚠ **`restore` is not `go-live`.** A record replays what this instance
actually had, so a birth-only field is exactly what it must write back;
conflating the two would empty every logged-out player's purse. Phases 2
and 3 are skipped by SELECTION rather than by special case — an
instruction or seed field never appears in a captured field slice.

### Once-ness: the applier owns WHEN, the ledger owns WHETHER

Phase 3 runs at mint only. But a re-clone after a destruct IS a new mint,
and only the ledger knows the history is already written — so each
`seed<Field>` keeps its own guard (`Cast.seedPrologue` skips if any
`claim` row exists; `RenownApi.seedTo` counts the evidence already on the
log and writes the difference). ⚠ A once-flag on the applier would be the
obvious simplification and would be wrong: it cannot tell *the same
object being re-filled* from *a new object at the same path*.

### ⛔ An unapplied key is reported now, not discarded in silence

Both the property and instruction loops `continue` on a key the class does
not declare, so an author's line simply had no effect — no throw, no
warning, nothing in a log. The bill, all of it found by DRIVING:
`material:` instead of `_materialPath:` on **49 rows across eight packs**
(a sack of wheat that was "not grain" at the mill); `name:` and
`description:` on the bar; `primaryKeyword` on a family of rooms; a
zone's whole `deposit:` orebody, so `hew` refused in a room with a seam
visibly in the face.

At mint the applier now writes one `DiagnosticApi` record per
`(templatePath, key-set)` per process, readable at the `errors` verb.
⚠ A **warning**, not a throw: a catalogue class that parses its own row's
`data` directly is a legitimate authoring act and ~161 rows do it.

### Reading what will fill a row in

`TemplateApi.describeFill(spec)` answers it in three lists — the keys the
applier **applies** (with phase, and whether the field is birth-only),
the keys nobody **applies**, and what an instance will also **remember**
from a declared source. Four surfaces print it (`cat`, `write`, the
Studio's create disposition, the CMS's `templateMeta.fill`) from that one
method. ⭐ "Nothing" is printed out loud rather than omitted, because an
absent line is indistinguishable from a surface that forgot to write one.

### Where a cross-field rule goes

On the class, not in a subclass of the applier. A per-field shape rule
belongs on the field's setter (the applier routes through it for free); a
cross-field invariant ("if `isLocked`, `lockKey` must reference a real
key") belongs in the host's own `set<Field>` / `apply<Field>`. ⚠ There is
no second applier to subclass: a row cannot name one and the engine
resolves exactly this one.

**Bracket-assign (the Phase 1 fallback) IS still part of the contract
surface.** It invokes accessor pairs when present. So if a field has
a shape invariant ("must be boolean", "lowercase / trim / dedupe"),
put the rule on the setter accessor and hydration routes through it
for free even when no `setX` method exists. Don't add
`normalize()`-style post-hydrate fixups for per-field rules — see
[antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks).

Cross-field invariants ("if `isLocked` is true, `lockKey` must reference
a real key") can't live on a single setter — that's the legitimate use
case for a custom `Hydrator` subclass. Override `hydrate()`; call
`super.hydrate()` first if you want the default two-phase dispatch
too.

### Property vs instruction fields

Two distinct field shapes ride on the Hydrator's two-phase dispatch
(per `feedback_property_vs_instruction_fields`):

- **Property fields** — declared `{ persistent: true }` in `fieldMeta`.
  Data IS the field's value: `setX(v); assert(getX() === v)`.
  Symmetric on shape. Side effects on the setter (e.g., `setCoords`
  registers with the zone) don't change the field's identity. Shape:
  optional backing slot `_x`, optional accessor pair
  `get x / set x` for shape invariants, public methods
  `getX()` / `setX()` for the inter-Stuff contract. Hydrator
  dispatch: Phase 1 — `setX` if defined, else bracket-assign.
- **Instruction fields** — declared `{ instruction: true }` in `fieldMeta`
  (new sibling to `fieldMeta`'s persistent entries). The data is a **recipe**
  consumed to produce or modify *separately-named* runtime state —
  no "value" to set/get on the spec, only a verb's argument. The
  canonical example is `exits` on `ExitableMixin`: the YAML data is
  a `Record<string, ExitInstruction>` recipe, `applyExits` consumes it to
  populate the runtime `exits: Map<string, Exit>` collection (which
  has its own established `getExit` / `addExit` / `removeExit`
  surface). No paired getter for the spec. Hydrator dispatch:
  Phase 2 — `applyX` required.

  Other appliers shipped by the substrate:
  - `applyContainer` on `ContainableMixin` — resolves a templatePath
    via `StuffApi.singleton` and moves self into it via
    `ContainmentApi.move`. Compare-and-move idempotency: no-op when
    the current container's templatePath matches the declared path.
    Target must be singleton-shaped (validated at template-save
    time by `TemplateApi.validateSingletonContainerTarget`).
  - `applyProps` on `StagedMixin` — iterates a list of
    entries, each either a bare templatePath or a
    `{template, onto}` object. Dispatches per-entry by
    source-template singleton-shape: singletons resolved via
    `StuffApi.singleton` (skip-when-already-elsewhere);
    non-singletons cloned via `StuffApi.clone`. A bare entry is
    moved into self (`ContainmentApi.move`); an `{template, onto}`
    entry is placed on an already-populated sibling surface
    (`ContainmentApi.place`), keyed by the placement key's source path —
    so the surface fixture must be listed before its resting
    items (the back-bar before its bottles). See
    [crafting.md](./crafting.md).

The two shapes split apart cleanly: properties have symmetric shape
and storage IS the value; instructions are commands whose outcome
lives elsewhere. Trying to use `set/get` for instructions produces
shape asymmetry (setter takes specs, getter returns runtime state)
— "marshaller work in the wrong place." Recognizing them as separate
concepts dissolves that.

The constant `TemplateApplier.templatePath` is the single source of truth
for the applier's row path. ⚠ No call site passes it to `saveTemplate` any
more — the engine resolves the applier itself — so it is read by the
pipeline, the restore paths, the boundary exemption and the three
call-security gates, and by nothing in content.

## The Context Bag

`StuffApi.clone(path, context?)` accepts an opaque `context` bag that
gets threaded through to `onCreate`. It exists to carry runtime
setup that cannot come from the template — typically references to other
runtime objects.

The canonical example is Avatar:

```typescript
// Avatar.ts
export interface AvatarInitContext {
  user?: User;
  playerId?: string;
}

class Avatar extends AvatarBase {
  override async onCreate(context?: unknown): Promise<void> {
    const ctx = context as AvatarInitContext | undefined;
    this.user = ctx?.user;
    // ...
  }
}

// Application.ts
const avatar = await StuffApi.clone<Avatar>(
  Avatar.getTemplatePath(playerId),
  { user, playerId }
);
```

Subclasses narrow the context to a concrete type locally. The clone path
itself stays generic — no type parameter on `StuffApi.clone`.

## `onCreate`

`Stuff.onCreate(context?)` is the post-registration hook, and it is a
**terminal no-op on `Stuff`** — the twin of `onDestruct` at the other end
of a life, named for the two events the engine already emits
(`stuff.created` / `stuff.destructed`). It runs after registration, so any
resolver that walks the registry sees the in-flight instance, and after
every declared hydration source, so it may rely on remembered state.

⛔ **It was `onCreate` on an opt-in `onCreate` until
2026-10-01, and the mixin's retirement removed a failure class rather
than renaming one.** The mixin's default was a *non-chaining* no-op, so
composing it anywhere but innermost SWALLOWED every layer inside it —
`KeptAnimal` shipped that way, and `Bonded.onCreate` never ran on a
live animal. With the terminal on the root there is no layer to shadow
and no composition order to get wrong; the only way left to shadow a
layer is to forget `super` in your own override. The full account is in
[lifecycle.md](./lifecycle.md).

## TemplateApi & the Folder/Leaf Invariant

`TemplateApi.saveTemplate(path, spec)` is the typed writer, taking the
RAW row an author means to state:

```typescript
await TemplateApi.saveTemplate('/narnia/castle/foyer', {
  class: '/lib/location/CartesianLocation',
  data: { /* persistent fields */ },
});
```

It looks up an existing `_id` for upsert semantics and delegates to
`PersistenceManager.save(Collections.Domain, doc)`.

The **folder/leaf invariant** (Phase 7 Decision 12) constrains the
`domain` collection paths:

- **Folders** = Zone-class templates. Detected structurally by
  `ZoneApi.isFolderClass(classPath)` — a class is a folder iff its
  `prototype instanceof Zone`. Spatial Zones (`CartesianZone`,
  `SphericalZone`) AND non-spatial Zones (`Clade` — taxonomic) all
  qualify. Folders MAY have descendant templates.
- **Leaves** = any non-folder template. Must NOT have descendant
  templates.

The folder check (`ZoneApi.isFolderClass`) is a strict superset of
the spatial-zone check (`ZoneApi.isSpatialZoneClass`) — the latter
is `prototype instanceof SpatialZone`, the set of classes whose
templates stamp `Stuff.zone` via `ZoneApi.resolveZoneForPath`.
Non-spatial folders (Clades) are folders for the invariant but
**never** become a `Stuff.zone`. The structural-check approach
means content devs add new folder or spatial-zone classes by
extending the right base — no central allow-list to edit. See
[spatial.md § Zones](./spatial.md) and
[race.md § Clade](./race.md#clade--taxonomic-scope).

The rule is enforced by `DomainHook` (`obj/hooks/DomainHook.ts`), which
composes `AroundSaveHookMixin` and `AroundDeleteHookMixin` and registers
against `Collections.Domain`. The hook calls
`TemplateApi.validateFolderLeafSave` / `validateFolderLeafDelete`, which
reject:

1. Path doesn't start with `/`.
2. Doc shape isn't a template (missing `path` or `class`).
3. Leaf save with existing children.
4. Save under a non-Zone ancestor — "Ancestor `A` is a leaf template,
   not a zone folder."
5. Delete of a Zone with surviving descendants.

The validation fires at the PM chokepoint, so calling
`PM.save(Collections.Domain, doc)` directly is equivalent to
`TemplateApi.saveTemplate` — both go through the hook.

Zone classification uses the runtime `class` field only.

## Avatar Template Convention

```typescript
class Avatar {
  static readonly TEMPLATE_PATH_PREFIX = '/platform/agent/Avatar/';

  static getTemplatePath(playerId: string): string {
    return `${this.TEMPLATE_PATH_PREFIX}${playerId}`;
  }
}
```

Avatar templates are stored at `/platform/agent/Avatar/<playerId>` and created
automatically when a Player is added to a User. Cloning happens at user
connect (see `Application.handleUserConnect`).

The per-player Avatar template carries the **initial** (char-gen) state at
first clone. It is no longer written back to on save: Avatar persists through
the **self-persistence spine** into its own `holder_snapshots` record (fields
+ inventory + gear + spawn location), NOT the template — see
[persistence.md § The self-persistence spine](./persistence.md#the-self-persistence-spine-persistable).
The template is authoritative only on first-ever login (no record yet), then
the record is (seed-then-persist).

## Restore-from-template (content go-live)

`TemplateApi.restoreFromTemplate(stuff)` re-applies a live Stuff's backing
row — looks the row up by the host's runtime-stamped `getTemplatePath()`
and runs `applier.apply(host, tpl.data, { mode: 'go-live' })`. Operates on
the existing live instance; preserves identity / stuffId / wired
Interactives. Phase 2 appliers re-fire (`applyContainer` moves the host
via compare-and-move).

⚠⚠ **`go-live` is the mode, and the mode is load-bearing.** It skips
every `birthOnly` field and runs no seed phase, because this pushes an
author's edit onto objects ALREADY IN THE WORLD. Before that mode
existed, a save on the coin row re-applied its authored `quantity: 1` to
every live coin stack in the game. See § The TemplateApplier.

Its consumers are the **content go-live** paths: `CmsLogic` and `PackLogic`
re-hydrate live clones after an author edits a template. It is NOT a
persistence-of-runtime-state mechanism — that is the spine.

The **snapshot** direction (`snapshotToTemplate`) was **retired** with the
Avatar migration onto the spine; its capture behavior (per-mixin field
marshalling, the `WarrenMember`-reconciled location capture, the
sync-prefix-before-first-await ordering) now lives in `PersistableLogic` and
is covered by `lib/persistence/__tests__/persistence-spine.test.ts`.

## `create` and `createSync` (Sister APIs)

`StuffApi.clone()` is the production path. Two sister APIs cover cases
where templates aren't right:

- **`StuffApi.create(factory, context?)` (async)** — caller-supplied
  factory, no row lookup, no content step, and no hydration sources
  (nothing names a template). Same register + `onCreate` tail. Used for
  runtime-only objects whose construction
  needs explicit arguments and don't round-trip through the CMS pattern.
  `Interactive` is the canonical example: `socketId`, `sessionId`, `user`
  all flow through the closure, not a template.

- **`StuffApi.createSync(factory)` (sync)** — same sentinel-flip + Proxy
  wrap + register guarantees as `create`, but no content step and no
  `onCreate` await. **Throws if the constructed Stuff OVERRIDES
  `onCreate`** — silently skipping it would yield a half-initialised
  object. ⚠⚠ The comparison is `raw.onCreate !== Stuff.prototype.onCreate`
  on the **raw target, before `ProxyApi.wrap`**: the proxy's get trap
  returns a fresh interception wrapper for every callable access, so a
  comparison through the proxy is always unequal and would throw on every
  `createSync` — taking `singletonSync` and every lazy logic-singleton
  resolver with it. Used inside sync helpers where
  awaiting would force the caller (and its callers) to become async too;
  `Exitable.addBidirectionalExit`'s `new Exit(...)` calls are the typical
  trigger.

Reach for `create()` whenever async hydration or post-registration
matters; `createSync()` is the narrow-use sister.

## `singleton(path)` and the Clone Pre-Flight

`StuffApi.singleton<T>(templatePath, context?)` returns the existing
instance for a path when one is already loaded; otherwise it routes
through `clone()`. The lookup hits the `byTemplatePath: Map<string,
Set<Stuff>>` index that `register` / `unregister` keep up to date —
see [lifecycle.md](./lifecycle.md#what-registration-actually-does).
A non-empty bucket with more than one entry throws (the caller mixed
`clone()` and `singleton()` on a class that does NOT compose
`SingletonMixin`).

`SingletonMixin` (`lib/stuff/Singleton.ts`) is a marker mixin (no
public surface; see
[mixins.md § Marker mixins](./mixins.md#marker-mixins-empty-public-surface)).
Composing it opts a class into a clone-time pre-flight: `clone()`
checks `byTemplatePath` first and throws if any instance already
exists for that path. The pre-flight depends on `unregister`'s
empty-bucket cleanup running before the next clone — which it does,
because `Stuff.destroy()` synchronously calls `unregister`.

Use `singleton()` for shared-state Stuff (the starting room, the
EventRegistry, well-known service objects); use `clone()` for
per-instance Stuff (avatars, items, NPCs) that should multiply.

## Failure Modes

- Template not found → `Error("Template not found: ${path}")`
- Class path validation fails → `Error("Class path must…")`
- Dynamic import fails → `Error("Failed to import class…")`
- Class name not exported by module → `Error("Class … not found in module…")`
- A declared `instruction` or `seed` field with no `apply<Field>` /
  `seed<Field>` method → throws, naming the field and the class. ⚠ Never
  a silent skip; that is the whole point of the required dispatch.
- A data key the class does not declare → **a `warn` diagnostic on the
  `template` channel**, one per `(path, key-set)` per process, readable at
  the `errors` verb. ⛔ Before 2026-10-01 it was discarded in total
  silence, which cost `material:` on 49 rows across eight packs.
- The content step, a required hydration source, or `onCreate` throws →
  object is unregistered, then the original error propagates
- `createSync` on a class that OVERRIDES `onCreate` → throws before
  registration. ⭐ The predicate compares the override against the
  terminal on the raw target, which is more accurate than the retired
  marker mixin: a class that composed the marker without overriding was
  refused for no reason, and one that overrode without composing passed
  and never had its hook called.

## Deferred / known limitations

The declarative-content substrate (the `container:` field, `applyProps`,
the structural field shapes) is shipped, but a few constraints are known and
left for future work:

- **Reset / respawn shares substrate with `props:` (own slate).** A
  future "reset this game item — destroy the current one, respawn fresh"
  feature needs three things `applyProps` doesn't yet provide: (a) a
  runtime ref from the Container back to the children it spawned, so it can
  destroy-and-respawn later — the ref must evaporate on a child's `onDestruct`
  so an already-gone child doesn't haunt the list; (b) an ownership
  distinction between "I spawned this and own it for reset purposes" and "a
  player put this here, leave it alone on reset"; (c) a declarative reset
  cadence on the entry (a richer entry form like `{ path, resetCadence }`).
  `applyProps` v1 is fire-and-forget; it extends to track the ref and
  respect the policy when reset lands as its own slate. Flagged so the
  props entry shape doesn't get locked into something that fights reset
  later.

- **`container:` can't target a multi-room facade (Lounge case).** A
  `container:` value must resolve to a singleton-shaped Container (a canonical
  Stuff). That can't express "land in the Lounge" when the Lounge is
  conceptually singular but implemented as a dynamic multi-room assembly that
  grows and shrinks with occupancy — there's no single canonical Stuff to
  point at. A known Phase-2 limitation; the eventual fix (a facade Stuff that
  IS a Container and routes internally, a master/slave room, or Pattern-C
  resolve-at-clone-time) isn't pre-baked. Cross this bridge when the Lounge
  ships.

- **`container:` vs `props:` conflict detection (linter / boot
  validation).** The same Containable-belongs-in-Container relationship can be
  declared from either side — `container:` on the occupant or `props:` on
  the host. Consistent declarations coexist (first to fire moves the
  singleton, the second no-ops via the existing-container check). But
  contradictory declarations (X says "I belong in A", yet A's `props:`
  omits X while B's includes X) aren't caught today — they need linter-level
  or boot-time validation that walks the template collection and flags the
  conflict. Not required for runtime correctness (refs still resolve lazily);
  a dev/CI affordance if conflicting declarations become an operational pain.

## Cross-References

- [lifecycle.md](./lifecycle.md) — full create → register → hydrate →
  onCreate → destroy lifecycle, construction sentinel, onDestruct
  hook
- [persistence.md](./persistence.md) — `Document`, around-save/delete
  hooks (the mechanism `DomainHook` rides on)
- [call-security.md](./call-security.md) — `ProxyApi.wrap`,
  `ExecutionContextApi.run`, `FrameKind.Constructor`,
  `SecurityApi.decorateApiClass`
- [state-model.md](./state-model.md) — why game-world objects use
  templates and Avatar's "self-contained" design
- [antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
  — setter contract for hydration
