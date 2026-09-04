# Document store (the owned-JSON tree)

The document store is the **third path-addressed tree** — the slate's
**source / template / document** triad. It is the generic, owner-claimed
tree of arbitrary JSON the runtime hands a user to fill: claim a branch
and put any JSON there, tagged with a `kind` that says what kind of
object lives at that path.

It generalizes the original scripting player-script store. Scripts are now
just **one kind** of stored document (`msh`); since content-packs wave 2
the store also holds the **pack-installed reference data** that used to
live in per-kind collections — emotes, recipes, name banks, the curated
blueprints, and every engine verb's command view — each a declared
kind. Dorm-room customization will be another (the first non-script
runtime consumer — see the Warren/lounge work).

### The closed kind vocabulary

`lib/document/DocumentKinds.ts` declares every **pack-installable** kind
— its `kind` string, natural key (`verb`, `recipeId`, `key`,
`blueprintId`, or none for a path-keyed kind), pack subdir, extension
and vanish policy. Editing it is a platform act: a kind needs a code
consumer and a go-live hook ([content-packs.md](./content-packs.md)).
`PersistenceManager` creates one **unique partial index**
`{kind, data.<naturalKey>}` per flat-key kind (partial on the kind — a
path-keyed kind never collides), and the nightly reset **keeps** every
declared kind beside `release` (`ResetPolicy`) — pack-installed rows are
world content, not player state. `release` is press-owned; `wiki` is
deliberately not a document kind (a page has a revision log and a CAS
edit path of its own).

⭐ **`water-right` is the worked example of why the vocabulary is closed**
(water build). A prior-appropriation right is a *record* — dated,
transferable, and meaningless if it can be quietly lost — so it is
path-keyed (`naturalKey: null`, `contentDir: 'water-rights'`,
`ext: 'yaml'`) and, critically, `onVanish: 'keep'`: a right survives the
pack that seeded it going away, because the whole point of recording an
appropriation is that it outlives the paperwork. A capability pack
**cannot** declare a kind — adding this one was a platform edit, and that
is the gate working, not friction to route around.

## What a record is

`StoredDocument` (`lib/document/StoredDocument.ts`, collection `documents`):

| field   | meaning |
|---|---|
| `path`  | canonical key; owner/scope encoded in the path — see the namespace taxonomy below |
| `owner` | the owner's durable `templatePath` (set from context, never the caller) |
| `kind`  | **what kind of object lives here** — `'msh'` (a script), `'emote'`, `'recipe'`, `'name-bank'`, `'blueprint'`, `'command-view'`, `'release'`, `'water-right'`, later `'dorm'`, … |
| `data`  | the arbitrary JSON payload |

The store is **kind-agnostic**: it persists and serves `{path, owner,
kind, data}` and never inspects `data`. Each `kind`'s consumer owns the
*meaning* of `data` and any **go-live** behavior (re-parse, re-hydrate).

`StoredDocument extends Document` only for persistence — distinct from the
persistence `Document` base class it shares a word with: a `Document` is
*how* a row is saved; a `StoredDocument` is the *owned-JSON-in-the-tree*
concept.

## ⭐ The four namespace kinds

The path prefix says what kind of thing a branch **is**, and there are
four — not three. The fourth was surfaced by needing somewhere for the
Compact to publish, and the taxonomy did not have it:

| namespace | what it is |
|---|---|
| `/home/<self>` | **workspace** — personal |
| `/studio/<group>` | **workspace** — the multiseat `/home` |
| `/world/<locality>` | **content** — a *place*, with rooms and NPCs, locally administered |
| `/trade/<industry>` | **content** — an *industry*: what a trade introduces (its stations, stock, recipes), held by the trade's own group |
| `/compact` | **an institution's own branch** — the Compact's; it keeps publications there |
| `/corpo/<key>` | **an institution's own branch** — a company's |

> **An institution's branch is what it owns that is not a place.**

⚠ **The fourth row was first written as "publications", and that was
describing the Compact's *contents* rather than the *kind*.** Giving each
corpo a branch of its own showed what the kind actually is: a body that
is not a place still needs ground it holds title over — because that title
is what makes it a **committee**, and a committee is how it appoints
([corpo.md](./corpo.md), [employment.md](./employment.md)). The Compact
keeps publications on its branch; a corpo keeps a chart on its. Neither
makes rooms.

Two wrong turns are recorded here because the next person will reach for
them too:

- ⚠ **`/studio` is not the precedent for a publishing namespace.** It is a
  *workspace* namespace, the multiseat `/home`. Its parcel **row shape** is
  the right model — a core-held path-branch title deliberately omitting
  `landUse`/`areaM2`, so the inheritance walk answers `wild` and admits
  nothing — but its *concept* is not.
- ⚠ **`/compact` is not a federal enclave.** The diegetic premises of
  Compact institutions are Saxonberg's rooms, **administered locally** (DC
  has its own police under a seated city government). `/saxonberg` will be
  *content*. `/compact` is not where the buildings go; it is where the
  documents go.

See [parcel.md](./parcel.md) for the `/compact` title.

## ⭐⭐⭐ When a collection should be a document instead (audit, 2026-09-03)

The cluster caps at **500 collections** and a world is **46**, so *"should
this be a collection?"* is a real question with a real budget behind it.
All 48 were audited against the criteria below. **The answer is three**,
and two of those are defects rather than migrations.

### The rubric — all five must hold

| # | test | why |
|---|---|---|
| 1 | **path-addressable** — every read by path or prefix, never *find all X where Y = Z* | the store is a tree |
| 2 | **grows with things, not actions** | ⚠⚠ a document is **one row**; concurrent appends to one row **fork** |
| 3 | **no revision log, no compare-and-set** | ⭐ the standing exclusion: *`wiki` is deliberately not a kind — a page has a revision log and a CAS edit path of its own* |
| 4 | **≤ 1 secondary key** | `PersistenceManager` creates exactly **one** unique partial index per flat-key kind (`{kind, data.<naturalKey>}`) and nothing else |
| 5 | **a titled owner** | `canAtPath` gates on parcel title, and `lint:untitled` says an unclaimed path is one **nobody can ever edit** |

⚠ **Criterion 2 is the one that does the work, and row counts lie.**
`renown_events`, `beliefs`, `accountability_events` and `player_frames`
all measured **zero rows** in the live worlds on 2026-09-03 — because
nobody has played, not because they are small. **Growth law, never
volume.** Every `*_events` collection, `bank_ledger`, `chronicles`,
`transcripts` and `diagnostics` fail here regardless of what they hold
today.

### ⭐⭐⭐ The verb test — `warm()` vs `rebuild()`

A third disposition exists beside *move* and *keep*: **a collection that
is a cache of derivable state should not persist at all.** And there is a
one-word tell, visible in every singleton's `postRegister`:

| the call | what it means | verdict |
|---|---|---|
| `warm()` · `warmCache()` · `boot()` | **the collection is the truth**, memory is the cache | keep |
| `rebuild()` | **the collection is the cache** | it should not exist |

⭐ Swept across all 39 singletons: **`BlueprintCatalogue` is the only one
that calls `rebuild()`.** `MaterialCatalogue`, `RecipeCatalogue`,
`SoulCatalogue`, `ChannelCatalogue`, `SubjectCatalogue`,
`ArchetypeCatalogue`, `HelpCatalogue`, `FermentProfileCatalogue` and the
rest all **warm** — they load, and their collections are real state.

### The audit result

| collection | disposition | evidence |
|---|---|---|
| ⭐⭐⭐ **`blueprints`** | **delete — derivable** | the only `rebuild()` in the tree; `Blueprint.find()` appears **twice**, both inside `BlueprintCatalogue.rebuild()`; the class says *"A CACHE: `rebuild` regenerates it at every boot"* |
| ⭐⭐⭐ **`descriptor_banks`** | **move — should be a kind** | ⚠ **a documented inconsistency** (below) |
| ⭐⭐ **`office_holders`** | **move** | 0 indexes, 5 sparse rows, `postRegister` does **no DB work**, and `holderOf` = *explicit row else the founder default* — a path lookup with a fallback |
| ⭐ `media_assets` | move | 0 indexes; provenance records, addressable by asset |
| ⚠ `bank_supply` | **leave** | 1 row, 0 indexes, looks perfect — **it is money.** The conservation chokepoint's counter does not go in a generic JSON store where `canAtPath` decides who may write. |
| ⚠ `pack_installs` | **leave** | circular: it is the *installer's* ledger, and documents are what the installer writes |
| ❌ the rest | leave | real state, or criterion 2 |

**Ceiling: 48 → 45.** Modest, and the point is less the count than that
two of the three are **bugs**.

### ⚠ The two defects

**`blueprints` is a cache that persists.** 147 rows survive a restart and
are then immediately discarded and re-derived by the only code that reads
them. ⭐ It is **not** a duplicate of the `blueprint` document kind — the
two hold disjoint populations by design (147 machine-minted
`bp-<class-path>` skeletons vs 9 hand-authored curated overlays at
`/platform/blueprints/<id>`; **zero id overlap** in both live worlds).
⚠ The seam to prove before removing it: `rebuild()` reads the previous
generation for *drift-safe on id* and the signature dedup — `derivedId()`
is deterministic, so that stability may already come from the function
rather than the stored rows.

**`descriptor_banks` should be a kind, by its own class comment:**

> *"The **`NameBank` shape verbatim**, and for the same reasons: these are
> bulk authored **content**, not code, so they live as plain-JSON
> `Document`s in their own collection and arrive through the `PackApi`
> reconcile installer as the `descriptor-banks` content kind."*

`name-bank` **is** a document kind. `descriptor-bank` is not — and
`PackLogic` parses `content/descriptor-banks/*.yaml` right beside the
kinds that land in `documents`, then routes these into a collection.
Same shape, same installer, same stated reasons, different store.

### ⭐⭐ A cross-cutting register is not a new namespace

The recurring ask is a global ledger — *every herd in the world*, every
right, every licence — and the reflex is a root like `/herds`. ⚠ That
fails criterion 5 immediately: **nobody holds title to it, so nobody can
write it.**

The taxonomy above already answers it, in two parts:

> ⭐ **The record lives with its owner. The register lives on the branch
> of the institution that keeps it.** The deed is yours; the register is
> the county's.

Which is the house pattern already — `ParcelRecord` + `ParcelRegistry`,
and chattel's *"ownership is a row in a separate, gated registry, stored
apart from the editable good it gates."* The herd's record sits under the
rancher's parcel where they can write it; the register entry sits on
`/corpo/<association>` or the realm's own branch, holding a pointer and
whatever the register must answer globally.

⭐ And it forces the question the design should be asking anyway — **who
keeps this register?** — instead of inventing a rootless namespace that
nobody can be accountable for.

## The Api (`api/document.ts` → `platform/idea/api/DocumentLogic.ts`)

A thin gated forwarding shell over a hot-reloadable logic singleton at
`/platform/idea/api/document`, the `ScriptLogic`/`CraftingLogic` precedent:

- `DocumentApi.read(path)` → the `StoredDocument` at `path`, or null.
- `DocumentApi.list(prefix)` → every doc at/under `prefix` (the CMS tree's
  input).
- `DocumentApi.list(prefix)` / `DocumentApi.listOfKind(kind)` — the second
  is the release window's rebuild input.
- `DocumentApi.save(path, kind, data)` → find-or-create. **Access-gated**,
  **owner-stamped from context**, **provenance-recorded** (below). For
  `kind: 'command-view'` it also runs the **code-naming gate** and the
  schema check, and reloads the view (below).
- `DocumentApi.delete(path)` → the same gate as `save`; no provenance row
  (a deletion is not authorship); returns whether a row existed. The
  emote catalogue's `delete` rides it.
- ⚠⚠ `DocumentApi.saveRelease(publisher, path, data)` — **the named write
  transport**, and an ownership bypass by construction. See below.

The acting **owner is always derived from `ExecutionContextApi`** (the
in-world command-frame giver, or a transport's `tagActingAuthor` stamp),
**never a parameter** (memory: gated-api-actor-from-context).

### The gate — parcel title

`DocumentApi.save` / `delete` gate on **parcel title** (content-packs
wave 2, D11), with one base case on top:

1. **An owner owns their own `/home/<self>/` branch** — keyed on the
   durable-path basename, so a player owns exactly the subtree the runtime
   banks under their name (recorded recipe-scripts today, a dorm's
   customization tomorrow). No broader grant needed. As of property phase
   0a this rule is **shared, not forked**: `isOwnHomePath` consumes
   `ParcelApi.selfHomeOwnerOf` (the single implementation, also rung 2 of
   the parcel `ownerOf` chain — see [parcel.md](./parcel.md)).
2. else **`AccessApi.canAtPath(actor, 'write-document', path)`** — the
   covering title through `ParcelApi.ownerOf(path)` (rung 1 a parcel,
   rung 2 the self-home, rung 3 the state) and that owner's `can()`
   dispatch, verbatim. No zone walk, no `core` literal: the state default
   IS `ownerOf`'s third rung. A pack's document branch (`/expression`,
   `/generic-objects`) is untitled, so the state's group decides; an
   author-minted emote or blueprint lands on the platform's own
   `/emotes/` / `/blueprints/` branch for the same reason.

### The command-view code gate

A `kind: 'command-view'` save names TypeScript: its `controller:` (at
the verb or any subcommand) and every validator / `requires` reference
are code references. A save that changes any of them — **the set, not a
subset**: a validator is a gate, and removing one widens dispatch just
as adding one narrows it — requires `AccessApi.isWizard` (the code-trust
axis, [access.md](./access.md)); so does a new view naming a controller.
A cosmetic edit (help text, descriptions) is content authoring. Every
command-view save is validated against the command schema at the
chokepoint and then `CommandApi.reload(path)` puts it live — the CMS
edits `look.yaml`'s help and `help look` changes without a restart. The
installer never sees this gate (it writes through `PersistApi`,
bootstrap-exempt like templates).

### Provenance

Every save appends an `AuthoringEvent` keyed on the path —
`DocumentLogic` (`/platform/idea/api/document`) is a named authoring transport in
the `ProvenanceApi.recordAuthoring` gate (alongside the template
chokepoint). Authorship is *derived*, not a mutable stamp.

## ⚠⚠ The release write transport — a named ownership bypass

`kind: 'release'` documents ([press.md](./press.md)) are owned by the
**publisher organization**, not by the person who wrote them. The gate
above cannot express that: it admits the **parcel owner**, and making
every comms director a landowner is obviously wrong.

So the press path writes through `DocumentApi.saveRelease`, which stamps
an owner the ordinary `save` gate would never admit — the same shape as
`PersistableApi` routing capture as the owning principal. **If it is
loose, the whole document-store ownership story is broken, not just for
releases.** Four things keep it narrow, and each is load-bearing:

1. **Gated to one calling module** (`PressLogic`). ⚠ The gate lives on the
   **Api static**, not the logic method: every logic method's caller is
   its own Api face, so a policy there would name `DocumentApi` and narrow
   nothing (the `CompactApi.assignOffice` → `OfficeController` precedent).
2. **It takes no caller-supplied owner** — it takes the publisher and
   derives one, so there is no parameter to lie in.
3. **It refuses a path outside that publisher's own feed branch.**
4. **It pins the `kind`**, so it cannot write anything else.

⚠ And because the tree is a **shared store**, the complement matters as
much: `kind: 'release'` is a tag anyone who can write a document can
apply, so **every read of the release set re-verifies what the transport
guarantees** (`Release.publishedByOwner` — the owner must resolve to a
live publisher, at a path under that publisher's feed). Without it, a
document written through the ordinary `save` path by whoever owns the
covering branch would ride the ticker as a release.

## Scripts as a `kind`

`ScriptLogic` keeps the script **semantics** — parse source → AST, the
per-path AST cache, the script-specific go-live — and delegates **storage**
to `DocumentApi`:

- `ScriptApi.saveScript(path, source)` → `DocumentApi.save(path, 'msh',
  { source })`, then invalidate the AST cache (the script go-live).
- resolve-by-path → `DocumentApi.read(path)`, take `data.source` when
  `kind === 'msh'`, parse + cache.

The kind is **`msh`** — the language's name. The `saxonberg-lounge`
pack ships them as `content/msh/*.msh`.

The generic store deliberately does **not** keep an AST cache (an AST is
script-specific) and runs **no** kind-specific go-live — that all lives
with the kind's consumer.

## CMS — the third tree

The CMS exposes the store as the **`'document'`** backend (`CmsBackend =
'content' | 'source' | 'document'`), the third root in the explorer
alongside content (templates) and source (engine TS). The record's `kind`
drives the editor treatment:

- `kind: 'release'` → a press release ([press.md](./press.md)), under its
  publisher's declared feed branch, owned by the publisher.
- `kind: 'msh'` → the source text (`data.source`) as a plain-text code
  leaf; a write funnels through `ScriptApi.saveScript` (the script
  chokepoint: gate + provenance + AST go-live). Scripts are the one
  runtime-**creatable** kind through the CMS.
- any other kind → `data` pretty-printed as JSON; a write parses the JSON
  and persists via `DocumentApi.save` under the doc's existing kind (no
  live re-hydration consumer yet — that lands with dorm).

Author-tier read gate (like content). Wired end-to-end: the REST
`parseBackend` admits `'document'`, and the client explorer lists the
`documents` root.

## Deferred

The fuller per-`/home/` access model (sharing, grants beyond the
self-owner), non-script document **creation** via the CMS, and the first
non-script runtime consumer (dorm-room customization data, hydrated onto
a Warren constituent — base-template + customization-document) all land
later. The substrate is built to accommodate them; scripts, the
pack-installed kinds, and releases ride it today.
