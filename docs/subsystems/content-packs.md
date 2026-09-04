# Content packs

The **content-pack** substrate: game content lives in standalone,
git-versioned packages — `content/` (YAML, the installer's jurisdiction)
and, at the **capability rung**, `src/` (the TypeScript classes that
content names) — and a single **installer** reconciles the content into
MongoDB at boot and on demand while the kernel's class-path resolver
serves the code. The pack files are the **source of truth**; the database is a
*derived install* of them. It replaced `SeederManager` outright (wave
3) — it gives content real version control (git, on the files) and a
clean deliverable boundary, decoupled from the kernel's release cycle.

**Nineteen packs ship today** (capability packs, arcana first, 2026-08 —
the rung after wave 4b's *pack zero, and the end of `core`*), and
**every template row in the world comes from one of them**: there is no seeder, no code
manifest, no `seeds/` tree. The table is in *§ The shipped packs*. The
short version: **platform** is pack zero — the controller templates, the
registries and catalogues, the marshallers, the closed vocabularies, the
Compact's institutions (`/compact/executive`, `/compact/press`), the
namespace roots and the landing shell (`/platform/location/void`), plus the
settings, the standing subjects, the curated blueprints and every engine
verb's command view — and **it alone is a bootable world** (the
platform-only e2e proves it every pipeline). Everything else `dependsOn`
it: the substrate packs (**base-library**, **species-and-names**,
**arcana** — magic's substrate and the first *capability* pack —
**arcane-library**, **generic-objects**), the
social packs (**expression**, **wiki-starter**), the five
**corpo-{aevex,goodkin,hollis,veshko,vionne}** (each now an
*organization* with its own chart), the three industries
(**trade-smithing**, **trade-cooking**, **trade-hospitality**),
three localities (**newbie-wilds**, **saxonberg-lounge**,
**hearthworks**), and **world-seed** — the TRANSITIONAL pack holding
the locality rows the retired `SeederManager` used to insert, deleted
piecewise as wave 5 homes each remaining locality.

A pack no longer only ships rows. Its manifest **declares what the rows
need** (*§ The requires phase*): the groups it needs to exist, the
**title** it claims over the extents it ships into (a pack's paths must
lie under a claim — its own or a host's — or the install refuses), who
**maintains** it (a group, or an organization — the executive holds the
platform; a corpo holds its own branch), and which of its rows are
**eager at boot** and why (*§ The boot union*). The seeders'
`config/groups.yaml` and `config/parcels.yaml` are gone: a group or a
title is declared by the pack whose content needs it.

**The DB is a cache of the packs.** Since the pack-installer build
(2026-08) the templates collection is named **`content`**,
and every install is **three-way** against a per-deployment install
record (`pack_installs`): a row the pack changed is updated, a row the
database changed is kept, a row both changed is a **conflict** —
reported, never merged. The `pack` verb is
gated on **title over `/compact/executive`** — the executive's own
staff (the Prime Minister and whoever the office appoints), never the
wizard axis; the wave-2 `pack-installers` committee folded into the
executive.

> **Scope.** Two rungs of the downloadable-content ladder (roadmap
> Framework 13: Content / Capability / Full; the content-packs slate's
> *data pack · capability pack · mod*). A **data pack** ships content and
> assumes the kernel already provides the classes its rows name. A
> **capability pack** ships those classes too, under `src/`, imported by
> the kernel's class-path resolver exactly as the kernel's own — and
> **most packs will be capability packs**: the rung is a fact about a
> `src/` directory, not a pack's identity. What neither may do is add an
> Api or reach past the kernel's author surface; that is the *mod* rung,
> a kernel MR. The `requires-kernel` check enforces the line — a pack's
> classes must resolve, and it records *where* (§ The capability rung).

## The shape of a pack

A pack is a monorepo package (`packages/content/<pack>/`, its own repo
once the monorepo is broken up) named `@saxonberg/content-<id>`:

```
packages/content/base-library/
├── package.json          # workspace member; `dependencies` name the packs it depends on
├── pack.yaml             # the manifest the installer reads
├── src/                  # CAPABILITY packs only: the classes its rows name, <root>/<branch>/…
└── content/              # content root; MIRRORS the template-path namespace
    └── lib/
        ├── material/spirit/gin.yaml   →  template path /trade/distilling/idea/material/gin
        └── biome/…
```

The `content/` root mirrors the template-path namespace: a file's path
relative to `content/`, minus `.yaml`, prefixed with `/`, **is** its
template path (`content/trade/distilling/idea/material/gin.yaml` →
`/trade/distilling/idea/material/gin`; `content/world/newbie-wilds/crossroads/hub.yaml`
→ `/world/newbie-wilds/crossroads/hub`; `content/corpo/aevex.yaml` →
`/corpo/aevex`; `content/home.yaml` → `/home`) — the rule the retired
`SeederManager` used. So the path is a pure namespace identifier,
decoupled from where the file physically sits, and moving a tree between
packs is a `git mv` (newbie-wilds out of `seeds/` was exactly that: 21
files, bytes and paths unchanged; wave 3's 439-file move of the engine's
own rows into the platform pack was the same act at scale).

The same rule names every row in the install record — the **record
key**: the content-root-relative path with a leading slash and no
extension. For the template kind that *is* the template path; for a
descriptor bank it is `/descriptor-banks/<key>`; for a document kind
`/<contentDir>/<name>` (`/emotes/grin`, `/msh/martini`,
`/name-banks/common`); for a settings section `/settings/<section>`; for
a subject `/subjects/<name>`; for a wiki page `/wiki/<ns>/<slug>`; for a
command view its document path (`/platform/cmd/perception/look`). One uniform
address for every kind (`pack diff <id> <path>`).

### The manifest — `pack.yaml`

```yaml
id: saxonberg-lounge      # the sourcePack stamp value; stable
version: 0.2.0            # reserved release label — nothing reads/enforces it
description: …
root: /world/lounge      # the DOCUMENT root (optional; defaults to /<id>; must start with /)
maintainers: lounge       # a group name, { group: <name> } or { organization: </path> };
                          # default `<id>-maintainers`
requires:                 # what the rows need — see § The requires phase
  groups:
    - { name: lounge, purpose: the lounge team }
  title:
    - { extent: /stuff/idea/lounge,    holder: { group: lounge } }
    - { extent: /world/lounge, holder: { group: lounge } }
boot:                     # which rows are eager, and why — see § The boot union
  - { template: /world/lounge/thing/terminal, role: producer, reason: "…" }
```

**The key set is closed** (`MANIFEST_KEYS` in `PackLogic`): `id`,
`version`, `description`, `root`, `requires`, `boot`, `maintainers`.
**`dependsOn` is not a key** — it is *derived* from the pack's
`package.json`: every `@saxonberg/content-<id>` dependency is a pack id
(the `@saxonberg/server` and `@saxonberg/types` lines are the kernel).
One graph: the line that lets a pack's code import another pack's
classes is the line that orders its install and names its hosts. Any other key — a typo, a field from a future wave — is an
**error at read**, and so is an unknown key inside `requires`
(`groups` / `title` only) or inside a boot entry (`template` / `role` /
`reason` / `dependsOn`). A manifest that parses is a manifest the
installer understands in full.

`root:` is where the pack's **document-kind** rows land — every
`documents` row a pack ships has `path = root + recordKey` and `owner =
root` (`/expression/emotes/grin`, owned by `/expression`). It is the
pack's branch in the document tree; the template kind ignores it
(template paths are their own namespace). `saxonberg-lounge` declares
`root: /world/lounge` so its scripts land under the lounge's branch.

Still deliberately minimal about *rows*: the pack's kernel requirement
is **derived**, not declared (*requires-kernel* is the install-time
class-resolve check, below), and *owned paths* are recorded by the
`sourcePack` stamp at install time — the manifest never enumerates
paths. What it does enumerate is **extents** (`requires.title`), which
is a different thing: a claim is a title, and title is declared, never
inferred from a file tree.

`server`'s `package.json` `@saxonberg/content-*` dependencies are the
**single source of truth** for which packs a build ships — adding a pack
to a build is adding a dependency line.

## Discovery

`PackLogic` discovers packs by reading the `@saxonberg/content-*` keys
from the **deployment manifest** — the root `package.json` of the
workspace, or wherever `SAXONBERG_DEPLOYMENT_ROOT` points (default: the
nearest ancestor of the server package whose `package.json` carries a
`@saxonberg/content-*` dependency). **The server never depends on a
pack**; installing one is `pnpm add @saxonberg/content-<id>` in the
deployment, and when the packs are their own repos that manifest is the
only file that changes. Each name resolves to its on-disk root via Node
module resolution from that manifest (`createRequire(<root>/package.json)
.resolve('<pkg>/package.json')`), layout-robust across dev (`tsx`) and
prod (also `tsx`). A pack with a `src/` dir is a **capability pack**:
discovery registers its `src/` against every namespace root it holds
(the manifest `root` + each `requires.title` extent) in the class-source
table (`ModuleApi.registerPackSource`) before anything imports —
`PackApi.registerSources()` is that half alone, which `test-bootstrap`
runs so a test resolves `/system/arcana/…` without an install. Packs are then
ordered by the derived `dependsOn` (a stable topological sort; throws on
a cycle; an unknown id is ignored).

**`SAXONBERG_PACKS`** (D10) filters the discovered set: a comma-separated
list of ids (`SAXONBERG_PACKS=platform`), applied *after* ordering. A
pack it names that no shipped pack provides **throws at boot**; a pack
it omits is **ignored** — not installed, and (because `bootManifest`
reads the same filter) not booted, even when an earlier unfiltered boot
left its record behind. Unset or empty means every discovered pack.
`platform` always sorts first: every other pack `dependsOn` it.

## The installer — reconcile by `sourcePack` stamp + content-kind

The installer's job each run is *make the DB match the pack files*. It
lives on the **Api + logic-singleton** pattern (not `backend/` like
`SeederManager`) because it has two callers — the boot path and the
runtime `pack sync` verb — and the gated `PackApi` is the shared surface:

- **`PackApi`** (`mud/api/pack.ts`) — the thin forwarding shell:
  `install()` / `sync(packId)` / `discoverPacks()` and the ops surface
  `status()` / `dryRun()` / `diff()` / `resolve()` / `pin()` / `unpin()`,
  plus the call-shape types (`PackManifest`, `PackReconcileResult`,
  `PackInstallRecord`, `PackConflict`, the report shapes).
- **`PackLogic`** (`mud/platform/idea/api/PackLogic.ts`, `/platform/idea/api/pack`,
  `@internal @Unshadowable`) — the reconcile core; every public method
  gated `FromModule('/api/pack#PackApi')`. All work in module-private
  functions (the `CraftingLogic` precedent), so no intra-singleton
  `this.x()` calls trip the gate.

A single private `reconcilePack(pack, {rehydrate})` is the one
implementation; `install` and `sync` differ only in the re-hydrate tail.
Inside it, every shipped kind rides **one strategy interface**
(`KindStrategy<F>`, module-private — the content-pack-units Part C
interface): the target collection, the record key, the db-key query, the
rendered row, the hash preimage, the go-live side effect, and an optional
`flatKeyOf` for kinds whose keys form a flat namespace. One
`computeKindPlan` (pure — reads only) and one `applyKindPlan` (the
writes) drive all three kinds, so the reconcile policy is written once.
**Dry-run is the planner with the apply half never called** — zero
writes by construction, not by a flag threaded through write sites.

### Content-kind dispatch

Content is classified by subdir convention and dispatched to the right
backend — the per-row stamp makes this work identically across backends,
which a path-prefix notion of ownership could not:

| Subdir | Kind | Backend |
|---|---|---|
| every `content/**/*.yaml` **outside the kind dirs below** | **domain** (the template kind) | reconciled into the `content` collection (stamped). Wave 3 widened the walk from the two enumerated roots (`obj/`, `domain/`) to *everything that is not another kind's directory*: a pack ships a row at the path its file mirrors, wherever in the tree that path lives — `content/corpo/aevex.yaml` → `/corpo/aevex`, `content/home.yaml` → `/home`, `content/wiki/main.yaml` → `/wiki/main` (the namespace ZONE rows; the wiki *pages* beside them are `.md`, a different extension, read by the wiki kind). The non-template dirs are **enumerated by kind** (`nonTemplateDirs()`: `settings`, `subjects`, `descriptor-banks`, `quantity`, and every yaml `DOCUMENT_KINDS` `contentDir`), never guessed. `cmd/` is skipped at **any** depth — a command view has no `class:` and is the command-view document kind |
| `content/quantity/quantity-tags.yaml` | **quantity** | loaded into the in-memory tag table via `QuantityApi.loadTagTables(path)` |
| `content/descriptor-banks/<key>.yaml` | **descriptor-banks** | reconciled into `descriptor_banks` (stamped), keyed on the basename = the item class; the appearance caches drop on change |
| `content/<contentDir>/<name>.<ext>` per **`DOCUMENT_KINDS`** — `emotes/*.yaml`, `recipes/*.yaml`, `name-banks/*.yaml`, `blueprints/*.yaml`, `archetypes/*.yaml` (the venue archetype — libations D11: `{ archetypeId, label, industry, capabilities: [{ key, needs, default? }] }`, `needs` one of `tool`/`heatK`/`bulkSource`/`surface`/`seating`/`coldStorage`; validated by `Archetype.fromData`, warmed into `ArchetypeCatalogue`; an archetype then describes and materializes **itself** — `archetype.describe()` is the effective floor (the authored residue + the rows derived from the industry's recipes) and `archetype.materialize()` is the derived test venue (a `FurnishableRoom` + each default binding). ⭐ **There is no `ArchetypeApi`**: the first cut shipped one whose every caller was a test, while the two production paths — this validator and the go-live re-warm — reached the value-object and the catalogue directly. A `checklist(venue)` reporting surface was deleted with it: D11 said *no runtime enforcement*, and nothing read it), `msh/*.msh`, `cmd/**/*.yaml` (+ every template tree's own `<tree>/**/cmd/*.yaml` — `world/…/cmd/`, `trade/…/cmd/`) | **document** (one strategy per declared kind) | reconciled into `documents` (stamped) at `root + key` — the closed vocabulary in `lib/document/DocumentKinds.ts`; a `.yaml` file's object is `data`, an `.msh` file is `data: { source }`; a flat-key kind gets its natural key from the basename (a disagreeing file fails at `read`); per-kind read validation (what the retired seeders validated) |
| `content/settings/<section>.yaml` | **settings** | merged into the `app_settings` singleton (**merge-missing**, below) |
| `content/subjects/<name>.yaml` | **subject** | `forum_subjects` + its channel/board surfaces (**archive-never-reap**, below) |
| `content/wiki/<ns>/<slug>.md` | **wiki** | submitted through `WikiRegistry` AS the pack (**CAS**, below) — never rows |

#### `DocumentKinds` — the closed vocabulary

`lib/document/DocumentKinds.ts` declares every pack-installable document
kind: its `kind` string, its natural key (`verb`, `recipeId`, `key`,
`blueprintId` — or null for a path-keyed kind), the pack subdir, the
file extension, and its vanish policy. **Editing it is a platform act**
(slate A11.5): a pack cannot declare a new kind, because the kind's
consumer (a catalogue, an engine) is code and the installer needs its
go-live hook. `PersistenceManager` builds one **unique partial index**
`{kind, data.<naturalKey>}` per flat-key kind (partial on the kind, so
path-keyed kinds never collide), and the nightly reset keeps every
declared kind beside releases — without that the world would lose its
emotes at 04:00. `wiki` is deliberately absent (a page is not a
document: it has a revision log and a CAS edit path); `settings` and
`subject` are contribution kinds with their own targets.

The **document strategy** is one factory (`documentStrategy(spec, root)`)
per kind: `dbKeyQuery` — `{kind, path}`, or `{kind, 'data.<naturalKey>'}`
for a flat-key kind (its identity IS its natural key: a row with that key
at *any* path is this row, and one this pack did not stamp is refused) —
and a
**`stampedQuery`** `{kind}` — ⚠ load-bearing: `{sourcePack}` alone over
`documents` returns every kind the pack ships, and each kind would reap
the others' rows as its own vanished files. Baselines and conflicts
carry the label `document:<kind>` so `pack diff` output names it. The
preimage is `{ data }` only — `path`/`owner`/`kind`/`sourcePack` are
bookkeeping.

> **The template trees are `content/<root>/<branch>/…`** — `platform/`
> for the platform pack, `stuff/` for every other pack, `trade/<industry>/`
> for an industry — because a pack ships content and
> content is instanceable — nothing a pack installs may live under
> `/lib/`, which is substrate-only. This is a **breaking format change**
> the second segment is the Stuff branch the row's class descends from.
> repoint every `class:` value. Shipped packs needed no data migration
> because the installer **reconciles** (unlike `SeederManager`, which is
> insert-only) — the old rows are stamped, their files vanished, so
> the reconcile deletes them and inserts the new paths on the next
> boot. `PackLogic.readContent` reads the subdir names.

**Descriptor banks** are the one remaining *side-collection* kind — a
flat `Document` keyed on the item class in `descriptor_banks`; the
appearance caches drop on change. Name banks were the first such kind
and now ride the `name-bank` document kind (the same
`content/name-banks/<key>.yaml` files, read as `data: {key, given,
surname, style?}`; the species-and-names pack needed no file change).

#### The per-kind reconcile policy

| Policy | Kinds | What it does |
|---|---|---|
| **three-way** (default) | domain, descriptor-banks, every document kind, subject | the A10.4 matrix below; a vanished file deletes a clean row (subjects: archives it) |
| **merge-missing** | settings | a key the `app_settings` singleton lacks is merged in (`merged`); a key the operator tuned with `config` is **kept — never a conflict**; a vanished file keeps every value and drops its baseline. The baseline is the file body, so `pack diff` shows the pack's default against the operator's value |
| **cas** (compare-and-swap) | wiki | a page is created or edited **through `WikiRegistry` as the pack** (`asInstaller: <packId>` — the revision author is `pack:<id>`, the namespace protection walk is skipped); an edit carries the baseline's page `rev` as the CAS token, and a page somebody edited since is a **`wiki-cas` conflict** the registry throws — recorded with the baseline / live / pack hashes, one diagnostic; `pack diff` renders the live page; `resolve --take-pack` edits over the CURRENT rev (the history keeps both); a vanished file **keeps** the page. A page that pre-dates its record (a seeder-made one) adopts with the live `rev` in the baseline — no edit submitted |

**Vanish policies** (`onVanish`): `delete` (the default), `keep`
(settings, wiki — the baseline drops, the row stays), `archive`
(subjects — `Subject.state = 'archived'`, `Channel.archived` /
`Board.archived = true`; the catalogues and the board listing skip
archived rows, the entries stay; a returning file re-activates the row
in place; never a delete).

**Subjects** (D6) are the one kind written *through the seeder's own
shape*: a Subject row (owned by `pack:<id>`) plus its `open-chat`
channel and/or `open-forum` board, with optional name overrides and an
audience group resolved **by name** at the pre-write gate (a missing
group fails the pack). Both sides render to one preimage
(`{name, description, audience, board, channel, channelName,
boardName}`) so the three-way compares like-for-like. ⚠ No `SubjectCatalogue.installSubject`: at boot the installer
runs *before* `BootstrapManager` clones the catalogues, so the rows are
written as Documents through `PersistApi` and the resident catalogues
are invalidated only on a live sync.

**Command views** (D8) are the `command-view` document kind at the view
key's document path — `content/platform/cmd/perception/look.yaml` →
`/platform/cmd/perception/look`, and a **content tree's own** views — a
locality's `content/world/<…>/cmd/<verb>.yaml` → `/world/<…>/cmd/<verb>`,
an industry's `content/trade/<industry>/cmd/<verb>.yaml` →
`/trade/<industry>/cmd/<verb>` — by **one rule** (wave 4a): a view key
whose first segment is not `cmd` and that carries a `cmd` segment is a
content-tree key at `/<key>`; `cmd/<rel>` is an engine key at
`/platform/cmd/<rel>`. The installer walks every top-level `content/` dir that is
not a kind dir for `cmd/` at any depth; no tree is special-cased (no
`root` join: the dispatcher's key is the same string, so
`CommandApi.reload(path)` finds it). Every view is
validated against the command schema at `read`. **There is no disk
fallback** (wave 3): once `CommandApi.preloadAll` has served the views
from a document store, the store is the *only* source and a miss is a
miss (`getCommand` returns null and says so). The wave-2 counted residue
(*N command view(s) still served from disk*) is gone with the last seven
domain-local views, which moved into `world-seed`. **Offline** — a unit
test, a stripped boot, anything with no store ever preloaded — the views
are read straight from the packs' own files (`PackApi.contentRoots()`,
cached; `cmd/<key>` for an engine key, `<key>` for a content-tree key —
the same rule: the same files the installer reads), so the kernel tests
keep working with zero seeding. The module-level `servedFromStore` flag
is what tells the two apart; `clearCache` resets it. A CMS save of a
view that changes its `controller:` or its validator set is **wizard
code trust** (`DocumentLogic`); a cosmetic edit goes live through
`CommandApi.reload` without a restart.

### The `sourcePack` stamp

Each installed row carries a **top-level `sourcePack` field** (a `content`
template, a `name_banks` bank — every stamped backend). On a template row
it is a sibling of `data` (`{path, class, hydratorClass, data, sourcePack}`).
It is **not** inside `data`, and the clone pipeline passes only
`template.data` to the Hydrator — so the stamp is structurally
unreachable by the instance (a `Material`/`Biome` never sees it). It is a
pure DB-row provenance marker, written directly by the installer through
the `PersistApi` chokepoint (`save` = `$set`-by-`_id` for update, or
insert), never via `TemplateApi.saveTemplate` (which has no slot for
it).

### The install record — `pack_installs`

One row per pack per deployment, written only by `PackLogic` through the
`PersistApi` chokepoint; policy `refuse` (a circle never writes installer
state), `keep` on the nightly reset, unique on `packId`. Its own
collection so no contribution kind can ever reach it (the
`parcels`-not-in-`content` reasoning, slate A17.1). Schema —
`PackInstallRecord` in `mud/api/pack.ts`:

- `packId`, `version` (the manifest's), `appliedAt`, `principal` (the
  context-derived acting author's templatePath, or `bootstrap` at boot —
  never a parameter), `status` (`applied` | `failed`; `staged` reserved),
  `failure` (`{step, error, file?}` — `read` / `flat-key` /
  `requires-kernel` / `topics` / `reconcile`), `parameters` (`{}`),
  `pins`, `conflicts`, `sideEffects.kinds` (`['quantity']` when the
  RAM-only kind ran — noted, never baselined).
- **`rows[recordKey] = {kind, hash, body}`** — the per-row **baseline**
  as installed. `hash` is `sha256:` over the canonical body; **`body`
  is the hash's preimage, stored beside it** (a planner's-choice
  extension over A17.1's minimum; a CAS kind adds **`rev`**, the page
  revision the baseline was taken at): `pack diff` must render three bodies,
  and in the one cell where diff matters most — both changed — the
  baseline content is recoverable from nowhere else (not the file, not
  the DB, not a git ref the DB is not pinned to). Size is bounded by the
  `content` collection itself.
- **`conflicts`** — the open conflicts, **recomputed every reconcile**
  (never accumulated, so it cannot rot into a stored to-do). A `title`
  conflict (the claim's extent is held by somebody else) sits beside the
  row conflicts, its `dbHash`/`packHash` the two holders' descriptions.
- **`requires`**, **`boot`**, **`maintainers`** (wave 3) — the manifest's
  three declarations **as applied**: what the nightly `reprovision`
  re-grants, what `bootManifest` reads (`[]` on a failed pack — a pack
  that did not install boots nothing), and who a `pack.<id>` diagnostic
  routes to. The record is the deployment's memory of the manifest, so a
  pack that is no longer in the build (its record still here) still
  boots its rows and still has maintainers.

**Canonical hashing.** The preimage is the rendered content only —
`{class, hydratorClass, data}` for the template kind, the bank body for a
bank, `{data}` for a document, `{front, body}` for a wiki page, the
rendered subject shape for a subject — key-sorted, cycle-safe,
`undefined`-normalized (`JSON.stringify`
drops it, so absent-vs-undefined hashes identically on the file side and
the BSON-round-trip side); never `_id`, `path`, `sourcePack`, timestamps,
nothing random. A future field on a row shape is decided at the
strategy's `canonicalBody` site: content (in) or bookkeeping (out).

### The three-way reconcile

For every file and every stamped row of a kind, with the record's
baseline for that key (slate A10.4):

| file vs baseline | DB vs baseline | action |
|---|---|---|
| same | same | nothing |
| changed | same | **update** the row (baseline := file), silently — reported in `updated` |
| same | changed | **keep** the DB — reported in `kept` |
| changed | changed, file == DB | **converged** — baseline := the shared hash, any conflict cleared, no write (this is what closes an `--export` round-trip) |
| changed | changed, file ≠ DB | **conflict** — the row is untouched, a `PackConflict` (`both-changed`) lands in the record, one diagnostic fires, the reconcile continues. Never merge, never block. |

Plus: a **vanished file** deletes a clean row (DB == baseline) and
conflicts (`deleted-vs-edited`) on an edited one — an operator-edited row
is never silently deleted. **Pinned** keys (`record.pins`) are skipped
before any comparison and counted; every reconcile result, boot line,
and `pack status` reports `N rows pinned, skipped` — pins are loud,
every time. A stamped row with **no baseline** (the requires phase's
own pre-written registries — see *The grants*) is normalized from what is
written, counted, and logged. A **file at a key with a row this pack did
not stamp** — another pack's, or nobody's — **fails the pack at
`reconcile`**: the packs are the only writer of these rows, so there is
nothing to adopt and nothing to clobber. A file at a key with no row is
inserted. With **no record** (a fresh database) every row is inserted and
its baseline is what was written; the second boot is a no-op — zero
changes, record hashes unchanged, no line.

**Conflict surfacing.** Each *newly detected* conflict (deduped against
the prior record's set — a persisting one is not re-fired) lands one
`DiagnosticApi.record` — severity `warning`, explicit channel
`pack.<packId>` (the `sandbox.boundary` precedent), the message carrying
the copy-pasteable `pack diff` / `pack resolve` commands. `pack status`
lists every open conflict with the same next-command line. No new
notification machinery.

### Per-pack failure isolation

`install()` (boot) reads every pack first, runs the flat-key check over
the whole install set, then reconciles each pack inside its own
try/catch: a failure records `status: 'failed'` + `failure` on that
pack's row (keeping any prior baselines), logs loudly, and **continues
with the remaining packs** — a failed pack boots *without* the pack; it
never bricks the boot (A17.1 / A10.10). `sync` (one pack, an operator at
the keyboard) throws instead.

#### ⚠⚠ `owned by pack X` when X *is* the pack — two installers, one DB

A reconcile that fails with `E11000 duplicate key`, or with
`pack 'trade-cooking' wants /trade/cooking/... but it is owned by pack
'trade-cooking'`, is **not** a content bug and no amount of reading the
YAML will find it. It means two server processes were installing into the
same database concurrently: each read a baseline the other had not
finished writing, so each sees rows it does not remember claiming. The
message reads exactly like a stamp collision because at the row level
that is what it is.

The fix is procedural, never a code edit: **one database per worktree,
one server on it, drop and boot ONCE.** (Found the hard way by the
food-safety drive, which spent a round chasing it as a pack defect.)

### requires-kernel

Before any write, the installer resolves every distinct `class:` (and
`hydratorClass:`) the pack's content names, via the standard
`StuffApi.loadClassByPath`. A missing class fails the pack (before any
of its writes) with a message naming the pack, the class, and the
offending file. This is the enforced content-pack ↔ mod boundary. The
static half of the requires phase (`gateRequires`, *§ The requires
phase*) runs under the same step name, and the topic gate (leaves under
a core root only; the platform pack is exempt because it *authors* the
root descriptors) runs next, in the same pre-write position.

### The flat-key uniqueness check

Kinds whose keys form a **flat namespace across the install set** (the
banks today; later kinds — emote verbs, recipe ids — plug in by giving
their strategy a `flatKeyOf`) must see each key claimed once. The check
runs after discovery + content read and **before any pack's writes**: a
second claimant — cross-pack, or two files in one pack — marks the
*claiming* pack failed (`failure.step: 'flat-key'`) with an error naming
the kind, the key, and **both** `(packId, relFile)` pairs; earlier packs
in the topo order still install. Never first-wins, never silent (A17.2).
The different-pack-stamp refusal in the reconcile stays as the belt to
this check's suspenders.

## The capability rung — a pack ships classes
### ⭐ …and where a trade's VOCABULARY lives

The same rule one level down: **the kernel keeps no list of a trade's
words.** Three closed tables were opened during the libations review, all
for the same reason — a pack must never need a kernel LIST edit:

| was | is now |
|---|---|
| `ToolCapability`'s closed kinds + a table of the verbs each confers | any kebab string; **the tool row authors its verbs** |
| `Technique`'s `shaken \| stirred \| built \| muddled` + an effect table | any kebab string; **the tool row authors the working and its numbers** |
| `CAPABILITY_TECHNIQUES`, mapping capability → technique | gone; the instrument that performs the working names it |

The tell is the same each time: the kernel naming a *word* only one trade
uses. A capability, a technique, a material tag and a census key are all
open strings two pieces of **content** agree on.

### ⭐ Where a class lives — the same rule as a brain's

> **A class lives in the pack whose content is the only thing that names
> it.** The kernel keeps what several trades name, or what nothing names
> because it is substrate.

The rule already stated for brains (below), applied to classes — which is
what the capability rung is *for*. The test is the row set, not the
inheritance: `GlassRack` was named by hospitality alone and named for a
bar fixture, so it ships in hospitality; `CookPot` likewise in
cooking; `SpiritBottle` and `Still` were always distilling's.

⚠ **One current consumer is not the test — the NAME is.** `Crate` is
named only by produce today, and stays kernel: a crate is a case of
anything, and brewing or bottling would plausibly ship one. `Bottle`
(four packs) and `Tablet`/`Screen` (display substrate) stay for the
ordinary reason.

⭐ A useful tell: **a kernel class only ever imported by TESTS is
suspect.** Both `GlassRack` and `CookPot` had zero production kernel
importers — the only kernel references were tests reaching for shipped
trade content, which `lint:test-content` already discourages. Moving the
class forces those tests onto synthetic fixtures, which is where they
belonged: the kernel knows no rack class, only that the gather walk
descends open `Container`s.


A pack's `src/` has the kernel's taxonomy and nothing else: instanceable
classes under the four branches (`thing/`, `idea/`, `agent/`,
`location/`), controllers at `idea/cmd/<category>/`, brains under
`behavior/` (the Brain category's home inside a pack, mirroring the
kernel's `lib/behavior/` — see [behavior.md](./behavior.md) § Brains in
packs), tests under `__tests__/`, and **`lib/` for substrate that is only
ever inherited** — mixin factories and value objects, the kernel's own
`lib/` rule applied to a pack. No free-floating helpers, no logic
singleton, no Api (a pack that needs an Api needs a kernel MR — the
*mod* rung).

⭐ **The `lib/` ban was lifted by the TPA reform (P2a)**, because it made
a pack-owned mixin **unrepresentable**: a mixin is not instanceable, so
no branch folder is honest for it, and promoting it to the kernel is
wrong for substrate only that pack composes. The tree already carried the
consequence — trade-mining's `WorkingMixin` was parked in the *Location*
branch because invariant 8 left it nowhere else.

The headline invariant is untouched: *nothing instances `/lib/`* is
checked against **template paths** literally starting `/lib/`, and a pack
mixin (`/system/arcana/lib/ManaPowered`) has no template row at all.
`classFileOf` resolves it by longest prefix like any other pack path.

⭐⭐ **The test for the kernel instead of a pack `lib/`: whether the
substrate's composers have a common pack ancestor.** Arcana's lamp and
tpa's terminal do — tpa depends on arcana anyway, it is magic — so
`ManaPoweredMixin` is arcana's. A third pack wanting mana-powered devices
*without* depending on arcana is the signal to promote it, and that is a
review question, not a lint.

### ⭐⭐ How a kernel VERB reaches pack behaviour — the declared shape

> **A kernel verb may not import a pack. It may declare the SHAPE it
> will talk to, and let whatever answers that shape answer.**

The import boundary runs one way (`lint:imports`, and `classFileOf`
resolves a pack root into that pack's `src/` and *never* falls back to
the kernel), so a kernel controller cannot name a pack class. But a verb
often needs to *reach* one: `analyze water` needs the water pack's
conduit, `teleport` needs the tpa pack's terminal. The answer is a
kernel-side interface plus a one-method holder that tests for it
structurally:

| shape | declared in | who answers it | the verb |
|---|---|---|---|
| `SupplyReporting` (`supplyReport()`) | the water pack's own substrate, read structurally | any conduit-like host | `analyze water` |
| `TravelNode` (`ride()` + `renderDepartures()`) | `lib/travel/TravelNode.ts` (kernel) | `FastTravelMixin` (the `tpa` pack) | `teleport` |

`TravelNodes.of(stuff)` is the whole mechanism: a duck-type probe
returning `Stuff & TravelNode` or `null`. The kernel imports nothing; the
pack implements a shape it never imports either — it just has the two
methods. Both sides are checkable, because the shape is a real
`interface` and the pack's tests assert against `ride()`.

⭐ **The forks are the design, not the plumbing.** `teleport` keeps the
forks that are the *kernel's* — free movement inside an extent you hold,
and the anchored spell — and delegates only the forks that are the
*network's* (the timetable, the fare, the clearance, the mana leg). That
split is the test for whether a shape seam is honest: **remove the pack
and the verb must still do something correct**, not merely fail politely.
The `tpa`-less kernel still teleports the people entitled to.

⚠ A shape seam is NOT the general answer to "the kernel needs a pack
thing". It works here because the *capability* is the kernel's (movement)
and only an *implementation* is the pack's. When the capability itself
belongs to the pack, the verb belongs to the pack too — that is the
ordinary case, and `content/<root>/cmd/` is where it goes.
`packages/content/<pkg>/src/<rel>.ts` backs `<root>/<rel>` for every
namespace root the pack holds — source mirrors path, inside a pack as in
the kernel — so `packages/content/arcana/src/thing/Wand.ts` IS
`/system/arcana/thing/Wand`, and a pack controller's `FromModule` gate reads the
same string as its template path.

**The loader.** `StuffApi.resolveClassFile(classPath)` is the one place
a class path becomes a file: a path whose namespace root a capability
pack registered resolves into that pack's `src/` (longest root first,
`<srcRoot>/<rel>.ts`; a missing file is an error naming the pack —
**never a fallback to the kernel**); every other path resolves from the
kernel tree as before. The clone pipeline, `loadClassByPath` and the
brain resolver (`resolveExport`) import the file by absolute `file://`
URL — the shape `HotReloadApi` already used — so Node's cache and the
`?hmr=` override both hold, and `reload /system/arcana/thing/Wand` reloads the
pack file. Prod and dev are both `tsx` from source; the server's
`tsconfig.json` `include`s `../content/*/src/**/*` so tsx applies
`experimentalDecorators` to pack files (tsx honours a tsconfig only for
files it includes), and `pnpm build` typechecks them. The call-security
loader transform stamps `packages/content/<pkg>/src/**` like `mud/**`,
importing the registry by **package specifier**
(`@saxonberg/server/mud/api/module`); `ModuleApi` normalises a pack file's
URL through the class-source table to `/<root>/<rel>` before the kernel
root hints. **Pack code writes absolute gate strings** — the transform
cannot know a pack's root, so a relative `FromModule('./x')` in a pack
file is left alone and `lint:gates` reports it.

**The pack import profile is the server's `exports` map.** A pack
imports the kernel **only by package specifier** —
`@saxonberg/server/mud/lib/stuff/Thing`, never a relative path into the
monorepo — and `packages/server/package.json`'s `exports` map is the
author surface: `mud/lib/*` (mixin factories, value objects, abstract
roots), `mud/api/*` (the facades, single level — `api/mql/**` and
`api/mml/**` blocked), `mud/platform/{thing,idea,agent,location}/*` with
`platform/idea/api/**` (the logic singletons) and `platform/idea/hooks/**`
blocked, `test-bootstrap`, and the vite plugin. Nothing else is
exported — not `backend/`, not `world/**`, not `config/` — and Node,
tsx, vitest and tsc all refuse a subpath the map does not list, so the
boundary is enforced by the package, not by a script's opinion; widening
it is a deliberate, reviewable edit to one file. `lint:imports`' **pack
tier** checks what the map cannot see: a relative escape out of the
pack's `src/`, a pack-to-pack import without the dependency line, any
other package or Node built-in.

**The rung check** (`requires-kernel`, before any write). The installer
resolves every class a pack's rows name and records where it resolved
(`PackReconcileResult.classOrigins`; the boot line prints a capability
pack's). Keyed on **resolution origin, not path prefix**:

1. a class in the pack's own namespace that resolves nowhere, from a
   pack with no `src/`, fails — *"claims data but ships code"* — while a
   data pack whose rows name parked **kernel** classes under its own
   `/world/<x>` claim (`saxonberg-lounge`, `hearthworks`) passes;
2. a class that resolves into another pack's `src/` requires that pack
   in the importer's derived `dependsOn`, else fails naming both;
3. a pack with `src/` records `rung: 'capability'` (else `'data'`), and
   after the install every `src/` class no row of any installed pack
   names is **reported** (console + a diagnostic) — dead code in a pack
   is a review finding, never a failure.

**Code and the record.** The record stores `codeVersions` (`src/`-relative
file → sha256 at install). `pack status` prints `capability | data`,
the derived `dependsOn`, and `code: current | stale — restart owed` by
comparing disk to the record; `pack sync <id>` first hot-swaps every
changed `src/` file (`HotReloadApi.reload`, the same machinery as
`reload`) and then re-hydrates the rows, so a re-hydrated clone gets
the new class. In prod a code change rides deploy + restart like the
kernel's; `status` says when one is owed.

**Two rules the rung applies to magic.** *A discipline row ships with
the pack whose code derives or teaches its key* — the 18 `magic-*` rows
are arcana's (its `Grid` derives them); the trades' stay in the
platform until their pack reaches the rung. And *arcana holds what other
packs' content names; a class only one pack's own rows name is that
pack's* — the glowlight mote and the spark locus are the arcane
library's, named by its spell rows' `locus:`. The catalogues warm **by
class** (`Template.findByClass`), never by root prefix, so a second pack
shipping a discipline or a spell needs no kernel edit. One ratified
exception to *a kernel module never names a pack's row*: a kernel mixin
that declares a capability names that capability's command view
(`Caster` → `arcana/cmd/magic/cast.yaml`, `Charged` → `zap`/`recharge`)
— the view's key is its document path, and the affordance belongs where
the capability is declared.

**Tests travel with the code.** A pack's `src/**/__tests__/` runs under
its own `vitest.config.ts` (`pnpm -r test` picks it up; `pnpm test:near`
routes a changed pack file to the pack's suite), imports
`@saxonberg/server/test-bootstrap`, and is held by `lint:test-bootstrap`;
`lint:test-content` treats it as a content test beside its content. The
lint family walks pack `src/` throughout (`scripts/pack-roots.ts` is the
scripts' shared table): `instanceable` (invariant 3 through the table,
invariant 8 — every module under a branch, `behavior/` or `lib/`, and a
`lib/` module that is only inherited substrate), `gates`,
`imports`, `module-scope`, `field-meta`, `blessed-bands`, `arg-kinds`
(every `cmd/` under a pack's content), `untitled` (`/system/arcana` is a title
root because arcana claims it — the roots derive from the claims).

## The two entry points

### Boot — `PackApi.install()`

`AppBootstrap.run` calls `PackApi.install()` in the pre-`loadHooks`
region. It **writes rows only** —
`BootstrapManager` clones singletons *afterwards*, so nothing is live yet
and there is no re-hydrate at boot.

Ordering: `PersistenceManager.connect` (the indexes) →
`PackApi.install()` → `loadHooks` → `CommandApi.preloadAll` →
`BootstrapManager.run()` (the boot union, below) → the Api boots. The
install runs before `loadHooks` because everything — the DomainHook
template, the marshallers, the quantity table, the registries the
catalogues warm from — *is* the packs' content now; there is nothing
else that writes a row at boot. The boot line per pack reads
`PackApi: '<id>' installed — N inserted, … , requires: G group(s) (C
created), T title(s) (granted, kept, conflict)[, S row(s)
skipped (extent sold)], boot: A sync-read + B producer, staffed|UNSTAFFED`
— the second boot of a settled deployment is all zeros on every pack.

#### The install read window

A pack's plan needs the rows THAT pack stamped, and the natural query
says exactly that: `{ sourcePack, ...stampedQuery }`, one round trip per
(pack, kind). On the shipped world that is **thirty-one packs times
eight document kinds = 248 queries** against a collection of a few
hundred rows, plus a per-pack read of `content`, `forum_subjects` and
`descriptor_banks`. With the template cache in
([persistence.md](./persistence.md)), it was the largest remaining cost
of a boot.

Inside one `install()` the `sourcePack` term moves out of the query and
into a filter: each `(collection, stampedQuery)` is read **once** and
each pack takes its own slice in memory. 200 queries become 8.

It is safe for a rule the installer already enforces: **a row is owned
by exactly one pack**, so what pack A writes can never appear in pack
B's slice, and no pack is reconciled twice in a window. Outside a window
— an operator's `pack sync` or `pack diff` — nothing changes and the
read goes to Mongo.

⚠ **Two per-pack reads stay.** The **settings singleton** is *written*
during the install, so a pack may legitimately need a value an earlier
pack merged; caching that is a correctness question, not a performance
one. The **subject** kind's `loadStamped` mints across collections and
renders its own preimage.

⚠ **The installer's content WRITES cannot be batched the same way.**
They look like a serial loop worth parallelising (987 saves × ~33 ms on
a fresh boot) but they are order-dependent *through the hook chain*:
`content`'s around-save hook walks the row's ancestors for the
folder/leaf invariant, and `validateSingletonContainerTarget` requires
the row named by `data.container` to already exist — both read rows an
earlier save in the same install just wrote. Batching means validating
serially and bulk-writing after, which is a change to the hook contract.

### Runtime — the `pack` verb

`platform/idea/cmd/author/PackController.ts` + the platform pack's
`content/platform/cmd/author/pack.yaml` (declarative `subcommands:` + `options:`;
afforded on `AuthorMixin`'s operator surface, **authorized by
`requiresPackInstaller`** — `AccessApi.canAtPath(giver, 'install',
'/compact/executive')`, title over the executive, never the wizard axis;
see below):

- `pack status [<packId>]` — joins the discovered manifests with the
  records: status, version, applied-at/by, failure, **the maintainers
  line** (`maintainers: <group|organization> — staffed` /
  `UNSTAFFED — routes to the executive`), any **title conflict** (an
  extent the pack claims that somebody else holds), every open conflict
  with its next command, and the pin line — always, even at zero. A pack
  with a record but not in this build is headed `(NOT in this build)`.
  After the packs, one line for the **orphans** (D9): `N template row(s)
  under no pack: …` — every `content` row with no `sourcePack` stamp,
  listed, never deleted (`PackApi.orphans`).
- `pack install <packId> --dry-run` — the exact change set a sync would
  apply, zero writes. `install` **without** `--dry-run` is rejected:
  boot installs; a live apply is `sync`'s job (staging is a non-goal).
- `pack sync [<packId>]` — the **same** reconcile, then re-hydrate: for
  each changed path it calls `TemplateApi.restoreFromTemplate` on every
  live instance at that path (the `CmsLogic._writeContent` go-live
  pattern); for each deleted path it destructs the orphaned instance.
  The quantity kind uses `reloadTagTables` (diff-apply); a changed bank
  drops its cache. The document kinds' go-live is one module-private
  switch (`invalidateDocumentKind`): `emote` drops a resident
  `SoulCatalogue`'s cache, `recipe` re-warms a resident
  `RecipeCatalogue` (its reads are sync), `name-bank` clears
  `NameBank`'s cache, `blueprint` re-warms the `BlueprintCatalogue`,
  `command-view` reloads each changed view (`CommandApi.reload`) and
  invalidates each deleted one, `msh` needs nothing (scripts resolve by
  path per call). Settings re-warm `AppSettings`; subjects drop the
  resident subject/channel catalogues; the wiki needs nothing.
- `pack diff <packId> [<path>]` — the wiki three-body shape, three
  labeled sections: *baseline (as installed)* / *yours (database)* /
  *theirs (pack file)*, each with its hash. No machine merge. With no
  path, every open conflict. For a settings file *yours* renders the
  singleton's values for the file's keys; for a wiki page the live page.
- `pack provision <packId>` — the pack's requirements **as the
  registries hold them now**: the maintainers and their members, each
  declared group's member count, and each claimed extent with its
  current holder and an outcome — `held` (the claim's holder), `unheld`
  (no parcel at exactly that extent, or nobody holds it), `conflict`
  (somebody else). The read-side twin of the requires phase.
- `pack resolve <packId> <path>` with **exactly one** mode:
  `--take-pack` (write the file's row, rebaseline, clear, re-hydrate);
  `--keep --pin` (keep the DB row AND claim it — pinned rows never
  compare again; **`--keep` without `--pin` does not exist** — keeping
  means claiming); `--export` (serialize the DB row back to the pack's
  **workspace** source file; the conflict stays open until the next
  `sync` observes file == DB, the converged cell, and clears it — the
  git round-trip). A `sync` of an **unstaffed** pack ends with the
  staffing prompt (D7): *This pack has no maintainers. You, or who?* —
  `PromptApi.text`, enter for yourself, a name for somebody else, decline
  and it stays unstaffed (and `status` says so). `PackApi.staff` adds the
  member to the maintainers **group**; an organization-maintained pack
  refuses (*appoint through the organization, not the pack*). `--export` writes are workspace-only by construction;
  in a dist deployment the write fails loudly. A text kind exports its
  text verbatim — an `.msh` its source, a `.md` its frontmatter + body —
  not YAML.
- `pack pin` / `pack unpin <packId> <path>` — direct pin management. A
  pin outside a conflict is a legitimate proactive claim; an unpin
  re-compares on the next reconcile and may immediately surface the
  conflict the pin was hiding, which is correct.

This is the iteration loop: edit a pack file → `pack sync` → live, no
restart. It is tractable for base-library because materials/biomes are
**singletons by path** (one instance per path; `Tangible.getMaterial()`
re-resolves the material singleton lazily on each read, so every object
made of it sees the new values immediately). Cloned (non-singleton)
content — newbie-wilds' rooms and NPCs — re-hydrates through the same
`findAllByTemplatePath` walk, but live players standing in a re-hydrated
room is lightly-trodden ground: **restart remains the universal go-live**
(A10.9).

### Who may run it — the executive

Installing content is the executive's work. The platform pack is
maintained by the **organization** `/compact/executive` (its own row,
booted as a `producer`), and `requiresPackInstaller`
(`lib/command/validators/`) gates the verb on
`AccessApi.canAtPath(giver, 'install', '/compact/executive')` — the
ordinary title walk: an organization-held extent admits whoever holds a
position in the organization (`EmploymentApi.holdsPosition`) or heads it
(`holdsAuthority` — the Prime Minister, founder default included; see
[access.md](./access.md)). So the founder installs as head of the
executive with **no group membership at all**, and the PM staffs the
office with the ordinary `appoint`; handing the seat hands the platform
with no data migration. No `isWizard` anywhere in the pack path (the
code-trust axis is the wrong axis — a **wizard who holds no position in
the executive is refused**); a non-holder gets a diegetic decline; a
non-resident organization fails closed with one diagnostic.

**A live `sync` also checks the claims.** Bootstrap is exempt from the
*precondition* (who may claim), never from the checks; a person syncing
a pack must hold `write-template` title over every extent the pack
claims (`canAtPath`), or the sync fails at `requires-kernel` naming the
extent.

## The requires phase

A pack's rows need three things the rows themselves cannot say: the
**groups** its content names, **title** over the extents it ships into
(so somebody can edit, broadcast over, teleport within, and clone under
them — with `core` gone an untitled path is a path *nobody* can act on:
`ParcelApi.ownerOf` answers `null` and every `can` there fails closed),
and a **maintainer** to route its diagnostics to. The manifest declares
them; the installer provisions them, adopt-by-name throughout.

### The declarations

- **`requires.groups[]`** — `{ name, purpose, owner?: { office },
  members?: [{ id, role? }] }`. `purpose` is mandatory prose (the group's
  reason to exist, shown nowhere yet, refused when blank). `owner`
  names the office whose holder owns the group (the platform's `soul`
  committee is `{ office: prime-minister }`); absent, the group is
  system-owned. `members` may **only enrol NPC rows the pack itself
  ships**, under an extent the pack itself claims, in a group the pack
  itself declares — the NPC-only membership fence (a pack can never
  enrol a player, nor reach into another pack's staff). `world-seed`
  enrols Katie in `duncan-hall`.
- **`requires.title[]`** — `{ extent, holder?, landUse?, areaM2?,
  parentParcel? }` — a `TitleClaim`. `holder` is `{ group: <name> }` or
  `{ organization: </path> }`; **absent, the maintainers hold it** (the
  corpo packs claim `/corpo/<key>` for the corpo organization by
  default). `landUse` must be one of the closed land uses; `areaM2`
  positive; `parentParcel` an absolute path (the Hinkley Hills lot under
  its estate). **There is no implicit root claim**: every claim is an
  explicit entry — the platform's own extents are nine explicit lines
  (`/platform`, `/stuff`, `/blueprints`, `/compact`, `/studio`,
  `/home`, `/world`, and `/wiki` for `wiki-editors`), and two packs may
  name the same extent for the same holder (until wave 4b `world-seed`
  and `saxonberg-lounge` both claimed `/world/lounge` for `lounge`: the
  second is `kept`).
- **`maintainers`** — a group name, `{ group }` or `{ organization }`;
  default **`<id>-maintainers`**, a group the installer mints
  **PM-owned** (`{ office: prime-minister }`) with zero members —
  UNSTAFFED until somebody staffs it.

### The gate (static, before any write)

`gateRequires` runs at `gatePack` under the `requires-kernel` step, over
the whole install set:

1. a `{ group }` holder is declared under `requires.groups` by this pack
   **or a host** (a transitive `dependsOn` pack); `expression` claims
   `/expression` for `soul`, which the platform declares;
2. an `{ organization }` holder or maintainer is a **row this pack or a
   host ships** (`/compact/executive` is the platform's row;
   `/corpo/aevex` is corpo-aevex's own);
3. the NPC-only membership fence (above);
4. **coverage** — every path the pack ships under a **title root**
   lies under a claim of the pack **or a host** (template paths,
   document paths and wiki pages alike). The title roots are
   **derived, never listed**: the first segment of every extent any
   pack in the install set claims (`titleRootsOf`; `lint:untitled`
   mirrors it over the manifests). A root nobody claims is a place no
   title reaches, so nothing under it can be untitled; a root anyone
   claims is a place, and everything under it must be covered. `/trade/<industry>` (wave 4a) is an
   industry pack's root: what the trade **introduces** — its stations
   and stock under `/trade/<industry>/<branch>/`, its controllers under
   `…/command/`, its recipes at `/trade/<industry>/recipes/<id>` (the
   document root is the pack root), its verbs at `/trade/<industry>/cmd/`
   (reserved; none ship yet); `lint:instanceable`'s invariant 7 keeps
   an instanceable row under the industry's `obj/` or `command/`
   segment. What a trade merely *uses* (fire stations, a cut of meat) is
   commons under `/obj`; where it is *practised* (the smithy) is a venue
   under `/world/`. `base-library`'s rows under `/obj` ride the platform's `/obj`
   claim; `generic-objects` claims its twelve `/stuff/<branch>/<cluster>`
   branches itself; `wiki-starter`'s pages ride `/wiki`. A pack's own
   document root *outside* the title roots (`/expression`,
   `/generic-objects`) is the pack's to claim or not. A pack whose whole
   host chain claims **nothing** is pre-wave-3 shaped and passes
   **vacuously** — `pnpm lint:untitled` is the static gate that keeps
   the shipped set at zero untitled paths.

### The grants (before planning)

`applyRequires` runs **first** in `reconcilePack` — before the planner —
so a title this claim grants is in place before the bounded
reconcile asks who holds each row's extent:

1. **Groups** — the maintainers group first (PM-owned), then each
   declared group, through `GroupApi.ensureGroup(name, owner)`:
   **adopt-by-name** — an existing group is found and **never
   re-owned**; a missing one is created (`groupsFound` / `groupsCreated`).
2. **Memberships** — `GroupApi.ensureMember(ref, id, role)` (gated to
   `PackLogic`; idempotent; `membersAdded`).
3. **Titles** — `ParcelApi.grant(claim)` (`ParcelRegistry.grant`, the
   installer's title seam), one of three outcomes: **`granted`** (absent
   → the row + a `grant` event), **`kept`** (present under the same
   holder — no write, no event), or **`conflict`** (present under a
   different holder — *whoever* that is — no write; a `title` conflict
   on the record and a `title conflict:` line in `status`). There is no
   migration outcome and no migration code anywhere in the boot: a
   database that predates a rename is **dropped**
   (deployment.md § The Mongo environment policy).

Then the rows are planned and written, and `finishRequires` runs after
them: the organizations the manifest names are **stood up resident**
(`StuffApi.singleton` mints-if-absent — the requires phase runs after
the rows are written and before `BootstrapManager` clones the boot
union, so `/compact/executive` is a row but not yet a Stuff, and an
organization-held title needs a resident organization to answer
`holdsPosition`); the record takes `requires` / `boot` / `maintainers`;
the boot line takes the role counts and the **staffed** bit.

**Staffed** means somebody actually holds the maintainer: a group with
at least one member, or an organization with at least one filled
*position* — **the head alone does not count**; an office with no staff
is unstaffed (the platform boots `UNSTAFFED` on a fresh world until the
PM appoints someone).

### The bounded reconcile — `skip-sold`

Title can move after a pack installs — the executive sells a branch, a
locality changes hands. The three-way reconcile is **bounded by title**
(CPS:308): `soldPredicateFor(pack)` computes the pack's **holder set**
(its maintainers, its own claims' holders, and its hosts' — the whole
`dependsOn` chain), and a domain row whose covering parcel is held by
nobody in that set was *sold out from under the pack*: the planner
emits **`skip-sold`** for it — skipped and counted (`skippedSold`, the
boot line's `N row(s) skipped (extent sold)`), **never written**, never
deleted. No resident registry → unbounded (a unit test).

### Staffing and routing

- **`pack.<id>` diagnostics** (a conflict, a sale, a failure) route to
  the pack's maintainers: a staffed group's members (`GroupApi.membersOf`),
  or a staffed organization's committee — else, unstaffed, to **the
  executive's committee** (`CompactApi.committeeMembersOf('/compact/executive')`).
  `DiagnosticLogic.packRecipients` reads `PackApi.maintainersOf`, which
  answers from the record (or the manifest, pre-install) with the
  `staffed` bit and the executive as the declared `fallback`. The
  `errors` list shows a pack's channel to whoever maintains that pack —
  no author tier.
- **`pack status`** prints the staffing line per pack; **`pack sync`**
  prompts to staff (above); `PackApi.staff` is the one write.

### Orphans

`PackApi.orphans()` — every `content` row **with no `sourcePack`
stamp**, sorted: seed inventory nobody claims, a CMS-authored template,
a row from a pack no longer in the build whose record was lost. Listed
by `pack status` (D9), **never deleted** — a row is somebody's until
somebody says otherwise.

### The nightly reprovision

`RecordApi.wipe` (the nightly reset, [record-layer.md](./record-layer.md))
takes `groups` and `parcels` with the rest; what re-seeds them is
**`PackApi.reprovision()`** — for every `applied` record, the same
`applyRequiresFor` over the record's stored `requires` + `maintainers`
as `bootstrap` (groups re-minted, titles re-granted or kept, one line
each). The seeders' boot-only mint is gone; the record is the memory.

## The boot union

What is eager at boot is declared by **the pack that ships the row**.
Each manifest's **`boot[]`** entry is `{ template, role, reason,
dependsOn? }`: `template` an absolute template path the pack ships;
`role` **`sync-read`** (a registry or catalogue some Api reads
synchronously — it must be resident before the first read) or
**`producer`** (a row whose `postRegister` *produces* something — warms
a cache, installs a stair, rebuilds a floor — or that nothing else would
ever instantiate: `/platform/location/void`, the TPA network's eager root);
`reason` mandatory prose (refused when blank — the manifest is where the
*why* lives, next to the row, readable in `git blame`); `dependsOn`
other boot templates that must clone first (`/platform/idea/ChannelCatalogue`
after `/platform/idea/SubjectCatalogue`, `/platform/idea/PressBoard` after both
organizations).

`BootstrapManager.run()` with no argument reads **`PackApi.bootManifest()`**:
the union of every **applied** record's `boot[]`, in install order
(shipped packs in topo order; under `SAXONBERG_PACKS` only the filtered
packs — a filtered-out pack's record is ignored, not carried forward;
unfiltered, a recorded-but-no-longer-shipped pack boots last, sorted).
A template two packs both list is an **error naming both** — one row,
one declarer. The code manifest (`mud/bootstrap.ts`) is **gone**; the
manager owns only the `BootstrapEntry` shape (`templatePathPrefix` /
`awaitInit` stay code-only and no YAML exposes them). A singleton
already resident when the union runs (a lazy `StuffApi.singleton` mint
earlier in the boot — the installer standing an organization up) is
**reused**, never cloned twice.

The platform declares the registries and catalogues (`/platform/idea/EventRegistry`
first — every emit resolves it; `/platform/idea/AccessRegistry`,
`/platform/idea/ParcelRegistry` after `/platform/idea/GroupRegistry`; the scheduler chain
under `/platform/idea/WorldClockRegistry`), the two organizations
(`/compact/executive`, `/compact/press`), `/platform/idea/PressBoard` and
`/platform/location/void`; each corpo pack its own organization;
`saxonberg-lounge` the TPA root (`/world/lounge/thing/terminal`);
`world-seed` the three remaining locality producers (the Duncan Hall
dorm-warren, the Hinkley Hills plat-book and lot-holder). `pnpm
lint:instanceable` checks each entry names a real row.

## Pack zero — the platform-only boot

**`SAXONBERG_PACKS=platform` is a bootable world**, and it is checked
every pipeline: `e2e/playwright.platform.config.ts` boots the server
with that filter on its own ports (2011 / 5174, stdout captured to a
log), and `tests-platform/platform-only.spec.ts` proves the founder — as
head of the executive, a member of no group — logs in and lands in the
shell room (`/platform/location/void`, the code fallback when no pack contributed
`defaultStartLocation`: that setting moved out of the platform into
`saxonberg-lounge`), that `pack status` knows exactly one pack **in this
build**, and that the boot logged no `error` / `failed` line. The config
mints its own founder handle (`e2e-platform-founder`, purged by
teardown) because a character minted under the full-pack world carries
a species the platform does not ship. CI runs it **first** against the
fresh `saxonberg_e2e`, then the main suite — same database, sequential
(the four-database rule, deployment.md). Root
`pnpm test:e2e:platform`; `e2e` `pnpm test:platform`.

One more CI gate keeps the wave's invariant: **`pnpm lint:untitled`**
(every path the packs ship under a title root has a claim as a prefix —
the installer's walk mirrored in a script; zero is green).

## The shipped packs

| Pack | `dependsOn` | Maintainers | Claims (`requires.title`) | Groups | `boot` |
|---|---|---|---|---|---|
| **platform** | — | organization `/compact/executive` | `/platform`, `/stuff`, `/blueprints`, `/compact`, `/studio`, `/home`, `/world`; `/wiki` → group `wiki-editors` | `wiki-editors`; `soul` (PM-owned — the soul committee) | 32 entries: the registries + catalogues (sync-read), the two organizations, `/platform/idea/PressBoard`, `/platform/location/void`, `/platform/idea/BlueprintCatalogue`, `/platform/idea/HelpCatalogue`, `/platform/idea/AddressRegistry` (producer) |
| **base-library** | platform | `base-library-maintainers` (default) | — (rides `/obj`) | — | — |
| **species-and-names** | platform | default | — (rides `/obj`) | — | — |
| **arcana** (CAPABILITY — magic's substrate) | platform | default | `/system/arcana` → group `arcana` (PM-owned): `src/` ships Wand, Scroll, Spellbook, Conduit, Ring, Amulet, Potion, PotionMaterial and the five casting controllers; rows: the 18 `magic-*` disciplines, the five controller templates, the five views, `settings/magic.yaml`, the six descriptor banks | `arcana` | — |
| **arcane-library** (CAPABILITY — magic's catalog) | platform, arcana | default | — (rides `/stuff`): the 12 spells, the 13 items at `/stuff/thing/magic/` (wands, scrolls, books, a conduit, the bench, three potions, a ring, an amulet), the two loci `glowlight-mote`/`spark-locus` whose classes `src/` ships, the three draughts | — | — |
| **generic-objects** | platform | default | twelve `/stuff/<branch>/<cluster>` branches (the magic items left for the arcane library): `items`, `arms`, `armor`, `clothes`, `gear`, `vessel`, `fixture`, `instrument`, `traps`, `surface`, `exits`, `room` (wave 4a: the hearthworks commons — cuts, roots, rations, hide, logs — moved into `/stuff/thing/items`; wave 4b: it ships **no recipes** — every recipe is a trade's) | — | — |
| **trade-smithing** (CAPABILITY — libations) | platform, generic-objects, trade-cooking (the cook-pot recipe's output row) | default | `/trade/smithing` → group `smithing` (PM-owned); `src/idea/cmd/crafting/` ships `forge`/`hammer`/`quench`/`sharpen` (views under `content/trade/smithing/cmd/crafting/`); the anvil and whetstone rows author the verbs they confer | `smithing` | — |
| **trade-cooking** (CAPABILITY + the pantry, libations; the cooking build) | platform, generic-objects, base-library, distribution, trade-farming (the roster presses olives and stews orchard fruit) | default | `/trade/cooking` → group `cooking` (PM-owned); `src/idea/cmd/crafting/` ships `cook`/`plate`; the `cook-pot` row (over `CookPot extends CraftVessel` — the pot is a member of the dish pool, which is what makes pot-as-last-resort a claim) and the `kitchen` bundle; **fourteen recipes** across the method grid — wet (boiled-roots · root-mash · hearty-stew · stewed-orchard-fruit · clear-broth · simple-syrup), dry (toasted-ration · roasted-roots · hearth-roast · fine-roast) and the fat chain (render-tallow · press-olive-oil · pan-fried-roots · crisp-fried-cutlet); the two fats (`tallow` 478 K · `olive-oil` 463 K) with their crock and bottle; the kitchen sieve and fruit press; the pantry: `sugar`/`salt`/`coffee`/`simple-syrup` materials, the `sack` preset, sacks + a `syrup` floor row, the `syrup-bottle`, the pantry outfit (`pantry-outfit` + `pantry-floor` + a consigning hand) | `cooking` | the pantry floor + outfit (producer) |
| **trade-distilling** (CAPABILITY — libations) | platform, base-library, generic-objects, corpo-veshko, corpo-hollis, distribution | default | `/trade/distilling` → group `distilling` (PM-owned): `src/` ships `SpiritBottle` (a `Bottle` preset in code) and `Still` (the furnace-family station, capability `still` — the distil/compound/brandy/grappa recipes run it daily since fermentation W6); ten spirit materials tagged by category; ten generic floor bottles at target + Crowsfoot (`gradeBand: fine`, the independent Brand row at `/stuff/idea/corpo/Brand/crowsfoot-gin`); two floor outfits with `consigns` hands (→ the **distribution** counter — the cash-and-carry moved out, fermentation D10); the **two corpo-owned yards** as the `location/veshko-yard` locality (the zone that authors `stocks:` — vodka 24, whiskey/rum/gin 12; Volk + the unbranded liquid) and the flat `hollis-*` rows (the bottling floor: `old-hollis` / `hollis-cane` over Veshko's material), each an outfit whose `parentOrganization` points up at its corpo pack — a corpo pack is capital + the mark, never products; the `warehouse` bundle; the `distilling` Discipline; the working still-house (tun, standpipe, wash vats, the still book) + the small still (the homebrew kit's) | `distilling` | the four rooms + the four Businesses (producer — the cash-and-carry's pair moved to `distribution`) |
| **distribution** (fermentation D10 — the decoupler) | platform, base-library | default | `/trade/distribution` → group `distribution` (PM-owned): the cash-and-carry (hall, racking, counter, Tam Ferrier, the independent Business) moved out of trade-distilling — a distributor is distribution infrastructure, so every producing trade and venue points HERE and the sibling trades share no edges; plus the **malt line** — malt the material is base-library's (the commons — a shared input whose owning trade does not exist yet), the sack row + its par on the counter the honestly-labelled imported-input faucet | `distribution` | the hall + the Business (producer) |
| **trade-winemaking** · **trade-brewing** (DE-STUBBED — fermentation W4/W5/W8) | platform, base-library, distribution | default | `/trade/<x>` → group `<x>` (PM-owned): the WORKING venues over the kernel ferment — winemaking: the 285 K winery (press, vats, conditioning bottles, the cellar book), crush recipes + pomace, wine/must/vinegar/lees materials, the red/white/sparkling-conditioning/wine-culture profiles, the vermouth fortifications (bought spirit — the B2B); brewing: the 288 K brewhouse + the 279 K cold store (the lager line), mash recipes + spent grain, the ale/lager/cask-conditioning + culture profiles, the cask as a Vat-family conditioning vessel (real ale). Floor faucets RETIRED (the switchover): vessel faucets (`wine-bottle`, `keg`) supply empties and the `cellars` beats make every drop; serving recipes stay. | `<x>` | the floor(s) + outfit (producer) |
| **trade-bottling** (STUB trade — libations) | platform, base-library, distribution | default | `/trade/bottling` → group `bottling` (PM-owned): everything downstream of production and nothing of production — the sodas' materials (`carbonated`; `ice` = frozen water with `density`/`meltingPoint`/`latentHeatOfFusion`), vessel presets (`can`, `mixer-bottle`, `ice-bag`), the floor product at target, the outfit trio on `consigns` → the distribution counter, `soft-drink`. Forced carbonation is its future de-stub (D11). | `bottling` | the floor + outfit (producer) |
| **trade-farming** (libations) | platform, base-library, distribution | default | `/trade/farming` → group `farming` (PM-owned). ⭐ **A trade is a PROCESS, and produce is one of farming's OUTPUTS** — so this pack owns BOTH halves, as trade-distilling owns the still and the bottle, and it is **not** a stub. **Production:** the growing apparatus the husbandry/smallholding substrate drives — `thing/{pot,bed,seed,plant,crop}/` (drained from generic-objects; a pot is horticulture's vessel the way a keg is brewing's, and a dorm owning one no more makes pots household content than the bar owning a keg makes kegs hospitality's). **Output:** ten food materials tagged by category (the eight bar fruit + grape and juniper, farming A5), each a graded `Provision`, the `crate`/`basket` presets over `/platform/thing/Crate` (the fruit are `Provision` rows the crate props), a crate of each at target in `farm-stock`, and the `farm-outfit` + `farm-hand` on the pack's **own `farms` brain** (`src/behavior/farms.ts` → `/trade/farming/behavior/farms`, the first shipped pack brain — tends, picks, consigns → the distribution counter). The **domesticated species rows ship here too** (the trade ships what it domesticates — eleven plantae rows under `content/stuff/idea/species/`). Horticulture is farming too: the ornamentals (peace lily, snake plant) are grown here rather than filed under food. No serving recipe — produce is an input (`press` is hospitality's) | `farming` | the farm + outfit (producer) |
| **trade-hospitality** (CAPABILITY — libations) | platform, base-library, generic-objects, the five stubs, cooking | default | `/trade/hospitality` → group `hospitality` (PM-owned): `src/` ships `IceBin` (an insulated Thermos of ice) and `Tap` (a Surfaced fixture that is a `tap` tool); the bar tools (muddler, bar-spoon, strainer, juicer), the stations (tap, ice-bin, water-tap, basin, glass-rack, well, house-tablet), the nine pool glasses over `CraftVessel` (coupe = cocktail-glass renamed) + the juice bottle, the four house-made juices; 21 cocktails + coffee + four presses; the `bar` and `cellar` bundles; `archetypes/hospitality.yaml`; `src/idea/cmd/crafting/` ships the bar's own steps `muddle`/`strain`/`garnish`/`mix`/`serve` (the shaker/mixing-glass/muddler rows author the verbs they confer); `menu.test.ts` materializes a venue from the archetype and orders all 24 lines | `hospitality` | — |
| **expression** | platform | group `soul` | `/expression` → group `soul` | — | — |
| **wiki-starter** | platform | default | — (rides `/wiki`) | — | — |
| **corpo-{aevex,goodkin,vionne}** | platform | organization `/corpo/<key>` | `/corpo/<key>` (holder = maintainers) | — | `/corpo/<key>` (producer) |
| **corpo-veshko** (libations: the one corpo that MAKES — in the TRADE pack) | platform | organization `/corpo/veshko` | `/corpo/veshko` only — capital + the mark (Volk's Brand row, the chart); Veshko's yard is `trade-distilling`'s `location/veshko-yard/` locality, owned via `parentOrganization` | — | `/corpo/veshko` (producer) |
| **corpo-hollis** (libations: PRIVATE-LABELS Veshko's liquid — in the TRADE pack) | platform | organization `/corpo/hollis` | `/corpo/hollis` only — capital + the mark (Old Hollis, Hollis Cane; the chart); the bottling floor is `trade-distilling`'s flat `hollis-*` rows, owned via `parentOrganization` | — | `/corpo/hollis` (producer) |
| **newbie-wilds** | platform | default | `/world/newbie-wilds` → group `newbie-wilds` | `newbie-wilds` | — |
| **saxonberg-lounge** (the lounge, whole, since wave 4b; **stays a DATA pack** — its classes are parked kernel classes under `mud/world/lounge/`) | platform, corpo-goodkin, corpo-vionne, corpo-aevex, corpo-veshko, distribution | group `lounge` | `/stuff/idea/lounge`, `/world/lounge` → group `lounge`: the venue rows under `/world/lounge/{location,thing,idea,agent}` + the FolderZone, the library root, the three `msh`, the landing setting. Libations: the `Bar` populates **no bottle** (the four `*-bottle` rows deleted) — it lists the hospitality `bar` bundle's fixtures + the house tablet (`principal: /world/lounge/idea/business`); the sports **booth** with a `Screen` + `Remote`; `business.yaml` gains the `keeper` position (`purchases: true`) and 45 `parLines`; Mara runs `restocks`; Dave's dialogue appoints the keeper; the menu offers 26 lines | `lounge` | `/world/lounge/thing/terminal` (producer — the TPA network's eager root) |
| **hearthworks** (a VENUE pack, wave 4b) | platform, trade-smithing, trade-cooking, corpo-goodkin | default | `/world/hearthworks` → group `hearthworks` (PM-owned): the `/world/hearthworks` CartesianZone + 12 rows under branch subdirs — the four rooms + the forge floor + `offstage`, the business, Berta and Odo, the two menus, the pantry; every station and recipe a `props:` reference to a trade's or the commons' row | `hearthworks` | — |
| **world-seed** (TRANSITIONAL remainder — moor, practicum, substation, common) | platform, saxonberg-lounge, corpo-goodkin, corpo-vionne | default | what has not yet been homed in its own pack: moor, practicum, substation, the common TPA travel card, the commons `stuff/` reference rows. Declares nothing — rides the platform's `/world` + `/stuff` claims | — | — |
| **terminus** (the CORE CITY locality, residences D18) | platform, saxonberg-lounge, residence, trade-farming, corpo-goodkin | default | `/world/terminus` homed whole out of world-seed: University Avenue, the terminal, Counting-House Row, the general store, the Registry, Mayfield Row + the city budget, the farmers market — content and parked `src/` together; municipal parcels + land uses | `terminus`, `mayfield-holdings` | Seznick House's building (producer) |
| **eternal-university** (the CAMPUS locality, residences D18) | platform, residence | default | `/world/eternal` homed whole: Duncan Hall (the residence ladder's first rung), Katie + her kit, the dorm rows; parked `src/` registers against the extent so `/world/eternal/…` class paths are unchanged | `duncan-hall` (enrols Katie) | the dorm-warren (producer) |
| **hinkley-hills** (the SUBURB locality, residences D18) | platform, residence, trade-farming | default | `/world/terminus/hinkley-hills`: the stop, the lane, the plat and the lots (~24 ha held by the Improvement District until lots sell); lot-1 pre-taken | `hinkley-hills` | the plat-book, the lot-holder (producer) |
| **residence** (CAPABILITY — title becomes a place, D18) | platform | default | `/system/residence`: the substrate classes the locality packs' residential rows name — PlatBook, PlatWarren, LotGateExit, HoldingWarren, the householder's kit — moved out of the kernel platform tree | — | — |
| **tpa** (CAPABILITY — the teleport network's WORKS, TPA reform) | platform, arcana | default | `/system/tpa` → group `tpa` (PM-owned): `src/lib/` ships `FastTravelMixin` (the node — routes, board, timetable) and the pack's paths; `src/thing/` the `TpaTerminal` that composes it over arcana's `ManaPoweredMixin`, plus the `TravelCard`; `src/idea/cmd/movement/` the `teleport` + `register` controllers and `src/idea/cmd/tpa/` the card clerk's; rows: the three views + controller templates, `settings/fasttravel.yaml`, the travel card, and the **self-governing Teleport Authority** (`{kind: committee, parcel: /system/tpa}` — it names no realm, which is what lets it ship here). ⭐ The classes are the MECHANISM and are the pack's; a **terminal** is the realm's, so every gate keeps its `/world/**` row in the locality it stands in | `tpa` | — |
| **water** (CAPABILITY — the watershed's WORKS, D1–D27) | platform | default | `/system/water` → group `water` (PM-owned): `src/` ships `Watercourse` + `WatercourseCatalogue` (topology authored, direction **derived**, compiled to a reachability set), `WaterRightRegistry` (prior appropriation recorded, riparian derived over the same records) and the three works — `Conduit` (the conveyance ladder, a sewer being the same object reversed), `ControlStructure`, `StorageNode`; rows: the two singletons, `settings/water.yaml`, and `fouled-water` in the commons. ⭐ The classes are the MECHANISM and are the pack's; a **river** is the realm's, so every reach ships at `/stuff/idea/Watercourse/<name>` in world-seed | `water` | — |

Twenty-nine rows over thirty-six packs (`arcane-descriptors` folded into
`arcana` — the pack that ships the class ships the bank; libations added trade-distilling,
trade-brewing, trade-winemaking, trade-bottling, trade-farming and made
hospitality a capability pack; the TPA reform added **tpa**, pack
thirty-six, and it is the first pack to ship a `src/lib/`). **A stub trade** ships everything
downstream of production and nothing of production — materials, vessel
presets, brands, the floor product on an authored consignor, the serving
recipe — so the bar's demand is met today while the ferment/still is the
distillery build's. **The generic drain rule:** a row lives in the pack
whose PROCESS makes it; `generic-objects` is the junk drawer, slimming
(no brand, no demo bottle, no crop since libations). The corpo packs became **organizations** in wave 3: each ships
`content/corpo/<key>.yaml` (its chart — authority the PM office, because
a chart whose authority is *the committee over `/corpo/<key>`* recurses
once the organization holds that very title) beside its mark and
brands, and the wave-2 board *groups* are the retired holders `grant`
migrates from. `world-seed` exists so the seeders could be deleted
without the localities sitting in the platform under a false owner; the
lounge and the hearthworks left it in wave 4b; wave 5 homes eternal,
terminus, moor, practicum, substation and common and deletes it.

**Rows are sorted by Stuff branch, resolved by lineage.** A locality
with more than about six template rows keeps them under
`<root>/{location,thing,idea,agent}/`; six or fewer stays flat; source
mirrors it (`src/mud/world/lounge/{location,thing,idea}/`, the
hearthworks flat at one file). The branch is the class's — `TpaTerminal`
and `CommerceMenu` are Things, so the lounge terminal and every menu row
sit under `thing/`, whatever a first guess says.

**Packs seed, they do not own.** A venue row is an initial condition:
once installed it is the title-holder's, and the three-way reconcile's
file-same / DB-changed cell *keeps* the owner's edit (a renamed bar, a
refit room) with the baseline untouched, while a file change against an
unedited row still lands. `PackLogic.venue-ownership.test.ts` asserts
that cell in the venue framing; no mechanism was added for it.

## Reconcile policy

The policy is **three-way** (above). It subsumes both of v1's designed
shapes: `replace` (file is truth) is the *changed / same* cell, and the
deferred `seed-missing` (the DB wins thereafter) is the *same / changed*
cell — an operator's in-DB edit survives until the pack changes that
row too, at which point it is a conflict to settle rather than a policy
to pick. The opt-in answer to "content I *do* want to edit live in the
DB" is a **pin**.

## Versioning

A pack is files in a repo, so **git is the versioning** — history, diff,
branch, rollback, PRs, all on the content, all free. The manifest
`version` is a coarser release label and **inert in v1** (nothing reads
it). It becomes load-bearing only when packs evolve on independent
timelines (separate repos / third-party packs / a marketplace) — the
same boundary as the repo split — at which point it tracks the pack's
**public surface** (the paths and tags other content references), not its
values: editing gin's density breaks nothing (it re-hydrates); renaming
`/trade/distilling/idea/material/gin` breaks every pointer.

## Deferred

The slate (`docs/slates/builds/content-packs-slate.md`) holds the full
design surface and remaining build waves: **wave 5** — homing the rest
of `world-seed` (eternal as the capability exemplar with `DormThemes` /
`Footlocker` / `Gus`, terminus, moor, practicum, substation, common)
and deleting it; the **authorable-composition bridge** (a template
declaring its mixin stack in YAML — A23's biggest lever; 4b
genericized the composition-only classes into concrete `platform/`
classes instead); the **venue archetype** (A13.5 / A14 — the
`archetype` document kind, derive-on-read, the derived test venue:
declined until a third industry brings its own kernel gap);
hearthworks' inbound exit (still `goto`-only); the
`AppSettingFallbacks` code default for `defaultStartLocation`;
`requires.kinds:` (a pack declaring the document kinds it needs);
manifest version machinery + cross-pack dependency validation
(`dependsOn` on an unknown id is ignored today, deliberately); the
`staged` record status + runtime install / uninstall / marketplace;
third-party namespacing; and the repo split. `generic-objects` is the
junk drawer — expected to slim as trade packs take their objects.

Left by the capability-packs build (arcana first): the **parked
venue code** (`mud/world/{hearthworks,lounge,eternal}/**` — the
capability rung exists; moving each venue's TS into its pack is a
per-pack sweep as the reorg reaches it); the **third-party trust
story** (signing — every capability pack is first-party and
MR-reviewed); **first-instance minting** (`clone` fails closed when
no live representative exists, and the spawn sweep sees only live
candidates, so a never-instanced catalogue row is unreachable live
until something mints one — a `canAtPath` fallback in
`CloneController` is the shape); and `lint:instanceable`'s invariant-7
root set, the one literal left (`/platform`, `/stuff`, `/trade/<x>`
follow `<root>/<branch>/`, localities and `/wiki` do not — nothing in
a manifest declares which, so the script still says).

## Key files

- `packages/content/platform/` — pack zero: `content/platform/<branch>/` (the
  controllers, registries, catalogues, marshallers, vocabularies, the
  Avatar seed), `content/compact/` (the executive, the press),
  `content/platform/location/void.yaml`, the namespace roots (`home.yaml`,
  `studio.yaml`, `wiki.yaml`, `content/wiki/*.yaml`), `content/settings/`,
  `content/subjects/`, `content/blueprints/`, `content/platform/cmd/` (every
  engine verb's view).
- `packages/content/base-library/` — materials, biomes, quantity-units.
- `packages/content/species-and-names/` — the species/clade tree
  (`content/stuff/idea/species/**`) + the name banks (`content/name-banks/**`).
- `packages/content/arcana/` — the first capability pack: `src/thing/`
  (Wand, Scroll, Spellbook, Conduit, Ring, Amulet, Potion),
  `src/idea/material/PotionMaterial.ts`, `src/idea/cmd/magic/` (the five
  casting controllers), `content/system/arcana/idea/Discipline/` (the 18
  `magic-*` rows), `content/system/arcana/cmd/magic/` (the views),
  `content/settings/magic.yaml`, `content/descriptor-banks/`.
- `packages/content/arcane-library/` — `src/thing/{GlowlightMote,SparkLocus}.ts`,
  `content/stuff/idea/magic/Spell/**`, `content/stuff/thing/magic/**`
  (every clonable magic item), `content/stuff/idea/material/potion/`.
- `packages/server/scripts/pack-roots.ts` — the lint family's shared
  reader of which packs ship `src/` and which roots each backs.
- `packages/content/generic-objects/` — the object clusters under
  `content/stuff/<branch>/<cluster>/` and the loose objects; no recipes
  (every recipe is a trade's since wave 4b).
- `packages/content/trade-smithing/` — `root: /trade/smithing`:
  `content/trade/smithing/thing/` (anvil, whetstone, workbench, the ingots — a template row sits at the path its FILE mirrors; only documents derive from `root`) +
  `content/recipes/` (fire-poker, smiths-hammer, belt-knife, cook-pot,
  leather-jerkin).
- `packages/content/trade-cooking/` — `root: /trade/cooking`:
  `content/recipes/` (toasted-ration, root-mash, fine-roast, hearty-stew).
- `packages/content/trade-hospitality/` — `root: /trade/hospitality`:
  `content/trade/hospitality/thing/` (shaker, mixing-glass,
  cocktail-glass, back-bar, tip-jar — the templates a bar populates by
  reference) + `content/recipes/` (daiquiri, martini).
- `packages/content/expression/` — the emote roster (`content/emotes/`).
- `packages/content/wiki-starter/` — `content/wiki/<ns>/<slug>.md`.
- `packages/content/corpo-<key>/` × 5 — `content/corpo/<key>.yaml` (the
  organization) + `content/stuff/idea/corpo/**` (mark + brands).
- `packages/content/newbie-wilds/` — `content/world/newbie-wilds/**`.
- `packages/content/saxonberg-lounge/` — the lounge whole:
  `content/world/lounge.yaml` (the FolderZone) +
  `content/world/lounge/{location,thing,idea,agent}/` (the 22 venue
  rows, sorted by branch), `content/stuff/idea/lounge.yaml`,
  `content/msh/`, `content/settings/lounge.yaml` (`root: /world/lounge`);
  source under `src/mud/world/lounge/{location,thing,idea}/` + the
  locality-root `LoungeMixin.ts` / `paths.ts`.
- `packages/content/hearthworks/` — the venue pack:
  `content/world/hearthworks.yaml` (the CartesianZone — a Room's coords
  resolve against it, so the venue's zone row ships WITH the venue) +
  `content/world/hearthworks/{location,thing,idea,agent}/` (12 rows
  whose `props:` name the trade packs' and commons rows); source
  `src/mud/world/hearthworks/SealedCellar.ts` (flat).
- `packages/content/world-seed/` — the remaining locality rows under
  `content/world/**` (eternal, terminus, moor, practicum, substation,
  common), including the domain-local command views
  (`content/world/<…>/cmd/`) and their controllers.
- `mud/lib/employment/Offstage.ts` (`OffstageMixin`, the off-shift
  parking role) + `mud/platform/location/Offstage.ts` (the clonable every
  venue's `offstage` row names); `mud/lib/time/MechanicalMovement.ts`;
  the commons classes `mud/platform/thing/{CraftVessel,GradedReceptacle,
  NeonSign,CocktailShaker,TipJar,Menu}.ts` — the wave-4b graduations.
- `mud/lib/paths.ts` — `NON_TEMPLATE_DIRS` (enumerated from
  `DOCUMENT_KINDS`), the one list the installer, `CommandLogic`'s
  offline reader, `lint:untitled` and `lint:instanceable` share. (The
  title roots are not a list — they derive from the claims.)
- `mud/lib/document/DocumentKinds.ts` — the closed document-kind
  vocabulary (kind, natural key, dir, extension, vanish policy).
- `mud/api/pack.ts` — `PackApi` + the manifest / result / record /
  conflict / report types.
- `mud/platform/idea/api/PackLogic.ts` — discovery, the per-kind `KindStrategy`
  table, the pure planner + the applier, the record, the three-way
  machine, the flat-key check, `requires-kernel`, the ops surface, the
  re-hydrate tail.
- `mud/platform/idea/cmd/author/PackController.ts` + the platform pack's
  `content/platform/cmd/author/pack.yaml` — the `pack` verb suite;
  `lib/command/validators/requiresPackInstaller.ts` — its gate (title
  over `/compact/executive`).
- `mud/platform/idea/ParcelRegistry.ts` — `grant` (the three outcomes);
  `mud/api/group.ts` — `ensureGroup` /
  `ensureMember`.
- `backend/BootstrapManager.ts` — `run()` over `PackApi.bootManifest()`.
- `mud/platform/idea/api/CommandLogic.ts` — the store-only rule (`servedFromStore`)
  + the offline read over `PackApi.contentRoots()`.
- `mud/platform/idea/api/DiagnosticLogic.ts` — `packRecipients`, the maintainer
  routing.
- `packages/server/scripts/check-untitled-paths.ts` — `lint:untitled`.
- `e2e/playwright.platform.config.ts` +
  `e2e/tests-platform/platform-only.spec.ts` — the platform-only boot.
- `backend/PersistenceManager.ts` — the kind-scoped `documents` indexes
  (a derived loop over `DOCUMENT_KINDS`, not an authored list);
  `packages/server/src/schema/pack_installs.yaml` — that collection's
  policy, index and description; `src/schema/documents.yaml` — the
  declared kinds survive the night. ⚠ `lib/persistence/Collections.ts`
  and `ResetPolicy.ts` are **generated** from `src/schema/`; edit the
  YAML.
- `packages/server/scripts/check-test-content.ts` +
  `test-content-allowlist.txt` — `lint:test-content`, the shrinking
  allowlist of kernel tests that still name shipped content.
- `backend/AppBootstrap.ts` — the boot install pass + the per-pack boot
  line.
- `mud/platform/idea/api/QuantityLogic.ts` — the quantity-kind loader; its no-arg
  default lazily resolves the pack copy (test-only fallback; production
  always passes the path).

## History

Built on `feature/content-packs-build` (the content-packs v1 slice).
First substrate where canonical game content lives outside `packages/server`
as a versioned deliverable.

`feature/species-and-names-pack` (2026-06-29) added the second pack,
**species-and-names** — the `Species`/`Clade` tree + the char-gen name
banks moved out of the kernel seed tree / `NameBankSeeder` — and with
it the third content kind, **name-banks** (the first side-collection
kind).

`design/pack-installer` (2026-08-25) — the **pack-installer substrate**
(waves 0+1 of the content-pack program — slate addenda A10/A17/A24/A25
in `docs/slates/builds/content-packs-slate.md`; the requirements and
plan retired at the sweep): the
`domain` → `content` rename; the
`pack_installs` record with body-beside-hash baselines; the three-way
reconcile with conflicts and pins; per-pack failure
isolation; the flat-key check; the plan/apply split behind dry-run; the
`pack` verb suite; office-owned groups + the `pack-installers` committee
(`requiresWizard` left `pack`); and **newbie-wilds** as the fourth pack —
the first locality shipped as content.

`design/content-pack-wave-2` (2026-08-25) — **wave 2** (the requirements
and plan retired at the sweep): the `document` contribution kind over the
closed `DocumentKinds` vocabulary; the three per-kind collections
(`emotes`, `recipes`, `name_banks`) folded into `documents`; the `settings`
(merge-missing), `subject` (archive-never-reap), `wiki` (CAS submit as
the pack) and `command-view` (store-first with a counted disk fallback
and the wizard code-naming gate) kinds; `BlueprintCatalogue.rebuild()`
with the curated overlay as documents; `AccessApi.canAtPath` as the
document store's gate; `lint:test-content`; `mud/cmd` moved into the
platform pack; ten new packs and seven seeders retired.

`design/content-pack-wave-3` (2026-08-26) — **wave 3: pack zero, and the
end of `core`** (the requirements and plan retire at the sweep): the
manifest's `requires:` / `boot:` / `maintainers:`; the requires phase
(`GroupApi.ensureGroup` / `ensureMember`, `ParcelApi.grant` with its
four outcomes, the covered-extent rule, the bounded `skip-sold`
reconcile, staffing + maintainer routing, orphans, the nightly
`reprovision`); the boot union over every applied pack's `boot[]` (the
code manifest deleted); the `organization` parcel-owner kind (an
organization-held title admits its staff and its head); `core` deleted
— `ownerOf` nullable, every `can` fails closed on untitled, the author
tier / `:admin` / `requiresCoreAccess` / `pack-installers` gone, title
over `/compact/executive` gating `pack`; the template walk widened to
every non-kind `content/**/*.yaml`; `SeederManager` / `GroupSeeder` /
`ParcelSeeder` / `config/groups.yaml` / `config/parcels.yaml` / `seeds/`
deleted, 439 engine rows moved into the platform pack and the locality
rows into `world-seed`; no disk fallback for command views;
`SAXONBERG_PACKS`; the platform-only e2e; `lint:core-gone` +
`lint:untitled`.

**Wave 4a (2026-08-27) — the path surgery.** `/world/` → `/world/`
everywhere (content, `src/mud/world/`, tests, e2e, docs) with **no
migration** — the database is dropped; the `/trade/`
title root (then one of nine in a `TITLE_ROOTS` list; the list was
retired for a derivation from the claims in the capability-packs
build) and the
industry-pack shape (`/trade/<industry>/{obj,command,recipes,cmd}`,
`lint:instanceable` invariant 7); the hearthworks re-cut into
`trade-smithing` + `trade-hearth-cooking` (the trades own what they
introduce, the commons into `/stuff/thing/items`, the venue stays in
`world-seed`) — eighteen packs; the view-key rule generalised to every
template tree; wave 3's `migrated` grant outcome and both migration
branches deleted. **And the junk sweep:** every one-time boot migration
(`domain`→`content`, group owners, `script`→`msh`, the collection
collapse), the `developers`→`wizards` rename, the `adopt` reconcile cell +
`adoptQuery` + the adoption bridge, the migration scripts, `lint:core-gone`
— deleted. This game has never held data a boot of the same checkout did
not write; nothing is migrated, ever.

**Wave 4b (2026-08-27) — the venue packs.** The lounge is ONE pack
(`saxonberg-lounge` takes the 27 venue rows + the FolderZone out of
`world-seed`, with the TPA boot entry); **hearthworks** is a venue pack
of its own (12 rows, its group + claim, depending on the two trades and
its bank); the third industry **trade-hospitality** ships what the
lounge introduced (the four stations, the tip-jar template, the
cocktail recipes) and cooking's second pass rides along —
`generic-objects` ships no recipes. The A23 graduations: `Offstage` →
`lib/employment` (`OffstageMixin` + the concrete
`platform/location/Offstage`; both venues' casts park through it, the
hearthworks gaining the `shifts` brain and an `offstage` row),
`MechanicalMovement` → `lib/time`, and the composition-only classes
(`CraftVessel`, `GradedReceptacle`, `NeonSign`, `CocktailShaker`,
`TipJar`, the three menus collapsed to ONE `Menu`) → `platform/thing/`.
The locality rule applied: rows and source under branch subdirs by
lineage (>6) or flat (≤6). "Packs seed, they do not own" proven by
`PackLogic.venue-ownership.test.ts`, no mechanism added. Twenty packs.

**Capability packs, arcana first (2026-08-28).** A pack may ship `src/`
(§ The capability rung): the class-source table (`ModuleApi.
registerPackSource`), `StuffApi.resolveClassFile`, file-URL imports, the
loader transform over pack `src/`, the server's `exports` map as the
pack import profile, the pack list in the DEPLOYMENT manifest (the root
`package.json`; the server depends on no pack), `dependsOn` derived from
`package.json`, the rung check and `codeVersions`, `pack sync`'s code
tail, `reload` on a class path, the lint family over pack trees. Magic
proves it: **arcana** (the item classes, Ring/Amulet/Potion new, the
disciplines, the verbs, the banks — `arcane-descriptors` folded in) and
**arcane-library** (the spells, every clonable magic item, the two loci
named by `locus:`). The catalogues warm by class; `MaterialLogic.boot`
keeps a row by `instanceof Material`. Nineteen packs. The MR round
(MR !205) removed three hand-kept lists the build had grown: the
title roots now DERIVE from the claims (a template row is always
title-checked; a document only under a claimed root), `StuffApi` has
no class-namespace allowlist (resolution decides; the rung check is
keyed on what is on disk), and the test character joins every managed
group — with every test-only backend seam moved into
`backend/TestHooks.ts`.

**The path pattern (2026-08-28, on the wave-4a branch).** `/platform/… + /stuff/` is
gone. Every template path and every engine source file follows
`<root>/<branch>/…`: the root is the pack's (`/platform` for the
platform pack, `/stuff` — the commons — for every other pack,
`/trade/<industry>` for an industry), the branch is the Stuff branch the
class descends from (`thing` · `idea` · `agent` · `location`). Source
mirrors it: `src/mud/platform/` → `src/mud/platform/<branch>/`. `command` is
`cmd` everywhere: a controller is `<root>/idea/cmd/<category>/<Name>Controller`,
its view the document `<root>/cmd/<category>/<verb>` (`/cmd` is no longer a
root; the engine's 195 views live at `/platform/cmd/…` and their keys are
their paths, `platform/cmd/perception/look.yaml`). ONE walk rule: a `cmd`
dir holds views unless its parent is `idea`. The title roots were
`/platform /stuff /world /compact /studio /wiki /home /corpo /trade`
(a list then; derived from the claims since capability packs);
`lint:instanceable` invariant 7 checks the branch segment under every
rooted tree. Rosters that ship from two roots (Locality, Government)
scan `TemplatePathRosters`. Not applied inside `/world/<locality>` rows
(a place's rooms stay `/world/<locality>/<room>`) — only their
controllers moved to `<locality>/idea/cmd/`.
