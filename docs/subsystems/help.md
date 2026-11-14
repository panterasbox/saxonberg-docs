# Help system (Wave 1)

The in-game **rulebook**: a uniform, searchable, navigable index of how
the world *works* — its commands and its engine/API surface — served as a
structured server-side data contract a future client help card will
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
output and the future card both render through the existing MML pipeline.
The light `HelpIndexEntry` (`{ id, kind, title, summary, keywords }`) is
the instant-render + client-local-typeahead slice. The REST
request/response DTOs (`HelpIndexResult` / `HelpKindListResult` /
`HelpTopicResult` / `HelpSearchResult` / `HelpErrorBody`) and the
`HelpCategory` descriptor live alongside in `@saxonberg/types`.

## The lazily warmed index — harvest, don't register

`/platform/idea/HelpCatalogue` is a singleton `Idea`+`onCreate`, warming
in `onCreate`. ⭐ It has no pack `boot:` entry (the scan sweep,
2026-10-14); it is `HelpApi`'s declared `needs`, so `AppBootstrap` stands
it once at boot (load-on-use W10) and the `help` verb's sync reads find it
— no controller readies it. The index is
**harvested**: `warm()` *pulls* three projectors. No content-side hook
ever pushes a topic in (the substrate-no-content-hooks rule).

`warm(opts?)` is injectable so unit tests drive the projectors without
disk or boot:

- `commandDefs` defaults to `CommandApi.allDefinitions()` (the whole
  filename-keyed cache — a new Api accessor over `CommandLogic`).
- `surface` `undefined` → load `author-surface.json` from disk; `null` →
  simulate absent (degrade); an object → use it.
- `schema` follows the same three-state convention over the authored
  schema docs in `src/schema/`.

It holds `Map<id, HelpTopic>` plus a `byKind` index (kind → ids, sorted by
title). The catalogue's read methods are **ungated** (help is
transparency-by-default; a `FromModule` gate would be redundant noise) —
the single read chokepoint + the capability filter live in the `HelpApi`
facade, not here.

### Commands projector

One `HelpTopic` per loaded `CommandDefinition`: `id = command.<primaryVerb>`,
`title` = primary verb, `summary` = `def.description`, `keywords` =
`[verb, ...defaultAliases, 'command']`, and **`body` = `def.getHelpText()` verbatim** —
the single source that already assembles verb / aliases / syntax /
options / subcommands / examples. Commands are flat this wave (`relations
= []`). ⭐ The catalogue indexes them by `def.category` (D24) —
`commandCategories()` and `listCommandCategory()`, surfaced as
`HelpApi.commandCategories` / `commandCategory`.

A command view may carry a co-located **`help:` block** — multi-line,
player-facing prose ([command-spec.md](./command-spec.md) § the YAML
shape). `getHelpText()` renders it *below* the synthesized syntax, so the
command topic's body carries the authored prose without a second source,
and authored prose never restates what the syntax line already says.

⭐ **`help <verb>` lists who ANSWERS it** (controllers-are-io D8b). A command
topic whose view's slots `requires:` a verb protocol (`_mixinKind =
'protocol'`, [mixins.md](./mixins.md)) gains an *answers:* section — each
protocol, and under it the classes that compose it (`help dip`: `DipBathMixin
— DipPot, GlazeTub, Melt, PaintPot, PulpVat`), from `StudioApi.answerersOf`.
It is filled on the topic's **first read** and cached until the index is
rebuilt: the warm keeps only each view's required-mixin list
(`CommandDefinition.getRequiredMixins`), so the source scan never runs at
boot and no build artifact exists. ⚠ A verb whose object slot is `requires:
any` (narrowed in its controller — `turn`, `carve`) lists nothing until its
view names the protocol; a noun mixin a slot requires is not an answer.

### API projector — first-class mixins, graded kinds, complete roster

Reads the enriched `author-surface.json` (below) and the `Mixins`
registry, emitting three graded kinds:

1. **Mixins drive completeness.** The roster iterates the *entire*
   `Mixins` registry (`lib/mixin.ts`), not only types that happen to
   appear in a signature — so the capability model has no holes. Each
   value `'<Concept>Mixin'` → `mixin.<Concept>`, enriched from the
   author-surface consumer members whose `face === concept` (its
   conferred methods, rendered with signature + summary). ⭐ The topic's
   **summary is the mixin's own** — the first paragraph of the TSDoc on
   its exported `<Concept>Mixin` factory function, from the
   author-surface `mixins` section (client-surface #89); only a mixin
   whose factory has no TSDoc falls back to its first conferred member's
   summary, which used to describe one method rather than the interface.
   No authored match → **degrade to a bare topic** (`(no authored
   documentation)`). This is the centerpiece tier.
2. **Api faces** — `kind === 'api-static'` members grouped by face: a
   grouped landing `api.<Face>` **and** one member topic per static
   `api.<Face>.<member>` carrying the rendered signature + TSDoc body.
   `help api ContainmentApi.move` resolves the member topic.
3. **Types** — every `surface.types` entry not classified `api`/`mixin`
   → a lighter `type.<Name>` topic.

Classification: a name is `mixin` iff `'<Name>Mixin'` ∈ the registry; `api`
iff it ends in `Api` and lives under `mud/api/`; else `type`.

### Collections projector — the persistence layer, readable

One `HelpTopic` per authored schema doc (`src/schema/<collection>.yaml`,
one per Mongo collection — see
[persistence.md § Collections](./persistence.md)). `id =
collection.<name>`, `kind = 'collection'`, `subdivision = 'persistence'`.

The body is assembled from the doc's `purpose` and `invariants`, the
**fields harvested from the owner class's `fieldMeta`**, every index with
its authored `why`, and **both policies rendered in plain words**:

> Inside a sandbox circle, a write here is STAMPED with the circle: it
> happens for real while you are in there, field reads never see it, and
> leaving the circle discards it.
>
> The nightly reset EMPTIES this collection.

— never `{ verb: 'stamp' }`. The point of the whole build is that a
person can read it.

⭐ **The field list is harvested, never restated.** The schema doc carries
no fields; the projector resolves the doc's `ownerModule` through
`StuffApi.loadClassByPath` and reads `MixinApi.getAllFieldMeta` off the
class. Adding a persistent field to `LedgerEntry` changes
`help bank_ledger` with no edit to `bank_ledger.yaml`. A resolve that
fails degrades to no field list rather than no topic — the purpose and
the invariants are still worth reading.

⚠ **Spoiler flag `false`** for every collection topic. How the world
remembers things is not a secret, and the capability floor this cycle is
anonymous.

The `help` verb reaches these through the **bare fallthrough**, in the
order command → collection → api: `help bank_ledger` works, and a verb
wins over a collection of the same name (so `help wiki` is the verb;
`help collection.wiki` is the collection). `help verb <name>` stays
strict and does not fall through — asking for a verb by that name should
say the verb does not exist, not wander off into the persistence layer.

### The concept projector — authored `help-concept` documents

Every projector above reads something that exists for another reason,
which is why the index could never explain *what nitrogen is*. A **help
concept** is a player-facing topic about a CONCEPT — `title`, `summary`,
`body` (MML), `keywords`, `seeAlso` — authored as a **`help-concept`
document** ([document-store.md](./document-store.md)) at a pack's
`content/concepts/<word>.yaml`, stored at `<pack root>/concepts/<word>`.
The catalogue reads the kind (`DocumentApi.listOfKind('help-concept')`)
at `warm` and emits `concept.<word>` topics; there is no value class —
`HelpCatalogue` is the one reader. ⭐ **A concept's word is its path's
leaf** (`help nitrogen`), a typed word unique at install
([content-packs.md](./content-packs.md)); `seeAlso:` cites other concepts
by **path**, and a see-also relation targets `concept.<leaf>`. A concept
with no `title` is skipped and named. (It was a `HelpConcept` data Idea
under `<root>/idea/HelpConcept/<key>` until the documents path identity
build, 2026-11: a concept is authored reference data nothing writes,
nothing acts on and nothing finds with MQL — so a document.) `help api`
stays the author surface; the concepts are the pedagogy, and a build that
adds a whole agronomy owes both. Milling, baking, farming, ranching,
carpentry, coopering, ice and the spoilage build ship them; `help <word>`
reaches them through the bare fallthrough's concept rung.

## The enriched `author-surface.json` pipeline

`api/help <Type.member>` renders **real content** — a readable signature
string + the TSDoc summary, plus `@param`/`@returns`/`@example` when
authored. This is driven by extending the existing author-surface
projection (`scripts/project-author-surface.ts`) rather than a new
artifact — one pipeline, one file, right-sized at runtime.

⭐ The surface also carries **`mixins: { name, summary }[]`** — one entry
per exported function whose name ends `Mixin`, keyed by the name minus
the suffix, its summary the first paragraph of the function's TSDoc
(entries with no summary are skipped; sorted by name). It exists because
a mixin factory's *returned class* does not reflect into the author
surface, so without it a mixin had no summary of its own.
`HelpCatalogue` prefers it when it projects `mixin.<Concept>` topics.

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

Relations fall out of the projection so the card navigates a hierarchy,
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

A collection topic gets one derived `see-also`: the **Api face of its
owning subsystem**, computed from `subsystem: banking.md` →
`api.BankingApi`, because a subsystem doc and its Api face are named for
the same concept. The owner-class edge beside it is usually dormant — a
`Document` subclass is not Stuff and not an Api, so it has no topic of
its own to point at.

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

The `help` controllers (`HelpTopicController` for bare + topic, `HelpVerbController`, `HelpApiController`, `HelpSearchController`) render every form off the index through `HelpApi`, on
the existing `shell.result` topic:

- bare `help` → the landing: the verb categories with counts, then the
  rulebook's other kinds, then a hint line;
- `help <category>` → that category's verbs, one line each;
- `help <word>` → the **bare fallthrough** (the YAML gained
  `fallthrough: true` + a top-level optional `topic` positional, so the
  common case no longer needs the `verb` subcommand word), resolved in
  the order a player means the word: **verb → category → concept →
  collection → api**. ⚠⚠ The concept rung was missing until 2026-09-17: authored
  help concepts (then `HelpConcept` rows) were indexed and searchable from the farmstead
  build on, and `help extraction` answered *"no help topic"* — four
  packs' concepts were reachable only through `help search`. The
  grain-chain drive was the first to type one (`HelpApi.conceptTopic`,
  `HelpCatalogue.findConceptTopic`);
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
inspection→help cross-link ("this bed is `Placing` → open the `Placing`
topic") is a clean add, not a rework: a `mixin.<Concept>` id is exactly
what an inspection card already knows (the mixin name), and the relation
graph is the navigation substrate. Wiring that bridge is Wave 2 and is not
built here.

⭐ **The first half of the bridge shipped with the client-surface build.**
The inspection card names the interface that said each line of `look`
(a gutter stripe per source — see [message-rendering.md § Look
segments](./message-rendering.md)); a stripe's popover fetches
`mixin.<Name>` from the REST API (`GET /api/help/topic?id=mixin.<Name>`),
caches the summary per session, and offers `help mixin.<Name>` as a
command. The `mixins` summaries above are what make that popover say
what the interface IS.

## The line — help vs wiki vs inspection

Everything is an instanced Stuff with state, so the discriminator between
the three reading surfaces is **does it change at runtime?**

| | Scope | Surface | Maintainer |
|---|---|---|---|
| immutable-at-runtime definitions (Species, Clade, units, types) + systems + commands + API + concepts | system / type | **Help** | developer |
| mutable instances' narrative (this NPC, this area) | instance | **Wiki** ([wiki.md](./wiki.md)) | community |
| mutable instances' current truth | instance, viewer-relative | **Inspection** (`look` / `analyze`) | the game |

*The goblin as a species* is help; *this goblin bleeding in the corner*
is inspection; *the legend of the goblin king* is wiki. Mutability
decides — and it is why help is developer-maintained and co-locates with
the thing it documents, where the wiki is the community's.

## Deferred (later waves / non-goals)

- **The client React help card** — designed in a layout-system session;
  this build ships the server data contract it consumes.
- **The rest of the inspection↔help bridge** (Wave 2) — the card's
  source popover reads `mixin.<Name>` summaries already (above); a
  quiet interface's "how you'd find out" line does not exist yet.
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
| Command roster accessor | `mud/api/command.ts` + `mud/platform/idea/api/CommandLogic.ts` (`allDefinitions`) |
| Index singleton + three projectors | `mud/platform/idea/HelpCatalogue.ts` |
| Collection topics' source | `packages/server/src/schema/*.yaml` + `mud/lib/persistence/SchemaDoc.ts` (the parser) |
| Directory read for them | `SourceTreeApi.listResource` + `readYamlResource` |
| Read chokepoint + capability filter | `mud/api/help.ts` |
| REST surface | `backend/HelpRoutes.ts` (mounted in `services/Server.ts`) |
| Verb | `content/platform/cmd/system/help.yaml` + `mud/platform/idea/cmd/system/Help*Controller.ts` |
| Warm wiring | `HelpApi.needs`, stood by `AppBootstrap` (no pack `boot:` entry since the scan sweep), `mud/lib/paths.ts`, `platform/content/platform/idea/HelpCatalogue.yaml` |

## ⭐ The help card exists now

`help <topic>` opens the `help` card, carrying the `HelpTopic` the read
already resolved as its `payload`. It is the one Wave 7 surface that had
no client-side existence at all — the REST catalogue shipped and nothing
rendered it.

⚠ The card carries **the topic the controller resolved**, not a second
fetch. One read, two renderings: the prose it emitted to the terminal
and the card body, which renders the same already-gated MML through the
same `MmlRenderer`. That is why `shell.result: both` is safe here —
there is one rendering, shown twice, and the test asserts the two are
EQUAL.

⚠ Only the TOPIC read opens a card. The landing page, the verb list and
the search results open none, so filtering them would leave `help`
doing nothing.

See [card-surface.md](./card-surface.md).

## History

- **Client surface (2026-10-31)** — `project-author-surface.ts` emits a
  `mixins` section from the `*Mixin` factory functions' TSDoc, and
  `HelpCatalogue` uses it as each `mixin.<Concept>` topic's summary
  (#89); the inspection card's source popover reads those summaries over
  the REST API. See `docs/plans/client-surface-plan.md` (D10) until it is
  retired at the sweep.
- **Documents path identity (2026-11-09)** — help concepts moved from
  `HelpConcept` Idea rows to `help-concept` documents at
  `content/concepts/`, named by their leaf, with `seeAlso:` as paths. See
  git log for the `reqs/documents-path-identity` branch.
