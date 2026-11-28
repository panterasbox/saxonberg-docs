# MQL — MUD Query Language

MQL is the query language the dispatcher runs against `type: object` /
`type: objects` YAML fields. It replaces the original keyword-scoring
matcher with a real grammar: one root (the asker, `me`), relations
(`~name`), chain operators, set ops, filter expressions, scope-as-MQL
evaluation, pronoun memory. ⭐ MQL finds **runtime Stuff** and nothing
else; everything a player names that is not Stuff (a spell, a recipe, a
ship in a register) is named by the same keyword rule through
`GrammarApi.nameAmong` (§ Match-quality scoring).

This doc is the **developer reference** — pipeline, modules, internals,
extension seams. For the player/author-facing grammar reference, see
[../mql-grammar.md](../mql-grammar.md). For how it plugs into command
dispatch (the `scope:` field, `$focus`, `updates_focus`), see
[command-routing.md](./command-routing.md) and
[command-spec.md](./command-spec.md).

## Position in the pipeline

```
  text input
    → CommandLineApi.parse()             ─ shell tokenizer
    → giver.getAffordances() ⋯ by verb   ─ verb lookup (keeps source)
    → CommandApi.assemble()              ─ binds tokens to YAML, expands shell vars
    → CommandApi.resolveModel()          ─ MQL on type:object[s] fields
        (a typed query's head word first passes ShellApi.expandQueryAliases)
        for each MQL field:
          for each scope in def.scope:
            MqlApi.resolveOne / resolveMany(query, { commandGiver, scope })
            stop on first non-empty result
    → await CommandApi.preloadValidatorDeps()  ─ validator preload phase
    → CommandApi.runValidators()         ─ sync validators
    → controller.execute(model, ctx)
```

`MqlApi` does not know what cardinality the consumer wants. The
dispatcher splits on YAML `type: object` (one) vs `type: objects`
(many) and wraps the result in `MqlOneResult` / `MqlManyResult`. The
controller decides what no-match means.

## File layout

```
packages/server/src/mud/api/
  mql.ts                     ← thin facade (resolveOne, resolveMany)
  mql/
    types.ts                 ← public types + AST nodes
    desugar.ts               ← natural-language pre-parse pass
    lexer.ts                 ← raw text → Token stream
    parser.ts                ← Token stream → AST
    resolver.ts              ← AST + context → MqlMatch[]
    relations.ts             ← the built-in relation table (~here, ~peers, …)
    scope-walk.ts            ← candidate pools, facets, attribution (the
                               score itself is GrammarApi's)
    predicates.ts            ← bareword filter registry (living, mine, …)
    pronoun-memory.ts        ← per-Focused-giver stash read by ~it/~them/
                               ~him/~her/~last
    via-augment.ts           ← TS declaration merging for MqlMatchVia
  grammar.ts                 ← GrammarApi: the keyword tiers, scoreAll,
                               rank, name (the naming rule's one home)
  __tests__/
    mql.test.ts              ← end-to-end resolver
    mql.relations.test.ts    ← relations, @id, home paths, the listing
    mql.lexer.test.ts
    mql.parser.test.ts
    mql.desugar.test.ts
    grammar.test.ts / grammar.name.test.ts
    path-trie.test.ts
    path-pattern.test.ts
```

(`online-provider.ts` / `online-wire.ts` are deleted with the `online`
seed — naming overhaul D-B10.)

`mql.ts` is the only security-decorated entry point. The internal
modules under `mql/` are pipeline stages, not Apis — they're plain
TypeScript modules. `MqlApi` is the boundary; the rest is internals.

## Public surface (`MqlApi`)

```ts
class MqlApi {
  static resolveOne(query: string, ctx: MqlContext): MqlOne;
  static resolveMany(query: string, ctx: MqlContext): MqlMany;
}

interface MqlContext {
  commandGiver: (Stuff & CommandGiver) | null;  // null = system mode
  scope: string;                       // an MQL fragment, post-expansion
  attention?: number;                  // code-only perception term
}

interface MqlOne   { stuff: Stuff | null; via?: MqlMatchVia; relations?: RelationInfo[]; relationTie?: RelationTie; }
interface MqlMany  { stuff: Stuff[];      via?: MqlMatchVia; vias?: (MqlMatchVia | undefined)[]; relationTie?; }
```

`relationTie` says a name two of one holder's sets share (a contact list
and a group both called `bookclub`) — the binder prompts, a script
refuses (§ Relations).

Both methods route through the same internal `resolve()` in
`resolver.ts`; the difference is wrapping. `resolveOne` picks the
top-scored match (or `null`); `resolveMany` returns the full sorted
list.

`scope` on `MqlContext` is a **string**, not a structured policy. The
dispatcher takes the YAML `scope:` array (`['$focus', '~reachable']`),
expands each entry through `ShellApi.expandVariables`, and tries each
in order until one returns a non-empty result. From `MqlApi`'s point
of view it's one fragment per call.

### `MqlResult` wrappers (dispatcher-side)

The dispatcher lands a richer wrapper on `model[fieldName]` that
extends `MqlOne` / `MqlMany` with the player's text and any consumed
preposition:

```ts
interface MqlOneResult extends MqlOne {
  raw: string;        // post-desugar player input (always present)
  prep?: string;      // consumed preposition for this field, lowercased
  quantity?: MqlQuantity;  // from natural-language `2 X` or formal `:{N}`
}

interface MqlManyResult extends MqlMany {
  raw: string;
  prep?: string;
  quantity?: MqlQuantity;
}

interface MqlQuantity {
  value: { kind: 'count'; n: number } | { kind: 'all' };
  mode: 'strict' | 'lenient';
}
```

Controllers destructure directly:

```ts
const { stuff, via, raw, prep, quantity } = model.target;
```

`raw` is the source string for "you don't see any '<raw>' here"-style
messaging. `prep` is what the matcher consumed off this field's
`prepositions:` list — `look at flower` → `prep === 'at'`.

`quantity` carries the leading-number or formal `:{N}` hint when the
query supplied one — `5 coins`, `coins:{5}`, `all coins`,
`coins:{*}`. The `mode` discriminator is transport-only: controllers
pass the whole `MqlQuantity` through to `StackableApi.applyQuantity`
without branching on it. See stacks.md for the helper's
distribution algorithm.

## Pipeline stages

### Stage 1 — Desugar (`mql/desugar.ts`)

Translates common English phrasing to formal MQL **before** the
lexer. Three transforms:

1. **Article stripping.** Leading `the` / `a` / `an` are dropped.
2. **Ordinal prefix.** A leading ordinal marker rewrites the rest as
   a chain index — `second rose` → `rose:[2]`, `2nd rose` →
   `rose:[2]`, `last rose` → `rose:[-1]`.
3. **Quantity prefix.** A leading positive integer or the literal
   `all` followed by at least one keyword token rides on a
   side-channel quantity hint — `5 coins` → `{ rewritten: 'coins',
   quantityHint: { value: { kind: 'count', n: 5 }, mode: 'lenient' } }`.
   `all coins` → `{ kind: 'all' }`. The natural-language path is
   always lenient; strict-mode hints come from the parser's
   `QuantityNode` (formal `:{N}`).

The pass is **bypassed** when the input contains any formal-MQL
signal character (`:`, `[`, `,`, `'` at a token start, `{`, `~`, `@`). The `looksFormal()` guard
keeps dev-typed queries from getting their intent silently rewritten.

The desugar **return shape** is now structured (not just a string):

```ts
interface DesugarResult {
  rewritten: string;
  quantityHint?: MqlQuantityHint;
  error?: string;
}
```

When `error` is set (quantity + ordinal collision detected),
`rewritten` is the original input and `quantityHint` is undefined;
the resolver wraps the message in an `MqlDesugarError` that the
dispatcher surfaces as the command's "couldn't resolve" failure
prose.

The lexicon (articles, ordinal words, numeric ordinal pattern) lives
on `GrammarApi` (`api/grammar.ts`) — the natural home for English
artifacts. The desugar pass is a pure function of its input string.

The **conflict-fallback rule** (retry as a literal-keyword form when
ordinal interpretation hits zero matches) lives in the resolver, not
here, because match-count is observable only post-resolve.

#### Quantity + ordinal collision

Combining a quantity prefix with an ordinal in the same query
(`2nd 3 roses`, `3 2nd roses`, `all 2nd roses`) is ambiguous —
there's no unambiguous unbracketed reading. Desugar refuses with a
canonical error message that names three rephrasings (range, single
ordinal, plain count). The resolver throws `MqlDesugarError` with
that message; the dispatcher renders it as command failure prose.

### Stage 2 — Lex (`mql/lexer.ts`)

Tokenizes raw query text into a stream of typed `Token`s.

Reserved character set:

```
,  :  (  )  [  ]  {  }  .  -  #  $  *  /  '  ~  @
=  !=  >  <  >=  <=
```

⭐ **`~` and `@`** (naming overhaul D-B1): `~name` lexes as `tilde` then
the bareword; a bare `~` is the tilde alone; `~/x` is one `homePath`
token; `@abc123` is one `playerId` token (a bare `@` is a lex error).
`$$` still lexes — only so the parser can refuse it with *"'$$' is gone —
your last result is '~last'"*. MQL gives `::` no meaning: the command
line splits `source::verb` before MQL sees the source
([command-parsing.md](./command-parsing.md)).

`{`, `}`, and `*` (at top level — `*` inside a path is consumed by
the path scan) gained dedicated token kinds when the formal-quantity
syntax landed: `lbrace`, `rbrace`, `star`.

Tricky cases the lexer handles:

- **Hyphen.** `-` is set difference between expressions, but a literal
  inside a bareword. The lexer absorbs `-` into a bareword only when
  both the previous and next chars are bareword chars; standalone `-`
  is the operator (`oak-door` is one bareword; `swords - broken` is
  three tokens).
- **Hash.** Shape-dispatched: `#5` lexes as `hashInt` (chain-position
  ordinal); `#abc123` lexes as `hashId` (stuff id seed). A bare `#`
  not glued to digits or an identifier is a lex error.
- **Single-quoted literals.** `'…'` preserves whitespace; it opens
  only at the START of a token (`dave's` is one bareword). Inside it
  `\'` is a quote and `\\` a backslash, other `\X` verbatim.
  Unterminated is a lex error.
- **Backslash.** Outside a literal, `\X` makes `X` part of the word
  whatever it is; an escaped word lexes as a bareword, never an int or
  transform letter. A dangling trailing `\` is a lex error. `looksFormal`
  ignores escaped characters and a mid-word `'`.
- **Paths.** Start with `/`; absorb identifier-shaped characters plus
  `*`, `?`, `-`. Stops at whitespace, `.`, or any reserved character
  that isn't legal inside a path segment.
- **Numbers vs identifiers.** Greedy word scan; classified `int`
  (all-digit) or `bareword` (everything else, lowercased). `5th`
  lexes as one bareword, `5` as int.

### Stage 3 — Parse (`mql/parser.ts`)

Recursive descent over the token stream into the AST defined in
`mql/types.ts`. Grammar (informal):

```
query        = sublist ("," sublist)*
sublist      = chain ("-" chain)*
chain        = seed (":" chainElement)*
seed         = "me" | keywords | literal | path | stuffId
             | "~" name            ; ≡ me:~name
             | "~" | homePath      ; your home: /home/<you>[/…]
             | "@" id              ; a player's loaded avatar
             | "(" query ")"
chainElement = transform | keywords | predicate | ordinal
             | bracket | group | quantity | seed-shaped
             | "~" name            ; the set the left side keeps under name
             | "~"                 ; list what the left side answers
transform    = "i" | "I" | "e" | "E"
ordinal      = "#" int
bracket      = "[" bracketBody "]"
bracketBody  = ( "-"? int ) ranges? | filterExpr
quantity     = "{" ( int | "*" ) "}"

filterExpr   = orExpr
orExpr       = andExpr ("or" andExpr)*
andExpr      = notExpr ("and" notExpr)*
notExpr      = "not" notExpr | atomExpr
atomExpr     = "(" filterExpr ")"
             | "has" atom
             | atom (cmp value)?
atom         = ident "." ident   ; namespaced
             | "name" | "id"     ; bare
value        = int | quoted | ident
```

Positional rules the parser enforces (not reflected in the grammar):

- `#5` (ordinal) is rejected at chain head — there's no current set
  to index. Allowed mid-chain as a synonym for `[5]`.
- `[…]` is rejected at chain head — brackets index/filter an existing
  set.
- `{…}` (quantity) is rejected at chain head — quantities apply to
  an existing set. Body grammar is `int` or `*` only in v1; `{1..3}`,
  `{half}`, `{N unit}` (bulk) have real estate reserved but no
  implementation yet. Parsed into `QuantityNode { value:
  { kind: 'count' | 'all' } }`.
- Multi-bareword runs (`red rose`) collapse into one `KeywordsNode`.
- `@id`, a bare `~` and `~/x` are **head-only**: after a `:` they are a
  parse error (`x:~/y`). A head `~name` parses as `me:~name`.
- The `.X` detail-drill operator is gone; `.` is reserved for
  namespaced atoms inside bracket bodies (`prop.X`, `mixin.X`). The
  parser detects the old `.X` form at chain position and gives a
  diagnostic pointing at `:X`.

### Stage 4 — Resolve (`mql/resolver.ts`)

Walks the AST against an `MqlContext` and produces `MqlMatch[]`,
sorted by score (ties broken by insertion order — see §Stable
ordering below).

```ts
interface MqlMatch {
  stuff: Stuff;
  score: number;
  via?: MqlMatchVia;
}
```

The resolver is the largest module. Key entry points:

- `resolve(query, ctx)` — full pipeline: desugar → lex → parse →
  resolveQuery → finalize.
- `resolveWithQuantity(query, ctx)` — same, plus surfaces any
  quantity hint (from desugar's `quantityHint` or a parser-produced
  `QuantityNode`). `MqlApi.resolveOne` / `resolveMany` call this
  variant to populate `MqlOne.quantity` / `MqlMany.quantity`.
- `resolveQuery(ast, ctx)` — same, skipping desugar/lex/parse.
- `resolveSublist`, `resolveChain`, `resolveSeed`, `applyChainOp`,
  `applyTransform`, `filterByKeywords`, `intersectBySeed`, etc.

#### Quantity passthrough

When a chain contains a `QuantityNode` (formal `:{N}` / `:{*}`),
`applyChainOp` is a **no-op for candidate selection** — the resolver
does not slice the match list. The quantity hint is harvested
out-of-band by `extractQuantityFromAst` and attached to the wrapper
returned by `MqlApi`. Distribution is a controller/helper concern;
the resolver hands back the unmodified match list.

Strict-mode hints (from a parser `QuantityNode`) win over lenient
hints (from desugar). In practice both never co-occur — `looksFormal`
rejects the desugar pass entirely when the input contains `{`, so
the formal path always sees the hint slot empty.

No operator or seed is permission-gated (content-packs wave 3):
**resolving an MQL query is never a permission.** What an actor may
*do* with a resolved target is the verb's own gate — reachability
(`attack @<id>` is refused because the target is not reachable, not
because a player address was typed) and title. See [Resolving is not
permission](#resolving-is-not-permission).

#### Seeds — the root and the anchors (naming overhaul Stage B)

⭐⭐ **Every query is the asker's.** There is one root, `me`, and every
other way to reach something at a distance is a **relation** the asker or
a thing keeps. The fourteen reserved seed words (`here`, `peers`,
`reachable`, `inventory`, `person`, `horizon`, `online`, `it`/`him`/`her`/
`them`, `$$`, …) are gone from the grammar; a bare `here` is an ordinary
keyword unless a query alias rewrites it (§ Query aliases).

| Seed | AST node | Yields |
|---|---|---|
| `bareword` (or multi-word `red rose`) | `KeywordsNode` | scope-walk + scoring |
| `'literal'` | `LiteralNode` | exact keyword match (whitespace preserved) |
| `me` | `PronounNode` | the giver — the root |
| `~name` | `RelationNode` under `me` | ≡ `me:~name` — one of YOUR relations (§ Relations) |
| `~` · `~/x` | `HomePathNode` | the path seed `/home/<asker>[/x]` (head only) |
| `@<id>` | `PlayerNode` | the loaded avatar whose identity path is `/platform/agent/Avatar/<id>` (D17's identity, never lineage); nothing for an unknown or unloaded id (head only) |
| `world` | name-promoted `KeywordsNode` | ⚠ **refused** unless the run carries a registry-read grant — see *The registry-read grant* |
| `/path/with/*globs` | `PathNode` | template-path glob seed (PathTrie) |
| `#abc123` | `StuffIdNode` | direct stuff id lookup |
| `(query)` | `GroupNode` | grouping |

Mid-chain, the **fixed pools** — `me`, `world`, paths, stuff ids and
groups — **intersect** with the prior set (score and `via` preserved;
`intersectWithSeed`). A relation step **flat-maps** instead (below). The
old element-derivable seed words (`x:peers`, `x:reachable`,
`x:inventory`) and their per-seed exclusion rules are gone with the words.

#### ⭐⭐ Relations — `~name` (naming overhaul D-B1–D-B6; the listing is `which ~`)

`holder:~name` asks the Stuff on its left for the set it keeps under
`name`; a head `~name` asks you. Several things on the left: each is
asked, the sets are unioned and deduped, and every match carries
**`via.relation = { holder, name, meaning }`** (so `contacts remove`
knows which list without re-reading the typed words). Synchronous: a
holder answers from what is resident and never loads — an unloaded member
is simply absent.

**Built-ins come first and can never be masked** — one table,
`api/mql/relations.ts`, each entry `{ name, meaning, whose, members }`,
one answer each (no relation has two faces):

| relation | `whose` | the set |
|---|---|---|
| `~here` | anyone | the **place** the left side stands in — `getEnclosingPlace()`, past a boat, a cart or a bed; ONE thing (`me:e` is what immediately holds you) |
| `~inventory` | anyone | what is carried — the asker's whole tree, nearest first (hand · ready · packed); asked of another, read from OUTSIDE (what they hold and wear, one level into open things) |
| `~buildings` | anyone | (of a place) the buildings whose door is at hand from it — the landing row |
| `~peers` | self | what stands where you are, one level into open things there, and the vessel you sit in |
| `~reachable` | self | carried, standing here, and the place's surface — `PerceptionApi.reach`, on-person first |
| `~person` | self | your body, your apps, installed gear (+ apps), carried items (+ apps) |
| `~horizon` | self | what you can see from here that is not here |
| `~contacts` | self | every loaded person you keep — on any of your contact lists or in any managed group you belong to (not the party, which is `~party`). Was `~all` until naming-overhaul review: the name promised every built-in at once |
| `~it` · `~them` · `~him` · `~her` · `~last` | self | the pronoun-memory slots (§ Pronoun memory) |

`self` answers only when the left side is the asker — asked of anyone
else it is empty, visibly (`@bob:~peers` is nothing).

**Holders** compose **`RelatingMixin`** (`lib/relation/Relating.ts`):
`relationsFor(asker)`, `relatedBy(name, asker)` (empty = no such set;
more than one = a name two of its sets share) and `groupRefOf(name,
asker)` (the durable `GroupRef` a set stands for — what `notify` stores).
It composes ONCE beneath every contributor, which chain with `super`.
Shipped holders: contact lists (one relation per label, owner-only —
`~friends`), the party (`~party`), your managed groups (one per group,
resident on you through `ContactsMixin`, beside your lists — `~bookclub`), a warren
(`~rooms`, its live members), a charter book (`~ships`), a travel card
(`~nodes`, the tpa pack), an aether host (`~apps`). A list named like a
built-in is warned about when made, and never reached by a query.

**The listing left MQL (the which build).** `thing:~` once listed the
relations a holder answers, behind a `lists: relations` view key only
`query` declared — a listing is not runtime Stuff, and MQL finds runtime
Stuff only. `which ~` / `which ~ <thing>` lists them now
(`CommandApi.whichRelations`, reading `lib/relation/BuiltinRelations.ts` —
the built-in words' name · meaning · whose, which `mql/relations.ts` keys
its members by — and `RelatingMixin.relationsFor`). A bare `:~` is an
`MqlParseError` whose message names `which ~`.

**A tie** — `relatedBy` handing back two sets of one name — rides
`relationTie`; the binder asks *which?* (the sets' meanings) and keeps the
chosen set's matches. A script never prompts: the tie is the refusal.

##### Why relations look like this

- **One root, because of immersion** (lens 3a): a thing you have no
  relation to is not a thing you can name. `/path`, `#id` and `@id` are
  the second mode — names you must know exactly, answered by an index,
  like an absolute path beside a relative one.
- **A sigil, never a reserved word.** A list called `friends` can then
  never collide with a thing keyworded `friends`, and `~here` never with a
  thing keyworded `here`. Rejected first: one `:members` word with
  per-list step words (`me:friends`) — every list name anyone made would
  have become a reserved word.
- **Built-ins win so nothing can be hidden**: a player cannot keep their
  inventory out of other people's queries with a list named `inventory`.
  Making such a list is a warning, not a refusal — a soft limit.
- **Each holder answers per asker.** Contact lists are owner-only; two
  holders may mean very different things by one `~foo`, and that is the
  design.
- **A verb that names a person reaches them through its view's default
  scope** (`dm` scopes `["~here", "~contacts"]`) — never a binder special
  case.
- **`~` is also *home* in the shell's path arguments** (`cd ~`). That is a
  path and never MQL; inside a query a home path is valid at the head only.

#### Query aliases (D-B7)

A **query alias** rewrites the HEAD word of a typed query when the whole
query is that word or it is followed by `:` (`ShellApi.expandQueryAliases`,
called by the binder for every typed `object`/`objects` slot and for a
`source::verb` source — never for a view's authored `scope:` or
`default:`). Entries are `?name` keys in the alias table (`alias set
--query here ~here`). The defaults, each removable with `alias unset
--query`, ship as `static defaultAliases` on the mixin that owns the
thing: `here` → `~here` (`MobileMixin`), and `it`/`them`/`him`/`her` →
`~it`/`~them`/`~him`/`~her` (`FocusedMixin`). Those five are the whole
default set; `inventory`, `peers`, `reachable` typed bare are keywords.
See [shell-alias.md](./shell-alias.md).

⭐ **`~peers` descends ONE level into an open container** (see § Reach is
one rule) standing in the
room — the glass rack's coupes, the floor stock's kegs, a crate's limes —
so a bare noun can name what is plainly in front of you without a
`from <holder>` clause. The openness test is `MixinApi.isOpenContainer`,
the same rule `PerceptionApi.canReach` asks, so the scope can never offer
a candidate that reach then refuses (nor hide one it would allow). A shut
`Sealable` and anything that is somebody stay opaque. See
[perception.md](./perception.md) § *`canReach`*.

The dispatch lives in `mql/resolver.ts:applyColon`: `:~name` routes
through `flatMapRelation` (built-in first, else `RelatingMixin.relatedBy`),
the place steps through `flatMapPlaceStep`,
and the fixed pools (`pronoun`, `path`, `stuffId`, `group`, `world`)
through `intersectWithSeed`.

#### Chain operators

| Operator | Effect | Notes |
|---|---|---|
| `:keyword` | narrow current set by keyword | auto-extends with detail names at current via depth |
| `:predicate` | narrow by predicate (`living`, `mine`, `here`, …) | reserved by name match |
| `:~name` | the set each prior element keeps under `name` | flat-map; built-in first (§ Relations) |
| `:exits` · `:to` · `:from` | the place steps | flat-map; `:to` live-only |
| `:i` | descend one level (no via: immediate contents; with via: immediate child details) | via-aware |
| `:I` | descend deep (no via: `getDeepContents()`; with via: every detail descendant from the tip, DFS) | via-aware |
| `:e` | ascend one level (no via: container; with via: pop one detail level) | via-aware |
| `:E` | ascend to root (no via: walk container chain to the topmost via `Containable.getRootContainer()`; with via: drop the entire detail path, anchor stays on the same Stuff) | via-aware; asymmetric — see below |

**`:E` two-step asymmetry.** When a detail-tree match meets the
container walk, `:E` does the work in two steps: the first `:E`
drops the via (anchor stays on the host Stuff); a *subsequent* `:E`
hits the no-via branch and walks the container chain. So
`me:i:sword:engraving:E` lands on the sword, and
`me:i:sword:engraving:E:E` walks from the sword to its root
container. The mental model: `:E` first finishes "leaving the
detail," then "leaves the container." Folding both into one step
would lose the natural pause at the detail's host. `:I` doesn't
need the symmetry rule because it descends one direction at a
time — into contents (no via) or into the detail subtree (via set),
never both in a single step.

`Containable.getRootContainer()` is the underlying walker — exposed
for controllers that want to find the world / zone / outermost room
without rolling their own loop. Returns `null` when the receiver is
already at the root.
| `:[N]`, `:[N..M]`, `:[N..]`, `:[-N]` | ordinal index / range | 1-based |
| `:[expr]` | filter expression | bracketed body that isn't pure-numeric |
| `:seed` | intersect with the seed's candidate pool | fixed pools: `me`, `world`, paths, ids, groups |

The chain narrows; it never broadens. `bookcase:rose` is empty
because rose isn't a detail of the bookcase. To re-anchor across
buckets, the dispatcher's scope try-list (`scope: ['$focus',
'~reachable']`) is the right tool — fall back to a broader scope when
the narrow one comes up empty.

The detail-keyword extension in `:keyword` is what makes drill-additive
focus chains resolve coherently:

| Step | Result |
|---|---|
| `~here` | (room, no via). Keyword space = room's perceptible keywords + top-level detail names. |
| `~here:bookcase` | (room, via=[bookcase]). Detail name "bookcase" matched; keyword space at this depth is bookcase detail's display + child detail names. |
| `~here:bookcase:book` | (room, via=[bookcase, book]). Child detail "book" matched. |

##### `:~rooms` and the keyed-member locator (residences D16)

A **Warren** is a relation holder: `warren:~rooms` is its **live**
members (`getMembers()` — a sync query cannot materialize a dormant
holding; the locator is the operator/author surface and a drive admits
first). The reserved chain word `:members` and `flatMapMembers` are
deleted (naming overhaul B2). Holding programmes are warren-shaped one
level down, so `warren:~rooms:~rooms` walks institution → holding →
rooms.

Two filter atoms complete the locator, beside `name`/`id` in
`evalAtom`: **`key`** — the explicit persistence key via the
`Persistable` narrow (`getPersistenceKey() ?? undefined`), and
**`address`** — the declared Locality address via the `Addressable`
narrow. Both read `undefined` off an unkeyed/unaddressed object, so a
comparison never false-matches (`docs/mql-grammar.md` § Filter
expressions).

⭐ **`material.X`** is the fifth namespaced atom, beside `prop.` /
`mixin.` / `class.` / `keyword.` / `template.`. It narrows with
`MixinApi.isTangible` and delegates to `Tangible.hasMaterialTag(tag)` —
the walk belongs to the host, not the resolver, because it spans the
bulk `_materialPath` **and** every per-Detail override (an axe is both
`wood` and `metal`). ⚠ Made of, never CONTAINS: a waterskin is leather,
and its bulk slot is `BulkableApi`'s question. Being **viewer-free** it
is legal in system mode, unlike the bareword predicates.

#### Set operations

| Form | Means |
|---|---|
| `A, B` | union |
| `A - B` | set difference |
| `(…)` | grouping |

Dedup is automatic at the boundary of every set operation — `A, A`
yields `A` once.

Set intersection has no dedicated character — it falls out of mid-chain
seed-shaped tokens whose right side is a fixed-pool seed (`me`, `world`,
a path, an id, a group). `(A):(B)` between two
grouped fixed-pool seeds is intersection by virtue of the chain rule.

#### Stable ordering

The resolver's `finalize` step sorts by score descending, with
insertion order as the tiebreak — so ordinal queries (`[5]`, `last`)
mean the same thing call-to-call, and single-cardinality consumers
picking "the first match" get the highest-scored one. Insertion order
is determined by the scope-walk traversal, which is deterministic for
fixed pools (location contents in insertion order, exits in declared
order).

#### Match-quality scoring — keyword tiers (the name is not special)

⭐⭐ **The rule lives in `GrammarApi` (`api/grammar.ts`), not in MQL**
(naming overhaul D-C1): the tiers (`GrammarApi.TIER_PHRASE` ·
`TIER_WORDS` · `TIER_PREFIX` · `TIER_PARTIAL`), `scoreTier`, `scoreAll`,
`rank`, `name` and the public `pluralWord`. `scope-walk.ts` keeps what is
MQL's — the candidates, the facets, the attribution — and asks
`GrammarApi.scoreAll(lists, words)` for the scores **once per pool** (every
Api static pushes a security frame, so a per-candidate call would put one
on every candidate in a crowd; the pure per-list rule is a module function
in `grammar.ts`). ⭐⭐ **The scorer reads ONE list — the candidate's
keywords** (controllers-are-io D22, one-resolution slate § 4). There is no
name field on a candidate: a thing's name is one of its keywords
(`NamedMixin.getKeywords` folds the name and its bare form, so *the One
Ring* answers to `one ring`). The tiers:

- **100 — a keyword IS the phrase** (`look clay` at the pit is the lumps,
  keyworded `clay`; the pit answers to `clay pit`)
- **60 — every query word is a WHOLE word of a keyword** (fishing B8:
  `net` is a word of *drift net* and begins no word of *keepnet*)
- **40 — every query word BEGINS a keyword word**
- **partials — 10 × the words that begin one** (a fallback only)

⚠ A word matches where a keyword WORD begins, never mid-word (ceramics
run 28: `shelf consign "plant pot"` once sold the aether imPLANT). The
whole-phrase cut and the tier cascade are one rule now: the highest tier
present cuts every tier below it, and anything at 40+ answers every word.
`GrammarApi.scoreTier(score)` (3 · 2 · 1 · 0) is what the binder compares
ACROSS scope tries (`MqlMany.tiers`; see [command-spec.md §
scope](./command-spec.md)).

⭐⭐ **The same rule names what is not Stuff.** `GrammarApi.nameAmong(text,
entries)` takes a holder's entries (`{ value, keywords, title }`) — a
quay in a harbor, a spell, a reading channel, a recipe, a document by its
kind's words, a list label — strips a leading article, reads a leading
ordinal, treats a quoted phrase as phrase-tier only, lets an entry answer
to its plurals, ranks by these tiers, and answers `{ kind: 'one' }` ·
`{ kind: 'tie', tie, titles }` · `{ kind: 'none', offered }`. ⚠ One
difference from MQL: **a partial names nothing** — some words matching
nothing is `none` (`reef bend` is no knot), where MQL keeps partials as
its own fallback. It never prompts and never refuses; the I/O half is
`CommandController.named(context, answer, noun, what)` — the value, a
prompt among the titles on a tie (a script: refused), or the
`no-such-name` row listing what could have been named. A hand-rolled
matcher anywhere else is what `lint:hand-matchers` counts, at a hard zero.

⭐ **Every drive finding that once made a NAME tier keeps its outcome by
keyword data**, each asserted in `scope-walk.score.test.ts` on the real
rows' keywords: the field patch answers to `top end` outright while its
stand answers to `top end wheat` (`Plant.getKeywords` leads its own words
with its patch's name — a bare `top end` on the stand would tie the
patch); the clay pit carries `clay pit`, never bare `clay`; the keepnet
`keepnet`/`net-bag`; the latex cups gained bare `latex`; the implant and
the brick table were already word rules. ⚠ Where the old 100 name tier
was the only thing separating two answers, the collapse leaves them tied
and the binder PROMPTS — the honest outcome; the fix is keyword data,
never a tier.

⭐ **A whole-phrase match CUTS the partials** (`scoreCandidates`): when
any candidate answers to EVERY query word (tier 40 or better), the
candidates naming only some are dropped — scores only order, they never
cut, so a plural verb took every partial too. Partials stay the fallback
when nothing names the whole phrase. Found by the rubber build's browser
drive (2026-10-18): `get "latex cup"` also picked up a coagulating pan
keyworded *latex pan*.

⚠⚠ **The scorer was inert for every `scope:word` query until 2026-09-22.**
A scope seed (then `reachable` / `inventory` / `peers`, now `~reachable` …
— the default arg
shape of most verbs) stamps each candidate 100, and the keyword narrow
(`filterByKeywords`) kept `max(prior, keyword)` — always 100 — so every
`reachable:net` tied and the `top` policy fell to **pool order**
(inventory before the room): `haul net` with a keepnet in hand hauled
the keepnet however the scorer ranked them, and every "substring
collision" the drives kept finding (creel/eel, ledger/edge, keepnet/net)
was really this. The narrow now carries the keyword's own score
(`scope-walk.score.test.ts` resolves it in situ). The scores above only
mean something because of that.

### Scope-walk (`mql/scope-walk.ts`)

Given a scope name, builds a `ScopeCandidate[]` pool:

```ts
interface ScopeCandidate {
  stuff: Stuff;
  keywords: string[];
  via?: MqlMatchVia;
  facets?: { keywords: string[]; via: MqlMatchVia }[];
}
```

⭐⭐ **MQL reads a Stuff ONLY through what it says it answers to**
(one-resolution slate § 5). `candidateFor(stuff, viewer)` builds the one
candidate an object makes: its keywords are `perceivedKeywordsFor(viewer)`
(system mode: the object's own `getKeywords()`), and every word in them is
supplied by the object — nothing is synthesized query-side:

| word | supplied by |
|---|---|
| authored keywords | `PerceptibleMixin.getKeywords(viewer?)` |
| the name + its bare form | `NamedMixin.getKeywords` (derived) |
| top-level detail ids the viewer may address | `DetailedMixin.getKeywords(viewer)` (derived; the D34 gate) |
| the held substance (`drink coffee` binds the cup) | `BulkableMixin.getKeywords` → `getHeldKeywords(affordance)` (derived) |
| a direction and its short forms | `Exit.getKeywords` |

The **facets** are attribution, not words: when the words that matched
are a detail's id or the held substance's (and the rest of the keywords
do not reach the same score), the match carries `via.detailPath` /
`via.bulk` as a fact. `pushDetails` / `pushBulkMaterials` — which built a
second candidate per detail id and per held material — are retired. A
side effect worth knowing: an item in your INVENTORY now answers to its
detail ids too (the old detail candidates were never pooled there).

Multiple candidates point at the same Stuff only inside a detail tree (a
chain step's tip and its children). The resolver dedups by `(stuff, via)`
after scoring.

**Candidate keywords are viewer-relative** (the recognition name-leak
gate): a living being resolves only by what the viewer perceives
(`knownAs` if recognized, salient features if not, the disguise's
descriptors if masked, never the true name); items keep their own
keywords. `look bob` resolves iff the room view shows the viewer "Bob".
Ordinal disambiguation (`stranger:[2]`) is an index into the snapshot and
carries no identity. See [belief.md](./belief.md).

A view's **scope entry** (one part of a comma-union, after `$focus`
expansion) becomes a pool by these rules (`candidatesForScopePart`):

| Scope part | Pool |
|---|---|
| a bare built-in — `~reachable`, `~peers`, `~inventory`, `~person`, `~horizon`, `~here` | that built-in's own pool, asked of the giver — detail subcandidates included (`sea`, a sign's `inscription`) |
| a query that **ends in a relation** (`~reachable:[class.CharterBook]:~ships`), or `@id` | the query's MATCHES are the candidates — nothing else, no neighborhood walk |
| `me` | `[giver]` |
| a `/path` · `#id` | the instances it names |
| `world` | every Stuff — ⚠ gated; see below |
| empty | the `~reachable` backstop |
| anything else (`$focus` expanded, a drilled focus `~here:bookcase`, a `[class.X]` anchor) | the slow path — § Scope-as-MQL evaluation |

The pools the built-ins read:

| Pool | Members |
|---|---|
| `~here` | the giver's place — ONE thing (its details among its keywords). ⚠ There is no scope word `here` and no "place surface" scope any more: a view says what it meant (`~here`, `~here:exits`, `~here:~buildings`, `me:e`, `~peers`) |
| `~peers` | the place's contents and the vessel the giver sits in, minus the giver, one level into the open things there |
| `~inventory` | the giver's whole carried tree, never into another body, nearest first (hand · ready · packed) |
| `~person` | the **on-person** pool: the giver + their own apps + installed slot occupants (+ each occupant's apps) + carried inventory (+ each carried attuned host's apps). Bearer semantics — no location leg; the anchor for `presentsKey` / payment-credential / hosted-app resolution (a matching key on the floor is never "presented") |
| `~reachable` | `~person` + the place's surface (the place, its ways and their doors, the buildings on its landing) + `~peers`, emitted **on-person-first** (the former `ContainmentApi.findReachable` contract, absorbed into the seed): first-match consumers and tie-broken targeting prefer your own gear over the floor's |

### ⭐⭐ Reach is one list and ONE rule (controller-liquidity W20; size-and-carrying D17, 2026-11-11)

`~inventory` · `~person` · `~peers` · `~reachable` are **filters over
`PerceptionApi.reach`** — the one definition of what the giver can lay
hands on, each thing tagged with its pool (reach's `here` pool = `around`
+ `here`: the holders themselves, then their surface — still part of
`~reachable`, though the relation `~here` is the place alone). The scope walk decides no
WHICH; it turns each reached thing into a candidate (the keyword face,
the facets). ⭐ **There are no rungs any more** (`ReachDepth`, `depth:`,
`--top`/`--open`/`--deep`, `MqlContext.depth` and the binder's
`refuseIfShutAway` are gone — D17):

- **the person** — the giver's whole carried tree, never into another
  body, ordered **nearest first by tier**: *hand* (a hand slot) · *ready*
  (worn, hung from what is worn, in a pocket or an open worn container
  — nothing sits loose on a body since the carrying flip) · *packed*
  (inside a worn container in the `back` slot). The tier read is
  `Carrying.tierOf` (the carrier answers; W4). A thing
  behind a shut lid or in the pack on your back **is in the list**;
- **the place** (and the vessel you sit in, and a named holder) — its top
  level and one level into the open things there
  (`Container.getReachableContents()`, breadth-first);
- the ways you can see, their doors, the buildings whose landing it is,
  the fixtures.

**`PerceptionApi.canReach` is membership AND the object's own refusal**
(`Containable.reachRefusal` — `enclosed-shut`: *"The box is shut."*;
`packed`: *"The rucksack is on your back — take it off first."*), so the
binder BINDS the apple in your shut box (it is the nearest thing you
perceive that answers) and the verb's `canReach` validator refuses it in
the object's words — **after** binding, and only for a thing the giver
perceives (the validator `mustNotBeShutAway` folded into `canReach`'s
wording). `~reachable` as a scope is exactly `canReach`'s membership
pool (`reach.sameSet.test.ts`, `reach.oneRule.test.ts`). Asked of
another (`@bob:~inventory`, `crate:~inventory`) the carried set is read
from OUTSIDE — what it holds and wears, its top level and one level into
the open things there (`candidatesHeldBy`), the same set whoever asks.

### ⭐⭐⭐ MQL names; the acts perceive (size-and-carrying D23, 2026-11-11)

*"MQL stays simple — the things and the interfaces make the
determinations; MQL only asks them, passing whatever information they
need (the asker)."* The resolver returns, for any non-system giver,
exactly what it returns in system mode (`mql.system-mode.test.ts` pins it
for every viewer-free seed, transform and class filter;
`mql.universal.test.ts` for `them:I:letter` across the bearer, a stranger
and system mode). It holds **no perception rule of its own**:
`pushDirect`'s `perceives` gate, the `#id` gate, the `visible` predicate's
perceives clause and `visibleInside`'s viewer filter are gone, and so is
`MqlContext.attention` (a detection consumer — the `wary` brain — names
with MQL, then `PerceptionApi.perceives(viewer, x, attention)`). The asker
reaches the resolver only (a) through the words that are the asker's
(`me`, a head `~name`, the `self` relations, `me:e`, `:mine`, `:here`) and
(b) as an argument to the OBJECT's own methods — what a thing answers to
is `Stuff.perceivedKeywordsFor(viewer)` (the recognition face for a
being, the thing's own `getKeywords(viewer)` otherwise), which detail ids
it offers is `getDetailIds(…, viewer)`.

**Perception is applied by the SURFACES through one filter,
`PerceptionApi.perceived(viewer, list, ordinal?)`** (order-preserving, over
`perceives`), held at zero exceptions by `lint:perceived-surfaces`: the
binder (each scope try, before tiers, kinds, `--strict` and the
cardinality policy — so there is no *which one?* over a covered thing),
`query`, `find`, the script `( )` island, the MQL subscriptions and the
card query source (before the projection and the diff), `look`,
`inventory`, `sense`, the card reads (`contents`, `worn`). ⭐ **The head
ordinal** (`2nd knife`) is no longer desugared into an index: it rides
`MqlMany.ordinal` and the surface picks the nth AFTER the filter, so an
ordinal never counts a thing you do not perceive (`resolveOne` alone picks
it from the universal list — system semantics). The chain index `:[n]` is
unchanged and universal. The `MqlApi` surface is still exactly
`resolveOne` + `resolveMany`.

### ⭐⭐ The registry-read grant

`world` is the only seed whose cost is *the size of the realm*, so it is
the only one that is not simply available. ⭐⭐ **The permission is a
fact about the person at the helm, and it lives in the execution
environment** — never in the Api surface, never in a parameter, never in
the shape of the call stack:

```
ExecutionContextApi.getWorldReadGrant()   // 'office:prime-minister' | null
```

That single read is everything the resolver consults. `null` — which is
what every ordinary execution reads — makes `world` throw
`MqlPermissionError`, naming the anchored alternatives. Non-null lets any
`world` shape resolve, and the run records a `RegistryScan` (`scanned`,
`indexed`, `shape`) which the binder turns into a `registry-scan` note.

**Who may hold the grant is decided in exactly one place**:

```
CompactApi.readWorldAs(subject, fn)
```

— which asks the **executive** about `subject`, and only on a yes plants
the grant on a frame around `fn`. Asking and granting are one act on
purpose: a grant primitive that could be planted without the question
would be a permission anybody in the engine could hand themselves.
Today the executive's answer is its head, the Prime Minister's seat,
derived at the moment of asking so authority follows a handoff in both
directions with no restart. When the executive wants to carve it up
further — a standing group, a per-shape allowance, a delegate who may
scan but not act — it is carved up *there*, and nothing in the query
engine moves.

⚠ **There is no second resolve method and no caller gate.** `MqlApi` is
`resolveOne` / `resolveMany`, differing only by cardinality; the
entitled query and the refused one go through the same door. Two
rejected shapes, and why:

| rejected | why |
|---|---|
| a `permission:` field on `MqlContext` | a context is supplied by the caller, and a permission a caller can hand itself is not a permission |
| a separate `resolveWorldForSeat` gated to the binder | that makes the permission a property of *which function called*, which says nothing about *who is acting* — a calling convention standing in for an authority |

The binder's arm is **catch-and-retry**: it lets the resolver refuse
from wherever `world` actually appears (a chain head, a mid-chain
intersect, a scope keyword), then asks the executive and re-runs the
same query inside the granted environment. So nothing in the binder
knows the grammar, a new place `world` can appear is covered the day it
exists, and an ordinary command never pays for the authority lookup.

⚠ **The grant is one-shot by construction.** It lives on a frame around
a single dispatch's retry; a subscription's re-resolve runs on no such
frame, so `world:` is refused on every subscribing surface for everyone,
the seat included — a standing rescan nobody would remember starting is
worse than the query it began as (see [governance.md § What else the
seat confers](./governance.md)).

⭐ **The engine's registry reads are not here.** `StuffApi.findByMixin`
is where they live, gated to its own `(template, method)` pair list —
because a wide read of the registry is the registry's business, not the
query language's, and the eleven callers never wanted a query.

**The composition index** (`StuffApi.findByMixin`, maintained inside
`register`/`unregister`) is what makes the indexed shape cheap. It is
**composed-only**: a mixin granted by a shadow or conferred by an
augment does not bucket its host, which is what `world:[mixin.X]` has
always meant. A runtime-grant selector would be its own thing.

⭐ **Why one index, and why it is composition.** A registry index is
legitimate when it describes an axis of the *substrate* — every Stuff
has a mixin composition, so `byMixin` describes the type system, not one
subsystem's content. That is the test for any future one: *does this
describe every object, or does it describe my feature?* ⛔ A
`byClassName` index was rejected on exactly that test, so the
`[class.X]` readers went to their owners (`residence.md § The PLACE
names its institution`, `antipatterns.md § Bespoke Object-Search
Algorithms`). The index is cheap because `hasMixin` is a pure function
of the constructor: `MixinApi` memoizes the lowercased name set per
class (`#lowercaseMemo`, a `WeakMap` keyed on the ctor), so the bucket
insert at `register` costs a set lookup. ⛔ A global event bus feeding
per-cache subscribers was rejected as the alternative: it taxes every
object creation with N predicate dispatches, and a subscriber's answer
to *"what existed before I subscribed?"* is the scan itself.

`~here` is the place and nothing else; what stands in it is `~peers`.
The split lets `get` declare the surgical scope it actually wants
(`~peers`) without picking up the room itself or its exits.

##### Doors and exits — both candidates, each as itself

The place's surface (`PerceptionApi.reach`'s `here` section, inside
`~reachable` — and a slow-path anchor's neighborhood) emits boundaries
and exits through the same generic candidate path (harbors review 3
retired the asymmetry; ⚠ since W20 a boundary is offered only on a way the
viewer can see, as `canReach` always had it):

- **A way's boundary is Stuff** with keywords, descriptions and details
  — a door, a window, a curtain, a gate. It is emitted beside its way,
  **once** however many of the room's exits are its sides (deduped by
  `stuffId` — one boundary, two sides; [boundary.md](./boundary.md)). A
  door at the north exit is addressable by its own keywords (`oak`,
  `door`), and `wash window` / `open door` bind the boundary itself.
- **Exits are candidates in their own right.** Each exit the viewer can
  see (`obviousExitsFor(viewer)` — on the slow path too) is emitted with
  `stuff: exit`, scored over its name and keywords. ⚠ Both are DERIVED
  on the object and named there: the name is the direction
  (`Exit.presentationPhrase`), the keywords are the authored ones plus
  the direction and its short forms — `n`, `u` (`Exit.getKeywords`). MQL
  reads an exit only through that interface and rewrites no word. An exit composes `Visible`, so
  `look north` describes it (an authored `longDescription`, else *a way
  north; the oak door is closed* — never where it leads).

There is no `via.exit`. A verb that needs the boundary on a way asks
`MqlApi.effectiveTarget` (the bound thing, else — when the bound thing is
an exit — that exit's boundary, `exit.getBoundary()`), so `open north`
reaches the door standing in the north way; movement verbs scope to ways
(`me:e:exits`, `~here:exits`), so a word can only mean one. The place
steps `:exits` / `:to` / `:from` (`flatMapPlaceStep`, reserved chain
words; `:to` is live-only) are in
[mql-grammar.md](../mql-grammar.md#places-and-the-ways-between-them).

Collisions between a door's keywords and an exit's direction (a
door author who adds `north` as a keyword on the door at the north
exit) TIE at the top keyword tier and prompt — the name tier that once
made the exit win is gone. Authors should leave directions out of door
keyword lists.

#### Scope-as-MQL evaluation

When the YAML's `scope:` fragment isn't recognized by the fast path
above (a bare built-in, a relation-ended query, `me`, a path, an id,
`world`), the resolver parses it as an MQL query (inner scope
`~reachable`), resolves it, and walks each resulting `(Stuff, via)`
anchor's neighborhood for candidates. That's what makes a focus like
`"~here:bookcase:book"`
evaluate cleanly as a scope — the candidate pool is the book detail's
children, if any. The fast path stays as the common case; arbitrary
fragments go through the slow path.

The neighborhood walk for an anchor:

- The anchor's own keywords + display name (or the detail tip name
  when `via.detailPath` is set).
- Detail names at the current via depth (top-level when no via,
  child-of-tip when via set).
- With **no via** AND the stuff is a Container: include its contents.
- With **no via** AND the stuff is Exitable: include doors and exit
  directions.

The "no via" gate matches the via-aware transform rule (`:i`, `:e`):
a candidate inside a detail tree has its detail tree as the
neighborhood, not the host's contents.

## `MqlMatchVia` — sub-feature attribution

Tells callers which path produced the match — a Detail name, a held
substance, a kept relation, or a direct hit. The base shape is
intentionally empty; subsystems augment via TypeScript declaration
merging. `mql/via-augment.ts` collects the resolver's own:

```ts
declare module './types' {
  interface MqlMatchVia {
    detailPath?: string[];   // names walked through the host's Detailed descriptors
    relation?: { holder: Stuff; name: string; meaning: string };  // the kept set (D-B6)
  }
}
```

`lib/bulk/Bulkable.ts` adds `bulk?: { affordance }`. There is no `exit`
augmentation: a direction binds the Exit itself as the match's `stuff`.

### Adding a new via key

1. In the owning subsystem file (or the augment collector), open a
   `declare module '<path>/api/mql/types' { interface MqlMatchVia { … } }`
   block and add an optional field.
2. The producing seed/transform stamps the field on `MqlMatch.via`.
3. Consumers narrow on the optional field — `result.via?.detailPath`,
   `result.via?.exit` — and ignore it when absent.

The list of registered via keys lives below in
[Registered via keys](#registered-via-keys); update it when you add
one.

### Registered via keys

| Key | Carries | Set by | Read by |
|---|---|---|---|
| `detailPath?: string[]` | names walked through `Detailed` descriptors | the detail facet of `scope-walk.candidateFor`, the chain detail-tree steps | controllers that drill into details |
| `bulk?: { affordance }` | the slot a held-substance match landed on | the bulk facet of `candidateFor`, the `:b` transform | the bulk verbs |
| `relation?: { holder, name, meaning }` | the kept set a match was reached through | `flatMapRelation` (every `:~name` step, built-ins too) | `contacts remove` (which list); the binder's tie prompt (the meaning tells two sets of one name apart) |

### The mixed-path consensus rule

`MqlApi.resolveMany` returns a single query-level `via` only when
every match arrived through the same sub-feature path; mixed paths
produce `via: undefined`. The shallow-equal is
`MqlLogic.consensusVia`.

⭐ **And each match keeps its own** (maritime browser walk, 2026-10-14):
`MqlMany.vias` is index-aligned with `stuff`, present whenever any match
came through a sub-feature. The mixed case was not dev-only after all —
`query all sea` on a deck matched the deck's `sea` feature AND a sea
chest, the consensus was nothing, and the deck was named instead of its
sea. The binder keeps `vias` aligned through `requires:` filtering
(`walkScopes`) and the cardinality policy (`viasAligned`), and a
`type: object` slot taking the top of several reads the top's own via
from it instead of resolving a second time.

## Pronoun memory (`mql/pronoun-memory.ts`)

Per-Focused-giver stash behind the built-in relations `~it`, `~him`,
`~her`, `~them` and `~last` (`$$` is gone — it is `~last`). Updated
post-resolve by `CommandApi.resolveModel`; read by the relation table's
`stash(slot)` entries. The typed words `it`/`him`/`her`/`them` reach it
through the default query aliases (§ Query aliases).

```ts
type PronounSlot = 'it' | 'him' | 'her' | 'them' | 'last';

class PronounMemory {
  read(slot: PronounSlot): MqlMany | null;
  readFragment(slot: PronounSlot): string | null;   // input fragment that produced the entry
  update(result: MqlMany, fragment: string, slotFor: (s: Stuff) => GenderedSlot): void;
  clear(): void;
}
```

Key semantics:

- **Empty results don't touch the stash.** The last successful query
  stays referenced.
- **Pronoun-input fragments don't update either.** A query like `look
  it` (or `look ~it`) doesn't overwrite the slot with the pronoun
  string — that would lose the original anchor.
- **Single-stuff results route to the gendered slot AND `last`.** The
  dispatcher passes a `slotFor` routing function that maps the matched
  Stuff to its gendered slot via `GenderedMixin.getPronouns()`. Stuff
  without `Gendered` defaults to `it`.
- **Multi-stuff results update `them` and `last` only** — there's no
  single-stuff anchor to route by gender.

Pronoun memory lives on `FocusedMixin`. NPCs without `FocusedMixin`
have no stash; the pronoun relations in their queries resolve to empty
match lists. The dispatcher gates `update` on `MixinApi.isFocused(giver)`.

The `#`-private slots in `PronounMemory` are fine because the class
isn't a Stuff and never crosses the call-security proxy. (See
CLAUDE.md § Member Privacy.)

## ⭐⭐ Descending a holder names everything inside it — for everyone (REVERSED 2026-11-11)

⚠ **Reversed by size-and-carrying D23.** Until 2026-11-11 the `:i` / `:I`
step yielded what the VIEWER might see inside a holder
(`PerceptionApi.visibleContentsOf`, maritime revision 2′). It now yields
the holder's contents — `getContents()` / `getDeepContents()` — for every
giver: `bob:i` names Bob's pockets, `chest:i` names what is in the shut
chest. What a viewer may SEE of them is decided where a result is bound or
shown (`PerceptionApi.perceived`): another body shows its hands, its
outermost layers, what hangs from what it wears, an open uncovered pocket
and the inside of an open worn container; a shut container light does not
pass (Beer–Lambert, `e^(−α·t) < 0.3`) hides what is in it; your own things
are always yours to perceive (see [perception.md](./perception.md)
§ *The holder rules*). The reason the rule moved out of the language is
the reason it was put in: `query bob:i`, `look bob:i:knife`, `query
--ledger bob:i` and a script must get the same answer — they do, because each is a
surface and each calls the one filter, and MQL can no longer disagree with
any of them.

⭐ **What went with it (2026-11-12, the carrying flip).** The reach
grammar is gone with the viewer filter: no `depth:` on a view, no
`--top`/`--open`/`--deep`, no `ReachDepth` — the binder binds the nearest
thing the giver perceives (tier order hand · ready · packed, then the
room) and the verb's `canReach` refuses a packed or shut-away one in the
object's words. A head ordinal rides `MqlMany.ordinal` and is applied
after `perceived`. MQL is universal; perceiving is the act's job.

## Resolving is not permission

There are **no permission tiers** (content-packs wave 3 deleted them
along with the `core` group and the "author tier" — see
[access.md](./access.md)). Every seed, relation, scope, namespace filter and
predicate resolves for every giver: a guest may type `~contacts:flower`,
`~reachable:[mixin.Door]` or `/platform/place/*:fountain` and get the honest
answer — the same answer anyone gets, which the verb then shows through
`PerceptionApi.perceived`, exactly as `look` is fogged. ⚠ **The one thing
that is gated is not a permission tier**: `world:` reads every object
the realm has ever made, and what gates it is *cost*, not privilege —
which is why the exception is an office with a receipt rather than a
tier. What that guest may
then *do* with a match is decided where it always was — by the verb:
`attack @<id>` fails on reachability, `teleport` on title over
the destination's extent, a mutation on the covering parcel's holder.
Enumeration is not capability; capability is title over a resource.

**Nor is resolving feasibility.** The same split holds for the physical
question. A view's `scope:` is a search hint — the try-list is tried in
order and the first non-empty result wins — never a guarantee: explicit
MQL bypasses it, and `eat @<id>:i:cake` resolves the cake wherever
it is. So the **per-verb validator is the contract**, and it must assume
nothing about how the target was named. `canReach`
(`lib/command/validators/canReach.ts`) is the exemplar — a membership
predicate over `PerceptionApi.canReach`, run on whatever was bound — and
a validator returns a *reason string*, not a boolean, because the prose
lives there: *sealed*, *out of reach* and *not food* are different lines.
See [perception.md § `canReach`](./perception.md) for the one
definition of reach.

What was retired: the `ctx.permission` snapshot (`isAuthor` /
`coreMemberIds`) the dispatcher stamped, `gateAuthor`, the
`PermissionTier` field on `MqlPredicate`, and the `:admin` predicate
(it tested membership of a group that no longer exists). The
`MqlPermissionError` class remains in `mql/types.ts` **as a name only**
— the subscription substrate and the client classify errors on it
(`'permission'` vs `'parse'` / `'resolve'`), but no resolver path throws
it any more.

⚠ `ctx.attention` (added by the stealth & deployables build) is **gone**
(size-and-carrying D23): it was the attention term of `pushDirect`'s
per-candidate `perceives` gate, and that gate is gone with it — MQL names
the same things for everyone. A detection consumer names with MQL and then
perceives at its own attention: the `wary` sentry brain resolves
`~peers:living` and keeps what `PerceptionApi.perceives(host, x, alertness)`
admits (see [stealth.md](./stealth.md)).

## System mode (`commandGiver: null`)

`MqlContext.commandGiver` may be `null` — the **code-only system
mode** (the `attention` precedent: the command dispatcher always
stamps a real giver, so player-typed MQL can never reach it). A null
giver is the engine's own viewer-blind enumeration for registry
sweeps and fixture indexes (`AttendantLogic.allPoints`,
`EmploymentLogic.allBusinesses`, and the rest of the pair list):

- the viewer-free seeds resolve (`/path` globs, `#id`, `@<id>`) with
  **no perception gate** and baseline (`getPresentation`) names —
  omniscient, not fogged;
- ⭐ **`world` is NOT among them.** A null giver says *nobody is
  looking*; it never said *and therefore you may read everything*. The
  two coincided, and every one of the seventeen engine scans came
  through that door. ⭐ The engine does not come through it at all any
  more: every one of those reads was `world:[mixin.X]`, which is
  `StuffApi.findByMixin` with extra steps, so they ask the registry
  directly and MQL is back to being only a query engine;
- the root `me` — and so every head `~name`, which is `me:~name` — a home
  path, and the bareword predicates that read the giver (`mine`/`here`)
  **throw** a clear resolver error — nothing guesses a viewer. ⚠ The
  connected-player listing that once rode the `online` seed is
  `PlayerApi.connectedAvatars()`, and an `mql:` group resolves over
  `/platform/agent/Avatar`;
- namespace filters (`[mixin.X]`/`[class.X]`) work — nothing gates
  them for anyone.

The one deliberate NON-consumer is `ResidencyLogic`'s pair of sweeps
— they operate on raw unwrapped proxies so enumeration never counts
as a residency touch, which MQL's proxy-mediated emission can't
express (the exemption is commented at both loops).

## Predicate registry (`mql/predicates.ts`)

Bareword filters that can appear in chain position
(`:living`, `:mine`, `:here`, `:visible`, `:loose`, `:held`,
`:worn`, `:carried`; `:online` left with the `online` seed). Each is a check function — no tier (there are none) — and
⭐ each DELEGATES to a method the model already answers
([architecture.md § MQL is a VIEW](../architecture.md)):

| predicate | delegates to |
|---|---|
| `visible` | a CLASS word: composes `Visible` (*presents to the eye*). ⚠ Since size and carrying D23 it does NOT ask `perceives` — the surface does |
| `loose` | the holder's `getLooseContents()` |
| `held` | on a carrier, a **hand** slot (`Attired.handSlots()`); a non-carrier holder keeps the wieldable-slot rule. A coupe in a rack is not held |
| `worn` | on a carrier, a **non-hand** slot (a coat in your hand is held, not worn) |
| `carried` | on the holder and not worn — a hand, a pocket, hung from a worn host — so `contents`/`worn` still partition everything (size and carrying W7-k) |

⭐ The room listing `look`, `sense` and the inspection card share is one
query — `~here:i:visible:[not mixin.AdornmentMixin] - me` (the card adds
`:carried`) — replacing three verbatim hand-rolled filters
(mql-predicate-parity slate). The orphan guard stays in
`getLooseContents(items)` (see [perception.md](./perception.md)).

⭐ **The discriminator for the next hand-rolled `getContents().filter`**
(the parity slate's census, 2026-09): *if a player would ever want to ask
it, MQL should be able to; if it is the engine talking to itself, a loop
is fine.* A brain counting the room's player-shaped occupants
(`Behaved.ts`) and a kiln reading its tangible charge (`FireController`)
are the engine's own questions and stay loops; a listing a verb shows a
player is a targeting rule and must be typeable.

```ts
interface MqlPredicate {
  check(target: Stuff, giver: Stuff & CommandGiver): boolean;
}
```

Unknown barewords in chain position fall back to keyword filtering —
predicates are reserved by *name match* on this published list, not
by parser keywords. Adding a predicate is a one-line append.

## `online` is retired (naming overhaul D-B10)

The `online` seed, scope and predicate are gone, and with them the
`online-provider` / `online-wire` seam that kept `ConnectionApi` off the
eager import chain. A view that scoped to `online` says what it meant:
`dm` → `["~here", "~contacts"]`; the money and invite verbs → `["~reachable",
"~contacts"]`; the add/invite/office verbs → `["~peers", "~contacts"]`; the speech
verbs → `["~peers"]`. A stranger elsewhere is reached by `@<id>` (`who` and
the profile card print it, withheld under a disguise that masks
identity). Engine code that needs the connected players asks
`PlayerApi.connectedAvatars()`.

## Scope-as-MQL and `$focus`

The dispatcher expands `$focus` (and any stored vars) in YAML
`scope:` strings via `ShellApi.expandVariables` before calling
`MqlApi`. So when a YAML declares `scope: ['$focus', '~reachable']`:

1. `$focus` expands to the giver's current focus fragment (defaults
   to `"~here"` if none).
2. The resolver runs with `scope: <expanded>`.
3. If the fragment isn't a recognized named seed, the scope-as-MQL
   slow path kicks in: parse + resolve the fragment, walk each
   resulting `(Stuff, via)` anchor's neighborhood for candidates.
4. If that produces no candidates / no scoring matches, the
   dispatcher tries the next entry (`~reachable`).

That's how a focus like `"~here:bookcase:book"` resolves coherently
both as a query (drill chain that lands on the book detail) and as a
scope (candidate pool = book's child details).

## Path-glob index (`PathTrie<T>`)

Path-glob seeds (`/platform/thing/Sword/*`, `/platform/… + /stuff/**/long*`) need to enumerate
template paths matching a pattern. `StuffApi`'s template-path index
is a `PathTrie<Stuff>`, exported from `api/path-pattern.ts`
alongside `PathPatternApi`. One node per path segment; pruning on
remove keeps the trie tight.

`PathPatternApi.compile` supports `*` (any non-`/`), `?` (single
char), `**` (recursive across `/`). The trie's `glob(pattern)` walker
short-circuits — `/platform/thing/Sword/*` only walks the `obj → Sword` subtree.

Operations:

- `findByTemplatePath(path)` / `findAllByTemplatePath(path)` — exact
  lookup, same singleton semantics as before.
- `findByPathGlob(pattern)` — glob expansion via the trie.

## Extension seams

Common extension points and where they live:

- **Add a predicate.** Append to `MQL_PREDICATES` in
  `mql/predicates.ts`. Write the check function.
- **Add a relation.** Almost always a holder's: compose `RelatingMixin`
  beneath the mixin or class that keeps the set and answer
  `relationsFor` / `relatedBy` (and `groupRefOf` when the set is a durable
  group), chaining to `super`. A **built-in** is a row in
  `api/mql/relations.ts` (`name`, `meaning`, `whose`, `members`) — one
  answer, no second face; it is a kernel change and masks every holder's
  set of that name.
- **Add a via key.** TypeScript-declaration-merge a new optional
  field onto `MqlMatchVia` in the owning subsystem (or the
  `via-augment.ts` collector during transition), produce it in the
  matching scope-walk / resolver path, document it in the
  [Registered via keys](#registered-via-keys) table.
- **Add a synthetic var (`$X`).** Declare `static syntheticVars:
  SyntheticVarEntry[]` on the mixin that owns the underlying state.
  See [shell-environment.md § Variable interpolation](./shell-environment.md#variable-interpolation).
- **Gate an operator.** Don't — resolving is not permission. Put the
  check on the verb that acts on the result (reachability, or
  `AccessApi.canAtPath` / `can` over the target's extent).

## Limitations / Non-goals

Documented decisions worth not re-litigating:

- **No method invocation (`->foo()`) or lambdas.** Permanent non-goal
  — admin scripting belongs in a different language.
- **No `obj.field` direct property access.** `.` is namespace-only
  (`prop.X`, `mixin.X`). Direct field access would leak the JS
  surface to players and contradict the inter-stuff contract
  (CLAUDE.md).
- **No sort operations** (`:sort.X`). Distinct syntactic shape; can
  be added without grammar churn.
- **No ad-hoc named groups** (`@@group`). A named set is a relation a
  holder keeps (`~friends`, `~bookclub`, `~party`); MQL mints none.
- **MQL never prompts.** It exposes the contract (several matches in a
  single-cardinality field; `relationTie`) and the binder asks.
- **No locale / translation.** Reserved words (`me`, `world`, the
  built-in relation names, `living`, etc.) are English. Architecture
  allows for it later.
- **English-only natural-language layer.** Article stripping and
  ordinal-prefix work on English forms only.

### What MQL can and cannot see — the three storage shapes

Three storage shapes, three answers, and the middle row surprises
people (graduated from the pets slate, 2026-09):

| storage shape | examples | MQL |
|---|---|---|
| Stuff + keyed snapshot | cultivated plants, a holding's rooms, pets | ✔ visible; `mixin.X` + `[key=…]` |
| **Document** | herds, water rights, bills of lading, rate cards | ✘ **not queryable at all** — MQL is over Stuff |
| **Seeded, unmeasured** | heads before `draft`, deposit samples, ground character | ✘ **invisible by construction** — not in the registry |

⚠ **The herdbook is not MQL-able.** Herds are filed records read through
their register; no query reaches them. Know this before designing a verb
that assumes otherwise.

⚠ **Do not infer state from key presence.** `has` is a documented no-op
outside `prop.K`, so key-presence is not cleanly testable in the grammar
— and inferring state from storage is the same dishonesty the city-watch
decision rejects. A state like *tamed* is a mixin or a property, so the
query language can see it and the abstraction carries it.

## Cross-references

- [../mql-grammar.md](../mql-grammar.md) — user-facing grammar
  reference: every anchor and relation, every operator, with examples.
- [shell-alias.md](./shell-alias.md) — query aliases (`?name`).
- [contacts.md](./contacts.md) · [social-graph.md](./social-graph.md) ·
  [grouping.md](./grouping.md) — the holders behind `~friends`,
  `notify ~bookclub`, and the durable `GroupRef` a relation stands for.
- [command-spec.md](./command-spec.md) — how to write a YAML view
  that uses MQL fields.
- [command-routing.md](./command-routing.md) — the dispatch pipeline
  including the resolve / preload / validate split (`resolveModel`
  → `preloadValidatorDeps` → `runValidators`), scope try-list
  semantics, focus management.
- [shell-environment.md](./shell-environment.md) — `$focus` /
  variable interpolation, `FocusedMixin`'s synthetic vars.
- [collections.md](./collections.md) — the Detailed / Container /
  Exitable surfaces the resolver walks.

## History — size and carrying (2026-11-11 → 2026-11-12)

`39d4ba23e..50f562ed4`. D17 removed the reach rungs (`ReachDepth`,
`depth:`, the three flags, `MqlContext.depth`); D23 made the resolver
universal (`visibleInside`/`visibleInsideDeep` now yield contents for
everyone, `pushDirect`'s `perceives` gate and `MqlContext.attention`
removed, `visible` a class word, the head ordinal on `MqlMany.ordinal`)
and moved perception to the surfaces (`PerceptionApi.perceived`,
`lint:perceived-surfaces`). The flip (W7) made `:held` a hand slot and
`:worn` a non-hand slot on a carrier.

## History — naming overhaul (2026-11-22)

`build/naming-overhaul`, Stage B and C. B0 made the affordance selector
`source::verb`; B1 added `~`/`@`, the built-in relation table and
`RelatingMixin`; B2 the holders (contact lists, parties, groups, warrens,
charter books, travel cards, aether hosts; `:members` deleted); B3a
respelled every view's scopes and defaults as relations, retired the seed
words, `$$` and `online`, and added query aliases; B3b split `here` — 116
slots now say what they meant, and the empty scope is the `~reachable`
backstop; B4 moved `notify` and `contacts` onto relations. C0 moved the
keyword tiers into `GrammarApi` and added `GrammarApi.nameAmong` +
`CommandController.named`; C1–C7 drove every hand-rolled matcher onto it
(`lint:hand-matchers` at zero). Decisions cited in code as `D-B*` /
`D-C*`.
