# Bootstrap & Seeding

This document covers the game-startup mechanics: how the template
rows the engine needs reach the `content` MongoDB collection, and how
runtime instances are cloned from those templates when the server
starts.

## Scope

How `/mud/` game state initializes when the server starts. Excludes
`/backend/` and `/services/` infrastructure setup (those are unrelated
concerns and are left alone here).

## Two distinct concerns

| Concern | When it runs | Lifecycle |
|---|---|---|
| **Install** | Every server start (a three-way reconcile; a no-op when nothing changed) | Reconciles the shipped content packs' rows into the `content` collection — see [content-packs.md](./content-packs.md) |
| **Bootstrap** | Every server start | Clones runtime instances from those templates, ordered by deps |

Sequenced (install before bootstrap) but distinct. Install answers
"what templates ship?" Bootstrap answers "which runtime instances must
exist for the game to function?"

## Server boot order

1. Connect to MongoDB
2. Framework Api decoration (existing)
3. **`PackApi.install()`** — the content-pack installer reconciles
   every shipped pack (the `platform` pack first) into `content`, and
   its requires phase mints the groups and title each pack declares
4. **`PersistenceManager.loadHooks()`** — clone hook templates and
   register them with the persistence pipeline. Runs AFTER install so
   the hook templates exist in `content` when this clones them.
5. **`CommandApi.preloadAll()`** — the command views, served from the
   store the installer just wrote.
6. **`BootstrapManager.run()`** — clone runtime instances from the
   manifest: the union of every applied pack's `boot:` list. Last
   because manifest entries reference templates installed in step 3.
7. HTTP / WebSocket listeners come up

---

## Seeding — now the content packs

There is no seeder. `SeederManager` (insert-only YAML → `domain`
rows from `src/mud/seeds/`) was retired by content packs wave 3: every
row the engine ships is **pack content** under
`packages/content/<pack>/content/**`, installed by `PackApi.install`
with a real three-way reconcile (an edited pack file DOES reach the
live row; a locally-edited row is a conflict, never an overwrite). The
engine's own rows — `/platform/idea/EventRegistry`, `/platform/agent/Avatar/seed`,
`/world/void`, every `/platform/idea/cmd/<category>/<Name>Controller`,
the registries and catalogues — ship in the **`platform`** pack
(`packages/content/platform/content/`); generic objects, species,
localities and the corpo charts ship in their own packs. The file path
under a pack's `content/` still determines the template path
(`content/platform/idea/EventRegistry.yaml` → `/platform/idea/EventRegistry`).

Layout, kinds, the reconcile, `sourcePack` stamps, the `pack` verb:
[content-packs.md](./content-packs.md).

### Path conventions

Singletons live at `/platform/<branch>/<ClassName>` — `/platform/idea/EventRegistry`,
`/platform/idea/TopicCatalogue`, etc. Multi-instance classes extend
the same namespace with a per-instance suffix:
`/platform/agent/Avatar/<playerId>`. The `/platform/agent/Avatar` segment is implicit
(no template at it); the validator's folder/leaf rules treat path
segments as opaque tokens, so this works without a separator
distinct from `/`.

### Orphan templates (forked at runtime)

Most shipped rows are end-state singletons — `/platform/idea/EventRegistry`
ships and that's it. Some are **orphans** — templates that live in
the same namespace as their class's instances but with a reserved id
no real instance can collide with. The seed avatar is the worked
example:

- `platform/content/platform/agent/Avatar/seed.yaml` lands at `/platform/agent/Avatar/seed`.
  It's mechanically just an avatar template; the `seed` playerId is
  reserved (`Avatar.SEED_PLAYER_ID`) — 4 chars, nanoids are 21,
  no collision with a real player.
- Every avatar-mint site (`EnrollController.commit`, the test-auth
  `Application.createDefaultCharacter`, `Login.mintRandomGuestAvatar`)
  reads `Avatar.SEED_TEMPLATE_PATH`, copies its `class` /
  `hydratorClass` / `data`, overlays the character's own fields, and
  clones. A minted avatar is snapshot-backed (`holder_snapshots`), not
  a per-player template row — see [persistence.md](./persistence.md).

To change the defaults going forward, edit the seed row in the pack
and let the reconcile carry it. The pattern isn't an installer
feature — it's just "a row lands at a known path with a reserved id,
consumer code forks."

---

## Bootstrap

### BootstrapManager

Internal-only Manager. Called from `AppBootstrap.run` after
`PackApi.install` and after framework Apis are decorated. Reads a
typed manifest array — by default `PackApi.bootManifest()`, the union
of every applied pack's `boot:` list — topologically sorts,
instantiates each entry.

```ts
class BootstrapManager {
  static async run(manifest?: BootstrapEntry[]): Promise<void> {
    manifest ??= await PackApi.bootManifest();
    const sorted = topologicalSort(manifest);
    for (const entry of sorted) {
      const clone = await StuffApi.clone(entry.templatePath);
      // PostRegistration fires per existing Stuff lifecycle (sync)
      if (entry.awaitInit) {
        await entry.awaitInit(clone);
      }
    }
  }
}
```

`StuffApi.clone(templatePath, context?)` is the clone primitive — it
loads the Template by path, dynamic-imports the backing class, runs
the optional Hydrator, and awaits any `postRegister`. The clone
always lives at its template path; there's no separate "destination"
concept (each bootstrapped singleton's identity IS its template
path).

Failure modes throw and prevent server start:
- Cycles in `dependsOn` graph
- Missing `dependsOn` references
- Clone failure (template doesn't exist, etc.)
- `awaitInit` rejection

### BootstrapEntry shape

```ts
interface BootstrapEntry {
  /**
   * Path of the template to clone. Exactly one of `templatePath` /
   * `templatePathPrefix` must be set.
   */
  templatePath?: string;

  /**
   * Prefix to expand into all strict descendants in the `domain`
   * collection (via `Template.findDescendants(prefix)`), each
   * cloned as if it were an explicit entry. The prefix node itself
   * is NOT cloned — add a sibling `templatePath` entry for it if
   * needed. Cannot be combined with `dependsOn` / `awaitInit` (the
   * expansion's depth-ascending order stands in for explicit deps,
   * and per-clone async init isn't expressible when one entry
   * fans out to N clones).
   */
  templatePathPrefix?: string;

  /** Other entries' templatePaths that must complete before this one. */
  dependsOn?: string[];

  /** Optional async init beyond PostRegistration's sync surface. */
  awaitInit?: (clone: Stuff) => Promise<void>;
}
```

Most entries will have just `templatePath`. `dependsOn` for
ordering; `awaitInit` only for entries that need async setup beyond
PostRegistration. `templatePathPrefix` is the bulk-bootstrap option
for clusters of singletons that all need to be live but shouldn't
bloat the manifest — species clades, materials, biomes. Expansion
sorts depth-ascending (shorter paths first) so ancestor clades land
before their descendants; collisions with explicit `templatePath`
entries let the explicit one win.

### The manifest — every pack's `boot:` list

There is no code manifest. `mud/bootstrap.ts` was retired by content
packs wave 3; the manifest is the **union of every applied pack's
`boot:` list** (`pack.yaml`), read through `PackApi.bootManifest()` in
install order. Each entry names a template the pack ships, a `role`
(`sync-read` — something resolves it synchronously and would read
null; `producer` — it must be live to do its job), a `reason`, and an
optional `dependsOn`. A template two packs both list is a boot error
naming both. Under `SAXONBERG_PACKS` only the filtered packs' entries
boot.

```yaml
# packages/content/platform/pack.yaml (excerpt)
boot:
  - { template: /platform/idea/EventRegistry, role: sync-read, reason: every EventApi emit resolves it synchronously }
  - { template: /world/void, role: producer, reason: the evacuation fallback ContainerMixin resolves synchronously on destruct }
  - { template: /platform/idea/TopicCatalogue, role: sync-read, reason: topic descriptors resolve synchronously from its lazy cache }
```

The `BootstrapEntry` type stays owned by `BootstrapManager` (the
consumer); `PackApi.bootManifest` maps each pack entry onto it
(`templatePath`, `dependsOn`, plus the `packId` / `role` for the boot
line). The platform pack carries the engine's registries and
catalogues; the corpo packs boot their charts; `world-seed` boots the
locality singletons it ships. Per-pack counts print on the boot line
(`boot: N sync-read + M producer`), and `BootstrapManager` reports the
total (`bootstrapped N entries`).

Note what's explicitly NOT in any `boot:` list:

- **Species clades** are not bootstrapped. `SpeciesApi.isAnimate` /
  `getKingdom` are sync and resolve clades via
  `findByTemplatePath`, but the `requiresAnimate` validator
  preloads the relevant chain via its async `preload` hook (see
  [command-routing.md § Validator preload phase](./command-routing.md#validator-preload-phase));
  the dispatcher awaits preloads between MQL resolution and the
  sync validator phase. Same pattern stays available for Materials
  / Biomes / etc. as they grow validator coverage.
- **No ModuleRegistry / CommandRegistry / WorldRoot.** Earlier
  designs sketched these; reality folded them into other surfaces
  (controllers are templates registered through the normal hook
  template flow; module info isn't externally consulted yet).

### Why no phases

Earlier design considered explicit phases (`PreEngine`, `Engine`,
`PreWorld`, `World`, `PostWorld`). Dropped because dep-ordering covers
every case we could imagine. Phases would only earn their keep if
dep-ordering had loops you needed to break with a coarse "everything
in phase A before anything in phase B" rule — we don't have that.

### Why no `BootstrappableMixin`

Earlier design considered a mixin that templates compose to declare
"I auto-spawn at startup." Dropped because:
- Templates are templates; specializing them via a mixin just to mark
  them as bootstrap-eligible is unnecessary indirection.
- The bootstrap manifest is the single source of truth for "what gets
  spawned" — easier to read one list than to crawl all templates
  looking for the marker.
- A pack appends to the same union through its `boot:` list; no
  special mod-specific mechanism needed.

### PostRegistration cooperates

Bootstrap orchestrates *what gets created and when*. PostRegistration
handles *how each instance initializes itself* — existing mechanism,
unchanged. For most entries, PostRegistration is enough. `awaitInit`
on the BootstrapEntry is only for entries needing async setup beyond
PostRegistration's sync surface.

The decision to put async setup on the bootstrap entry (not on a new
PostRegistration variant) avoids changing PostRegistration's contract
and keeps the async-init concern visible at the bootstrap manifest
level where the orchestration decisions live.

### Convention: template path = backing class module path

For singleton system Ideas (one template per class), the template path
matches the module path:

| Backing class | Template path |
|---|---|
| `mud/platform/idea/EventRegistry.ts` | `/platform/idea/EventRegistry` |
| `mud/platform/ModuleRegistry.ts` | `/platform/idea/ModuleRegistry` |
| `mud/platform/CommandRegistry.ts` | `/platform/idea/CommandRegistry` |

This aligns the on-disk seed location with the runtime template
path. Avatar is the existing exception
(per-user dynamic template paths) and remains so.

### Mod extensibility (future)

A mod IS a content pack: its rows install through the same
reconcile, its `boot:` entries join the same union, its `requires:`
declares the groups and title it needs. There is no mod-specific
bootstrap mechanism to build — see [content-packs.md](./content-packs.md).

---

## Registry pattern (referenced by both this doc and events.md)

### Definition

A **Registry** is:

- A long-lived **singleton Idea** (lives at a well-known template path).
- Holds a **named collection of declarations** (not Stuff instances).
- Exposes **lookup-by-name** + enumeration.
- **Gates per-entry access** through Propertied `checkAccess`.
- Composes `Propertied` on an `Idea` base (rebuilt from seed at boot —
  not a persisted record), plus whatever else the entries need.
- Is **bootstrapped at engine startup** — every Registry has a
  BootstrapEntry.

### Distinct from

| Word | Means |
|---|---|
| **Registry** | Singleton Idea; named collection of declarations; gated lookup |
| **Manager** | Subsystem-internal coordinator over a class of things (`PersistenceManager`, `BootstrapManager`) |
| **Api** | Security-gated public surface for callers (`EventApi`, `StuffApi`) |
| **Container** | Holds Stuff instances; participates in containment / movement |
| **Mixin** | Composition unit that adds methods / fields to classes |

Corollaries:

- A Registry is **NOT** a Container. It doesn't "contain" its entries
  in the spatial / containment sense; entries aren't necessarily Stuff.
- A Registry's entries can themselves *point at* other Stuff (e.g.,
  CommandRegistry entries point at controller classes), but the
  Registry itself owns only the declarations.
- A Registry is queryable by name and enumerable. Its surface is "look
  up by key" + "list all keys" + the per-entry `checkAccess` gate.

### Initial Registries

| Registry | Holds | Lives in |
|---|---|---|
| `EventRegistry` | Well-known events + emit/subscribe permissions | `/platform/idea/EventRegistry` |
| `ModuleRegistry` | Module info for hot-reload (future) | `/platform/idea/ModuleRegistry` |
| `CommandRegistry` | Command name → controller mapping (existing in some form; consider formalizing) | `/platform/idea/CommandRegistry` |

Future candidates: `AchievementRegistry`, `DamageTypeRegistry`,
`FactionRegistry`, `RecipeRegistry`, `PromptTypeRegistry`,
`MqlResolverRegistry`. None of these are on the immediate roadmap.

### `Mixins` constants object

The existing `Mixins` constant object in `lib/mixin.ts` is
informally called a "registry" today but doesn't fit the formal pattern
above (it's a constants object, not an Idea, not Propertied, not
bootstrapped). Either rename to avoid confusion (e.g., `MixinNames`)
or upgrade to a proper Registry. Probably the former — upgrading just
to fit a label is over-engineering for a pure constants object.

---

## What we explicitly decided NOT to do

- **`BootstrappableMixin`.** Templates are templates; bootstrap is a
  list. No need to specialize.
- **Phases.** Dep-ordering covers all required ordering.
- **Engine vs world manifest split.** Single mechanism. Every pack
  contributes to the same union.
- **`BootstrapApi` as a security-gated public surface.** It's a
  Manager, internal-only. Called from `main()` and (eventually) from
  hot-reload's mod-load path. No external callers.
- ~~**Auto-update of seeded templates** when YAML changes.~~ Reversed
  by the content-pack installer's three-way reconcile (waves 0–3).
- **Hot-reload re-running bootstrap.** Hot-reload is a separate
  subsystem with its own re-init story when it lands. Conflating the
  two added complexity for no clear benefit.
- ~~**YAML for the engine bootstrap manifest.**~~ Reversed by content
  packs wave 3: the manifest is the packs' `boot:` lists (YAML), so a
  pack can name what it needs live without a code change. `awaitInit`
  remains available to a code-supplied manifest override (tests).

---

## Implementation order (as it happened)

1. **`SeederManager` + initial seed YAMLs** for the engine's existing
   templates (Avatar, anything else currently inserted via
   ad-hoc startup code) — since replaced by the content-pack installer
   (waves 0–3, 2026-08).
2. **`BootstrapManager` + initial manifest** with no entries yet —
   wire it into `main()` boot sequence.
3. **First bootstrap entry: `EventRegistry`** — the events subsystem
   needs this. See [events.md](./events.md).
4. **Subsequent registries** as their subsystems land
   (`ModuleRegistry` with hot-reload, `CommandRegistry` if formalized,
   etc.).
5. **Mod extensibility** is a future phase, deferred until the modding
   subsystem itself lands.

---

## Cross-references

- [events.md](./events.md) — the EventRegistry is the first Registry
  that needs this bootstrap mechanism. The events subsystem and this
  one are tightly coupled in scope but designed independently.
- [templates.md](./templates.md) — TemplateApi.clone is the underlying
  mechanism BootstrapManager uses for each entry.
- [lifecycle.md](./lifecycle.md) — PostRegistration is what each
  bootstrapped instance uses for its own per-instance setup.
- [persistence.md](./persistence.md) — the `Document` track; registries
  are seeded Ideas rebuilt at boot, so a registry that needs to persist
  declarations would use a `Document` collection rather than its own
  runtime state.

## History

- **Content packs wave 3 (2026-08-27)** retired `SeederManager` and
  `mud/bootstrap.ts`. Rows ship as pack content and the boot manifest
  is the union of the packs' `boot:` lists (`PackApi.bootManifest`).
  The sections above that describe the seeder's insert-only semantics
  were removed; [content-packs.md](./content-packs.md) is the
  reference.
