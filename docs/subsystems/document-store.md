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
kind; and since the documents path identity build (2026-11) the
**authored metadata families** that used to be Ideas — aromas, dirt
kinds, pump mechanisms, joints, forms, operations, help concepts,
formations, placements and topics. Dorm-room customization will be another (the first non-script
runtime consumer — see the Warren/lounge work).

### ⭐⭐ Kinds are declared — free text, ten traits (naming overhaul, 2026-11)

There is **no closed list of kinds** (the old `DocumentKinds.ts` table is
deleted). A kind is a **declaration** — a document of kind `kind` at
`<root>/kinds/<kind>`, shipped as a pack's `content/kinds/<kind>.yaml`
(the platform's land at `/platform/kinds/<kind>`) or saved by an author
under their own extent. Its leaf is the kind's name, and the namespace is
global: two declarations of one leaf are refused, by the installer across
packs and by `lint:document-kinds`. Who may change a kind is decided by
title on its declaration's path, like any document. The one kind code
knows by name is `kind` itself (`DocumentKind.BOOTSTRAP`).

A declaration is read into a **`DocumentKind`** value
(`lib/document/DocumentKind.ts` — pure, so the installer and the lint
scripts read declarations off disk through it) and served by
`DocumentKindCatalogue`, reached as `DocumentApi.kinds()`. Every write
and every paper read asks its kind synchronously, so the whole roster
stands at boot as `DocumentApi`'s declared `needs` (one
`listOfKind('kind')`); a declaration saved later is read back at once.

A declaration composes **traits**, each parameterized by data, plus an
optional **`fields:` schema** (`type: string | number | boolean | list |
map | any`, `required`, and `index:`/`default:` for a card key, below).
An unknown trait or parameter is an error, never ignored:

| trait | says |
|---|---|
| `shipped` | packs ship it: `{ dir, ext?, vanish? }`, or `{ in-tree: cmd }` (any `cmd/` dir — command views). A kind with no `shipped` is runtime-only and reserves no pack dir |
| `catalogued` | a service reads it: `{ by: <singleton path>, load: per-path \| whole \| boot }` — its `parse` checks a document's meaning, its `forget` is how an edit goes live |
| `keyworded` | what words find one: `{ keywords?: <field>, leaf?: bool, unique?: bool }` — `unique` means a word names one document across every pack (`emote`, `formation`, `topic`, `kind`) |
| `titled` | what one is called: `{ field }` (no title → the leaf) |
| `dated` | when: `{ field, unit?: game-s \| real-ms, indexed?: bool }` |
| `filed` | written only by its transport — `register`, `business`, `release`, `instrument`, `map`, `stream-token`, `program`, `berth`, `edition` (closed: each is a security path) |
| `paper` | how it prints: `{ template }`, Liquid |
| `ledger` | writes may add, never alter: `immutable`, or `{ grows: [field…] }` |
| `tree` | `{ parent: folder \| <field> }` — `DocumentKind.parentOf` |
| `secret` | never listed, never paper, never in the CMS tree; read by exact path only |

⚠ **No trait is a class a document becomes.** The only code a declaration
names is a service (`catalogued.by`, resolved with the class-file rule at
install and at a live edit) and a filing transport. Shipped dirs derive
from the `shipped` traits alone — a runtime-only kind's name (`maps`,
`herds`) no longer silently reserves a template dir.

⭐ **Every document has a title, a date and a page.** `titleOf` reads the
`titled` field (else the leaf), `dateOf` the `dated` field (game seconds
unless `real-ms`), and `DocumentApi.pagesOf(path)` renders any non-secret
document as paper: the kind's `paper.template` (over its data plus
`title`, `date`, `path`, `owner`, `owner_leaf`; `<page/>` breaks a page),
else one summary line — *title — date*, the date on the world's calendar.
`ls` prints titles.

The nightly reset **keeps the whole collection** (`documents.yaml`
`reset: verb: keep`): an undeclared kind is refused at the write, so every
row is of a declared kind. `wiki` is deliberately not a document kind (a
page has a revision log and a CAS edit path of its own).

⭐ **Four decisions the declarations carry, and why** (naming overhaul):

- **Validation is not a trait.** Shape is the declaration's `fields:`;
  meaning is the reading service's `parse`. The installer no longer
  hydrates a document into a value class to check it — doing so was the
  sign of a kind that wanted to be a class.
- **A reader that caches is the kind's declared catalogue.** `forget` is
  the only way an edit reaches a cache, and `recipe`, `archetype`,
  `help-concept`, `blueprint` and `name-bank` had readers nothing could
  tell — a CMS edit to one never went live. Each has a declared catalogue
  now.
- **`ledger` is the constitution's A2 reaching the store** — *ledgers are
  append-only* (measurement.md) — for the bore log
  and an instrument's history (`grows`), an edition and a rate card
  (`immutable`).
- **A release and an edition are dated in game seconds** (`publishedAtS`,
  `expiresAtS`, `frozenAtS`): both are in-world. `real-ms` exists for an
  out-of-character kind and none declares it today.

`venue-eighty-six` (the list a venue's enforcement brain keeps) is declared
plain — a `fields:` schema and no trait, written by the plain `save` — so
that its write is checked instead of wiped at the next reset.

⭐ **A kind belongs to the pack that declares it.** The installer reads every
pack's `kinds/` first; a pack that ships documents of a kind must declare it
or depend on its declarer (its `package.json`, transitively) — else it fails
at the `kind-declarer` step, naming the declarer to add. Without the rule a
pack's documents install only when the declarer happens to be in the set.
The platform's kinds are everyone's (pack zero is always installed), and a
kind declared at runtime has no declaring pack to check.

⭐ **The authored metadata kinds** — each cited by path, none with a
second name; most read through a value class in `lib/` over a
`DocumentCatalogue` (below — `Topic` keeps a dictionary of its own, and help
concepts have no value class):

| kind | shipped `dir` | value class | named by | `keyworded` |
|---|---|---|---|---|
| `aroma` | `aromas` | `lib/perception/Aroma.ts` | path; the folder is the tree ([senses.md](./senses.md)) | — |
| `dirt` | `dirt` | `lib/cleanliness/DirtKind.ts` | path (`DirtKinds.*` constants for the kernel's) | — |
| `pump-mechanism` | `pump-mechanisms` | `lib/pump/PumpMechanism.ts` | a pump's `mechanism:` path | — |
| `joint` | `joints` | `lib/craft/Joint.ts` | a bill's `joints[].method` path | — |
| `form` | `forms` | `Form` in `lib/craft/FormSpec.ts` | an instrument's `forms:` paths (the word resolves against that list, so two wheels may each know a `brick`) | — |
| `operation` | `operations` | `lib/vitals/Operation.ts` | `operate … for <word>` | leaf + title |
| `help-concept` | `concepts` | none — `HelpCatalogue` reads the kind | `help <word>`; `seeAlso:` paths | leaf + title |
| `formation` | `formations` | `lib/combat/Formation.ts` | a party's path; `party adopt <word>` | leaf + title, `unique` |
| `placement` | `placements` | `lib/spatial/Placement.ts` | a host's `placements:` paths | leaf + its `prepositions` |
| `shape` | `shapes` | `lib/spatial/Shape.ts` | a thing's `shape:` — a bare word is `/platform/shapes/<word>`, a path any pack's ([size.md](./size.md)); ⚠ not the potter's `form` | — |
| `reason` | `reasons` | `lib/command/Reason.ts` | a refusal's code — a bare word is `/platform/reasons/<word>`, a path any pack's; its `facts` and Liquid `says` word it at emit ([response-envelope.md](./response-envelope.md)); ⚠ not `standby-reason` | — |
| `topic` | `topics` | `lib/message/Topic.ts` | code (`.topic('act.combat')`); the dotted key is the leaf | leaf + title, `unique` |
| `cleanser` | `cleansers` | `lib/cleanliness/Cleanser.ts` | a material's `cleanser:` path ([cleanliness.md](./cleanliness.md)) | — |
| `fabric` | `fabrics` | `FabricSpec` in `lib/material/Construction.ts` | a garment's `constructionForm:` path (kernel forms stay words — [textiles.md](./textiles.md)) | — |
| `maturation-profile` | `maturation-profiles` | `lib/maturation/MaturationProfile.ts` | a vessel's charge by tag; the vessel keeps the path ([maturation.md](./maturation.md)) | — |
| `fraction-schedule` | `fraction-schedules` | `lib/fractionation/FractionSchedule.ts` | a still's charge by tag; the still keeps the path ([fractionation.md](./fractionation.md)) | — |
| `dyestuff` | `dyestuffs` | trade-dyeing's `src/lib/Dyestuff.ts` | the card key `material` (the dye bath's material) | — |
| `app-listing` | `apps` | none — `AppStore` (`TemplateApi.appStore()`) reads `app:` | `apps install <word>`: a pack PUBLISHES an app by shipping `content/apps/<key>.yaml` naming its `App` row ([aether-devices.md](./aether-devices.md)) | leaf, `unique` — a clash is refused at install |

`emote` (`unique`), `recipe` and `archetype` are `keyworded` too; a
`blueprint` is named by its leaf. ⭐ None of the four carries a key
line in its file any more (`verb:`, `recipeId:`, `archetypeId:`,
`blueprintId:` are gone; `Emote/Recipe/Archetype.fromData(data, path)`
read the word from the path).

⭐⭐ **The logistics build added three, all on the `water-right`
pattern**: `bill-of-lading`, `warehouse-receipt` and `rate-card` — all
path-keyed, declared `filed: business` by the haulage pack (`rate-card`
also `ledger: immutable`), and filed **under the filing business's
own branch**. That last part is the design rather than the plumbing:
a depot's records cover exactly what it handled and nothing else, which
is the honest consequence of shipping without customs — ⭐ *private
books do not aggregate*, and the first institution that can see across
is not the state but the depot, whose coverage is its market share.
⚠ `DocumentApi.saveAsBusiness` is the gate: derived owner, a path under
the business's branch, and a kind declared `filed: business` (the closed
allowlist it once kept is the trait now).

⭐⭐ **The persistence model added three, all HISTORY** (2026-11-24 — a
thing is kept by its container, history by a ledger): `cap-table` and
`hull-sale` (platform, `filed: business` — a company's owners after every
sale, gift or transfer, at `<company>/cap-table`; a yard's sold hulls at
`<house>/hull-sales/<book>`) and `water-ledger` (the water pack,
`filed: register` — one document per subject an aquifer, gate, conduit or
reach files through `WaterLedger`, a `Registrar`, under
`/system/water/ledger`). Each is its owner's hydration source, read once as
it stands up. They are declared documents, so the nightly reset keeps
them (the plan's per-kind `onReset: 'wipe'` was dropped when kinds became
rows). See [persistence.md](./persistence.md) § Who keeps what.

⭐ **`ship` — a ship's register entry** (harbors P14). One document per
ship a register holds, at `<register prefix>/<name slug>`
(`/platform/ships/terminus-city/petrel`), written only by its
`ShipRegister` through `DocumentApi.saveToRegister` (the herdbook's
transport, below — the second `Registrar`). The slug is the uniqueness:
a register refuses a second ship of a name because the path is taken.
The entry is TITLE only — `{name, kind, key, flag, registeredAtS,
owner}`, the owner being her company; who holds the company is its cap
table ([employment.md](./employment.md)). Pack-shipped for the authored
ships, runtime-written by a yard sale, `onVanish: 'keep'`. See
[ship.md](./ship.md).

⭐ **`berth-book` — a harbor's berth record** (the paper build, reconciling harbors, 2026-11-05): the berth
book's lines and hails, one document per harbor at
`<registerPrefix>/record`, filed by the harbor through
`saveToRegister` (the harbor is a `Registrar` of its own branch). The book
on the desk loads it on use and writes through; burning the book forgives
nothing. Runtime-written, `onVanish: 'keep'`. See
[harbor.md](./harbor.md).

⭐ **`berth-roll` — a landing's list of the ships lying at it** (harbors
review 1). It was a live `Persistable(Idea)` per landing with no behavior
beyond add/remove/list and nothing targeting it — so it is a document:
one per landing at `/platform/berths<landing path>` (the path keeps its
slashes; a harbor's berths are a prefix read), owned by the landing,
`data: { lying: [handle…] }`, runtime-written on the `map` pattern,
`onVanish: 'keep'`. ⚠⚠ Its write, `DocumentApi.saveBerth(ship,
landingPath)`, is an ownership bypass narrow by construction: gated
`FromMixin(Voyaging, caller === args[0])` (the ship herself, writing her
own line — the register transport's relational shape), and it takes **no
lying/gone flag** — the store reads whether she is made fast there now off
the ship, so even she cannot write a false line. Writes to one roll are
chained per path. The landing reads it with `DocumentApi.berthRollOf` on
first use and stands up every ship on it that is not loaded
([expanse.md](./expanse.md), `FrameSited`).

⭐ **`water-right` is the worked example of a filed record** (water
build; re-declared by the naming overhaul). A prior-appropriation right is
a *record* — dated, transferable, and meaningless if it can be quietly
lost — so it is path-keyed, `titled`, `dated` (`priorityDateS`) and
**`filed: register`**: `WaterRightRegistry` composes `Registrar` and writes
through `saveToRegister`, and the plain `save` that once reached it
ungated is refused (`document-filed`). The water pack declares the kind
itself (`content/kinds/water-right.yaml`) — ⚠ *a capability pack cannot
declare a kind* was true of the closed table and is not any more: a pack
ships its kinds' declarations; what stays closed is the set of filing
TRANSPORTS, because each is a security path.

⭐ **`aroma`** (vocabularies-to-rows, 2026-11): what a nose can read; authored
metadata ([senses.md](./senses.md#one-smell-source-air-nose)). It was an Idea
for one build and moved here because nothing writes one, nothing acts on one, and MQL
should not find one. It was the precedent the nine families of the
documents path identity build followed (the table above).

⭐⭐ **A document's identity is its PATH** (owner, 2026-11-08). A pack's document is
stored at `<pack root>/<shipped dir>/<rel>`, so a reference names that full path
(`/base-library/aromas/smoke`, `/species-and-names/name-banks/common`) and is read
with `DocumentApi.read`; the path is the scoped namespace, and two packs shipping
the same file name ship two documents. `naturalKey` — a second identity beside
the path, with a unique partial index per flat-key kind — is **gone** (documents
path identity, 2026-11): the path is unique (the `{path: 1}` index is `unique`),
the installer adopts a row by `{kind, path}` only, and what a document is
otherwise FOUND by is its derived catalogue card (below).

## ⭐⭐ Idea or document — the test

A row family is an **Idea** when any one of these holds; otherwise it is a
**document kind** (owner, 2026-11-07/08: *"Ideas are MQL addressable and
they carry mutable state, documents just hold a buncha metadata"*):

1. it carries **mutable state of its own** — a thing that is live, holds
   behavior over time, and is changed in place by acts on it. ⚠ **Being
   written at runtime is not this limb** (restated by the naming
   overhaul, 2026-11): a record written after it loads — a herd entry, a
   water right, a bore log, a release that is retracted — is what the
   `filed` and `ledger` traits mean, and its behavior lives in the service
   that reads it. `herd`, `water-right` and `warehouse-receipt` were
   weighed against this limb and stay documents (owner, 2026-11-20);
2. something **acts on** it — it is shadowable, a command targets it, or
   its behavior varies by class;
3. ⭐ it could be **a link in a query chain** — an MQL path a person would
   write to reach something else (`here:business:owner`), whether or not a
   controller targets it today. The governance palette will reach through
   institutions and referents, so Discipline, Spell, Corpo/Brand and
   Government stay Ideas (loaded on reach) while DirtKind, PumpMechanism,
   Joint, Form, Operation, HelpConcept, CombatFormation, Placement, Topic
   and Aroma are documents — and, since size and carrying W10j, the size
   `Shape` (the `Form` Idea that build first shipped).

A document may still carry **pure computation over its own data** — a value
class in `lib/` (`Aroma`, `Joint`) holds `rateAt(tempK)` as happily as a
threshold. Behavior that varies by CLASS is limb 2; computation is not.

## ⭐ The catalogue card — `keys`, and `findByKey`

Every document carries a **card**, `keys: Record<string, DocumentKeyValue>`
(`string | number | boolean | Array<string|number>`), each **derived
from the document's own data and never authored**, so it cannot drift
from the value it repeats. The card is DATA on the declaration — no
derivation function survives: a `fields:` entry with `index: <key>`
writes that key (with its `default` when the field is absent), `dated:
{ indexed: true }` writes `keys.date`, and `keyworded` writes `keys.word`
(every derived keyword, lowercased — the leaf unless `leaf: false`, the
`keywords` field, and the title with and without its article when the
kind is also `titled`). One method derives it, `DocumentKind.cardOf(path,
data)`, called by the write funnel (below) and by the pack installer's row
builder; `StoredDocument` stays kind-agnostic. An empty path throws.

Declared today, each with its reader: `release → {date, pinned,
releaseId}` (the press — [press.md](./press.md)), `recipe → {output}`
(`recipeFor`), `dyestuff → {material}`, and `word` for every `keyworded`
kind (emote, recipe, archetype, operation, help-concept, formation,
placement, topic, herd, `kind`). ⚠ A key nobody reads is a declaration to
delete.

⭐ **A document is NAMED through its kind's words** —
`DocumentCatalogue.named(text, among?)` hands a catalogue's documents to
`GrammarApi.nameAmong` with `keywordsOf` as their keywords (the one naming rule
— [mql.md](./mql.md) § Match-quality scoring), path order the tiebreak;
the controller's `named` prompts on a tie or refuses `no-such-name`.
`party adopt wedge`, `operate … for <op>` and the press's `callUp` name
this way.

**One index serves every key**: `{kind: 1, 'keys.$**': 1}`, a compound
wildcard, authored in `schema/documents.yaml` beside `{path: 1}` (unique)
and `{kind: 1}`. A wildcard keys each card field natively, so a sort is a
real index order (an attribute array `[{k, v}]` would sort by the array's
extreme element). The PM's old derived per-kind index loop is deleted —
the store's indexes are authored facts like every other collection's.

**`DocumentApi.findByKey(kind, key, q)`** is the one query surface over
the card — a read, ungated like `read`. `q: {eq?, min?, max?, prefix?,
order?: 'asc' | 'desc', limit?}`: `eq` for equality, `min`/`max` an
inclusive range, `order` reads in the key's order (an unbounded ordered
read constrains the key with `$exists`, so a document with no value for
it is not returned), `prefix` keeps the read under one path (a path
range, never a regex), `limit` caps. `findByKey('release', 'date',
{order: 'desc', limit: 30})` is the press, newest first (the release's
date is game seconds, `publishedAtS`).

## ⭐⭐ Resident by construction — `DocumentCatalogue`

⭐ The owner's rule (2026-11-09): *"you're responsible for loading objects
into the game when they're needed; you can't just say 'they're not loaded
yet' and give up."* So **a cited document is resident whenever the thing
that cites it is resident**, and a synchronous read always finds it.
Nobody above the model — not a controller, not a pack author — ever warms
anything, and no family is on a `boot:` list.

⭐⭐ **No class readies its Api's data** (owner, load-on-use W10,
2026-11-12). A dictionary an Api's reads stand on is declared in the Api's
`static readonly needs` (an `ApiNeed`: a catalogue path, or a loader for a
row-less catalogue) and `AppBootstrap` stands every decorated Api's needs
once, before the manifest; the async call gate still awaits them. No
setter returns such a load, no `onCreate` warms one, and no act calls
`StuffApi.ready` (it is `@internal`). Today: `AddressApi`, `GovernmentApi`,
`CorpoApi`, `HelpApi`, `MagicApi` (spells), `ConditionApi` (the roster),
`BulkableApi` (profiles + schedules), `ContainmentApi` (the kernel's
placements), `MaterialApi` (the kernel's dirt kinds — **not** the material
roster), `CraftingApi` (recipes + disciplines). Resident-by-construction
below now covers only the **per-path** citations (an authored or runtime
field naming one document) and material rows.

⭐⭐ **A family is SERVED by a catalogue singleton, reached through its
domain Api** — `CraftingApi.joints()`, `ContainmentApi.placements()`,
`MaterialApi.dirtKinds()`, `PartyApi.formations()`,
`ConditionApi.operations()`, `BulkableApi.pumpMechanisms()`,
`CraftingApi.forms()` / `.recipes()`, `MessageApi.topics()`,
`PerceptionApi.aromas()`, `MaterialApi.cleansers()` / `.fabrics()`,
`BulkableApi.maturationProfiles()` / `.fractionSchedules()` (the
`dyestuff` catalogue is the dyeing pack's, reached by its own controller —
a pack owns no Api). The catalogue at `/platform/idea/<X>Catalogue` is
the one place that family's documents are looked up; the value class keeps
only what the thing IS (no statics). The per-path families extend
`lib/document/DocumentCatalogue` (its `parse` is the installer's validator
too); its per-kind memory is a hard-private static slot, so a catalogue
stood up again reads the same memory. **A memory, never a roster**: a path
enters when something names it. Each kind names its service in its
declaration (`catalogued.by`), which is how the go-live finds it —
`DocumentApi.catalogueFor(kind)` — and how a document's meaning is checked
(the service's `parse`, read off its class when the service is not
resident).

⚠ Why not statics on the value classes (W5d, at review): that is where the
build first put these lookups — tagged `@internal` to get under
`lint:lib-statics`, with no caller audit — and it left a family's lookups
on a class nobody thinks to search. A catalogue reached through its Api is
discoverable, documented, and part of the author surface.

| door | what it does |
|---|---|
| `ensure(path)` (async) | read and build the document if it is not resident; concurrent first reads share one load; throws naming the path when no document of the kind is there |
| `need(path)` (sync) | the resident value, or **`DocumentNotResidentError`** (`lib/errors.ts`, naming the kind and the path) — an engine defect, loud for us, never a player-facing refusal and never a skip |
| `peek(path)` (sync) | resident or `null`, starting nothing — for a reader whose honest answer to "not read" is "not mine to read" |
| `remember(path, value)` | the one way a value enters (unit fixtures use it too) |
| `all()` / `byWord(word)` | the lazy dictionary, only for a kind a player names by a typed word or a verb lists: one `listOfKind`, or one `findByKey(kind, 'word', {eq})` (a resident leaf answers first) |
| `forget(paths?)` | the go-live: the catalogue forgets those paths (or all). The installer and a CMS save reach it as `DocumentApi.catalogueFor(kind)` — by the `catalogued.by` path the kind declares |

(The sync door is named `need`, not `require`: `lint:imports` reads a
`require(` declaration as a dynamic require.)

What makes "resident" true, per family:

1. **An authored citation loads through the field's setter**, which
   returns the load for the template applier to await — in clone and in
   restore mode alike (`Placing.setPlacements`, `Assembled.setBill`, a
   pump's `mechanism`, an instrument's `forms`). A row is not live until
   what it cites is.
2. **Runtime state that cites a document is restored through the same
   awaited setters** (`Soilable.setDeposits`/`setStains`,
   `Assembled.setJoints`, a placed item's `ContentPlacement` in the
   restore pass, a party's formation in `materializeParty`).
3. **A runtime write** happens in an act that already holds the document
   (a word was resolved to it; a pack's own dirt kind is ensured by the
   act that lands it), so the written path is resident by induction.
4. **The kernel's own built-in citations stand at boot** as their Api's
   `needs` — the kernel placements (`ContainmentApi`) and dirt kinds
   (`MaterialApi`), one prefix read each. (Until W10 a host's `onCreate`
   loaded them.)

⚠ **The one engine-wired load is topics**: named by code and cited by no
holder, so the message pipeline warms the dictionary on the first scene
and queues dispatch in order behind it ([topics.md](./topics.md)). The
dictionaries that stay Stuff split two ways: those an Api declares
(`RecipeCatalogue`, `DisciplineCatalogue`, `SpellCatalogue`,
`CorpoCatalogue`, `GovernmentCatalogue`, `HelpCatalogue`) stand at boot as
its `needs`; the rest (`SoulCatalogue`, `ArchetypeCatalogue`,
`BlueprintCatalogue`) warm on their first async use. A sync read of a
cold one throws `DocumentNotResidentError` rather than answering empty.

⭐ **Two Idea families serve the same doors by hand** (load-on-use round
two): `MaterialApi.materials()` and `ConditionApi.conditions()` reach
`platform/idea/MaterialCatalogue` / `ConditionCatalogue` by `singletonSync`
— no `onCreate`, no yaml row, no boot line — and answer `ensure` · `need` ·
`peek` over **live Stuff** (the rows stay Ideas: query links, class-varying
behavior). A cold `need` throws `DocumentNotResidentError('material' |
'condition', path)` — the error's message says *"not a missing row"*, so an
Idea kind reads right. The roster (`ensureRoster()` → `roster()` /
`byTag`) is only for the few reads that ask for every row — the
conditions' stands at boot as `ConditionApi`'s need; the materials' never
does, and is reached only through async `MaterialApi.all()` /
`findByTag()`, which load it themselves — and its state
is the class's, so a suite's `clearAll` cannot un-stand it. The material
catalogue adds `ensureCited(paths)` / `ensureCitedIn(value)`, the load a
restoring setter returns for a runtime field that carries material paths
(it answers `undefined` when nothing is cold and never rejects). How each
material reaches residence — holder, relation, act, roster — is
[race.md](./race.md)'s; conditions stand at boot
([vitals.md](./vitals.md)). Two implementers by hand, not a shared base:
the third promotes it.

## What a record is

`StoredDocument` (`lib/document/StoredDocument.ts`, collection `documents`):

| field   | meaning |
|---|---|
| `path`  | canonical key and the only identity (unique index); owner/scope encoded in the path — see the namespace taxonomy below |
| `owner` | the owner's durable `templatePath` (set from context, never the caller) |
| `kind`  | **what kind of object lives here** — `'msh'` (a script), `'emote'`, `'recipe'`, `'name-bank'`, `'blueprint'`, `'command-view'`, `'release'`, `'water-right'`, later `'dorm'`, … |
| `data`  | the arbitrary JSON payload |
| `keys`  | the catalogue card, derived from `data` by the kind on every write (above) — never authored |

The store is **kind-agnostic**: it persists and serves `{path, owner,
kind, data}` and never inspects `data` — except that a declared kind's
card is derived from it on write. Each `kind`'s consumer owns the
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
| 4 | **secondary keys are a derived card** | anything besides the path a row is found by is a key its kind DERIVES from `data` onto `keys`, served by the one wildcard index — no per-kind index, no authored second name (originally *≤ 1 secondary key*, the `naturalKey` unique partial, retired 2026-11) |
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
one-word tell, visible in every singleton's `onCreate`:

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
| ⭐⭐ **`office_holders`** | **move** | 0 indexes, 5 sparse rows, `onCreate` does **no DB work**, and `holderOf` = *explicit row else the founder default* — a path lookup with a fallback |
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

### ⭐⭐ The register transport — a shape, not a name

*"Who keeps this register?"* has a mechanical answer as well as a design
one, and it is `Registrar` (`lib/document/Registrar.ts`).

A register needs a write the ordinary gate cannot give it: `save` admits
the **branch owner**, and the whole point of a register is that the
**subject is not the owner** — *you file; you do not hold the pen.* A
keeper filing a head against the herdbook is not the ranching trade, and
granting them the branch would hand them the pen.

> ⚠⚠ **The first cut was a pinned `saveHerd`, with
> `/trade/ranching/herds`, `/trade/ranching` and a
> `FromTemplate('/trade/ranching/idea/HerdRegistry')` gate AS KERNEL
> CONSTANTS.** That is a pack's namespace hardcoded in the engine — the
> thing the pack system exists to prevent — and it did not survive the
> second register: a studbook, a claims book or a land registry each
> wanted another method and another three constants.

So the kernel learns the **shape**, exactly as `saveRelease` learns
`Publisher` rather than a press path. A register declares three things
and the transport derives everything from them:

| declaration | the herdbook's |
|---|---|
| `getRegisterPrefix()` | `/trade/ranching/herds` |
| `getRegisterOwner()` | `/trade/ranching` |
| `getRegisterKind()` | `herd` (a declared kind that is `filed: register` — a register files only a kind declared for it) |

⭐ **What stops a class declaring itself registrar of your home
directory is structural, not an allowlist:**

> **the owner must be a prefix of the register's own template path, and
> the register's branch must lie under that owner.**

`/trade/ranching/idea/HerdRegistry` may own `/trade/ranching` and file
under `/trade/ranching/herds`; it may not own `/home/somebody`. The
capability is *"the society keeps its own book"* — expressible without
the kernel knowing which societies exist.

⚠ And the caller contract is relational: `FromMixin(Registrar, { where:
caller === register })` — the register must be the thing calling, and it
must be writing for itself. Nobody else can reach the transport at all.

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
  input) — a range over the unique path index, never a collection scan.
- `DocumentApi.listOfKind(kind)` — every document of a kind (a lazy
  dictionary's warm).
- `DocumentApi.findByKey(kind, key, q)` — documents of a kind found by one
  key of their card (above).
- `DocumentApi.kinds()` — the `DocumentKindCatalogue` (every declared
  kind); `DocumentApi.catalogueFor(kind)` — the service a kind declares.
- `DocumentApi.pagesOf(path)` — any non-secret document as paper (above);
  `DocumentApi.callUp(reader, word)` — a release or a paper of your own,
  named by the one rule.
- `DocumentApi.save(path, kind, data)` → find-or-create. **Access-gated**,
  **owner-stamped from context**, **provenance-recorded** (below), and
  through the one write funnel (below) — so a `command-view` save is
  schema-checked by its service and goes live through its `forget`.
- `DocumentApi.delete(path)` → the same gate as `save`; no provenance row
  (a deletion is not authorship); returns whether a row existed. The
  emote catalogue's `delete` rides it.
- ⚠⚠ `DocumentApi.saveRelease(publisher, path, data)` — **a named write
  transport**, and an ownership bypass by construction. See below.
- ⚠⚠ `DocumentApi.saveToRegister(register, path, data)` — the **generic**
  one: a `Registrar` files into the branch it keeps the book for. See
  *The register transport* below.
- ⚠⚠ `DocumentApi.saveBerth(ship, landingPath)` — a ship's own line on a
  landing's `berth-roll`, truth read off her; `DocumentApi.berthRollOf`
  reads the roll. See *`berth-roll`* above.
- The other filing transports — `saveAsBusiness`, `saveInstrument`,
  `saveMap`, `saveStreamToken`, `saveProgram`, `freezeEdition` — each the
  one writer of the kinds declared `filed:` by its name. `saveProgram`
  (`/platform/programs/<name>`, kind `program`, catalogued per-path by
  `ProgramCatalogue`) is `StreamLogic`'s alone, writing `by: 'program'`
  ([livestream.md](./livestream.md)).
- `DocumentApi.mayWrite(actor, path)` — **the gate below, asked without a
  write**: would `save` let this actor write here? (stream arrival: a
  program's pen is `mayWrite` on `<owner>/programs/<name>`, so a player's
  own home counts — `AccessApi.canAtPath` alone refused it.)

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

### The command-view save

A `kind: 'command-view'` save names TypeScript — its `controller:` (at
the verb or any subcommand) and every validator / `requires` reference —
but **naming code is content**: the save is gated by title over the
path like any other document, and nothing asks the wizard axis. (Until
the wizard-axis cleanup, 2026-11-05, a save that changed that set ran a
code-naming gate, `codeFieldsChanged`, requiring `AccessApi.isWizard`;
it was deleted with the template gate — see [access.md § The code-trust
lockdown](./access.md). A validator is still a gate, which is why
a view's validators are its titleholder's to change and nobody else's.)
The kind is `catalogued` by `CommandLogic` (`/platform/idea/api/command`,
`shipped: { in-tree: cmd }`): its `parse` validates the view against the
command schema in the funnel, and its `forget` reloads the view — the CMS
edits `look.yaml`'s help and `help look` changes without a restart.
The installer writes through `PersistApi`, bootstrap-exempt like
templates.

### ⭐⭐ The one write funnel (naming overhaul D-A4)

Every writer — the plain `save` after its title gate, and each filing
transport after its own authority check and path scheme — ends in one
private `writeDocument` in `DocumentLogic`, the only code that constructs
or saves a `StoredDocument`. In order:

1. the kind is declared — else **`document-kind-undeclared`** (nothing is
   written that the nightly reset would silently wipe);
2. the writer matches `filed`: a plain `save` of a filed kind is refused
   **`document-filed`**, naming the transport (title on the path is not
   enough to write somebody's herd record); a transport writing a kind it
   does not file is a defect (a throw);
3. the data fits the kind's `fields:` — else **`invalid-document { path,
   why }`**;
4. the meaning check — the service's `parse`, when the kind is
   `catalogued` (the same check the installer runs);
5. the ledger — `ledgerFailures` against the stored row: an immutable kind
   is written once, a growing field keeps every earlier entry, else
   **`ledger-altered`**;
6. a declaration edit is vetted against the documents of its kind —
   **`kind-in-use`** when they would no longer fit its `fields:`, when a
   newly `unique` word would name two of them, or when its `filed` would
   change while any exist; deleting a declaration is refused the same way
   while documents of it that no pack shipped remain;
7. a `unique` kind's words must name no other document;
8. the card is derived, the row saved, and the service told to `forget`
   the path — the edit is live on the next read.

The title refusal is a `Refusal` too — `document-access-denied { path }`
(its `message` keeps the gate's sentence for a REST caller).

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
much. Since the naming overhaul the kind is declared `filed: release`, so
a plain `save` of a release is refused outright (`document-filed`); the
read still **re-verifies what the transport guarantees** (`Release.publishedByOwner` — the owner must resolve to a
live publisher, at a path under that publisher's feed) — defense in
depth, for a row written before the trait existed or by the installer.

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

A `secret` kind (the stream tokens) never appears in the tree, nor in any
`list` / prefix scan — it is read by exact path only. Author-tier read
gate (like content). Wired end-to-end: the REST
`parseBackend` admits `'document'`, and the client explorer lists the
`documents` root.

## Deferred

The fuller per-`/home/` access model (sharing, grants beyond the
self-owner), non-script document **creation** via the CMS, and the first
non-script runtime consumer (dorm-room customization data, hydrated onto
a Warren constituent — base-template + customization-document) all land
later. The substrate is built to accommodate them; scripts, the
pack-installed kinds, and releases ride it today.

## History

- 2026-11-09 — documents path identity: `naturalKey` and its per-kind
  unique partial indexes are gone; the path is unique; every document
  carries a derived `keys` card served by one wildcard index and read with
  `findByKey`; the nine metadata families became document kinds over
  `DocumentCatalogue`, resident by construction. See git log for the
  `reqs/documents-path-identity` branch.
- 2026-11-10 — review of !560: the per-family catalogues came back as
  `DocumentCatalogue` singletons reached through their domain Api
  (`CraftingApi.joints()`, `PartyApi.formations()`, …), and every Api
  gained one readiness routine (`static needs` + `StuffApi.ready` — see
  call-security.md). Code comments cite the build's decisions as
  `(documents Dn)`; the index:

  | D | decision | where it lives now |
  |---|---|---|
  | D1–D4 | the kind declares its keys; one wildcard index; typed words unique at install; `findByKey` | § The catalogue card |
  | D5 | the press reads newest first by key | press.md |
  | D6 | the metadata families become documents (the aroma recipe) | § Idea or document |
  | D7 | ⭐⭐ a cited document is resident whenever its citer is (per-path setters return their load; a cold sync read throws) — ⚠ for an Api's dictionaries retired by load-on-use W10: they stand at boot as the Api's `needs` | § Resident by construction |
  | D8 | the four flat-key kinds: the leaf is the word | content-packs.md |
  | D9–D10 | which catalogues survive; the Ideas that stay Ideas — loaded on reach then, at boot as their Api's `needs` since W10 | § Resident by construction; civics.md, corpo.md, magic.md |
  | D11 | disciplines: bare keys and a lazy dictionary | advancement.md |
  | D12 | Placement: the offering host loads it | spatial.md |
  | D13 | Topic: the engine-machinery exception, the ordered scene queue | topics.md |
  | D14 | the aroma folder is the tree | senses.md |
  | D15 | the lints follow the rows | lint-family.md |
  | D16 | a CMS edit reaches the catalogue (`catalogueFor(kind)?.forget`) | § CMS |
- 2026-11-11 — load-on-use round two (MR !575): five material-adjacent
  families became document kinds — `cleanser`, `fabric`,
  `maturation-profile`, `fraction-schedule`, `dyestuff` (the last keyed by
  its `material` card) — and their Idea classes and boot lines went; the
  material and condition catalogues gained the sync doors and stopped
  booting (§ Resident by construction). `DocumentNotResidentError`'s
  message now says *row*, not *document*. The plan's decisions are cited in
  code as `(load-on-use Dn)`. ⭐ In its review (W10) no class may ready its
  Api's data: every Api's `static needs` stand at boot (`AppBootstrap`),
  and the readying setters, `onCreate` loads and caller `StuffApi.ready`s
  went.
- 2026-11-22 — naming overhaul (`build/naming-overhaul`), Stage A: the
  closed `DocumentKinds` table is deleted; kinds are free-text
  declarations at `<root>/kinds/<kind>` read into `DocumentKind` (ten
  traits + `fields:`) and served by `DocumentKindCatalogue`
  (`DocumentApi.kinds()`); one write funnel; filed records, enforced
  ledgers, secret kinds; every document a title, a game-time date and a
  page; the reset keeps the collection; the Idea-or-document limb 1
  restated in trait terms. Decisions cited in code as `D-A*`.

### The history kinds (persistence-model, 2026-11-24)

`origin/master..HEAD` on `build/persistence-model` (MR !654): the
`cap-table`, `hull-sale` and `water-ledger` kind rows, and `WaterLedger`
as a register.
