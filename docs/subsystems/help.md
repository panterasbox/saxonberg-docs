# Help system (Wave 1)

The in-game **rulebook**: a uniform, searchable, navigable index of how
the world *works* — its commands and its engine/API surface — served as a
structured server-side data contract a future client help pane will
consume. Governing posture: **transparent by default, hidden only by an
explicit spoiler gate**. A specific product thesis rides it: the API
browse is the **player → contributor on-ramp** — the surface through
which a player learns the mixin/type model that powers the world.

Seeded by docs/slates/builds/help-slate.md
(Wave 1). Replaces the prior `HelpController` + the unconsumed TypeDoc
`author-surface.json` scaffold.

## The uniform `HelpTopic`

One schema spans both subdivisions; the index, search, typeahead, REST
contract, and `help` verb know *only* this shape (defined in
`@saxonberg/types`):

```ts
interface HelpTopic {
  id: string;            // 'command.look' | 'api.ContainmentApi.move' | 'mixin.Container' | 'type.Grade'
  kind: 'command' | 'api' | 'mixin' | 'type';
  title: string;
  summary: string;       // one line; '' when none
  keywords: string[];    // typeahead corpus: verbs/aliases, member name, concept, kind
  body: string;          // MML markup string (the rulebook entry)
  relations: HelpRelation[];
  spoiler: boolean;      // capability flag; false at the floor this cycle
  source: HelpSource;    // { subdivision: 'commands' | 'api', ref }
}
```

`body` is an MML markup string — the same field the verb's terminal
output and the future pane both render through the existing MML pipeline.
The light `HelpIndexEntry` (`{ id, kind, title, summary, keywords }`) is
the instant-render + client-local-typeahead slice. The REST
request/response DTOs (`HelpIndexResult` / `HelpKindListResult` /
`HelpTopicResult` / `HelpSearchResult` / `HelpErrorBody`) and the
`HelpCategory` descriptor live alongside in `@saxonberg/types`.

## The boot-warmed index — harvest, don't register

`/obj/HelpCatalogue` is a singleton `Idea`+`PostRegistrationMixin`
(bootstrap-manifest entry, no `dependsOn`), warming in `postRegister` —
the `TopicCatalogue` / `RecipeCatalogue` precedent. The index is
**harvested**: `warm()` *pulls* two projectors. No content-side hook ever
pushes a topic in (the substrate-no-content-hooks rule).

`warm(opts?)` is injectable so unit tests drive the projectors without
disk or boot:

- `commandDefs` defaults to `CommandApi.allDefinitions()` (the whole
  filename-keyed cache — a new Api accessor over `CommandLogic`).
- `surface` `undefined` → load `author-surface.json` from disk; `null` →
  simulate absent (degrade); an object → use it.

It holds `Map<id, HelpTopic>` plus a `byKind` index (kind → ids, sorted by
title). The catalogue's read methods are **ungated** (help is
transparency-by-default; a `FromModule` gate would be redundant noise) —
the single read chokepoint + the capability filter live in the `HelpApi`
facade, not here.

### Commands projector

One `HelpTopic` per loaded `CommandDefinition`: `id = command.<primaryVerb>`,
`title` = primary verb, `summary` = `def.description`, `keywords` =
`[...verbs, 'command']`, and **`body` = `def.getHelpText()` verbatim** —
the single source that already assembles verb / aliases / syntax /
options / subcommands / examples. Commands are flat this wave (`relations
= []`).

### API projector — first-class mixins, graded kinds, complete roster

Reads the enriched `author-surface.json` (below) and the `Mixins`
registry, emitting three graded kinds:

1. **Mixins drive completeness.** The roster iterates the *entire*
   `Mixins` registry (`lib/mixin.ts`), not only types that happen to
   appear in a signature — so the capability model has no holes. Each
   value `'<Concept>Mixin'` → `mixin.<Concept>`, enriched from the
   author-surface consumer members whose `face === concept` (its
   conferred methods, rendered with signature + summary). No authored
   match → **degrade to a bare topic** (`(no authored documentation)`).
   This is the centerpiece tier.
2. **Api faces** — `kind === 'api-static'` members grouped by face: a
   grouped landing `api.<Face>` **and** one member topic per static
   `api.<Face>.<member>` carrying the rendered signature + TSDoc body.
   `help api ContainmentApi.move` resolves the member topic.
3. **Types** — every `surface.types` entry not classified `api`/`mixin`
   → a lighter `type.<Name>` topic.

Classification: a name is `mixin` iff `'<Name>Mixin'` ∈ the registry; `api`
iff it ends in `Api` and lives under `mud/api/`; else `type`.

## The enriched `author-surface.json` pipeline

`api/help <Type.member>` renders **real content** — a readable signature
string + the TSDoc summary, plus `@param`/`@returns`/`@example` when
authored. This is driven by extending the existing author-surface
projection (`scripts/project-author-surface.ts`) rather than a new
artifact — one pipeline, one file, right-sized at runtime.

`projectAuthorSurface` stays **pure** (no IO). Each consumer member now
carries `signature`, `summary`, optional `params`/`returns`/`examples`,
and `signatureTypes` (the named project-types in its params + return — the
relation join key). The riskiest new logic is the **signature renderer**:
a pure walk over TypeDoc's serialized type JSON →
`name(p1: T1, p2?: T2, ...rest: R[]): Ret`, handling intrinsics,
references (generics via `typeArguments`), unions, intersections (e.g.
`Stuff & Container`), arrays, tuples, literals, and the optional/rest/
default parameter flags. Fixture unit tests assert the exact rendered
strings (generic, union, optional, rest, intersection) and the TSDoc
extraction (incl. the degrade-to-signature-only rule — a thin-TSDoc
member is **never dropped**, only stripped to its signature).

The artifact is a **gitignored build artifact**
(`packages/server/docs/api/author-surface.json`, produced by `pnpm docs`
→ `docs:project`). Wiring `docs:project` into the server build/deploy so
production always carries it is a noted follow-on ops item, not built
here.

## Typed relations — derived, never authored

Relations fall out of the projection so the pane navigates a hierarchy,
not a flat see-also list. Each edge denormalizes its `targetTitle`:

- **method-of** — every member topic → its parent face/mixin/type topic.
- **confers** — each `mixin` topic → its conferred members (the method
  name rides `targetTitle`). Conferred methods are rendered inline in the
  mixin body this wave rather than as standalone topics, so the edge
  resolves to the mixin topic that documents them; per-member drill-in
  topics are a later wave.
- **requires** — an `api`/member topic → the `type`/`mixin` topics named
  in its `signatureTypes`.
- **consumed-by** — the inverse of `requires` (a name→consumers index over
  every member's `signatureTypes`); `Container` → consumed-by
  `ContainmentApi`.
- **composes** — each `mixin` topic → mixin topics among its conferred
  methods' `signatureTypes` (`Container` → `Containable`). Best-effort
  (D5): a richer source — the augmentation `confers()` graph, the explicit
  base-chain — is deferred. Where a parameter's type is an alias
  (`ContainableStuff`) rather than a bare mixin name, the edge simply
  doesn't fire; the topic still exists.
- **see-also** — the catch-all: remaining same-module siblings.

## The capability floor — a real seam, single chokepoint

Spoiler/capability gating is a read-time filter present in the schema and
applied on **every** read, but a **no-op pass at the anonymous floor**
this cycle (everything visible). `HelpApi` is the thin facade over the
catalogue (resolved sync via `StuffApi.findByTemplatePath`); every public
read routes through one private `applyFilter` — the single place gating
happens. At the floor it is identity; it is shaped to drop
`topic.spoiler && !viewerCanSeeSpoilers(viewer)` once the spoiler slate's
ceiling lands (the viewer's tier already drives the decision, so the
ceiling is a one-place flip). The catalogue itself never filters. The
viewer shape stays server-internal — not yet a wire DTO.

## REST help data API

Read-only `GET` endpoints (the CMS REST precedent, minus CSRF + the
write-attribution bridge — there are no writes), registered in
`Server.setupRoutes()` before the SPA catch-all. Every route is
`requireAuth` this build (the public pre-auth face is a Wave 3 non-goal);
the session-derived viewer is the anonymous floor.

```
GET /api/help/index             → HelpIndexResult       (entries + categories)
GET /api/help/kind?kind=<k>     → HelpKindListResult     (400 on bad kind)
GET /api/help/topic?id=<id>     → HelpTopicResult        (404 on miss)
GET /api/help/search?q=<q>      → HelpSearchResult       (groups by kind)
```

`search` matches across `summary`+`body`; typeahead matches across
`title`+`keywords`+`kind`; both resolve across **both** subdivisions via
the uniform fields.

## The `help` verb — index-backed, light grammar polish

`HelpController` renders every form off the index through `HelpApi`, on
the existing `shell.result` topic:

- bare `help` → the landing/index (categories + counts + a hint line);
- `help <verb>` → that command's topic (**bare fallthrough** — the YAML
  gained `fallthrough: true` + a top-level optional `topic` positional, so
  the common case no longer needs the `verb` subcommand word);
- `help verb <name>` → the **legacy** form, still works;
- `help api <target>` → the API topic by id / `Type.member` (real
  signature + summary), or the api-kind landing;
- `help search <q>` → search, results grouped by `kind`.

The `topic` positional is named to avoid colliding with the `api`
subcommand's `target` field; `fallthrough` requires both top-level
`args:` and `subcommands:` (a `CommandDefinition` load-time invariant).

## Graceful degrade

`author-surface.json` is gitignored. If it is absent at boot, the API
projector yields **no** `api`/`mixin`/`type` topics and logs **exactly one**
boot warning (guarded); command topics and boot itself are unaffected.

## The shaped-not-wired world→model bridge

Topic ids and the `mixin` relations are designed so a later
inspection→help cross-link ("this bed is `Surfaced` → open the `Surfaced`
topic") is a clean add, not a rework: a `mixin.<Concept>` id is exactly
what an inspection pane already knows (the mixin name), and the relation
graph is the navigation substrate. Wiring that bridge is Wave 2 and is not
built here.

## Deferred (later waves / non-goals)

- **The client React help pane** — designed in a layout-system session;
  this build ships the server data contract it consumes.
- **The inspection↔help bridge wiring** (Wave 2).
- **Wave 2 subdivisions** — taxonomies / immutable defs / units, mechanics
  (formulas + constants), a standalone `help` **Document** collection, and
  **co-located `help:` prose**. Each is *adding a projector*; the
  interface does not change.
- **The public pre-auth face / standalone public docs render** (Wave 3) —
  the capability seam shipped here is what later gates it.
- **The full spoiler substrate** — owned by the spoiler slate; help only
  consumes the ceiling and stubs it at the floor.
- **`{{help:…}}` wiki transclusion** (Wave 2).
- **L3 developer source-surfacing** — topic `file:line` + "view source" +
  the CMS edit handoff (slate-deferred).
- **Replacing the TypeDoc HTML site** — this build consumes the JSON
  model; the HTML output stands as-is.

## Source map

| Concern | File |
|---|---|
| DTOs | `packages/types/src/index.ts` (Help block) |
| Enriched projection + signature renderer | `packages/server/scripts/project-author-surface.ts` |
| Command roster accessor | `mud/api/command.ts` + `mud/obj/api/CommandLogic.ts` (`allDefinitions`) |
| Index singleton + two projectors | `mud/obj/HelpCatalogue.ts` |
| Read chokepoint + capability filter | `mud/api/help.ts` |
| REST surface | `backend/HelpRoutes.ts` (mounted in `services/Server.ts`) |
| Verb | `mud/cmd/system/help.yaml` + `mud/obj/command/system/HelpController.ts` |
| Boot wiring | `mud/bootstrap.ts`, `mud/lib/paths.ts`, `mud/seeds/obj/HelpCatalogue.yaml` |
