# Wiki — the community encyclopedia of the world's nouns

The source of truth for the page model, the render pipeline, the
component/snippet contract, the permission anchoring, and the markup
additions. Read this before editing under `lib/wiki/`, `obj/Wiki*`, or
`cmd/system/wiki.yaml`.

Design lineage: the wiki requirements + plan docs (the complete design
and the build carve) were **retired at the pre-merge sweep** — this doc
superseded both. The open design surface that outlived them is
wiki-slate.md.

---

## What it is, and what separates it from `help`

A **community-maintained encyclopedia of the world's nouns** — materials,
biomes, species, places — plus lore, guides and system pages.

The difference from the help system is not subject matter but
**substance**: in this engine a thing's behaviour *is* its mixin
composition, so an article can carry a **live architecture panel**
beside its authored prose. Help tells you what `plant` does; the wiki
tells you what oak *is*, and therefore what you can do with it as a
consumer and how you work it as labour. The panel is derived at read
time and cannot go stale.

---

## Where the code lives

| Concern | Home |
|---|---|
| Page + revision rows | `lib/wiki/WikiPage.ts`, `lib/wiki/WikiRevision.ts` (Documents) |
| Namespace zone | `obj/WikiNamespaceZone.ts` — **instanceable, so `obj/`** |
| Render contract + level vocabulary | `lib/wiki/render.ts` (`SpoilerLevels`) |
| Budget | `lib/wiki/RenderBudget.ts` |
| Snippet syntax | `lib/wiki/Snippet.ts` |
| Sections + sticky anchors | `lib/wiki/Section.ts` |
| Word diff | `lib/wiki/SourceDiff.ts` |
| Components | `lib/wiki/components/<name>.ts`, sole export `component` |
| Page state + mutations | `obj/WikiRegistry.ts` (`extends Idea`) |
| Render pipeline | `obj/WikiRenderer.ts` (`extends Idea`) |
| Verb | `cmd/system/wiki.yaml` + `platform/idea/cmd/system/WikiController.ts` |
| Starter articles | the `wiki-starter` content pack (`packages/content/wiki-starter/content/wiki/<ns>/<slug>.md`) — the installer's `wiki` kind |
| Client card | `client/components/WikiCard.tsx`, fed by `publication.wiki` |

**No `*Api` was added**, by constraint. `obj/<Name>Registry.ts` is the
shipped shape for a gated, state-owning singleton with no Api face
(`AccessRegistry` is the precedent).

> ⚠ **`@CallSecurity` is INERT on a `Document`.** `ProxyApi.wrap` is
> called only from `StuffApi.create`/`clone`/`createSync`; a Document is
> constructed with `new`, never wrapped, and instance-method gating is
> enforced *by that proxy*. This is why behaviour lives on two `Idea`
> singletons rather than on `WikiPage` — a gate there would silently do
> nothing.

`WikiNamespaceZone` is in `obj/` because a template's `class:` names it
(`seeds/wiki/*.yaml`), and **nothing instances `lib/`** — enforced on
both axes by `pnpm lint:instanceable`. The Documents stay in `lib/`
because nothing instances *them* (`lib/parcel/ParcelRecord.ts` is the
precedent).

---

## The page model

`_id` is identity; `slug` is a label. A rename appends the old slug to
`aliases`, so **the set of names a page answers to only ever grows** —
which is what makes `pageId#anchor` a durable citation, and a broken
citation a broken lesson (the college slate's cite-never-restate rule).

Slugs and aliases share **one name space per namespace**: a name
resolves to at most one page, and claiming a held name is refused
*naming the holder* — "taken" tells an author nothing they can act on.

A subject-bound page and a free-standing one differ in exactly one
field, `subject`. There is no second kind and no marker mixin:
documentability is a property of a **template path**, not something a
game class opts into.

### The typed subject

```yaml
subject: { kind: template, ref: /stuff/idea/material/oak }
subject: { kind: mixin,    ref: CombustibleMixin }
subject: { kind: command,  ref: inventory/plant.yaml }
subject: null
```

Typed rather than a bare `domain` path because **not everything is
Stuff**. A Template describes Stuff; the things that are not are other
Documents and *compilation units* — YAML files, modules, and interfaces,
which is what a mixin is. The case this design started from (a player
understanding a thing's architecture as consumer and as labour) is a
**mixin**, which has no `domain` row at all. The kind set is open: a new
kind adds a resolver rather than reinterpreting a string.

### Revisions

`wiki` holds current state; **`wiki_revisions`** holds an append-only
log — the shipped `parcel_events`→`parcels` pattern. A separate
collection rather than an embedded array for two reasons, both about
scale: a heavily-edited page would eventually hit **Mongo's 16MB
document cap**, and every page *read* would otherwise drag its history.

Snapshots, not diffs: rollback is a copy, not a replay.

> ⚠ **A revision stores the RESULTING body**, not the "prior" body the
> requirements phrase. A draft is "a revision with `published: false`",
> and its point is holding the *proposed* text while the page serves its
> last published one — under a prior-body model a draft has nowhere to
> put what the author wrote. What the requirement is *for* is intact:
> every state the page has ever had is a row.

`rev` is a **compare-and-swap token**. An edit submits the rev it was
based on; a mismatch is rejected with all three bodies and **no
auto-merge**, because a machine-merged paragraph reads as somebody's
writing and is nobody's. Without it, last-write-wins silently destroys
work — and history does not save you: it faithfully records that B
overwrote A, and A's edit is still gone.

---

## ⭐ The reveal model — two axes, one gate

Every fragment carries an **effective spoiler level**, checked against
two different numbers:

| axis | what it is | what happens above it |
|---|---|---|
| **capability** | the reader's *ceiling*, derived from identity | **DELETED server-side** — never serialised, never on the wire |
| **appetite** | the reader's declared *preference* | **KEPT and TAGGED** `<spoiler level="n">` for the client to collapse |

Conflating them leaks. `level > ceiling` deletes; `ceiling >= level >
appetite` keeps and wraps. **Every test asserts on the emitted string** —
a tree can carry the right levels and still serialise the wrong bytes.

### ⚠⚠ What reveal does NOT answer — appetite is not epistemics

Both axes above are about **the reader at the keyboard**, not the
character in the world:

| | question | resolved by |
|---|---|---|
| **appetite** ✅ built | *does this reader want to be spoiled?* | a preference, and one click |
| **capability** ✅ built | *is this reader allowed to see it?* | identity / authoring authority |
| ⚠ **epistemics** ✗ **no substrate** | *does this **character** know this?* | nothing today |

⭐ **A collapse toggle is not a lock, and a lock is not knowledge.**
Nothing here models what a character has *learned* — there is no
per-viewer knowledge state over world facts, no way to earn a
measurement, and no way to be **wrong** about one. `spoiler: 1` on a
material's density means *"a reader may prefer to click for this,"* not
*"this character has never measured oak."*

⚠ This is the specific mistake to avoid: reading the level tables — the
one above, or *What else is level 1* further down — and concluding that
some surface which prints a level-1 value is **leaking withheld data**.
It is not. Level 1 is *collapsed by default, never forbidden*, and
reclassifying density and hardness to it was only coherent **because** of
the ordinary-player rung above. A surface that prints one is at most
failing to offer the collapse.

**The contract itself lives on the field, not here** —
`FieldMetaEntry.spoiler` in `lib/mixin.ts`, declared there precisely so
the level travels *"wherever it surfaces — a wiki panel, the Studio,
help, a future codex."* Read that docstring before reasoning about
levels; this doc describes the wiki's *use* of the model, not its
definition. A surface that drops the level on the wire (as
[char-gen](./char-gen.md)'s `SpeciesDossier` does) has a **wire-type**
bug, not a content one.

The epistemic half is the deferred
identification-slate —
`analyze X with Y`, real Material chemistry, partial identification,
misidentification: knowledge as something **earned per viewer** rather
than revealed by preference. The two are complements and should both
exist; what must not happen is one being mistaken for the other.

### The MAXIMUM rule

Four things can assign a level: the page's frontmatter default, an
inline tag, a component's declared floor, and a derived field's
`spoiler`. **The effective level is the MAXIMUM of all of them** — never
the nearest, never the innermost.

A level-0 fact inside a level-2 section is level 2, because knowing a
fact appears *under that heading* is itself the reveal. Max is also the
only rule that composes: under nearest-wins, wrapping content in a
lower-level container reveals it.

### The capability ladder

Derived from identity, never stored:

| level | who |
|---|---|
| 3 | a wizard |
| 2 | may mutate the namespace zone (its `'owner'` role) |
| 1 | may edit in the namespace — **or is simply a player** |
| 0 | a guest, and an unresolved principal |

#### ⭐ Why an ordinary player sits at 1

The upper rungs are **authoring authority**, and for a while every
rung was. A signed-in player sat at 0, which made `spoiler="1"` mean
*no ordinary player may ever see this*, and meant the **appetite** axis
— the click-to-reveal — could never fire for the people it was built
for. Two axes, one of them structurally dead.

Level 1 is therefore **ordinary reader content a reader may choose not
to be shown**: collapsed for a reader whose `wiki.spoilerAppetite` is
0, one click to open, plain for a reader who asked for everything.
Levels 2 and 3 are unchanged and stay authority-gated, so opening 1 to
everybody does not open the levels above it.

> ⚠ A **guest stays at 0**, and not only for symmetry with the write
> gate: a guest persists nothing, so the very setting that would let
> them opt in cannot be remembered for them. "Collapsed until you
> choose otherwise" is not an offer you can make to somebody whose
> choice evaporates at disconnect.

> ⚠ **This is not a knowledge model.** The ladder cannot express "this
> character has worked oak and therefore knows its density" — that is
> the Transcript/Competence axis, which the panel does not consult.
> Reaching for a spoiler level to model something a character should
> **earn** is the mistake worth naming here.

> ⚠ `AccessApi.can` **ignores its `action` argument** (the parameter is
> discarded). The ladder differentiates by *which predicate it calls*,
> not by the action string. Do not write a test expecting
> `can(a,'read',z)` and `can(a,'edit',z)` to differ.

### The reader is derived, never passed

`WikiRenderer.render` takes `(body, opts)` and `opts` has **no reader
field**. Both faces resolve the acting principal from
`ExecutionContextApi.getActingAuthor()` internally. With a parameter,
"over-capability content never crosses the wire" is a convention any
caller can break; derived, it is a property of the code path.

### `spoiler` on a field

`FieldMetaEntry.spoiler?: 0|1|2|3` — the reveal level of a field's
*value* wherever it surfaces. Declared on the **field**, because the
same field is a spoiler in a wiki panel, the Studio, help, and a future
codex alike; a policy table owned by the wiki would be wrong the moment
anything else rendered the same value.

`SpoilerLevels.ofField(ctor, field)` is the **one seam**, and the single
place the fail-open default lives.

> ⚠ **Untagged means level 0 — open.** Default-spoiler would empty every
> panel until several hundred mundane fields were tagged and would train
> authors to tag reflexively. Density and hardness are not spoilers; a
> species' resistances are.
>
> The cost is real: a newly-added spoilery field is visible until
> somebody tags it. It is covered **by enumeration, not inference** —
> `mud/__tests__/wiki-spoiler-fields.snapshot.test.ts` lists every
> surfaceable field with its level, so an untagged spoiler is a review
> diff rather than a production leak. **A diff there is a review item,
> not a breakage.** Blessing it without reading it is the one way that
> file stops working.

Today's tagged set: a trap's `trigger` / `delivery` /
`traverseConsequence` / `groundTriggered` / `dropDestination`, at level
2 — what the trap does and how to avoid it.

---

## The render pipeline

```
render(body):
  1. parse              markdown (article dialect) → MML tree
  2. expandSnippets     fixpoint, depth cap, cycle detect
  3. resolveLinks       [[Page]] → <link> / redlink
  4. resolveComponents  path-resolved, budgeted
  5. gate               MAXIMUM levels → omit / tag
  6. emit               serialise back to MML
```

### ⭐ Stage 1 is a **conversion**, and it lives at READ time

An article body is **markdown** — the article dialect below, exactly
what a player types into `wiki edit`. Stage 1 runs
`Mml.markdownToMml(body, undefined, { longForm: true })` and parses the
result.

Converting at **write** time was the alternative and is worse in a way
that compounds: the stored source would be a machine translation of
what the author typed, so `history`, `diff`, `--section` and the edit
box would all show them MML they never wrote, and every subsequent edit
would be an edit of the translation. Converting at read time costs one
parse per render and keeps **the stored source byte-identical to what
was typed**.

> ⚠ The consequence, and it bit twice during the build: anything that
> reads the **source** must understand markdown, not MML.
> `Sections` (headings, sticky anchors, `--section`) and
> `WikiPage.linkRefs` (the whole link graph) are both source scanners,
> and both failed **silently** — an empty heading list and a phantom
> redlink, not an error. Both now read either form. A third source
> scanner is the thing to check for whenever this file changes.

**The stage list is frozen.** Four things each want to be outermost and
the wrong order is a silent correctness bug rather than a crash, so the
order is asserted **by observation**: a snippet emitting a component
emitting a `[[link]]` leaves the link unresolved, proving 3 ran between
2 and 4.

> ⚠ **Documented limitation:** a component's *output* misses stage 3, so
> a `[[link]]` a component emits stays literal. The fix, if it becomes
> unacceptable, is re-running stage 3 — a stage *body* change, not a new
> stage.

### ⭐ Body, never a page id

`render` takes a **body**; `pageId` rides `opts` for self-reference
only. That is what makes `wiki preview` the same code path as a saved
read, for free. Keyed on a page id, preview becomes a second rendering
path — and second paths drift until preview lies about what saving will
do.

### `redactSource`

The same walk, ceiling only, no tagging: source with over-ceiling
fragments deleted. It feeds `history`, `diff` and the conflict payload,
so a reader cannot read past their ceiling by asking for a diff instead
of a page.

Above the ceiling a fragment is **absent** — not redacted, not counted,
no "1 change hidden". A redaction marker is the leak in miniature.

It returns its input **byte-identical when nothing was removed**: parse
/serialise normalise entities, so re-emitting would rewrite bodies it
was only meant to filter.

---

## Components and snippets — the capability line

**Snippets compose and parameterise; only components reach live state.**
An author can build any presentation they like and cannot reach game
state except through a component a developer wrote.

### Components

A module at `lib/wiki/components/<name>.ts` with a sole `component`
export — the **brain pattern**, applied to markup. Re-resolved per
invocation via `StuffApi.resolveExport`, so adding a component is
dropping in a file. No registry.

```ts
export const component = class WikiInfobox {
  static label = 'infobox';
  static spoilerFloor?: 0 | 1 | 2 | 3;   // for a component whose PRESENCE reveals
  static render(props, children, ctx): MmlNode[] | Promise<MmlNode[]>;
};
```

> ⭐ **A component receives no reader identity and no capability
> ceiling.** `ComponentContext` carries `budget` and an optional
> `pageId` and nothing else. If components gated, the reveal model would
> be enforced in N places written by N people and one buggy component
> would leak. With annotation-only there is exactly one gate, and a
> component *cannot* leak because it never learns what the reader may
> see. **Do not add a reader field.**

Returns **nodes, never a string** — a string would bypass sanitisation.
A throwing component yields an inline error naming it and the rest of
the page renders.

> ⭐ **No component can trigger a page render**, and it is a GATE, not a
> depth counter: `render`/`redactSource` are
> `AnyOf(FromModule(WikiController), FromTemplate('/platform/idea/WikiRegistry'))`,
> and `/lib/wiki/components/*` is in neither. There is no depth at which
> recursion becomes allowed.

A component's tag name becomes a **module basename**, so the charset
rule (`[a-z][a-z0-9-]*`, in `api/mml/tags.ts`) is load-bearing: `../`,
slashes and dots are unrepresentable before any resolver sees the
string.

Shipped: `infobox` (pure presentation), `image` (a key; **no upload
path**), `help` (transclude a verb's topic), `composition` (the live
architecture panel).

#### ⭐ `composition kind="template"` reports VALUES, not schema

The panel is a **game reference**. The schema view — what a field is,
how it persists, whether it is authorable — is what `help` and the
generated API docs are for, and putting it on an article makes an
encyclopedia entry about the engine instead of about the world.

It shipped rendering each field's declaration metadata, and the
failure mode is worth keeping written down: **every field on
`Material` declares exactly `{ persistent: true }`**, so a material's
panel was a column reading `persistent` twenty-six times — an
all-identical column carrying no information — while discarding the
`marshaller`, which is the **unit**, and the one thing a reader wants
beside a number. Now: `density · 750 kg/m³`, `autoignitionTemperature
· 570 K`, `edibility · no`.

Three rules behind that:

- **Units come from the marshaller singleton** (`getUnit()`), not from
  decoding its path — the `kg/m³` → `kg-per-m3` encoding has an owner
  and a second copy would rot. Best-effort: no panel fails for want of
  a unit.
- **A declared field the template never set is omitted.** Oak has no
  melting point because wood chars rather than melts; a blank row
  states that no better than its absence, and twenty of them bury the
  fields that do have values.
- **Iteration is over the DECLARED fields**, not the `data:` block's
  own keys, so the enumerating snapshot audit still covers everything
  a panel can surface — a stray key cannot slip a value onto a page
  without a declared level.

> ⚠ This is also what makes `spoiler` mean something. A field marked
> level 3 so a creature's weakness stays hidden was, until now, hiding
> the word "persistent". The gate was real and guarding nothing.

**`Material`'s measured properties carry `spoiler: 1`** — density,
hardness, the conductivities, autoignition, the heats, edibility,
nutrients, toxicity. What a material *is* (name, appearance, tags, and
the chemical identity anyone could look up) stays open; what it
*measures* is the reward for working with it, and a panel that hands
over every constant makes the material science a lookup instead of a
subject. At level 1 it is a **default about presentation, not a lock**:
one click, or `settings set wiki.spoilerAppetite 1` once and never
again.

#### ⭐ What else is level 1, and the line that decides it

> **Collapse what the WORLD measures. Never collapse what the PLAYER
> operates.**

| class | level 1 | left open |
|---|---|---|
| `Material` | density, hardness, conductivities, autoignition, the heats, edibility, nutrients, toxicity | name, appearance, tags, and the chemistry anyone could look up |
| `Species` | `vitalProfile`, `facultyProfile`, `innateMixins`, `naturalAttacks`, `affordedGambits` | the natural history a field guide prints — diet, lifespan, circadian band, vision, scent, reproduction, sentience |
| `Condition` | `progression`, `resolution`, `contagion`, `toxinBehavior`, `signature`, `mentalBands` | `observableSigns` — a sign nobody can read is not a sign — and the name |
| `Recipe` | inputs, tools, heat, difficulty, grade band, every output field | `name`, `keywords`, `discipline` — how the recipe is found at all |
| `Biome` | temperature, pressure, humidity, wind, gravity, ambient sound level | the ambient sound and smell MML: that IS the description of arriving |

Two classes were looked at and **deliberately left open**, recorded at
their own declaration sites so the next sweep does not silently reverse
them:

- **`Spell`** — magic here is a *published science*, one postulate and
  a price list ([arcane-science.md](../arcane-science.md)). A caster
  reasoning about the cost before paying it is the premise; collapsing
  the cost contradicts the fiction rather than protecting anything. A
  particular spell that IS a plot spoiler belongs behind a `<spoiler>`
  in authored prose, because a field level applies to every spell at
  once.
- **`LocomotionMode`** (and `CombatFormation`) — `speed`, `noiseLevel`,
  `costMultiplier` are numbers, but a locomotion mode is the player's
  own control surface. Hiding how your own legs work teaches nothing:
  a player learns sneaking is slower by sneaking, once.

> ⚠ Tagging only bites where a **template authors the field**, since
> the panel reads a template's `data`. Runtime state (`VitalsMixin`'s
> heart rate, `GrowingMixin`'s vigor) is never in a template's data, so
> a level there would change nothing on the wiki while still applying
> to the Studio and to anything else that renders the value later.
> That is why the sweep tagged authored data Ideas and left instance
> state alone.

#### Name and value can carry different levels

`spoilerName` sits beside `spoiler` on a field declaration and says
when a reader learns the field **exists**; it defaults to `spoiler`,
so name and value hide together unless a declaration splits them.

The distinction is **schema versus measurement**. "This material has a
density" is not a secret — it is exactly what `help` and the generated
API docs publish. `750 kg/m³` is the part worth working for. Split, a
reader gets the property list with the numbers collapsed; unsplit,
they get a table of blanks that says nothing about what is there to
find.

> ⚠ Splitting is **opting into a redaction marker**, which the reveal
> model refuses everywhere else — the empty value cell announces that
> something is there. That is coherent only because the name was
> already public. **Never split a field whose existence IS the
> reveal**: on a creature whose `fireVulnerability` is a spoiler,
> knowing it has one is most of the information, and the marker would
> hand over exactly what the level was protecting. That case is the
> default for a reason.

> ⚠ A name level above its value's is **clamped**, not trusted — it
> would render as a value in a row with no label. The enumerating
> audit records both levels (`Material.density = 1 (name 0)`), so a
> split shows up as a diff a reviewer has to look at.

`kind="mixin"` is untouched: *what in this world composes
`Combustible`* — i.e. **what can burn** — is a question about the
world that no template page can answer, and no help topic either.

### Snippets

A page in the `snippet` namespace, invoked `{{Name|pos|key=value}}`,
with `{{{key|default}}}` in the body. A snippet **is** a page, so it
inherits revisions, history, rollback and protection with no new code.

A missing parameter with no default renders `{{{name}}}` — **never
empty**. Silently-empty parameters are how wiki snippets rot: the page
looks fine and the data is gone.

Two cycle guards, because they catch different things: a **name stack**
catches self-inclusion *and* the mutually-including pair (which
alternates and never gets deep), while the budget's `snippetDepth` and
`maxSnippets` bound everything else. The error prints the whole chain.

---

## Permissions

`AccessApi.can` is zone-anchored and page data is deliberately not a
Stuff, so the **namespace tree** supplies the anchor: a
`WikiNamespaceZone` at `/wiki/<namespace>` under a `/wiki` root.

> ⚠ The requirements' `accessGroups` propagation describes machinery
> **removed in property phase 0a**. `ownerGroup`/`accessGroups` are gone
> from `Zone`; title lives in the gated `parcels` collection. There is
> also no "all signed-in players" group, and a parcel has exactly one
> owner — so the open edit floor **cannot be a group**.

The replacement is a `protection` field on the namespace zone, resolved
by the shipped `Zone.lookupField` inheritance walk:

| protection | check |
|---|---|
| `anyone` | any signed-in Avatar |
| `editors` | `AccessApi.can(actor, 'edit', nsZone)` |
| `moderators` | `AccessApi.canMutateZone(actor, nsZone)` |

`anyone` at `/wiki`; a per-page value takes the **stricter** of the two,
so a page tightens and never loosens. Narrowing a namespace is **one
seed field**, no code.

Ownership is a `parcels` row over `/wiki`, held by **one** managed
group, `wiki-editors`; moderators are its `'owner'`-role members. A
second group would need `GroupApi.isMember` in a controller (which the
antipatterns table forbids) and would invent a distinction the substrate
already draws.

`snippet` is the one namespace that tightens to `editors`, and the
reason is **blast radius**: an article is read by whoever opens it, but
a snippet is *transcluded*, so vandalising one vandalises every page
using it.

### ⚠ The guest floor — beneath every rung

**A guest may read the wiki and may not write to it.** Checked before
the ladder, the *wizard* rung included, because the objection is to the
**identity**, not the permission — no amount of authority makes an
unattributable edit attributable.

Open editing works here because **undoing is cheaper than reviewing**:
no queue, `wiki rollback` in about four seconds, and an edit that
belongs to somebody who is still there afterwards. A guest is an
anonymous throwaway (`/platform/agent/Avatar/guest-<uuid>`, persists nothing)
whose identity evaporates at disconnect — nobody to talk to, nobody to
refuse next time, and a revision log of names that mean nothing.
Admitting guests would not loosen the wiki; it would remove the thing
that makes its openness safe.

**Reading is untouched, deliberately** — a wiki nobody can read before
signing up cannot recruit its own authors.

Two properties worth keeping:

- The refusal **names the way out** ("make a character and it is yours
  to write"), not a rung. A would-be author who is told only "you may
  not" leaves.
- It arrives **before the composer opens**. `refusalToEdit` /
  `refusalToCreate` answer the permission question and its reason in
  one computation, so the verb can ask up front and the mutators can
  still throw — the printed reason and the thrown one cannot drift.
  Refusing somebody who has just written an article is technically
  identical and humanly very different.

`mayEdit` rides the `publication.wiki` frame, so the card simply does
not draw an Edit button for a guest.

**A review queue is deliberately absent.** Open editing plus fast
rollback is the wiki-classic bargain; a queue converts a commons into a
submission process. Revisit only if abuse actually appears.

---

## Deletion

`delete` sets `deletedAt`/`deletedBy`. The page stops resolving for
readers, stays visible to moderators, and undelete clears a field.
Revisions are untouched. A wiki whose delete is destructive has a
revision log that lies.

**`purge` is the one hard exception** — page *and* revisions, moderator
-only, irreversible. It exists because pretending otherwise does not
protect the log; it means the only remedy for illegal material is
somebody with database access doing it by hand, unlogged. It writes a
**tombstone revision** naming who purged what, with no body, which is
what keeps it accountable.

---

## Markup additions

`api/mml/` gained an **article dialect**, selected by
`MarkdownOptions.longForm`. An options bag, not a fork: `parseMarkdown`
runs on every utterance, and a forked long-form parser would drift from
the chat one silently.

Added: `<h1>`–`<h3>` with **sticky `{#anchor}` suffixes**, indent-nested
lists, pipe tables, `<spoiler level="n">`. Every tag defines `flatten`.

**Tag passthrough.** The article dialect emits a recognised tag
verbatim instead of escaping it, so markdown and MML mix in one body —
which is what lets an author write `<composition of="…"/>` or wrap a
paragraph in `<spoiler level="2">` while writing markdown everywhere
else. A tag's *children* keep getting markdown treatment;
`<code>`/`<pre>` are consumed whole, so a page documenting the syntax
renders the syntax.

> ⚠ Two conditions, both load-bearing. **Shape**: the slice must be a
> well-formed tag, or `a < b and c > d` reads as a `<b>` tag and three
> words of prose vanish. **Name**: it must pass the caller's
> `TagPolicy`. A legal-but-unknown name still passes on the wiki;
> stage 4 looks it up, fails, and emits an inline "unknown component"
> error. Shape is the dialect's job, existence is the renderer's, and
> neither pretends to be the other.

#### The tag policy — what each surface admits

The passthrough is what makes `<spoiler>` writable, and it is safe
here because the wiki renders through a **gated pipeline**: components
resolve against a charset-restricted namespace (a component name
becomes a module basename), spoilers pass the ceiling gate, output is
budgeted. A surface without that pipeline has to admit only what is
safe on its own, so `MarkdownOptions.tags` names how much literal
markup an author may write:

| policy | admits | surface |
|---|---|---|
| `none` | nothing — everything escapes | `say`, emotes, broadcast |
| `spoiler` | `<spoiler>` only | channels, `dm`/`tell` |
| `inert` | presentation: emphasis, lists, headings, tables, spoiler | forum posts |
| `all` | known markup **plus component candidates** | the wiki |

Defaults preserve what every caller had before the option existed:
`all` under `longForm`, `none` otherwise.

> ⚠ The excluded tags are excluded for a reason worth keeping written
> down. `<link>` and `<mention>` are **affordances that act** — the
> client renders a clickable that ISSUES A COMMAND. `<name>`,
> `<player>`, `<speech>`, `<msg>`, `<chan>` are **identity claims** the
> composer emits on the server's authority, so a player who could write
> `<speech>` could attribute words to somebody else. `<color>` is mere
> presentation and still excluded: staff and system styling are
> recognisable by it.

Two conversational surfaces take `spoiler` because **chat is where a
spoiler gets blurted** — a channel hears "the boss is a mimic" long
before anybody writes the article — and only the *appetite* half of the
reveal model applies there: a chat line carries no authored capability
level, so there is nothing to gate, only something to fold. `say` stays
at `none` on purpose: it is acoustic speech, and you cannot speak a
table.

Forum posts moved from the **chat** dialect to `longForm` + `inert`.
They are long-form authored prose composed through the same prompt a
wiki page uses, and until this pass a forum post could not carry a
heading.

> ⚠ **`api/__tests__/mml.corpus.test.ts` pins the chat path byte-for
> -byte**, including its defects (a `[label](mudcmd:…)` link after a
> space stays literal — the sentinel passthrough copies whole
> space-delimited words). A diff there means chat output changed, which
> is a failure regardless of whether the new output reads better. The
> article dialect narrows the passthrough; the chat dialect does not.

`<spoiler>` **flattens to its content**, not to a `[spoiler]` marker:
flatten is a failsafe projection of an already-gated body, so what
survives is content the reader may see. Hiding it would blank text on
exactly the surfaces with no client to un-hide it.

### The editor opens on what is there

`wiki edit` sends the **current** text into the composer —
`ComposePromptOpts.initial`, threaded through the `prompt-compose`
note to a client that seeds the draft with it. A section edit sends
that section; a full edit sends the article; `create` sends nothing,
because there is nothing to open on.

> ⚠ Without it, "edit" silently means "retype": the box opens empty
> and whatever is posted **replaces the whole body**. This shipped that
> way and only the live drive found it — every test submitted `--body`,
> which skips the prompt path entirely. The three-line test that now
> covers it drives `PromptApi.compose` and asserts what it was handed.

### Sticky anchors

An anchor is **minted once and held**, not derived from heading text: a
derived anchor changes when the words do, silently breaking every
citation. `Sections.reconcile(prior, next)` carries anchors forward
positionally and mints from text for new headings — **in the syntax the
heading was written in**, `## Uses {#uses}` for markdown and
`<h2 anchor="uses">` for MML, so minting never rewrites the author's
source into a form they did not choose. Fenced code blocks are skipped,
or a `# comment` in a shell sample acquires an anchor.

> ⚠ When the heading **count** changes, anchors are re-minted rather
> than matched positionally — a mis-aimed citation is worse than a
> broken one, because it is silent.
>
> ⚠ Rolling back restores that revision's anchors, which may be fewer. A
> citation to a newer section then dangles. Correct, but it reads as a
> bug.

---

## The client — one card, everything a command

`wiki <page>` sends the article's prose to the scroll on
`shell.result.wiki` **and** a structured twin on `publication.wiki`.
`WikiCard` reads the twin: title, handle/rev/author, the outline, the
body, the subject binding, tags and see-alsos.

> ⭐ **The frame carries the body that was already RENDERED.** That is
> the security argument for having a card at all: the payload inherits
> the pipeline's gate instead of re-deriving it, so there is no second
> path to the reader. A card fed from the source would be exactly that
> path — and the leak would be invisible from the scroll, which would
> still look correct. The card holds no article source and cannot
> render one.

Every affordance is a **command the card composes and emits** — `wiki
edit`, `wiki history`, `wiki links`, and per-section `wiki edit …
--section <anchor>` — on the shared bus, previewing on hover like every
other clickable. Bus-primacy: there is no private wiki channel from the
client, so anything the card can do can also be typed.

An outline row opens **that section for editing** rather than scrolling
to it: section editing is the wiki's real concurrency control, and a
reader who only wants to look has the whole body directly below. For a
reader who may not edit, the same row re-reads the page.

`mayEdit` rides the frame so the card does not draw an affordance that
is certain to be refused. It is a display hint — **the verb re-checks
on arrival**, which is where the decision lives.

A `wiki preview` frame is flagged, and its outline comes from the
draft: a preview indistinguishable from the saved page is a page
claiming to be something it is not.

**No live preview of an in-flight draft.** Markdown is parsed
server-side, so that is a round trip per keystroke; it belongs to the
shared viewer substrate in the client-shell slate. The client here is
scoped to "read a page, follow a link, submit an edit" — rudimentary by
decision, not omission.

---

## Maintenance

Four reports, all derived on read from the link graph rather than
maintained as an index — a **stale** backlink index is worse than a slow
one, because it reports links that are not there.

- `wiki links <page>` — backlinks. Follows aliases.
- `wiki wanted` — redlinks ranked by demand. The best authoring to-do
  list a wiki has, and it is written by its readers.
- `wiki orphans` — pages nothing links to.
- `wiki dangling` — articles whose subject template is gone. The CMS
  renames and deletes templates and the wiki points at paths, so this
  *will* happen; without the report a stale article quietly documents
  nothing.

`[[refs]]` inside code are excluded — MML `<code>`/`<pre>`, markdown
fences and backtick spans alike — or an author writing *about* the
syntax creates phantom demand for a page called `Page`. The guide page
documents `` `[[Oak]]` ``, so this is not hypothetical: it put a
`guide:oak` nobody could ever satisfy at the top of `wiki wanted`.

---

## The installer as a writer — the `wiki` content kind

A pack ships pages as `content/wiki/<namespace>/<slug>.md` — YAML
frontmatter (`title`, `subject`, `tags`, `related`, `spoilerLevel`)
over a **markdown** body, the article dialect. The installer never
writes rows: it **submits** through `createPage` / `editPage` **as the
pack** (`asInstaller: <packId>` — the gate on exactly those two methods
admits `PackLogic` beside `WikiController`; the revision author is
`pack:<id>`, never `system`; the namespace protection walk is skipped
because there is no acting player at boot; `WikiController` never sets
the option, and a test asserts it). An edit carries the install
record's baseline `rev` as the **compare-and-swap** token: a changed
pack file over a page nobody touched is a new revision; over a page
somebody edited it is a `WikiConflict` → a **`wiki-cas` pack conflict**
with all three bodies in `pack diff`, settled by `pack resolve
--take-pack` (an edit over the current rev — the history keeps both)
or `--keep --pin`. A page that pre-dates its record (the retired
`WikiSeeder`'s) adopts with its live `rev` and no edit; a renamed page
resolves by alias, so it is never re-created; a vanished file **keeps**
the page. See [content-packs.md](./content-packs.md).

## Collections

| collection | holds | sandbox policy |
|---|---|---|
| `wiki` | current page state | `pass` (unmarked) |
| `wiki_revisions` | the append-only edit log | `pass` (unmarked) |

**PASS, beside `domain`** — the wiki is authored truth and a
**communications surface**. An article cannot affect advancement, cannot
mint anything, and cannot be spent; it is people writing to each other,
so there is nothing for the sandbox to contain. It is also strictly less
powerful than `domain`, which passes: a circle that may edit a room
template should not be refused an encyclopedia edit about one.

⚠ **Not STAMP**, which would be actively harmful: a scoped page
reverting on circle exit is a page an author watched themselves write
and then lose, and its scoped revision rows would collide with the
unique `{pageId, rev}` index. **Not the epistemic mark**, which is for
"what happened to *you*" — an article is not a personal record.

Authorization is unaffected: the protection ladder resolves through
`AccessApi`, which is circle-independent, so a circle confers no editing
right its occupant did not already hold.

Indexes: `{namespace, slug}` and `{namespace, aliases}` (the one name
space), `subject.ref` (the total reverse lookup), `tags`, and a **unique**
`{pageId, rev}` — two rows at one rev would mean a lost edit that
history records as having happened.

> The name index is deliberately **not** unique: `aliases` is an array,
> so a unique index would be multikey-unique across the collection and
> would reject two namespaces legitimately holding the same name.
> Uniqueness is enforced at the `WikiRegistry` write chokepoint, which
> is also where the refusal can name the holder.

---

### The Wave 6 pass — what was hatched, and what was cut

The card's guarantees are unchanged: the body still arrives **already
rendered and already gated**, and every affordance is still a command the
card composes. Wave 6 touched only what the design handoff added.

**Search is hatched.** Track C records the wiki search port as *not
wired*, and the convention's first preference is *ship the surface, hatch
the value*. ⚠ Deliberately **not a disabled input** — that still reads as
a search box that is temporarily off, and it is the shape this would
quietly drift back into, so `wikiSearch.test.tsx` asserts no input
element exists at all. The field names the way in that *does* exist: the
tree.

**Four blocks are CUT, not hatched.** The handoff's page furniture —
the `OFFICIAL` standing badge, *what it affords*, *seen in play*, and
*composed by* — has no server half at all:

| Block | Why cut |
|---|---|
| `OFFICIAL` page standing | No canon/standing concept exists anywhere in this subsystem or in `WikiPageFrame`. The mock's own rationale ("pages the Make chamber has adopted") makes it an unbuilt **governance** feature, not an unwired read. |
| *What it affords* | Class-level affordance introspection — a real derivation, and closer to Studio's `describeClass` than to anything here. |
| *Seen in play* | Per-viewer encounter memory. Nothing records it. |
| *Composed by* | A reverse composition index. Nothing builds it. |

⭐ The reasoning is the unbuilt-state convention's **second** preference:
*cut the widget if it says nothing without data*. Hatching all four would
make one page carry four hatches, which reads as a broken page rather
than an honest one. Page standing in particular belongs to the wiki tail
as a governance question — *who adopts a page, and does adoption mean
anything the reveal model has to respect?*

## What this build does NOT do

Three shared substrates the wiki depends on and does not own:

- **Media ingest** — `<image key>` references an existing asset; there
  is no upload path. Three producers want one bucket (wiki, CMS,
  `illustrate.ts`), the route and S3 credentials belong in the backend
  tier, and building ingest here would make the wiki the second of three
  and the one recording provenance differently. Criteria 41/42 belong
  with it.
- **The search port** — the index must be over **source** with levels
  intact, because indexing renders is a spoiler leak by construction (a
  render is per-reader). Criteria 43/44/44a.
- **Durable notification** — nothing in the engine answers *"what
  happened to the things I care about while I was away."* The wiki emits
  nothing yet and **grows no inbox** (criterion 70, asserted); watching
  is the substrate's job. Criteria 69 and 71 belong with it — the
  publish event itself, and its carrying the level of what changed so
  delivery can gate on it. See
  notification-slate.md.

The one half-criterion: **39** is satisfied for `<image key>`
resolution and not for a subject-bound page surfacing its subject's
`illustration` with no authoring — that half rides media ingest with
41/42.

Also absent by decision: `wiki search` (a subcommand answering "not
available" would put a lie in `help wiki`), an `<mql>` component
(`resolveMany` needs a context carrying an actor, which C1 forbids
handing a component), and `recordAuthoring` wiring (`wiki_revisions`
*is* the wiki's authorship ledger, with the same context-derived author
rule).

---

## Cross-references

- [messaging.md](./messaging.md) — MML and the composer stack
- [message-rendering.md](./message-rendering.md) — the client renderer
- [mixins.md](./mixins.md) — the composition substrate the panel reads
- [studio.md](./studio.md) — `describeClass`/`describeMixin` (⚠ author
  -gated, which is why the panel does not use them)
- [access.md](./access.md), [parcel.md](./parcel.md) — the zone-anchored
  walk and the title registry
- [behavior.md](./behavior.md) — the path-resolved module pattern
  components reuse
- [document-store.md](./document-store.md),
  [persistence.md](./persistence.md) — Document vs Stuff
- [command-spec.md](./command-spec.md) — the verb shape
- [sandbox.md](./sandbox.md) — the collection policy table
