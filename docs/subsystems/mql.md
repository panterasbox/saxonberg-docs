# MQL — MUD Query Language

MQL is the query language the dispatcher runs against `type: object` /
`type: objects` YAML fields. It replaces the original keyword-scoring
matcher with a real grammar: seeds, chain operators, set ops, filter
expressions, scope-as-MQL evaluation, pronoun memory.

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
    scope-walk.ts            ← named-scope candidate pools, scoring
    predicates.ts            ← bareword filter registry (living, online, …)
    pronoun-memory.ts        ← per-Focused-giver stash for it/him/her/them/$$
    online-provider.ts       ← injection seam for online holders
    online-wire.ts           ← optional ConnectionApi wire-up
    via-augment.ts           ← TS declaration merging for MqlMatchVia
  __tests__/
    mql.test.ts              ← end-to-end resolver
    mql.lexer.test.ts
    mql.parser.test.ts
    mql.desugar.test.ts
    path-trie.test.ts
    path-pattern.test.ts
```

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
  commandGiver: Stuff & CommandGiver;
  scope: string;                       // an MQL fragment, post-expansion
  permission?: {                       // AccessApi-derived snapshot,
    isAuthor: boolean;                 // stamped by the dispatcher
    coreMemberIds?: ReadonlySet<string>;
  };
}

interface MqlOne   { stuff: Stuff | null; via?: MqlMatchVia; }
interface MqlMany  { stuff: Stuff[];      via?: MqlMatchVia; }
```

Both methods route through the same internal `resolve()` in
`resolver.ts`; the difference is wrapping. `resolveOne` picks the
top-scored match (or `null`); `resolveMany` returns the full sorted
list.

`scope` on `MqlContext` is a **string**, not a structured policy. The
dispatcher takes the YAML `scope:` array (`['$focus', 'reachable']`),
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
pass the whole `MqlQuantity` through to `GlobbableApi.applyQuantity`
without branching on it. See [glob.md](./glob.md) for the helper's
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
signal character (`:`, `[`, `,`, `'`, `{`). The `looksFormal()` guard
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
,  :  (  )  [  ]  {  }  .  -  #  $  *  /  '
=  !=  >  <  >=  <=
```

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
- **Single-quoted literals.** `'…'` preserves whitespace and has no
  escape syntax — closing quote is the only terminator. Unterminated
  is a lex error.
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
seed         = pronoun | keywords | literal | path
             | stuffId | "$$" | "(" query ")"
chainElement = transform | keywords | predicate | ordinal
             | bracket | group | quantity | seed-shaped
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

Each operator/seed carries a permission tier (predicates declare
theirs inline in `predicates.ts`); the resolver's
`gateAuthor(operator, tier, ctx)` throws `MqlPermissionError` when
`ctx.permission` is present with `isAuthor` false. The dispatcher's
outer try/catch converts the throw into a command-level failure.

#### Seeds

| Seed | AST node | Yields |
|---|---|---|
| `bareword` (or multi-word `red rose`) | `KeywordsNode` | scope-walk + scoring |
| `'literal'` | `LiteralNode` | exact name/keyword match (whitespace preserved) |
| `me` | `PronounNode` | the giver |
| `here` | `PronounNode` | the giver's location |
| `peers` / `reachable` / `person` / `inventory` / `online` / `world` | name-promoted `KeywordsNode` | the corresponding scope-walk pool |
| `it` / `him` / `her` / `them` | `PronounNode` | pronoun memory read |
| `$$` | `LastResultNode` | pronoun memory `last` slot |
| `/path/with/*globs` | `PathNode` | template-path glob seed (PathTrie) |
| `#abc123` | `StuffIdNode` | direct stuff id lookup |
| `(query)` | `GroupNode` | grouping |

Seed-shaped chain elements split by whether the seed is
**element-derivable**:

- `peers`, `reachable`, `inventory`, and the transforms `:i` / `:I`
  / `:e` / `:E` are element-derivable. Mid-chain they **flat-map**
  over the prior set: for each `x in prior`, compute `seed(x)` and
  union the results.
- `me`, `here`, `online`, `world`, paths, stuff ids, `$$`, and
  groups are **fixed pools**. Mid-chain they **intersect** with the
  prior set. `reachable:online` is "of the giver's reachable set,
  which are online."

**Set-aware exclusion is per seed.** `peers` and `inventory` exclude
the prior set from the union — their per-element definitions
already exclude the focal Stuff (peers excludes the focal by
definition; `inventory(x)` is `x`'s contents, not `x`), and without
the exclusion the union cross-pollinates: `(bob, joe):peers` would
pull bob in via `peers(joe)` and vice versa. The exclusion gives the
expected "peers of these people, minus these people" answer.

`reachable` does **not** exclude the prior set, because its
per-element definition INCLUDES the focal (you can reach yourself —
see {@link candidatesForReachable}). Excluding the prior set there
would silently drop legitimate self-reachable matches:
`(bob, joe):reachable` keeps bob and joe in the result.

For fixed-pool seeds (intersection), the prior set's `via`
attribution is preserved — intersection only decides membership.
For element-derivable seeds (flat-map), each derived match takes
its `via` from the candidate's via (e.g., detail subcandidates
carry `via.detailPath`, exit candidates carry `via.exit`); the
prior `via` is replaced by the derived origin, and score is
inherited from the prior match (same convention as
{@link applyTransform}).

The dispatch lives in `mql/resolver.ts:applyColon`:
`ELEMENT_DERIVABLE_SEEDS` routes through `flatMapBySeed` (which
calls `candidatesForElementDerivable` to anchor the seed walk on
each prior Stuff); everything else in `NAMED_SEED_KEYWORDS` and the
seed-shaped chain element kinds (`pronoun`, `path`, `stuffId`,
`lastResult`, `group`) routes through `intersectWithSeed`.

#### Chain operators

| Operator | Effect | Notes |
|---|---|---|
| `:keyword` | narrow current set by keyword | auto-extends with detail names at current via depth |
| `:predicate` | narrow by predicate (`living`, `online`, `mine`, …) | reserved by name match |
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
| `:seed` | intersect with the seed's candidate pool | seed-shaped chain elements |

The chain narrows; it never broadens. `bookcase:rose` is empty
because rose isn't a detail of the bookcase. To re-anchor across
buckets, the dispatcher's scope try-list (`scope: ['$focus',
'reachable']`) is the right tool — fall back to a broader scope when
the narrow one comes up empty.

The detail-keyword extension in `:keyword` is what makes drill-additive
focus chains resolve coherently:

| Step | Result |
|---|---|
| `here` | (room, no via). Keyword space = room's perceptible keywords + top-level detail names. |
| `here:bookcase` | (room, via=[bookcase]). Detail name "bookcase" matched; keyword space at this depth is bookcase detail's display + child detail names. |
| `here:bookcase:book` | (room, via=[bookcase, book]). Child detail "book" matched. |

#### Set operations

| Form | Means |
|---|---|
| `A, B` | union |
| `A - B` | set difference |
| `(…)` | grouping |

Dedup is automatic at the boundary of every set operation — `A, A`
yields `A` once.

Set intersection has no dedicated character — it falls out of mid-chain
seed-shaped tokens whose right side is a fixed-pool seed (per the
element-derivable / fixed-pool split above). `(A):(B)` between two
grouped fixed-pool seeds is intersection by virtue of the chain rule.

#### Stable ordering

The resolver's `finalize` step sorts by score descending, with
insertion order as the tiebreak — so ordinal queries (`[5]`, `last`)
mean the same thing call-to-call, and single-cardinality consumers
picking "the first match" get the highest-scored one. Insertion order
is determined by the scope-walk traversal, which is deterministic for
fixed pools (location contents in insertion order, exits in declared
order).

#### Match-quality scoring

Lives in `scope-walk.ts:scoreCandidate`. Rules:

- Exact-name match (case-insensitive, single-word query): 100
- Whole-name AND-narrow on lowercased name: 50
- Word-of-name AND-narrow: 40
- Keywords AND-narrow (all matched): 25
- Partial keyword match: 10 × matched-count
- Partial name-word match: 5 × matched-count

The rule that matters: **all query keywords must match for a
candidate to be considered.** AND-narrow, no exceptions. The exact
multipliers are arbitrary; they preserve Phase-4 behavior so existing
tests keep passing.

### Scope-walk (`mql/scope-walk.ts`)

Given a scope name, builds a `ScopeCandidate[]` pool:

```ts
interface ScopeCandidate {
  stuff: Stuff;
  name: string;
  keywords: string[];
  via?: MqlMatchVia;
}
```

Multiple candidates can point at the same Stuff (a location appears
once for itself, once per Detail, once per Exit-direction). The
resolver dedups by `(stuff, via)` after scoring.

**Candidate names/keywords are viewer-relative** (the recognition
name-leak gate). `pushDirect` builds each candidate's `name` from
`RecognitionApi.describe(giver, stuff)` and its `keywords` from
`RecognitionApi.perceivedKeywords(giver, stuff)` — so a living being
resolves only by what the viewer perceives (`knownAs` if recognized,
salient features if not, the disguise's descriptors if masked, never the
true name); items keep their ordinary keywords. `look bob` resolves iff
the room view shows the viewer "Bob". Ordinal disambiguation
(`stranger:[2]`) is an index into the snapshot and carries no identity.
See [belief.md](./belief.md).

Built-in scopes:

| Scope | Members |
|---|---|
| `me` | `[giver]` |
| `here` | location + its details + exits/doors |
| `peers` | location's contents (each item + its details), minus the giver |
| `inventory` | giver's contents (Container check) |
| `person` | the **on-person** pool: the giver + their own attunement-hosted updates + installed slot occupants (+ each occupant's hosted updates) + carried inventory (+ each carried attuned host's hosted updates). Bearer semantics — no location leg; the anchor for `presentsKey` / payment-credential / hosted-app resolution (a matching key on the floor is never "presented") |
| `reachable` | `person` + `here` + `peers`, emitted **on-person-first** (the former `ContainmentApi.findReachable` contract, absorbed into the seed): first-match consumers and tie-broken targeting prefer your own gear over the floor's |
| `online` | every connected interactive's holder Stuff |
| `world` | every Stuff |

`here` deliberately does NOT include peers; that's `peers`' job. The
split lets `get` declare the surgical scope it actually wants
(`peers`) without picking up the room itself or its exits.

##### Doors vs exits — an intentional asymmetry

`candidatesForHere` walks doors and exits along **different paths**,
and that's deliberate, not a workaround:

- **Doors are Stuff.** They have keywords, descriptions, props, and
  participate in the lifecycle like any other thing. They flow
  through the standard `pushDirect` candidate emission (own
  keywords + display name) plus `pushDetails` (any details on the
  door). A door at the north exit is addressable by its own
  keywords (`oak`, `door`, `iron`).

- **Exits are descriptors of the room**, not Stuff. They have a
  direction, an inverse, optionally a door reference — but they
  aren't lifecycle-attached objects with keywords of their own. The
  candidate walk synthesizes one per exit:
  `{ stuff: location, name: direction, keywords: [direction], via: { exit } }`.
  The matched Stuff is the location; `via.exit` carries the
  attribution.

This mirrors how players think about the world: doors are *things*
("the oak door") and exits are *features of the room* ("you can go
north"). The resolver's two paths encode that asymmetry — a single
"give exits keywords too" generalization would obscure it. The
direction vocabulary stays the canonical surface for "name an exit"
because it's already the framework's closed set of direction tokens.

Collisions between a door's keywords and an exit's direction (a
door author who adds `north` as a keyword on the door at the north
exit) resolve through scoring: the synthesized exit candidate
scores 100 (exact-name match on the direction), the door's keyword
hit on a multi-word display name scores 25. Exit wins. Authors who
don't want this should leave directions out of door keyword lists.

#### Scope-as-MQL evaluation

When the YAML's `scope:` fragment isn't a recognized named seed (or
comma-union of them), the resolver parses it as an MQL query, resolves
it, and walks each resulting `(Stuff, via)` anchor's neighborhood for
candidates. That's what makes a focus like `"here:bookcase:book"`
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

Tells callers which path produced the match — Detail name, Exit, or
direct hit. The base shape is intentionally empty; subsystems augment
via TypeScript declaration merging. v1 ships two augmentations,
collected in `mql/via-augment.ts`:

```ts
declare module './types' {
  interface MqlMatchVia {
    detailPath?: string[];   // names walked through the host's Detailed descriptors
    exit?: Exit;             // exit reference for direction matches
  }
}
```

Phase 7 plans to move the `detailPath?` declaration into
`lib/description/Detailed.ts` and the `exit?` declaration into
`lib/spatial/Exit.ts`. The augment file is a temporary collector to
keep the resolver from depending on subsystem files.

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
| `detailPath?: string[]` | names walked through `Detailed` descriptors | `scope-walk.pushDetails`, the chain detail-keyword extension | controllers that drill into details |
| `exit?: Exit` | the exit a direction match landed on | `scope-walk.candidatesForHere` (direction synthesis) | `GoController`, `OpenController`, `CloseController` for door access |

### The mixed-path consensus rule

`MqlApi.resolveMany` returns a single query-level `via` only when
every match arrived through the same sub-feature path; mixed paths
produce `via: undefined`. The cheap shallow-equal is in
`mql.ts:consensusVia` — JSON.stringify each match's via and compare
to the first.

For the mixed-path case (a comma-union like `north, sword`), the
controller has to handle the absence: that's a documented limitation
that bites only dev queries.

## Pronoun memory (`mql/pronoun-memory.ts`)

Per-Focused-giver stash for the dynamic pronouns (`it`, `him`, `her`,
`them`) plus the `last` slot returned by `$$`. Updated post-resolve
by `CommandApi.resolveModel`; read by the resolver when those
seeds appear in a chain.

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
  it` doesn't overwrite the slot with the literal string `"it"` —
  that would lose the original anchor.
- **Single-stuff results route to the gendered slot AND `last`.** The
  dispatcher passes a `slotFor` routing function that maps the matched
  Stuff to its gendered slot via `GenderedMixin.getPronouns()`. Stuff
  without `Gendered` defaults to `it`.
- **Multi-stuff results update `them` and `last` only** — there's no
  single-stuff anchor to route by gender.

Pronoun memory lives on `FocusedMixin`. NPCs without `FocusedMixin`
have no stash; pronoun seeds in their queries resolve to empty match
lists. The dispatcher gates `update` on `MixinApi.isFocused(giver)`.

The `#`-private slots in `PronounMemory` are fine because the class
isn't a Stuff and never crosses the call-security proxy. (See
CLAUDE.md § Member Privacy.)

## Permission tiers

Three tiers: `public`, `authoring`, `admin`. Operators tag their
tier; the resolver gates them with `gateAuthor(operator, tier, ctx)`,
which throws `MqlPermissionError` when `ctx.permission` is present and
its `isAuthor` is false. (The standalone `mql/permissions.ts` module —
along with `checkTier(tier, operator, giver)` and the
`_MqlAdminFlag.granter` stub — was retired by the access build; the
`MqlPermissionError` type now lives in `mql/types.ts`.)

`ctx.permission` is an `AccessApi`-derived snapshot stamped on the
`MqlContext` by the command dispatcher before sync resolve runs.
`isAuthor` gates the pre-resolution operators (path seeds, `:online`,
`mixin.X` / `class.X` / `template.X` / `prop.X` filters);
`coreMemberIds` feeds the per-target `:admin(target)` predicate. When
the snapshot is **absent** the gate permits — server-internal callers
that build an `MqlContext` directly (without the dispatcher) continue
to work unchanged.

`ctx.attention` (added by the stealth & deployables build) is the
perception attention the scope-walk resolves concealed candidates at —
the `PerceptionApi.perceives` gate's attention term, threaded into
`pushDirect` across every seed builder (`peers` / `reachable` / `here` /
`inventory` / `flat`) and their mid-chain element-derived forms. It is a
**code-only** field: the command dispatcher never sets it, so player-typed
MQL always resolves at the passive baseline and **honest fog is unchanged**
(concealed things a player can't passively perceive stay un-targetable).
Only in-code detection consumers opt in — the `wary` sentry brain passes
its active `alertness` so `peers:living` enumerates the movers it perceives
*when actively watching*, folding detection into the query (see
[stealth.md](./stealth.md)). Absent → the passive baseline (byte-identical
to prior behavior). No grammar change — it rides the context struct, not the
query string.

Tier defaults:

- **public** — bareword keyword search, `me`, `here`, pronouns,
  `$$`, ordinals, ranges, set ops, `inventory` / `here` / `peers` /
  `reachable` scopes, `:i` / `:e`, `living` / `mine` / `here` /
  `visible` predicates.
- **authoring** — `mixin.X`, `class.X`, `template.X`, `prop.X`,
  `keyword.X`, bare `id` / `name` atoms, `/path/...` seeds,
  `#stuffid` seeds.
- **admin** — `world` / `online` scopes, `online` / `admin`
  predicates.

Permission failure is loud — fail at parse-time or resolve-time with
a message naming the tier ("You don't have permission to use
'mixin' filters here."). We don't silently strip operators; that
creates surprising behavior.

## System mode (`commandGiver: null`)

`MqlContext.commandGiver` may be `null` — the **code-only system
mode** (the `attention` precedent: the command dispatcher always
stamps a real giver, so player-typed MQL can never reach it). A null
giver is the engine's own viewer-blind enumeration for registry
sweeps and fixture indexes (`AttendantLogic.allPoints`,
`EmploymentLogic.allBusinesses`, `LocomotionLogic.allModes`,
`SlotLogic.findOccupiedSlots` — the former `getAllObjects` scans):

- the viewer-free seeds resolve (`world`, `/path` globs, `#id`,
  `online`) with **no perception gate** and baseline
  (`getPresentation`) names — omniscient, not fogged;
- the giver-anchored seeds (`me`/`here`/`peers`/`reachable`/`person`/
  `inventory`) and the bareword predicates (`visible`/`mine`/`admin`
  all read the giver) **throw** a clear resolver error — nothing
  guesses a viewer;
- namespace filters (`[mixin.X]`/`[class.X]`) work: the permission
  snapshot is absent for server-internal contexts, which permits
  (the pre-existing internal-caller rule).

The one deliberate NON-consumer is `ResidencyLogic`'s pair of sweeps
— they operate on raw unwrapped proxies so enumeration never counts
as a residency touch, which MQL's proxy-mediated emission can't
express (the exemption is commented at both loops).

## Predicate registry (`mql/predicates.ts`)

Bareword filters that can appear in chain position
(`:living`, `:online`, `:mine`, `:here`, `:visible`, `:admin`). Each
declares its tier and a check function:

```ts
interface MqlPredicate {
  tier: PermissionTier;
  check(target: Stuff, giver: Stuff & CommandGiver): boolean;
}
```

Unknown barewords in chain position fall back to keyword filtering —
predicates are reserved by *name match* on this published list, not
by parser keywords. Adding a predicate is a one-line append.

## Online-holders provider seam

The `online` / `world` seeds and the `:online` predicate need to
enumerate connected `Interactive`s back to their holder Stuff.
Importing `ConnectionApi` from anywhere on the eager
`command.ts → MqlApi` chain forces the
`ConnectionManager → Interactive → Idea` cycle to deadlock when test
files lead with `import { Idea }`. The two-step indirection:

- `mql/online-provider.ts` — module-private setter; consumed by the
  resolver and predicates.
- `mql/online-wire.ts` — imports `ConnectionApi`, registers the
  provider. **Not** imported by any module on the eager chain.

Production wires once at boot via `AppBootstrap`; tests that need
live online data import `online-wire` directly. Without the wire,
`online` queries resolve cleanly but yield no matches.

## Scope-as-MQL and `$focus`

The dispatcher expands `$focus` (and any stored vars) in YAML
`scope:` strings via `ShellApi.expandVariables` before calling
`MqlApi`. So when a YAML declares `scope: ['$focus', 'reachable']`:

1. `$focus` expands to the giver's current focus fragment (defaults
   to `"here"` if none).
2. The resolver runs with `scope: <expanded>`.
3. If the fragment isn't a recognized named seed, the scope-as-MQL
   slow path kicks in: parse + resolve the fragment, walk each
   resulting `(Stuff, via)` anchor's neighborhood for candidates.
4. If that produces no candidates / no scoring matches, the
   dispatcher tries the next entry (`reachable`).

That's how a focus like `"here:bookcase:book"` resolves coherently
both as a query (drill chain that lands on the book detail) and as a
scope (candidate pool = book's child details).

## Path-glob index (`PathTrie<T>`)

Path-glob seeds (`/obj/Sword/*`, `/obj/**/long*`) need to enumerate
template paths matching a pattern. `StuffApi`'s template-path index
is a `PathTrie<Stuff>`, exported from `api/path-pattern.ts`
alongside `PathPatternApi`. One node per path segment; pruning on
remove keeps the trie tight.

`PathPatternApi.compile` supports `*` (any non-`/`), `?` (single
char), `**` (recursive across `/`). The trie's `glob(pattern)` walker
short-circuits — `/obj/Sword/*` only walks the `obj → Sword` subtree.

Operations:

- `findByTemplatePath(path)` / `findAllByTemplatePath(path)` — exact
  lookup, same singleton semantics as before.
- `findByPathGlob(pattern)` — glob expansion via the trie.

## Extension seams

Common extension points and where they live:

- **Add a predicate.** Append to `MQL_PREDICATES` in
  `mql/predicates.ts`. Pick a tier, write the check function.
- **Add a named scope.** Add a `candidatesForX(giver)` builder in
  `mql/scope-walk.ts`, dispatch on the new name in the resolver's
  scope handler (`candidatesForScope`, fast path).
- **Add a via key.** TypeScript-declaration-merge a new optional
  field onto `MqlMatchVia` in the owning subsystem (or the
  `via-augment.ts` collector during transition), produce it in the
  matching scope-walk / resolver path, document it in the
  [Registered via keys](#registered-via-keys) table.
- **Add a synthetic var (`$X`).** Declare `static syntheticVars:
  SyntheticVarEntry[]` on the mixin that owns the underlying state.
  See [shell-environment.md § Variable interpolation](./shell-environment.md#variable-interpolation).
- **Add a permission tier check.** Tag the operator's tier and route
  it through `gateAuthor(operator, tier, ctx)` in the resolver; the
  gate reads the `AccessApi`-derived `ctx.permission.isAuthor` snapshot
  the dispatcher stamps. (The real zone-aware authoring check already
  landed — it's the `AccessApi` source of that snapshot, not a
  `granter` stub.)

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
- **No named groups** (`@@group`). Authored or player-saved object
  groups. Out.
- **No disambiguation prompts.** Depends on the future prompt stack.
  MQL exposes the contract (`MqlMany.stuff.length > 1` in a
  single-cardinality field); the prompt UX is built later.
- **No locale / translation.** Reserved keywords (`me`, `here`,
  `living`, etc.) are English. Architecture allows for it later.
- **English-only natural-language layer.** Article stripping and
  ordinal-prefix work on English forms only.

## Cross-references

- [../mql-grammar.md](../mql-grammar.md) — user-facing grammar
  reference: every seed, every operator, with examples.
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
