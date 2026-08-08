# Content packs

The **content-pack** substrate: foundational game content lives in
standalone, git-versioned packages of **pure data** (zero TypeScript),
and a single **installer** reconciles them into MongoDB at boot and on
demand. The pack files are the **source of truth**; the database is a
*derived install* of them. This is the long-term replacement for
`SeederManager` — it gives content real version control (git, on the
files) and a clean deliverable boundary, decoupled from the kernel's
release cycle.

Two packs ship today: **base-library** (materials, biomes,
quantity-units) and **species-and-names** (the `Species`/`Clade`
taxonomy tree + the char-gen name banks), plus the full iteration loop
(boot install + runtime `pack sync`). They coexist with `SeederManager`
and the remaining per-collection seeders, which keep serving every
not-yet-migrated tree.

> **Scope.** This is the *content-only* corner of the downloadable-content
> taxonomy (roadmap Framework 13: Content / Capability / Full). A content
> pack ships **no code**; it assumes the kernel already provides the
> classes its data names. That assumption — enforced by the
> `requires-kernel` check — *is* the content-pack ↔ mod boundary: a pack
> assumes classes, a mod brings them.

## The shape of a pack

A pack is a monorepo package (`packages/content/<pack>/`, its own repo
once the monorepo is broken up) named `@saxonberg/content-<id>`:

```
packages/content/base-library/
├── package.json          # workspace member: name, version, private — no code
├── pack.yaml             # the manifest the installer reads
└── content/              # content root; MIRRORS the template-path namespace
    └── lib/
        ├── material/spirit/gin.yaml   →  template path /obj/material/spirit/gin
        └── biome/…
```

The `content/` root mirrors the template-path namespace: a file's path
relative to `content/`, minus `.yaml`, prefixed with `/`, **is** its
template path (`content/obj/material/spirit/gin.yaml` →
`/obj/material/spirit/gin`) — the same rule `SeederManager` uses. So the
path is a pure namespace identifier, decoupled from where the file
physically sits, and migrating a tree out of `seeds/` is a `git mv`.

### The manifest — `pack.yaml`

```yaml
id: base-library          # the sourcePack stamp value; stable
version: 0.1.0            # reserved release label — nothing reads/enforces it in v1
description: …
dependsOn: []             # ids of packs that must install first
```

Deliberately minimal: the pack's two real requirements are **derived**,
not declared. *Requires-kernel* is the install-time class-resolve check
(below). *Owned paths* are recorded by the `sourcePack` stamp at install
time. The manifest never enumerates paths.

`server`'s `package.json` `@saxonberg/content-*` dependencies are the
**single source of truth** for which packs a build ships — adding a pack
to a build is adding a dependency line.

## Discovery

`PackLogic` discovers packs by reading the `@saxonberg/content-*` keys
from `server`'s `package.json` (the names), then resolving each to its
on-disk root via Node module resolution (`require.resolve('<pkg>/package.json')`).
Reading the *names* from `package.json` keeps one source of truth;
resolving *paths* via module resolution is layout-robust across dev
(`tsx`) and dist. Packs are then ordered by `dependsOn` (a stable
topological sort; throws on a cycle). With one pack this is a passthrough,
but the read-and-honor of `dependsOn` is present so multi-pack ordering
is already wired (full missing-dependency validation is deferred).

## The installer — reconcile by `sourcePack` stamp + content-kind

The installer's job each run is *make the DB match the pack files*. It
lives on the **Api + logic-singleton** pattern (not `backend/` like
`SeederManager`) because it has two callers — the boot path and the
runtime `pack sync` verb — and the gated `PackApi` is the shared surface:

- **`PackApi`** (`mud/api/pack.ts`) — the thin forwarding shell:
  `install()` / `sync(packId)` / `discoverPacks()`, plus the
  `PackManifest` / `PackReconcileResult` call-shape types.
- **`PackLogic`** (`mud/obj/api/PackLogic.ts`, `/obj/api/pack`,
  `@internal @Unshadowable`) — the reconcile core; every public method
  gated `FromModule('/api/pack#PackApi')`. All work in module-private
  functions (the `CraftingLogic` precedent), so no intra-singleton
  `this.x()` calls trip the gate.

A single private `reconcilePack(pack, {rehydrate})` is the one
implementation; `install` and `sync` differ only in the re-hydrate tail.

### Content-kind dispatch

Content is classified by subdir convention and dispatched to the right
backend — the per-row stamp makes this work identically across backends,
which a path-prefix notion of ownership could not:

| Subdir | Kind | Backend |
|---|---|---|
| `content/obj/**.yaml` | **domain** | reconciled into the `domain` collection (stamped) |
| `content/quantity/quantity-tags.yaml` | **quantity** | loaded into the in-memory tag table via `QuantityApi.loadTagTables(path)` |
| `content/name-banks/<key>.yaml` | **name-banks** | reconciled into the `name_banks` collection (stamped), keyed on the file's basename = the bank key |

> **The domain subdir was `content/lib/` until the lib/obj taxonomy
> refactor.** It is `content/obj/` now, because a pack ships content and
> content is instanceable — nothing a pack installs may live under
> `/lib/`, which is substrate-only. This is a **breaking format change**
> for any out-of-tree pack: rename `content/lib/` to `content/obj/` and
> repoint every `class:` value. Shipped packs needed no data migration
> because the installer **reconciles** (unlike `SeederManager`, which is
> insert-only) — the old rows are stamped, their files vanished, so
> `reconcileDomain` deletes them and inserts the new paths on the next
> boot. `PackLogic.readContent` reads the subdir name.

The **name-banks** kind is the first *side-collection* kind — a flat
`Document` (`{key, given, surname, style?}`) rather than a path-addressed
template, so its reconcile (`reconcileNameBanks`) mirrors the domain one
but keys on the bank `key` and writes to `name_banks`. It skips the
`requires-kernel` check (a name bank names no backing class). Because
banks are immutable reference data the char-gen suggester caches by key,
a `sync` that changes any bank calls `NameBank.clearCache()` so the edit
goes live (banks aren't held by live instances, so there is nothing to
re-hydrate). The general document/side-collection vein (emotes, recipes,
channels) remains a deferred generalization of this one kind.

### The `sourcePack` stamp

Each installed row carries a **top-level `sourcePack` field** (a `domain`
template, a `name_banks` bank — every stamped backend). On a `domain` row
it is a sibling of `data` (`{path, class, hydratorClass, data, sourcePack}`).
It is **not** inside `data`, and the clone pipeline passes only
`template.data` to the Hydrator — so the stamp is structurally
unreachable by the instance (a `Material`/`Biome` never sees it). It is a
pure DB-row provenance marker, written directly by the installer through
the `PersistApi` chokepoint (`save` = `$set`-by-`_id` for update/adopt,
or insert), never via `TemplateApi.saveTemplate` (which has no slot for
it).

### The domain reconcile

For the rows stamped with this pack (`PersistApi.find({sourcePack: id})`):

- **file at an already-stamped path** → update (only if `data`/`class`/
  `hydratorClass` actually differ — a no-op reconcile reports nothing);
- **file at a path with a pre-existing *unstamped* row** → **adopt**:
  stamp it and overwrite to match the file. This is the migration bridge
  — a live DB seeded by the old `SeederManager` is adopted on first
  install **without a wipe**, by `$set`-on-`_id` (never an insert, so the
  `path` unique index is never challenged). A row stamped by a *different*
  pack is refused (throws) — packs never clobber each other;
- **file at a path with no row** → insert (stamped);
- **stamped row whose file vanished** → delete. (A dangling reference
  from a live object is the author's to fix — documented, not guarded.)

### requires-kernel

Before any write, the installer resolves every distinct `class:` (and
`hydratorClass:`) the pack's content names, via the standard
`StuffApi.loadClassByPath`. A missing class **aborts the whole install**
(before any write, so the DB is untouched) with a message naming the
pack, the class, and the offending file. This is the enforced
content-pack ↔ mod boundary.

## The two entry points

### Boot — `PackApi.install()`

`AppBootstrap.run` calls `PackApi.install()` in the pre-`loadHooks`
region (it replaces the migrated trees' seeding and folds in the former
standalone `QuantityApi.loadTagTables` call). It **writes rows only** —
`BootstrapManager` clones singletons *afterwards*, so nothing is live yet
and there is no re-hydrate at boot.

Ordering and coexistence: the install pass runs after `SeederManager`
(which still walks the shrunken `seeds/` tree) and before `loadHooks`
(the migrated content — domain templates + the quantity table — is all
pre-hooks content the marshaller/`tag()` consumers and the DomainHook
clone depend on). The installer and `SeederManager` touch **disjoint
sets**: the installer only ever touches `sourcePack`-stamped (or
adopts-then-stamps) rows for paths its packs ship; `SeederManager` is
insert-only on the paths still in `seeds/`. Conflict-free by construction.

### Runtime — `pack sync <packId>`

The `pack` author verb (`obj/command/author/PackController.ts` +
`cmd/author/pack.yaml`, wizard-gated via the `requiresWizard`
validator, dispatch-on-subcommand — `sync` is the only verb in v1) runs
the **same** reconcile, then re-hydrates: for each changed path it calls
`TemplateApi.restoreFromTemplate` on every live instance at that path
(the `CmsLogic._writeContent` go-live pattern); for each deleted path it
destructs the orphaned instance. The quantity kind uses
`reloadTagTables` (diff-apply) instead of `loadTagTables`.

This is the iteration loop: edit a pack file → `pack sync` → live, no
restart. It is tractable for base-library because materials/biomes are
**singletons by path** (one instance per path; `Tangible.getMaterial()`
re-resolves the material singleton lazily on each read, so every object
made of it sees the new values immediately). Cloned (non-singleton)
content — many copies of one template — would need per-instance
re-hydration; that harder wave is deferred with world packs.

## Reconcile policy

v1 implements one policy, **`replace`** (the file is truth — overwrite
the row). The counter-case, **`seed-missing`** (install if absent, the DB
wins thereafter — the operator-tunable shape `AppSettingsSeeder` uses,
where in-DB edits survive), is designed but deferred until app-settings
migrates. It is also the opt-in answer to "content I *do* want to edit
live in the DB."

## Versioning

A pack is files in a repo, so **git is the versioning** — history, diff,
branch, rollback, PRs, all on the content, all free. The manifest
`version` is a coarser release label and **inert in v1** (nothing reads
it). It becomes load-bearing only when packs evolve on independent
timelines (separate repos / third-party packs / a marketplace) — the
same boundary as the repo split — at which point it tracks the pack's
**public surface** (the paths and tags other content references), not its
values: editing gin's density breaks nothing (it re-hydrates); renaming
`/obj/material/spirit/gin` breaks every pointer.

## Deferred

The slate (`docs/slates/builds/content-packs-slate.md`) holds the full
design surface and remaining build waves: the remaining packs (emotes,
recipes, channels) and generalizing the side-collection content-kind
beyond name-banks; retiring `SeederManager` and the per-collection
seeders (the strangler-fig end-state); the `seed-missing` policy; world
packs + cloned-instance
re-hydration; manifest version machinery + cross-pack dependency
validation; the round-trip / export lane (edit-in-game → file); migrations;
runtime install / uninstall / marketplace; third-party namespacing; and
the repo split.

## Key files

- `packages/content/base-library/` — the substrate pack (materials,
  biomes, quantity-units).
- `packages/content/species-and-names/` — the species/clade tree
  (`content/obj/species/**`) + the name banks (`content/name-banks/**`).
- `mud/api/pack.ts` — `PackApi` + the manifest/result types.
- `mud/obj/api/PackLogic.ts` — discovery, content-kind dispatch (domain /
  quantity / name-banks), the stamped reconcile (insert/update/adopt/
  delete), `requires-kernel`, the re-hydrate tail.
- `mud/obj/command/author/PackController.ts` + `cmd/author/pack.yaml` —
  the `pack sync` verb.
- `backend/AppBootstrap.ts` — the boot install pass.
- `mud/obj/api/QuantityLogic.ts` — the quantity-kind loader; its no-arg
  default lazily resolves the pack copy (test-only fallback; production
  always passes the path).

## History

Built on `feature/content-packs-build` (the content-packs v1 slice).
First substrate where canonical game content lives outside `packages/server`
as a versioned deliverable.

`feature/species-and-names-pack` (2026-06-29) added the second pack,
**species-and-names** — the `Species`/`Clade` tree + the char-gen name
banks migrated out of the kernel seed tree / `NameBankSeeder` — and with
it the third content kind, **name-banks** (the first side-collection
kind). The migration rode the adopt-don't-wipe path: a live DB's existing
unstamped species + `name_banks` rows are adopted in place on first
install, no wipe, no data migration (no class moved).
